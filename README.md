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

kubectl logs deploy/workflow-controller | grep workflow=hello-world

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
kubectl logs deploy/workflow-controller | grep workflow=coinflip
```

### ContainerSet Template

```
kubectl create -f examples/containerset.yaml
kubectl describe pod containerset
kubectl logs containerset -c a
kubectl logs containerset -c b
kubectl logs containerset -c init
kubectl logs containerset -c wait
```

### HTTP Template

```
kubectl create -f examples/http-template.yaml
kubectl describe pod http-template
kubectl logs http-template
kubectl logs deploy/workflow-controller | grep workflow=http-template
kubectl describe workflowtaskset http-template
```

## Inspect Database

```
# Get into the database pod
kubectl exec --stdin --tty `kubectl get pods --no-headers -o custom-columns=":metadata.name" | grep postgres` -- bin/bash

# Log in database
psql --username postgres

# List table schema
\d argo_archived_workflows

# List archived workflows
SELECT * from argo_archived_workflows LIMIT 5;
```

## Running Controller Locally

For quick debug and development.

TBD


