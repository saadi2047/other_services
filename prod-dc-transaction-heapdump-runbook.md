# Heap Dump + Histogram Runbook – prod-dc-transaction (txn-transactionservice)

**Purpose:** Capture a heap histogram and a heap dump from one Transaction Service pod, then analyse the dump in Eclipse Memory Analyzer (MAT). The goal is to find the GC root path that keeps `PushVerificationRequest` and the Reactor Netty HTTP client objects alive.

**Reference:** Red Hat case 07770429. The heap dump procedure comes from Red Hat (Prithviraj Patil, 29-Sep-2026).
**Run from:** `dcepayprodbastion` as `osuser`
**Namespace:** `prod-dc-transaction`  **Container:** `transactionservice`

> Run all steps **in the same terminal session**. Step 2 stores the chosen pod name in `$POD` and later steps reuse it.

---

## 0. Before you start (mandatory)

| # | Check | Owner |
|---|---|---|
| 1 | **Bank security approval** for capturing and moving a heap dump. The dump is a full copy of the pod's memory and **contains transaction/customer data**. | Namdev / Sourabh |
| 2 | Approved **low-traffic window** (change ticket raised) | Namdev / Sourabh |
| 3 | Agreed **storage location** for the dump and the analysis machine (encrypted, access-restricted) | Bank security |
| 4 | App team informed. The pod will **pause for about 20–60 seconds** during the dump, and requests on that pod may time out. | Saadi |

**Why a pause is tolerable here:** liveness and readiness probes are **disabled** for this service in prod-dc (`values.yaml`), so OpenShift will not kill or restart the pod during the pause. Other pods keep serving traffic.

---

## 1. Pre-checks

**1.1 Pod status, restarts, memory**
```bash
oc get pod -n prod-dc-transaction -l app.kubernetes.io/instance=txn,app.kubernetes.io/name=transactionservice -o custom-columns='NAME:.metadata.name,RESTARTS:.status.containerStatuses[*].restartCount,RUNNING_SINCE:.status.containerStatuses[*].state.running.startedAt,NODE:.spec.nodeName'
```
```bash
oc adm top pod -n prod-dc-transaction -l app.kubernetes.io/instance=txn,app.kubernetes.io/name=transactionservice --containers --sort-by=memory | grep transactionservice
```

**1.2 Bastion free space (the compressed dump is roughly 0.6–1.2 GB)**
```bash
df -h /home/osuser
```

---

## 2. Select the pod

**Rule:** choose the **highest-memory pod whose app container is below 3500Mi**. That pod has plenty of leaked objects to analyse, and enough headroom that the dump won't push it over the 4Gi limit. **Never dump a pod at or above ~3.7Gi.**

```bash
POD=$(oc adm top pod -n prod-dc-transaction -l app.kubernetes.io/instance=txn,app.kubernetes.io/name=transactionservice --containers --no-headers | awk '$2=="transactionservice"{m=$4; sub("Mi","",m); if (m+0 < 3500) print m, $1}' | sort -nr | head -1 | awk '{print $2}'); echo "Selected pod: $POD"
```
```bash
oc adm top pod $POD -n prod-dc-transaction --containers
```
If `Selected pod:` is empty, **stop**: every pod is either above 3500Mi or unreadable. Check with the team.

To choose the pod manually instead:
```bash
POD=txn-transactionservice-cf6ddc874-<suffix>; echo "Selected pod: $POD"
```

**2.1 In-container checks (`/tmp` space, `tar`, `gzip`, `jcmd`)**
```bash
oc exec $POD -n prod-dc-transaction -c transactionservice -- sh -c 'df -h /tmp; for t in tar gzip jcmd sha256sum; do command -v $t >/dev/null 2>&1 && echo "$t: present" || echo "$t: NOT present"; done'
```
- `/tmp` needs **at least 2 GB free**.
- If `tar` is **NOT present**, use the fallback copy in step 4.2. `oc cp` needs tar.

---

## 3. Heap histogram (before the dump)

This takes a full-GC histogram, which pauses the pod for 1–3 seconds, and saves it on the bastion with a timestamp.
```bash
TS=$(date +%Y%m%d_%H%M%S); oc exec $POD -n prod-dc-transaction -c transactionservice -- jcmd 1 GC.class_histogram > /home/osuser/histogram_${POD}_${TS}.txt 2>&1; echo "Saved: /home/osuser/histogram_${POD}_${TS}.txt"; tail -1 /home/osuser/histogram_${POD}_${TS}.txt
```

**Key rows** (push-verification and HTTPS client objects, plus `OrderStatus` as a control row):
```bash
grep -E "PushVerificationRequest|ReactorClientHttpConnector |JdkSslClientContext|reactor.netty.transport.ProxyProvider$|PooledConnectionProvider\\\$PoolKey|X509TrustManagerImpl|HttpClientConfig$|TrustAnchor|X509CertImpl|enums.OrderStatus$|^Total" /home/osuser/histogram_${POD}_${TS}.txt
```

**Optional: compare with the 24-Sep baseline.** This only works if `$POD` is `...-hcpb2` **and** its JVM has not restarted since 24-Sep.
```bash
awk 'FNR==NR && $1~/^[0-9]+:$/ {y[$4]=$2; next} $1~/^[0-9]+:$/ {printf "%12d %12d %10d  %s\n", y[$4], $2, $2-y[$4], $4}' /home/osuser/hcpb2_histogram.txt /home/osuser/histogram_${POD}_${TS}.txt | sort -k3,3nr | head -25
```
Columns: 24-Sep count · today count · increase · class.

---

## 4. Heap dump (Red Hat procedure)

**4.1 Capture the dump.** It is compressed inside the pod. `GC.heap_dump` runs a full GC first, so the dump holds live objects only.
```bash
DUMP=/tmp/heap_${POD}_${TS}.hprof.gz; echo "Dump file: $DUMP"; date; time oc exec $POD -n prod-dc-transaction -c transactionservice -- jcmd 1 GC.heap_dump -gz=5 $DUMP; date
```
Expected output ends with `Heap dump file created [...]`.

Check the file exists and note its size:
```bash
oc exec $POD -n prod-dc-transaction -c transactionservice -- ls -lh $DUMP
```

**Checksum inside the pod** (skip if `sha256sum` is not present):
```bash
oc exec $POD -n prod-dc-transaction -c transactionservice -- sha256sum $DUMP
```

**4.2 Copy the dump to the bastion**

Primary (Red Hat procedure, needs `tar`):
```bash
oc cp prod-dc-transaction/$POD:$DUMP /home/osuser/$(basename $DUMP) -c transactionservice
```
Fallback (no `tar` needed, streams the file):
```bash
oc exec $POD -n prod-dc-transaction -c transactionservice -- cat $DUMP > /home/osuser/$(basename $DUMP)
```

**4.3 Verify the copy**
```bash
ls -lh /home/osuser/$(basename $DUMP); sha256sum /home/osuser/$(basename $DUMP); gzip -t /home/osuser/$(basename $DUMP) && echo "gzip OK"
```
The size and checksum must match step 4.1, and the last line must be `gzip OK`.

**4.4 Clean up inside the pod (mandatory)**
```bash
oc exec $POD -n prod-dc-transaction -c transactionservice -- rm -f $DUMP; oc exec $POD -n prod-dc-transaction -c transactionservice -- ls -lh /tmp
```

---

## 5. Post-checks

**Pod still healthy (no new restart, container still running):**
```bash
oc get pod $POD -n prod-dc-transaction -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[*].ready,RESTARTS:.status.containerStatuses[*].restartCount,REASON:.status.containerStatuses[*].lastState.terminated.reason'
```
```bash
oc adm top pod $POD -n prod-dc-transaction --containers
```

**If the pod restarted during the dump (exit 137):**
- **Do not retry** on another pod in the same window.
- Record the time and report it to the team.
- The dump is lost. That is acceptable, since the pod restarts cleanly and releases its leaked memory.

---

## 6. Move the dump for analysis

- Move `/home/osuser/heap_<pod>_<ts>.hprof.gz` **only** to the location approved by bank security. Transfer it encrypted.
- **Do not** upload the `.hprof` to Red Hat, email or any shared drive. It contains transaction data.
- Record who has a copy and where it is stored.

---

## 7. Analyse in Eclipse Memory Analyzer (MAT)

**7.1 Setup**
- Use MAT 1.15 or later.
- In `MemoryAnalyzer.ini`, set `-Xmx8g`. A 2–3 GB heap needs about 6–8 GB of RAM for MAT.
- Unzip first: `gunzip heap_<pod>_<ts>.hprof.gz`, then open the `.hprof` with **File → Open Heap Dump**. Parsing takes several minutes the first time.

**7.2 Leak Suspects**
- On open, choose **Leak Suspects Report**.
- Expect one big suspect: a Reactor Netty pool registry (`PooledConnectionProvider` / `channelPools` map) or a similar holder retaining ~1.5 GB.
- Export: **File → Export Report → Leak Suspects** (HTML zip).

**7.3 Path to GC Roots (Red Hat's instruction)**
1. Open the **Histogram** (toolbar icon).
2. In the regex filter row, type `PushVerificationRequest` and press Enter.
3. Right-click the row → **List objects → with incoming references**.
4. Right-click any one instance → **Path To GC Roots → exclude weak/soft/phantom references**.
5. Expand the chain to the top. **Screenshot the full chain.** It shows which object or thread keeps the request alive.
6. Repeat steps 2–5 for `reactor.netty.resources.PooledConnectionProvider$PoolKey` and `io.netty.handler.ssl.JdkSslClientContext`.

**7.4 Dominator Tree**
- Open the **Dominator Tree** and sort by **Retained Heap**.
- Screenshot the top 10 entries. The top entry should be the holder found in 7.2.

**7.5 Optional OQL check (same counts as the histogram)**
```sql
SELECT COUNT(*) FROM com.epay.transaction.externalservice.request.PushVerificationRequest
```
```sql
SELECT COUNT(*) FROM reactor.netty.resources.PooledConnectionProvider$PoolKey
```

---

## 8. What to share (and what not to share)

| Share with Red Hat and the app team | Never share |
|---|---|
| Histogram text file (step 3) | The `.hprof` / `.hprof.gz` file |
| MAT Leak Suspects report (HTML) | Screenshots showing field **values** (card numbers, tokens, customer data) |
| Path to GC Roots screenshots (class names only) | |
| Dominator Tree screenshot | |

Before sharing any screenshot, **check that no object field values are visible**. Class names and sizes only.

---

## 9. Cleanup after analysis

- Delete the dump from the bastion once it has been moved to the approved location:
```bash
rm -f /home/osuser/heap_*.hprof.gz; ls -lh /home/osuser | grep -i hprof || echo "No dump files left on bastion"
```
- Delete the dump from the analysis machine after the RCA is closed, as agreed with bank security.
- Keep the histogram text files. They contain no sensitive data.

---

## 10. Quick reference

| Step | What | Pause on pod |
|---|---|---|
| 3 | Histogram (full GC) | 1–3 s |
| 4.1 | Heap dump, gz level 5 (full GC + write) | ~20–60 s |
| 4.2 | Copy to bastion | none |
| 4.4 | Delete dump in pod | none |
