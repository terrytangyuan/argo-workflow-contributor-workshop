# argo-workflow-contributor-workshop

## Prerequisites

* K3d
* Docker
* kubectl
* Golang 1.20


## Installation

Install Argo:

```
k3d cluster rm k3s-default; k3d cluster create k3s-default --image rancher/k3s:v1.25.3-rc3-k3s1; 

kubectl create namespace argo
kubectl apply -n argo -f install/quick-start-postgres.yaml
kubectl config set-context --current --namespace argo
```

Wait for successful deployments:
```
kubectl logs -n argo deploy/argo-server | grep 'Argo Server started successfully'
kubectl logs -n argo deploy/workflow-controller | grep 'Persistence Session created successfully'
```


Open another terminal and execute:
```
kubectl port-forward svc/argo-server 31298:2746
```

https://127.0.0.1:31298

Obtain a token:
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: argo-server
  annotations:
    kubernetes.io/service-account.name: argo-server
type: kubernetes.io/service-account-token
EOF

export SECRET=argo-server
ARGO_TOKEN="Bearer $(kubectl get secret $SECRET -o=jsonpath='{.data.token}' | base64 --decode)"
echo $ARGO_TOKEN
```

## Examples

### Hello World

```
kubectl create -f examples/hello-world.yaml
kubectl describe wf hello-world

kubectl logs -n argo deploy/workflow-controller | grep workflow=hello-world

kubectl logs hello-world -c init
kubectl logs hello-world -c wait
kubectl logs hello-world -c main
```

### K8s Resource Template

```
kubectl create -f examples/k8s-resource.yaml
kubectl describe pod k8s-jobs
kubectl logs k8s-jobs -c init
kubectl logs k8s-jobs -c main
```

### Coin-flip

```
kubectl create -f examples/coinflip.yaml
kubectl logs -n argo deploy/workflow-controller | grep workflow=coinflip
```

## Running Controller Locally

For quick debug and development.

