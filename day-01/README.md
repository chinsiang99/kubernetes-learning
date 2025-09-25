# This README file will include cluster, nodes, pods in kubernetes

# Installation Kind

> brew install kind

# Creating cluster

> kind get clusters

> kind create cluster --name=name

> kind create cluster --config kind-example-config.yaml

> kind delete cluster --name=clustername

# Commands to view cluster info

> kubectl cluster-info

```bash
kubectl cluster-info
kubectl config get-contexts                          # display list of contexts
kubectl config get-contexts -o name                  # get all context names
kubectl config current-context                       # display the current-context
kubectl config use-context my-cluster-name           # set the default context to my-cluster-name
```

# Rename file

> mv existing-file-name new-file-name

# Verify version of pod

> kubectl explain pod

# Create nginx in imperative way

> kubectl run nginx-pod --image=nginx:latest

# Create a pod in declarative way

> kubectl delete pod nginx-pod

> kubectl create -f pod.yaml

> kubectl apply -f pod.yaml

> kubectl edit pod podname

> kubectl exec -it nginx-pod -- sh

> kubectl get pods

> kubectl get pods -o wide

# creating a yaml file directly by dry running a pod

> kubectl run nginx --image=nginx --dry-run=client -o yaml

# Describing a pod

> kubectl describe pod podname

# To get the yaml file from existing pod

> kubectl get pod nginx-pod -n default -o yaml > test.yaml

note that this actually will output more than what we declared
