# IBMB (NPCI) Connectivity Test – Dev payment-service

Path tested: `payment-paymentservice` pod → SBI proxy `serverswg.sbi.co.in:9090` → `https://ibmbcert.npci.org.in` (mTLS + ECDSA-signed request)

---

## 1. Become root on bastion (kubectl works only as root)

```bash
sudo su -
```

## 2. Get current payment pod name

```bash
kubectl get pod -n dev-transaction | grep payment
```

## 3. Start debug container (JDK image – has keytool + java)

```bash
kubectl debug -it <payment-pod-name> -n dev-transaction \
  --image=artifactory.jfrog.sbi:443/itepaypg-sbiepay2-docker-local/custom-ci/ubi9/openjdk-21:1.23-6.1756793462 \
  --target=paymentservice -- /bin/sh
```

> App container files are visible under `/proc/1/root/...` from the debug shell.

## 4. Build client P12 from the mounted JKS (once per debug container)

```bash
keytool -importkeystore -srckeystore /proc/1/root/nbbl-certs/nbbl_client.jks -srcstoretype JKS -srcstorepass 'sbi@123' \
        -destkeystore /tmp/npci-client.p12 -deststoretype PKCS12 -deststorepass 'sbi@123'
```

Optional check:

```bash
keytool -list -v -keystore /tmp/npci-client.p12 -storetype PKCS12 -storepass 'sbi@123' | grep -E 'Alias|Entry type|Owner|Valid'
```

## 5. Write fresh request (payload + signature from app/dev team)

Paste the whole block at once. `EOF` must be on its own line.

```bash
cat > /tmp/req.json <<'EOF'
{
  "payload" : "<payload>",
  "signature" : {
    "signature" : "<MEUC... / MEQC...>",
    "protected" : "<a2V5SWQ9...>"
  }
}
EOF
wc -c /tmp/req.json
```

Expect ~1580 bytes. If a `>` prompt keeps waiting → `Ctrl+C` and paste again.

## 6. Fire the request (within 3 minutes of payload creation)

```bash
curl -v --proxy http://serverswg.sbi.co.in:9090 \
  --cert-type P12 --cert "/tmp/npci-client.p12:sbi@123" \
  -H "Content-Type: application/json" -H "User-Agent: Java/17.0.11" \
  --data @/tmp/req.json \
  "https://ibmbcert.npci.org.in/ibmb/ReqTxnInit/1.0/urn:referenceId:<refId>"
```

Optional – check payload is still valid before sending:

```bash
grep -o '"protected" *: *"[^"]*"' /tmp/req.json | cut -d'"' -f4 | base64 -d; echo; date +%s
# expires/1000 must be greater than date +%s
```

---

## Reading the result

| Output | Meaning |
|---|---|
| `curl: (52) Empty reply from server` | No client cert sent (missing `--cert`) – WAAP drops request |
| `could not parse PKCS12 ... mac verify failure` | Wrong p12 password (use `sbi@123`) |
| `Content-Length: 0` / `400 Bad Request` | `/tmp/req.json` missing or empty |
| `IBMBSIG010 Signature Verification Failed` | Payload expired (>3 min), refId reused, or signing key mismatch |
| `IBMBMER009 Merchant mcc code ...` | App/merchant data (MCC) – dev team |
| `"result":"SUCCESS"` | End-to-end OK |

Handshake must show: `TLSv1.2 (OUT), TLS handshake, CERT verify (15)`

---

## Useful checks

**App IBMB config (bastion):**
```bash
oc get cm payment-config -n dev-transaction -o yaml | grep -n -iE 'ibmb|nbbl|jks|keystore|proxy'
```

**Secret contents (key names only):**
```bash
oc get secret nbbl-cert -n dev-transaction -o go-template='{{range $k,$v := .data}}{{$k}}{{"\n"}}{{end}}'
```

**Mounted files in pod (debug shell):**
```bash
wc -c /proc/1/root/nbbl-certs/..data/*
```

**Verify signing private key matches NPCI-registered public key (bastion):**
```bash
openssl pkey -in sbi_private_key -pubout | diff - shared_pub.pem && echo MATCH || echo MISMATCH
```

**App logs for IBMB calls:**
```bash
oc logs <payment-pod-name> -c paymentservice -n dev-transaction --since=30m | grep -iE 'ibmb|nbbl|PSE09' | tail -30
```

---

## Updating nbbl-cert secret (e.g. after key rotation)

```bash
oc get secret nbbl-cert -n dev-transaction -o yaml > nbbl-cert-backup-$(date +%F).yaml

oc create secret generic nbbl-cert -n dev-transaction \
  --from-file=nbbl_client.jks=<(oc get secret nbbl-cert -n dev-transaction -o jsonpath='{.data.nbbl_client\.jks}' | base64 -d) \
  --from-file=ibmbcert_npci_org_in_new_24Dec.crt=<(oc get secret nbbl-cert -n dev-transaction -o jsonpath='{.data.ibmbcert_npci_org_in_new_24Dec\.crt}' | base64 -d) \
  --from-file=sbi_private_key=./sbi_private_key \
  --dry-run=client -o yaml | oc apply -f -

oc rollout restart deployment/payment-paymentservice -n dev-transaction
```

---

## Reference

| Item | Value |
|---|---|
| Namespace | `dev-transaction` |
| Proxy | `serverswg.sbi.co.in:9090` (10.191.191.39) |
| NPCI endpoint | `https://ibmbcert.npci.org.in/ibmb/ReqTxnInit/1.0/urn:referenceId:<refId>` |
| Secret | `nbbl-cert` → mounted at `/nbbl-certs` |
| Client keystore | `nbbl_client.jks` (JKS, pwd `sbi@123`) – **cert expired 10 Dec 2025, renewal pending** |
| Signing key | `sbi_private_key` (PKCS#8, EC P-256), keyId `PSE09` |
| NPCI public key | `ibmbcert_npci_org_in_new_24Dec.crt` (base64 EC public key) |
| Signature validity | 3 minutes (`expires - created`) |
| Last SUCCESS | 29 Sep 2026 12:37 IST, refID `PSE096272ZCRASVXYFCV` |
