# CKA Day 12 — Kubernetes **DaemonSet**, **Job** & **CronJob** (Notes)


## TL;DR
- **DaemonSet**: ensure a Pod runs on *every* (or selected) node — great for node‑local agents (e.g., log shippers, CNI helpers).  
- **Job**: run‑to‑completion workload; retries on failure; supports parallelism.  
- **CronJob**: creates Jobs on a **schedule**; control overlap with `concurrencyPolicy`, and keep history with `successfulJobsHistoryLimit`/`failedJobsHistoryLimit`.  
- Prefer minimal images, clear resource requests/limits; label everything; use `PriorityClass` for critical node services.

---

## Prereqs
- A working Kubernetes cluster and `kubectl` configured with appropriate context.
- Cluster role/permissions to create `DaemonSet`, `Job`, and `CronJob` objects.

---

## DaemonSet — run a Pod on each node
**Use cases:** metrics exporters, log forwarders, storage/network node agents.

**Example:** node exporter on all nodes, including control-plane.
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      serviceAccountName: node-exporter
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      tolerations:
        - key: "node-role.kubernetes.io/control-plane"
          operator: "Exists"
          effect: "NoSchedule"
      containers:
        - name: node-exporter
          image: prom/node-exporter:v1.8.1
          ports:
            - containerPort: 9100
              name: metrics
          resources:
            requests: { cpu: "50m", memory: "64Mi" }
            limits:   { cpu: "200m", memory: "256Mi" }
```

**Handy commands**
```bash
kubectl get ds -A
kubectl describe ds node-exporter -n monitoring
kubectl rollout status ds/node-exporter -n monitoring
kubectl rollout restart ds/node-exporter -n monitoring
```

---

## Job — run to completion
**Simple Job**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-2000
spec:
  backoffLimit: 4
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: pi
          image: perl:5.34
          command: ["perl","-Mbignum=bpi","-wle","print bpi(2000)"]
```

**Parallel Job** (N workers)
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: worker-job
spec:
  completions: 10
  parallelism: 3
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh","-c","echo processing $(date); sleep 10"]
```

**Commands**
```bash
kubectl get jobs
kubectl logs job/worker-job --all-containers=true
kubectl delete job worker-job
```

---

## CronJob — scheduled Jobs
**Safe, non-overlapping backup at 02:00 MYT**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
  namespace: ops
spec:
  schedule: "0 2 * * *"
  timeZone: "Asia/Kuala_Lumpur"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: alpine:3.20
              args: ["sh","-c","/scripts/backup.sh"]
```

**Operate**
```bash
kubectl get cronjobs -n ops
kubectl create job --from=cronjob/nightly-backup manual-backup -n ops
kubectl logs -l job-name=manual-backup -n ops --tail=100
```

> Notes:
> - `concurrencyPolicy: Forbid` prevents overlap; use `Replace` to cancel and start a fresh one.
> - `timeZone` ensures the schedule follows your local time (K8s ≥1.27).

---

## Best Practices (quick)
- **Idempotent jobs**; use app‑level locking only when necessary.
- **Resource requests/limits** for predictable scheduling; set **PodDisruptionBudget** for critical DaemonSets.
- Add **tolerations**/**nodeSelector**/**affinity** to target nodes precisely.
- Use **.spec.template.metadata.labels** consistently; they drive selectors, metrics, and cleanup.
- Rotate images and use **rolling updates** for DaemonSets; verify with `kubectl rollout status`.

---

## Troubleshooting
- Job stuck? Check **Events** and **backoffLimit**.  
- CronJob missed a run? Inspect **`startingDeadlineSeconds`**, controller time, and `timeZone`.  
- DaemonSet missing on a node? Verify node labels/taints and the DaemonSet’s selectors/tolerations.

---

## References
- Kubernetes DaemonSet concepts and tasks  
- Kubernetes Job and CronJob concepts + `kubectl create job/cronjob` refs

---


# README: Debugging Kubernetes CronJobs

## 🧠 What is a CronJob?
A **CronJob** in Kubernetes allows you to schedule **Jobs** to run periodically, similar to a Linux `cron` task.

Example:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
spec:
  schedule: "*/5 * * * *"  # Every 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: busybox
            command: ["echo", "Performing backup..."]
          restartPolicy: OnFailure
```
This runs a **Job** every 5 minutes.

---

## 🔍 Step-by-Step Debugging Guide

### 1️⃣ Check if the CronJob Exists
```bash
kubectl get cronjob
```
Look for the CronJob in the list:
```
NAME          SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
db-backup     */5 * * * * False     0        3m50s           1h
```
If `LAST SCHEDULE` shows a recent timestamp — your CronJob **is running**.

---

### 2️⃣ List Jobs Created by the CronJob
Each CronJob run creates a **Job** object.
```bash
kubectl get jobs --watch
```
Example output:
```
NAME                    COMPLETIONS   DURATION   AGE
db-backup-28389123      1/1           5s         3m
db-backup-28389124      0/1           10s        1m
```

If there are **no Jobs**, see step 5 below.

---

### 3️⃣ Inspect the Job Created
```bash
kubectl describe job db-backup-28389123
```
Check:
- Pod statuses (Succeeded/Failed)
- Start/Completion time
- Events section (for scheduling or image errors)

---

### 4️⃣ Inspect the Pod Logs
Each Job spawns a **Pod**. Check its logs:
```bash
kubectl get pods --selector=job-name=db-backup-28389123
kubectl logs <pod-name>
```
If your CronJob ran but failed inside the container, the logs will show why.

---

### 5️⃣ If No Jobs Are Being Created
If your CronJob isn’t spawning Jobs, check the following:

| Problem | Command | Fix |
|----------|----------|-----|
| CronJob suspended | `kubectl get cronjob db-backup -o yaml | grep suspend` | Set `suspend: false` |
| Invalid schedule | Review the `schedule:` field | Verify syntax with [crontab.guru](https://crontab.guru/) |
| Controller issues | `kubectl get events -n <namespace>` | Look for `FailedCreate` or controller errors |
| Missing permissions | Check RBAC / ServiceAccount | Ensure it can create Jobs |
| Timezone mismatch | Cluster uses UTC | Adjust schedule accordingly |

---

### 6️⃣ View Detailed History
```bash
kubectl describe cronjob db-backup
```
You’ll see:
- `Last schedule time`
- `Active jobs`
- `Successful/failed job history limits`

---

### 7️⃣ Run Manually for Testing
```bash
kubectl create job --from=cronjob/db-backup db-backup-manual
```
This runs the CronJob **immediately** as a Job — great for debugging.

---

## 🧰 Common Mistakes

✅ Using wrong timezone (Kubernetes CronJobs use **UTC**)  
✅ Not setting `restartPolicy: OnFailure`  
✅ Image pull issues — check `kubectl describe pod`  
✅ Misconfigured command syntax or entrypoint errors  

---

## ✅ Summary

| Step | What to Check | Command |
|------|----------------|----------|
| 1 | CronJob existence & schedule | `kubectl get cronjob` |
| 2 | Jobs spawned | `kubectl get jobs` |
| 3 | Job details | `kubectl describe job <job-name>` |
| 4 | Logs | `kubectl logs <pod-name>` |
| 5 | CronJob configuration | `kubectl describe cronjob <cronjob-name>` |
| 6 | Run manually | `kubectl create job --from=cronjob/<name> test-job` |

---

## 🧩 Tip
Use this command to quickly see the most recent CronJob logs:
```bash
kubectl logs $(kubectl get pods --sort-by=.metadata.creationTimestamp -l job-name=$(kubectl get jobs --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1:].metadata.name}') -o jsonpath='{.items[-1:].metadata.name}')
```
