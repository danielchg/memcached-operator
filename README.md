# Basic Tutorial Operator SDK

- [Basic Tutorial Operator SDK](#basic-tutorial-operator-sdk)
  - [Initialize](#initialize)
  - [Add API and Controller](#add-api-and-controller)
  - [Build Docker image](#build-docker-image)
  - [Deploy Operator](#deploy-operator)
  - [Enabling OLM](#enabling-olm)
    - [Install OLM](#install-olm)
    - [Creating a bundle](#creating-a-bundle)
    - [Testing bundle](#testing-bundle)
  - [References](#references)

## Initialize

```bash
mkdir memcached-operator
cd memcached-operator
operator-sdk init --domain danielchavero --repo github.com/danielchg/memcached-operator
```

## Add API and Controller

```bash
operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller
```

## Build Docker image

```bash
docker login
make docker-build docker-push IMG="danielchavero/memcached-operator:v0.0.1"
```

## Deploy Operator

In order to test our operator we are going to use `minikube`. Install `minikube` is not the goal of this tutorial, just refer to the official [minikube documentation](https://kubernetes.io/docs/tasks/tools/install-minikube/).

Once you have `minikube` installed, you can start it with:

```bash
minikube start
```

After finishing the starting of the cluster you can check that everything is fine listing for example the nodes:

```bash
$ kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
minikube   Ready    control-plane,master   83d   v1.22.3
```

With our local cluster ready we can deploy our operator:

```bash
make deploy IMG="danielchavero/memcached-operator:v0.0.1"
```

## Enabling OLM

### Install OLM

If the OLM operator is not installed yet in your cluster, you can install it with the following command:

```bash
operator-sdk olm install
```

The output of the command will be something like this:

```bash
$ operator-sdk olm install
INFO[0000] Fetching CRDs for version "latest"           
INFO[0000] Fetching resources for resolved version "latest" 
I0307 18:41:50.403556   35243 request.go:665] Waited for 1.045191839s due to client-side throttling, not priority and fairness, request: GET:https://192.168.39.91:8443/apis/scheduling.k8s.io/v1?timeout=32s
INFO[0008] Creating CRDs and resources                  
INFO[0008]   Creating CustomResourceDefinition "catalogsources.operators.coreos.com" 
INFO[0008]   Creating CustomResourceDefinition "clusterserviceversions.operators.coreos.com" 
INFO[0008]   Creating CustomResourceDefinition "installplans.operators.coreos.com" 
INFO[0008]   Creating CustomResourceDefinition "olmconfigs.operators.coreos.com" 
INFO[0008]   Creating CustomResourceDefinition "operatorconditions.operators.coreos.com" 
INFO[0008]   Creating CustomResourceDefinition "operatorgroups.operators.coreos.com"
[...]
```

In order to check the status of the OLM installation use the following command:

```bash
operator-sdk olm status
```

The output will be something like this:

```bash
$ operator-sdk olm status
INFO[0000] Fetching CRDs for version "v0.20.0"          
INFO[0000] Fetching resources for resolved version "v0.20.0" 
INFO[0001] Successfully got OLM status for version "v0.20.0" 

NAME                                            NAMESPACE    KIND                        STATUS
operatorgroups.operators.coreos.com                          CustomResourceDefinition    Installed
operatorconditions.operators.coreos.com                      CustomResourceDefinition    Installed
olmconfigs.operators.coreos.com                              CustomResourceDefinition    Installed
[...]
```

### Creating a bundle

The following commands will create a bundle, build it and push it to the the generarted container image based on `bundle.Dockerfile` :

```bash
make bundle bundle-build bundle-push
```

```bash
$ make bundle bundle-build bundle-push
/home/dchavero/Repos/Github/danielchg/memcached-operator/bin/controller-gen "crd:trivialVersions=true,preserveUnknownFields=false" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
operator-sdk generate kustomize manifests -q
cd config/manager && /home/dchavero/Repos/Github/danielchg/memcached-operator/bin/kustomize edit set image controller=controller:latest
/home/dchavero/Repos/Github/danielchg/memcached-operator/bin/kustomize build config/manifests | operator-sdk generate bundle -q --overwrite --version 0.0.1  
INFO[0000] Creating bundle.Dockerfile                   
INFO[0000] Creating bundle/metadata/annotations.yaml    
INFO[0000] Bundle metadata generated suceessfully 
operator-sdk bundle validate ./bundle
INFO[0000] All validation tests have completed successfully 
docker build -f bundle.Dockerfile -t danielchavero/memcached-operator-bundle:v0.0.1 .
Sending build context to Docker daemon  253.8MB
Step 1/14 : FROM scratch
 ---> 
Step 2/14 : LABEL operators.operatorframework.io.bundle.mediatype.v1=registry+v1
 ---> Running in 5e064bc489b1
Removing intermediate container 5e064bc489b1
 ---> 6a448f15bb85
Step 3/14 : LABEL operators.operatorframework.io.bundle.manifests.v1=manifests/
[...]
docker push danielchavero/memcached-operator-bundle:v0.0.1
The push refers to repository [docker.io/danielchavero/memcached-operator-bundle]
4fd725ef6bf7: Pushed 
98ea4d035322: Pushed 
7ba74528c9cf: Pushed 
v0.0.1: digest: sha256:594e8fb90a68bfde80ae6bfe646f33bb3b5e8b3f52321e4eb86eb86ab015566f size: 939
make[1]: Leaving directory '/home/dchavero/Repos/Github/danielchg/memcached-operator'
```

### Testing bundle

We required to allow `minikube` node to pull the images generated from our registry, in my case from Docker.io. To do this we have to run below command:

```bash
$ minikube addons configure registry-creds

Do you want to enable AWS Elastic Container Registry? [y/n]: n

Do you want to enable Google Container Registry? [y/n]: n

Do you want to enable Docker Registry? [y/n]: y
-- Enter docker registry server url: docker.io
-- Enter docker registry username: <your-user-name>
-- Enter docker registry password: 

Do you want to enable Azure Container Registry? [y/n]: n
✅  registry-creds was successfully configured
```

To test the bundle we can use the following command:

```bash
operator-sdk run bundle docker.io/danielchavero/memcached-operator-bundle:v0.0.1
```
The out should be something like this:\

```bash
$ operator-sdk run bundle docker.io/danielchavero/memcached-operator-bundle:v0.0.1
INFO[0010] Successfully created registry pod: docker-io-danielchavero-memcached-operator-bundle-v0-0-1 
INFO[0010] Created CatalogSource: memcached-operator-catalog 
INFO[0010] Created Subscription: memcached-operator-v0-0-1-sub 
INFO[0015] Approved InstallPlan install-wdt48 for the Subscription: memcached-operator-v0-0-1-sub 
INFO[0015] Waiting for ClusterServiceVersion "operators/memcached-operator.v0.0.1" to reach 'Succeeded' phase 
[...]
```

## References

- [Go Operators Tutorial](https://sdk.operatorframework.io/docs/building-operators/golang/tutorial/)
- [OLM Integration](https://sdk.operatorframework.io/docs/olm-integration/tutorial-bundle/)