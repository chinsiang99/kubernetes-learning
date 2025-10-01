# Kubernetes Namespaces Explained  

---

## 🎯 Overview  

In this lesson, you’ll learn about **Kubernetes Namespaces** — what they are, why they’re needed, and how they help you logically partition a cluster. The video also demonstrates a simple connectivity test between Services across Namespaces to show isolation and DNS resolution behavior.

---

## 📚 Topics Covered  

- Definition of Namespace in Kubernetes  
- Use cases: multi-tenancy, organization, isolation  
- Default Namespaces (`default`, `kube-system`, `kube-public`)  
- How to create and manage Namespaces  
- Resource scoping: how Namespaces impact which objects you can see/use  
- Inter-namespace access: how Services in one namespace can (or cannot) be reached from another  
- Demo: deploying a Service & Pod in one Namespace, then trying to reach it from another Namespace  

---

## 🧰 Demo / Example Commands  

### Creating Namespaces  
```bash
kubectl create namespace dev
kubectl create namespace prod
```

Or via YAML:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

### Deploy an application in a namespace  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: dev
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.23.0
        ports:
        - containerPort: 80
```

### Service in that namespace  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: dev
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### Accessing from another namespace  
```bash
kubectl exec -n other-ns <pod-name> -- curl http://nginx-svc
# Or with full DNS:
curl http://nginx-svc.dev.svc.cluster.local
```

### Listing and describing  
```bash
kubectl get namespaces
kubectl describe namespace dev

kubectl get pods -n dev
kubectl get svc -n dev
kubectl get all --all-namespaces
```

---

## ✅ Key Takeaways & Best Practices  

- Namespaces let you **partition** cluster resources without needing multiple clusters.  
- You can have the **same resource names** in different namespaces without conflict.  
- Resources in one namespace can’t see or interact with resources in another namespace *by default*.  
- When referring to Services across namespaces, use the full DNS path:  
  `<service>.<namespace>.svc.cluster.local`  
- Use Namespaces to separate environments (e.g. dev, staging, prod), teams, or logical modules.  
- Use RBAC, ResourceQuotas, and NetworkPolicies at the namespace level to further isolate and control.

> cat /etc/resolv.conf

---

## 🔁 How to Try It Yourself  

1. Create 2 or more namespaces (e.g. `dev`, `prod`).  
2. Deploy a service + pods in one namespace (`dev`).  
3. Try to access that service from a Pod in another namespace — see it fail.  
4. Use fully qualified DNS to access it across namespaces.  
5. Try listing resources with and without the `-n <namespace>` flag.  
6. Clean up when done:
   ```bash
   kubectl delete namespace dev prod
   ```

---

## 🔗 References & Further Reading  

- [Kubernetes Official: Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)  

---
