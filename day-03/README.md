# Kubernetes Service Exposure (Imperative Way)

This guide shows how to expose Pods, Deployments, or ReplicaSets using **imperative commands** in Kubernetes.  
These shortcuts are especially useful for the **CKA exam**, where speed is important.

---

## 1. Expose as ClusterIP (default)

Expose a Pod internally within the cluster:

```bash
kubectl expose pod mypod   --name=clusterip-svc   --port=80   --target-port=8080
```

- `--port=80`: Port exposed by the Service  
- `--target-port=8080`: Container port inside the Pod  
- Default `--type` = `ClusterIP`  

✅ Accessible **inside the cluster** at `clusterip-svc:80`

---

## 2. Expose as NodePort

Expose a Deployment externally via NodePort:

```bash
kubectl expose deployment myapp   --name=nodeport-svc   --port=80   --target-port=8080   --type=NodePort
```

- NodePort is auto-assigned (range: 30000–32767)  
- To see the assigned port:

```bash
kubectl get svc nodeport-svc
```

✅ Accessible **inside cluster** at `nodeport-svc:80`  
✅ Accessible **outside cluster** at `<NodeIP>:<NodePort>`

---

## 3. Expose as NodePort with Specific Port

If you want to fix the NodePort value:

```bash
kubectl expose deployment myapp   --name=nodeport-svc   --port=80   --target-port=8080   --type=NodePort   --node-port=30080
```

✅ Accessible externally at `<NodeIP>:30080`

---

## 4. Expose as LoadBalancer (Cloud Only)

```bash
kubectl expose deployment myapp   --name=lb-svc   --port=80   --target-port=8080   --type=LoadBalancer
```

- In cloud providers (AWS, GCP, Azure), this provisions an external load balancer  
- In **CKA exam clusters**, `EXTERNAL-IP` will usually stay `<pending>`

---

## 5. Expose as ExternalName

If you want to map a Kubernetes Service to an **external DNS hostname**:

```bash
kubectl create service externalname external-svc   --external-name=db.company.com   --port=5432
```

✅ Inside cluster, Pods can connect via `external-svc:5432`  
➡️ This resolves to `db.company.com:5432`

---

## 6. Test Service Connectivity

### Option A: Temporary Busybox Pod (quick one-off)

```bash
kubectl run tmp-busybox   --rm -it   --image=busybox:1.28   --restart=Never   -- wget -O- <SERVICE_CLUSTER_IP>:<PORT>
```

- Deletes itself after the test (`--rm`).  
- Useful if no other Pods are available.  

---

### Option B: Use `kubectl exec` on an existing Pod

If you already have a running Pod (e.g., your app Pod), you can test directly from inside it:

```bash
kubectl exec -it <POD_NAME> -- wget -O- <SERVICE_CLUSTER_IP>:<PORT>
```

Or using curl (if available in the Pod image):

```bash
kubectl exec -it <POD_NAME> -- curl <SERVICE_NAME>:<PORT>
```

✅ Faster than creating a new Pod.  
✅ Good practice for CKA/CKAD exams since it saves time.  

---

## 7. Test Service from Outside the Cluster

For **NodePort Services**, you can test access using the Node’s IP and NodePort:

```bash
wget -O- http://<NODE_IP>:<NODE_PORT>
```

- `<NODE_IP>` = IP of any worker node (use `kubectl get nodes -o wide`)  
- `<NODE_PORT>` = NodePort allocated (check with `kubectl get svc`)  

Example:

```bash
wget -O- http://192.168.1.101:30080
```
---

## Quick Reference

| Service Type   | Command Example                                                                 | Use Case                              |
|----------------|----------------------------------------------------------------------------------|----------------------------------------|
| ClusterIP      | `kubectl expose pod mypod --port=80 --target-port=8080`                          | Internal-only communication            |
| NodePort       | `kubectl expose deployment myapp --type=NodePort --port=80 --target-port=8080`   | External access via `<NodeIP>:NodePort` |
| NodePort fixed | `kubectl expose deployment myapp --type=NodePort --node-port=30080 --port=80`    | External with specific port             |
| LoadBalancer   | `kubectl expose deployment myapp --type=LoadBalancer --port=80 --target-port=8080` | Cloud external load balancer           |
| ExternalName   | `kubectl create service externalname external-svc --external-name=db.company.com --port=5432` | External DNS alias                     |

---
