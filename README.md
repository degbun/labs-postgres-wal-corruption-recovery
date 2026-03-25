#  labs-postgres-wal-corruption-recovery

> **Deliberately corrupt a PostgreSQL WAL on Kubernetes, watch it flatline, then bring it back with `pg_resetwal`.**

This lab walks you through a full disaster recovery scenario — from a healthy StatefulSet to a CrashLoopBackOff, and back to a running database. No magic, no shortcuts. Every step explained.

---

## 🧠 What You'll Learn

- How PostgreSQL's Write-Ahead Log (WAL) works and why its corruption kills the database
- What `pg_resetwal` actually does under the hood
- How to orchestrate an emergency repair on Kubernetes without losing your PVC
- Common pitfalls and how to avoid them (including the `cannot be executed by root` trap)

---

## ⚙️ Prerequisites

| Requirement | Version |
|---|---|
| Kubernetes cluster | minikube, kind, or any cluster |
| kubectl | any recent version |
| PostgreSQL image | `postgres:15` |
| Storage | ReadWriteOnce PVC support |

---

## 🗂️ Project Structure

```
postgres-flatline-lab/
├── postgres-sts.yaml     # StatefulSet + Headless Service + PVC
├── rescue-pod.yaml       # Emergency rescue pod (mounts the same PVC)
└── README.md
```

---

## 🚀 The Lab — Step by Step

### Phase 1 — Deploy PostgreSQL

Deploy a PostgreSQL StatefulSet with a persistent volume.

```bash
kubectl apply -f postgres-sts.yaml
kubectl wait pod/postgres-0 --for=condition=Ready --timeout=60s
```

> The PVC created by `volumeClaimTemplates` will be named **`postgres-data-postgres-0`**. Keep that name handy.

<details>
<summary>📄 postgres-sts.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          env:
            - name: POSTGRES_PASSWORD
              value: "password"
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 5Gi
```

</details>

---

### Phase 2 — Corrupt the WAL

Overwrite a WAL segment with random bytes using `dd`. This simulates a disk corruption or a forced volume detachment.

```bash
kubectl exec -it postgres-0 -- bash -c \
  "dd if=/dev/urandom \
     of=/var/lib/postgresql/data/pgdata/pg_wal/000000010000000000000001 \
     bs=1M count=1 conv=notrunc"

# Force a pod restart
kubectl delete pod postgres-0

# Watch it crashloop
kubectl get pods -w
```

Expected output:

```
NAME         READY   STATUS             RESTARTS
postgres-0   0/1     CrashLoopBackOff   3
```

Check the logs to confirm the corruption:

```bash
kubectl logs postgres-0
# FATAL:  could not open file "pg_wal/000000010000000000000001"
# DETAIL: Invalid magic number 0000 in log segment
# LOG:    startup process (PID 48) was terminated by signal 6: Aborted
```

PostgreSQL attempts crash recovery, reads the corrupted WAL, and dies. Every single restart. That's the loop.

---

### Phase 3 — Scale Down + Rescue Pod

The PVC is `ReadWriteOnce` — only one pod can mount it at a time. Scale down the StatefulSet first to release the volume.

```bash
# Release the PVC
kubectl scale statefulset postgres --replicas=0
kubectl wait pod/postgres-0 --for=delete --timeout=30s

# Confirm no pod is holding the volume
kubectl get pods
```

Now deploy the rescue pod. It uses the same image and mounts the same PVC, but starts with `sleep infinity` — preventing PostgreSQL from attempting (and failing) crash recovery automatically.

```bash
kubectl apply -f rescue-pod.yaml
kubectl wait pod/postgres-rescue --for=condition=Ready --timeout=30s
```

<details>
<summary>📄 rescue-pod.yaml</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-rescue
spec:
  securityContext:
    runAsUser: 999    # postgres user UID in the official image
    runAsGroup: 999
    fsGroup: 999
  containers:
    - name: rescue
      image: postgres:15
      command: ["sleep", "infinity"]
      env:
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
      volumeMounts:
        - name: pgdata
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: pgdata
      persistentVolumeClaim:
        claimName: postgres-data-postgres-0
```

</details>

> **Why `securityContext`?**
> `pg_resetwal` refuses to run as root. Without `runAsUser: 999`, `kubectl exec` opens a root session and the repair fails immediately.
> You can verify the UID beforehand with: `kubectl exec -it postgres-rescue -- id postgres`

---

### Phase 4 — Repair with pg_resetwal

```bash
kubectl exec -it postgres-rescue -- bash

# Inspect the corrupted state
pg_controldata /var/lib/postgresql/data/pgdata | grep -E "state|checkpoint"
# Database cluster state: in production  ← crash confirmed

# Run the repair
pg_resetwal -f /var/lib/postgresql/data/pgdata
# Write-Ahead Log reset

# Verify the repair
pg_controldata /var/lib/postgresql/data/pgdata | grep state
# Database cluster state: shut down  ← success

exit
```

> **Why `-f`?**  
> `pg_control` still shows `in production` instead of `shut down` because the pod was killed abruptly. Without `-f`, pg_resetwal refuses to act. The flag says: *"I know it's dirty, do it anyway."*

> ⚠️ **`pg_resetwal` is a last resort.** It discards all WAL segments and rewrites `pg_control`. Any transactions committed after the last checkpoint and before the corruption are **permanently lost.**

---

### Phase 5 — Rescale and Verify

```bash
# Release the PVC
kubectl delete pod postgres-rescue

# Bring the StatefulSet back
kubectl scale statefulset postgres --replicas=1

# Watch the startup
kubectl get pods -w
# NAME         READY   STATUS    RESTARTS
# postgres-0   1/1     Running   0         ← 

# Check the logs
kubectl logs postgres-0 --follow
# database system was shut down at ...
# database system is ready to accept connections

# Test the connection
kubectl exec -it postgres-0 -- psql -U postgres -c "SELECT version();"
```

After `pg_resetwal`, PostgreSQL starts in minimal recovery mode — it creates a fresh WAL timeline and comes back online. The key log line to look for:

```
database system is ready to accept connections
```

---

## 📊 What pg_resetwal Preserves vs. Destroys

| Element | Status | Why |
|---|---|---|
| Checkpointed tables | ✅ Preserved | Lives in `base/`, independent of WAL |
| Checkpointed indexes | ✅ Preserved | Same |
| Post-checkpoint transactions | ❌ Lost | Were only in the WAL |
| Sequences (`nextval`) | ⚠️ May diverge | pg_resetwal can reset them |
| Cluster parameters | ✅ Preserved | Read from `pg_control` |

---

## 🐛 Common Errors and Fixes

**`cannot be executed by root`**
```
pg_resetwal: error: cannot be executed by "root"
```
Fix: Add `securityContext.runAsUser: 999` to your rescue pod, or run `su - postgres -c "pg_resetwal -f ..."` inside the pod.

---

**`database server was not shut down cleanly`**
```
pg_resetwal: error: database server was not shut down cleanly
Use -f to force reset.
```
Fix: Add the `-f` flag — `pg_resetwal -f /path/to/pgdata`.

---

**`Multi-Attach error for volume`**
```
Multi-Attach error: volume is already exclusively attached to one node
```
Fix: You didn't wait for the StatefulSet pod to fully terminate before creating the rescue pod. Scale to 0 and wait.

---

**`No such file or directory`**
```
pg_resetwal: error: could not open file ... No such file or directory
```
Fix: Wrong PGDATA path. Check with `echo $PGDATA` or `ls /var/lib/postgresql/data/`.

---

## 💡 Production Best Practices

- **Use PITR backups** (pgBackRest, Barman) — they make pg_resetwal unnecessary in 90% of cases
- **Monitor `pg_control` state** — detect corruption before the full crash
- **Set a `PodDisruptionBudget`** on your StatefulSet — avoid brutal crashes during K8s maintenance
- **Use a `Retain` reclaim policy** on your StorageClass — your PVC survives even if the StatefulSet is deleted
- **Document a runbook** before you need it — reduces stress and mistakes at 2am
- **Run this lab in staging regularly** — keep the muscle memory fresh

---

## 🧾 Quick Reference

```bash
# Full repair flow — copy/paste ready
kubectl scale statefulset postgres --replicas=0
kubectl apply -f rescue-pod.yaml
kubectl exec -it postgres-rescue -- su - postgres -c \
  "pg_resetwal -f /var/lib/postgresql/data/pgdata"
kubectl delete pod postgres-rescue
kubectl scale statefulset postgres --replicas=1
kubectl logs postgres-0 --follow
```

---

## ⚠️ Disclaimer

`pg_resetwal` is a **last resort tool**. If you have a PITR backup, use it. Always. This lab is for learning and emergency preparedness — not a substitute for a proper backup strategy.

---

<p align="center">
  <em>The pod flatlines. You grab the defibrillator. It comes back.</em><br/>
  <strong>postgres-flatline-lab</strong>
</p>
