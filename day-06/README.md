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

## Change Log
- 2025-10-06: Initial notes generated for Day 12 video.
