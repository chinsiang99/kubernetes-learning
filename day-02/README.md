# This Readme file will include Deployments in kubernetes

# Deletion of whole pods or replicacontroller

> kubectl delete po/nginx

> kubectl delete rc/nginx-rc

# Kubernetes: Deployment, ReplicaSet & ReplicationController
---

## 📖 Overview

Kubernetes provides multiple controllers to ensure your applications run reliably:

- **ReplicationController (RC)** – older object, ensures a fixed number of pod replicas.
- **ReplicaSet (RS)** – modern replacement for RC, supports advanced label selectors.
- **Deployment** – manages ReplicaSets, enabling rolling updates, rollbacks, and scaling.

---

## 🚀 Concepts

| Object | Purpose | Notes |
|--------|----------|-------|
| **ReplicationController** | Keeps a set number of pod replicas running. | Legacy – avoid in new projects. |
| **ReplicaSet** | Manages pod replicas with flexible selectors. | Used by Deployments under the hood. |
| **Deployment** | High-level abstraction managing ReplicaSets. | Standard for app rollout, scaling, and updates. |

---

## 📝 Example: Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app-container
        image: nginx:1.27.0
        ports:
        - containerPort: 80
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

---

## ⚡ Commands Cheat Sheet

### Apply Deployment
```bash
kubectl apply -f deployment.yaml
```

### View Resources
```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

### Scale Deployment
- note that this will actually change the inner yaml file, not outside declared
```bash
kubectl scale --help

kubectl scale deployment my-app-deployment --replicas=5

kubectl scale rs/nginx-rs --replicas=10
```

### Update Image (Rolling Update)
```bash
kubectl set image deployment/my-app-deployment my-app-container=nginx:1.27.1
```

### Check Rollout Status
```bash
kubectl rollout history deployment/nginx-deploy

kubectl rollout status deployment/my-app-deployment
```

### Rollback to Previous Version
```bash
kubectl rollout undo deployment/my-app-deployment
```

---

## ⚠️ Best Practices

- ✅ Use **Deployment** (not ReplicationController) in modern clusters.  
- ✅ Always label pods properly to avoid selector conflicts.  
- ✅ Monitor rollout progress (`kubectl rollout status`).  
- ✅ Use `revisionHistoryLimit` to prevent too many old ReplicaSets.  
- ✅ Tune `maxSurge` and `maxUnavailable` for safe zero-downtime updates.  

---

## 📚 Resources

- 📘 [Kubernetes Docs: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)  
- 📘 [Kubernetes Docs: ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)  
- 📘 [Kubernetes Docs: ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)  


# Note

- please ntoe that the major difference between deployment and replicaset is that, if u want to upgrade the version of image, replica set will actually need to bring down all the pods at once, meanwhile deployment does not -> very important !!


- we can get the deployment yaml file also
> kubectl create deployment deployment/nginx-deploy --dry-run=client --image=nginx -o yaml > deploy-test.yaml

- we can create a change cause as well
> kubectl annotate deployment nginx kubernetes.io/change-cause="Pick up patch version"

- also need to note that only changing the template will have a new version, if change name, replicas etc, it will have no new version