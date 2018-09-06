# Console Operator 

An operator for OpenShift Console built
using the operator-sdk.

## Setup


```bash  
# you may need to fiddle around with $GOPATH
echo $GOPATH // /Users/bpeterse/go
GOPATH=$(pwd)
# NOTE: I need to reference gvm pkgset here
```


## Building

```bash 
# When you make changes be sure to run the command to generate code: 
operator-sdk generate k8s

# Then use the sdk to build your image. I've tended to build with a 
# simple name, then re-tag to push to hub.docker, quay.io or gitlab registry: 
operator-sdk build openshift/console-operator:v0.0.1

# if its going to the official registry on quay:
docker tag openshift/console-operator:v0.0.1 quay.io/openshift/console-operator:latest
# if you are going to your own for testing:
# (just be sure you update your yaml files to reference your image)
docker tag openshift/console-operator:v0.0.1 \ 
   quay.io/benjaminapetersen/console-operator:latest 

# then push the image to the registry of your choice 
docker push quay.io/benjaminapetersen/console-operator:latest

# create a namespace for testing your operator. In the future, this will 
# be pinned to the `openshift-console` namespace.
kubectl create namespace test-project-name
oc new-project test-project-name

# under the /deploy directory, 
# make sure the ./deploy/operator.yaml references your image
# under spec.containers[].image
# Then, deploy your operator container:
kubectl create -f ./deploy/operator.yaml
oc create -f ./deploy/operator.yaml

# also deploy RBAC 
kubectl create -f ./deploy/rbac.yaml
oc create -f ./deploy/rbac.yaml

# and the custom resource definition (CRD), otherwise the APIs 
# for your custom resource will not exist and your deployment will
# not function.
# do note that your CRD is not namespaced, this is created once 
# per cluster and represents the API for your resource.
# (however, your CR "console" is namespaced).
# with this created, you can query or your resource via api calls to 
# /apis/
kubectl create -f ./deploy/crd.yaml
oc create -f ./deploy/crd.yaml  

# finally, you can create an instance of your custom resource (CR)
# once an instance exists, your operator will take over managing it 
kubectl create -f ./deploy/cr.yaml
oc create -f ./deploy/cr.yaml 

# now, check to ensure that your operator creates all of the resource
# that you expect for your custom resource to function.  
# for the console, this should be at least a deployment, service, route,
# configmap, and a secret.  
kubectl get deployments     // verify "console" exists and is happy
oc get deployments          // verify "console" exists and is happy
# etc.

# you can now declaratively make changes to your custom resource on the 
# fly by updating ./deploy/cr.yaml.  For example, update the number of 
# replicas generated by changing spec.count.  
kubectl apply -f deploy/cr.yaml
oc apply -f deploy/cr.yaml

# Do the same verification check to ensure your change was applied and 
# everything is happy:
kubectl get console/<name-of-console-pod> -o yaml   // this "get" references kind/name
oc get console/<name-of-console-pod> -o yaml        // this "get" references kind/name
# or, use the group as well:
kubectl get consoles.console.openshift.io
# and the name 
kubectl get consoles.console.openshift.io openshift-console
```

## Run Locally 

Its much easier to dev & run the binary locally (rather than build it, 
put it in a container, push the container, then deploy the container...repeat.)

Follow local dev instructions from `operator-sdk` including a few extras:

```bash 
# you need a cluster!
oc cluster up # or minishift, etc
# you also need a namespace:
oc create -f deploy/namespace.yaml
# create the custom resources definitions for the api 
oc create -f deploy/crd.yaml
# rbac is a good idea 
oc create -f deploy/rbac.yaml 
# 
# now to get the operator ready to roll
# generate manifests
operator-sdk generate k8s 
# build the binary, etc
# this will appear in tmp/_output/bin
operator-sdk build console-operator:vX.X.X
# run the binary locally 
operator-sdk up local # this should do the trick
#
# finally, create a custom resource for the operator to watch:
oc create -f deploy/cr.yaml
# the operator will look at this manifest & use it to generate 
# all of the deployments, etc that are needed for the resource to function.
```

## Dependencies

If OpenShift resources are needed (they will be), do the following:

```bash
# dep ensure will drop these if ran a second time, but no imports
# exist in the project.
dep ensure --add github.com/openshift/api
dep ensure --add github.com/openshift/client-go
```