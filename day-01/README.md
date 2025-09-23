# Installation Kind

> brew install kind

# Creating cluster

> kind create cluster --name=<name>

> kind create cluster --config kind-example-config.yaml

# Commands to view cluster info

> kubectl cluster info

```bash
kubectl cluster info
kubectl config get-contexts                          # display list of contexts
kubectl config get-contexts -o name                  # get all context names
kubectl config current-context                       # display the current-context
kubectl config use-context my-cluster-name           # set the default context to my-cluster-name
```

