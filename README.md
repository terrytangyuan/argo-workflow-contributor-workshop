# Contributor Workshop

## Prerequisites

* K3d
* Docker
* kubectl
* Golang 1.20


## Installation

Install Argo:

```
k3d cluster rm argo-workshop; k3d cluster create argo-workshop --image rancher/k3s:v1.25.3-rc3-k3s1;
kubectl create namespace argo
kubectl apply -n argo -f install/quick-start-postgres.yaml
kubectl config set-context --current --namespace argo
```

Wait for successful deployments:
```
kubectl logs -n argo deploy/postgres | grep 'database system is ready to accept connections'
kubectl logs -n argo deploy/argo-server | grep 'Argo Server started successfully'
kubectl logs -n argo deploy/workflow-controller | grep 'Persistence Session created successfully'
```

Open another terminal and execute:
```
kubectl port-forward svc/argo-server 31298:2746
```

Obtain a token:
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: argo-server
  name: argo-server
type: kubernetes.io/service-account-token
---
apiVersion: v1
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: default
  name: default.service-account-token
type: kubernetes.io/service-account-token
EOF

echo "\n\n"
export SECRET=argo-server
ARGO_TOKEN="Bearer $(kubectl get secret $SECRET -o=jsonpath='{.data.token}' | base64 --decode)"
echo $ARGO_TOKEN
```

Access the UI: https://127.0.0.1:31298

## Examples

### Workflows with Single Template

#### Hello World

```
kubectl create -f examples/hello-world.yaml
kubectl describe wf hello-world
kubectl logs deploy/workflow-controller | grep workflow=hello-world
kubectl logs hello-world -c init
kubectl logs hello-world -c wait
kubectl logs hello-world -c main
```

#### K8s Resource Template

```
kubectl create -f examples/k8s-resource.yaml
kubectl describe pod k8s-jobs
kubectl logs deploy/workflow-controller | grep workflow=k8s-resource
kubectl logs k8s-jobs -c init
kubectl logs k8s-jobs -c main
```

#### ContainerSet Template

```
kubectl create -f examples/containerset.yaml
kubectl describe pod containerset
kubectl logs containerset -c a
kubectl logs containerset -c b
kubectl logs containerset -c init
kubectl logs containerset -c wait
```

#### HTTP Template

```
kubectl create -f examples/http-template.yaml
kubectl logs -l workflows.argoproj.io/workflow=http-template
kubectl logs deploy/workflow-controller | grep workflow=http-template
kubectl describe workflowtaskset http-template
```

### Workflows with Multiple Templates

#### Coin-flip

Defined as steps:

```
kubectl create -f examples/coinflip.yaml
kubectl describe wf coinflip
kubectl logs deploy/workflow-controller | grep workflow=coinflip
```

#### Diamond

Defined as DAG:

```
kubectl create -f examples/dag-diamond.yaml
kubectl describe wf dag-diamond
kubectl logs deploy/workflow-controller | grep workflow=dag-diamond
```

### Debugging

#### Debug Pause

```
kubectl create -f examples/debug-pause.yaml

# Create a shell in the container of interest of create a ephemeral container in the pod, in this example ephemeral containers are used.
kubectl debug -n argo -it debug-pause --image=busybox --target=main --share-processes

# Create the marker file to allow the workflow step to continue
touch /proc/1/root/run/argo/ctr/main/after

kubectl describe pod debug-pause
```


#### Inspect Database

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

```
kubectl delete namespace argo
kubectl create namespace argo
kubectl apply -n argo -f install/quick-start-simple.yaml
kubectl config set-context --current --namespace argo

git clone https://github.com/argoproj/argo-workflows.git $GOPATH/src/github.com/argoproj/argo-workflows
cd $GOPATH/src/github.com/argoproj/argo-workflows
LEADER_ELECTION_DISABLE=true go run ./cmd/workflow-controller/main.go -n argo --gloglevel 6 --namespaced --loglevel debug

kubectl create -f examples/hello-world.yaml
```

If there are changes in executor code, we need to rebuild the executor image:

```
docker buildx build --no-cache -t argoproj/argoexec:test-oss-1 --target argoexec --progress plain .
k3d image import argoproj/argoexec:test-oss-1

LEADER_ELECTION_DISABLE=true go run ./cmd/workflow-controller/main.go -n argo --gloglevel 6 --namespaced --loglevel debug --executor-image argoproj/argoexec:test-oss-1 --executor-image-pull-policy IfNotPresent

kubectl create -f examples/hello-world.yaml
kubectl describe pod hello-world
```

## Exercises

1. Modify `examples/coinflip.yaml` so that the 'flip-coin' step is recursively repeated until the result of the step is "heads". Hint: modify `coinflip.tails` template.
2. Add another step in `examples/k8s-resource.yaml` to print out the name of the K8s Job from the resource template. Hint: create a multi-step template where the new step takes the output of the previous step as the input parameter.
