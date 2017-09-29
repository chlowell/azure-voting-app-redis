# Azure Container Service Hands-On Lab
## Prerequisites
* Docker tools
  - [Docker for Windows](https://download.docker.com/win/stable/Docker%20for%20Windows%20Installer.exe)
* [kubectl](https://storage.googleapis.com/kubernetes-release/release/v1.7.0/bin/windows/amd64/kubectl.exe), the Kubernetes CLI, somewhere on your path

## Getting started with the Azure CLI
We'll use `az`, the new Python-based Azure CLI, to create resources for this lab.
It's conveniently available as a Docker container, eliminating the need to install
its dependencies on your machine.

Run the CLI container:
```sh
    $ docker run -it --rm -v c:/temp/az:/root azuresdk/azure-cli-python
    bash-4.3#
```
The volume mount simplifies moving files (e.g. Kubernetes credentials)
out of the container.

Log in to the CLI:
```sh
    bash-4.3# az login
```

You'll need a unique name for your container registry. Check candidates
with the CLI:
```sh
    bash-4.3# az acr check-name -n myregistry
```

Define some convenient environment variables:
```sh
    bash-4.3# export RESOURCE_GROUP=acs-hands-on CLUSTER_NAME=k8s-handson ACR_NAME=myregistry
```

Create a resource group:
```sh
    bash-4.3# az group create -l southcentralus -n $RESOURCE_GROUP
```

## Creating a Kubernetes cluster with Azure Container Service
Continuing with the same shell, let's deploy a cluster. The CLI is convenient
for this because it will handle creating SSH keys and a service principal. The
deployment will take a few minutes.

```sh
    bash-4.3# az acs create --orchestrator-type kubernetes -g $RESOURCE_GROUP -n $CLUSTER_NAME --agent-count 1 --generate-ssh-keys
```

Get credentials for the cluster:
```sh
    bash-4.3# az acs kubernetes get-credentials -g $RESOURCE_GROUP -n $CLUSTER_NAME
```
This will create `/root/.kube/config`. You'll need it to access the cluster from
your local machine. Verify the file exists on the local side of the volume mount
you created with `docker run`. If it doesn't, you can copy it somewhere manually
with `docker cp` (from a shell on your local machine):
```sh
    $ docker ps
    CONTAINER ID        IMAGE                       ...
    01864a9daf3e        azuresdk/azure-cli-python   ...
    $ docker cp 018:/root/.kube/config c:/temp/handson-kubeconf
```
If you later want to hack on the cluster's machines directly, you'll need the
SSH keys `az` generated; they're in the container's `/root/.ssh`.

## Creating an Azure Container Registry
This is straightforward with either the Azure Portal or the CLI. We're already
using the CLI, so let's continue with it:
```sh
    bash-4.3# az acr create -g $RESOURCE_GROUP -n $ACR_NAME --sku Basic --admin-enabled true
```

Images you want to push to this registry must be tagged with the registry's
login server name. That's of the form `$ACR_NAME.azurecr.io`; you can fetch it
with the CLI:
```sh
    bash-4.3# az acr list -g $RESOURCE_GROUP --query "[].{acrLoginServer:loginServer}" -o tsv
    myregistry.azurecr.io
```

You'll need credentials to use the registry:
```sh
    bash-4.3# az acr credential show -n $ACR_NAME -o table
```

With that, we're done with this container. You can Ctrl-D its shell.

## Pushing images to the registry
With the infrastructure in place, let's prepare the artifacts. The app
has two components: a Flask app and a redis cache.

Clone the code:
```sh
    $ git clone https://github.com/chlowell/azure-voting-app-redis
```

Build an image for the Flask app:
```sh
    $ cd azure-voting-app-redis/azure-vote
    $ docker build -t myregistry.azurecr.io/azure-vote:1.0 .
```

Log in to your registry with the information obtained from `az` above:
```sh
    $ docker login -p password -u username myregistry.azurecr.io
```

Push the image:
```sh
    $ docker push myregistry.azurecr.io/azure-vote:1.0
```

## Deploying the app to Kubernetes
First you must configure `kubectl` for your cluster. The easiest way to do this
is to define an environment variable, `KUBECONFIG`, with the path to the
configuration file from `az acs kubernetes get-credentials`:
```powershell
    $ $env:KUBECONFIG="C:/temp/az/.kube/config"
```

Verify the configuration:
```sh
    $ kubectl get no
    NAME                    STATUS                     AGE       VERSION
    k8s-agent-c4b4072e-0    Ready                      18m       v1.6.6
    k8s-master-c4b4072e-0   Ready,SchedulingDisabled   18m       v1.6.6
```

Deploy a redis pod named `backend` with `kubectl run`:
```sh
    $ kubectl run backend --image=redis:3.2.11-alpine --port=6379
    deployment "backend" created
    $ kubectl get deploy
    NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    backend      1         1         1            1           8s
    $ kubectl get po
    NAME                               READY     STATUS    RESTARTS   AGE
    backend-1732908694-jgfz9           1/1       Running   0          12s
```

That works, but it's more typical to specify resources in YAML files. Let's
delete the deployment and make a durable, versionable artifact specifying it:
```sh
    $ kubectl delete deploy backend
```

`kubectl` can output a YAML representation for any resource it creates:
```sh
    $ kubectl run backend --image=redis:3.2.11-alpine --port=6379 -l 'app=azure-vote,tier=backend' --dry-run -o yaml
```

We can redirect that output to a file:
```sh
    $ kubectl run backend --image=redis:3.2.11-alpine --port=6379 -l 'app=azure-vote,tier=backend' --dry-run -o yaml > deployments.yaml
```

We need a deployment for the Flask app as well:
```sh
    $ printf '---\n' >> deployments.yaml
    $ kubectl run frontend --image=myregistry.azurecr.io/azure-vote:1.0 --port=80 --env='REDIS=backend' -l 'app=azure-vote,tier=frontend' --dry-run -o yaml >> deployments.yaml
```

Now we can create the deployments from the file:
```sh
    $ kubectl create -f deployments.yaml
```

That gets Kubernetes deploying the pods we need, but the frontend can't
communicate with redis, and nobody can reach the frontend. We need to expose
the pods with services. A service is a durable interface to a collection of
pods. The pods are transient, but the service's name can always be used to
find them via DNS.

Let's use `kubectl expose` to get YAML describing a service for the backend:
```sh
    $ kubectl expose deploy backend --port=6379 --type=ClusterIP -l 'app=azure-vote,tier=backend' --dry-run -o yaml > services.yaml
```

A service of type `ClusterIP` will only be accessible from within the cluster.
This is the default, and what we want here because the backend shouldn't be
exposed externally.

However, we want the frontend exposed to the internet. For that we need a
`LoadBalancer` service:
```sh
    $ printf '---\n' >> services.yaml
    $ kubectl expose deploy frontend --port=80 --type=LoadBalancer -l 'app=azure-vote,tier=frontend' --dry-run -o yaml >> services.yaml
```

Now we can create the services:
```sh
    $ kubectl create -f services.yaml
```
It may take Azure a few minutes to set up a load balancer with a public IP for
the `frontend` service.

## Monitoring the app
`kubectl` can provide broad overviews of Kubernetes objects as well as deeply
detailed descriptions of them:
```sh
    $ kubectl get deploy,po,svc -l app=azure-vote
    NAME                DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    deploy/frontend     1         1         1            1           4m
    deploy/backend      1         1         1            1           4m

    NAME                             READY     STATUS    RESTARTS   AGE
    po/frontend-3100128220-3j313     1/1       Running   0          4m
    po/backend-2290028658-t9ntf      1/1       Running   0          4m

    NAME             CLUSTER-IP    EXTERNAL-IP    PORT(S)        AGE
    svc/frontend     10.0.116.96   13.84.159.21   80:30267/TCP   4m
    svc/backend      10.0.9.168    <none>         6379/TCP       4m
    $ kubectl describe po/frontend-3100128220-3j313
    ... much detail ...
```

Kubernetes also includes a web dashboard:
```sh
    $ kubectl proxy
    Starting to serve on 127.0.0.1:8001
```
Browse to [http://localhost:8001/ui](http://localhost:8001/ui) to see the dashboard.
