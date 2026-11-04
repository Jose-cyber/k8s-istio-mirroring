## 


## Create a kind cluster:

Apply the code below to create a kind cluster:

```bash
kind create cluster --config=k8s/cluster.yaml
```

## Install argocd

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

```bash
initial_password=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)  && echo -e "\nUSER:admin\nPASS: $initial_password\n"
```


kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
