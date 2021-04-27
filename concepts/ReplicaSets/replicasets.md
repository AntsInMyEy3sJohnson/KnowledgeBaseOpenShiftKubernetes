# ReplicaSets

* A _ReplicaSet_ considers a replicated set of Pods to be a single entitiy to be defined and managed, and in so doing, it acts as a cluster-wide Pod manager that makes sure the right types and number of Pods are active
* Pods under management of a ReplicaSet automatically get rescheduled under failure conditions such as node failures and network partitions
* The definition of a ReplicaSet contains three things:
    * A specification for the Pods to be created
    * The desired number of Pods to run (i. e., the number of _replicas_)
    * A set of labels that specifiy how to find the Pods the ReplicaSet should manage
* Act of managing replicated Pods serves as good example for reconciliation loop

## Reconciliation loops

* Basic idea: desired state vs. actual state
* "Desired state" in case of ReplicaSet: desired number of replicas + definition of Pod to be replicated
* The reconciliation loop -- which is constantly running -- takes action to align the observed cluster state with the desired state 

## How to link ReplicaSets to Pods

* In accordance to the key theme of _decoupling_ that runs through all of Kubernetes, ReplicaSets and the Pods they are supposed to manage are loosely coupled
* This means ReplicaSets, although responsible for managing a set of Pods, do not own those Pods
* Instead, ReplicaSets make use of label queries the identify the set of Pods they are responsible for

Decoupled nature between ReplicaSets and Pods entails several advantages:
* Existing Pods can simply be "picked up" by a ReplicaSet -- if the ReplicaSet owned those Pods, then the only way to switch from manual Pod management to ReplicaSet-based would be to delete all affected Pods and re-create them under the management of a new ReplicaSet, but that wouldn't be possible without service disruption
* Pods can be decoupled from the ReplicaSet managing them by simply modifying their labels -- the ReplicaSet controller will notice the cluster's observed state does not match the desired state anymore and create a new Pod, but the disassociated Pod will remain available so it can, for example, be debugged

## Designing with ReplicaSets

* A ReplicaSet should represent a single, scalable microservice inside a cluster's architecture of Kubernetes objects
* Pods should be stateless -- one Pod should be replacable by any other, and it should not matter to the service as a whole which Pod gets deleted when the ReplicaSet is scaled down

## Specification for ReplicaSets

* ReplicaSets must be uniquely named and, like all Kubernetes objects, carry a `spec` section
* A ReplicaSet's `spec` section describes the number of replicas and a template containing defining the Pod to be created whenever the observed number of replicas is less than the desired number of replicas

Sample manifest:

```
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: hello-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-service
  template:
    metadata: 
      labels: 
        app: hello-service
    spec:
      containers:
      - name: hello-service
        image: "antsinmyey3sjohnson/hello-container-service:1.0"
        ports: 
        - containerPort: 8081
```

* ReplicaSets use a set of Pod labels to monitor cluster state 
* Upon initial creation, the ReplicaSet will fetch a list of Pods from the Kubernetes API server using the labels given in its `spec.selector.matchLabels` section, and based on the number of Pods returned in the list, it will either delete some Pods or create new ones
* Caution: Make sure selector in `spec` section of ReplicaSet is proper subset of labels provided in Pod template!

## ReplicaSet creation

```
# Sample manifest to create a ReplicaSet
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: awesome-namespace
---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: hello-service
  namespace: awesome-namespace
spec:
  selector:
    matchLabels:
      app: hello-service
  replicas: 1
  template:
    metadata:
      labels:
        app: hello-service
    spec:
      containers:
      - name: hello-service-container
        image: antsinmyey3sjohnson/hello-container-service:1.0
        ports:
        - containerPort: 8081
          protocol: TCP
EOF
```

## Inspecting a ReplicaSet

`kubectl -n awesome-namespace describe rs hello-service`

### Finding a ReplicaSet from a Pod

The entity that created another entity is automatically added to the list of owner references on the created entity by the Kubernetes control plane. In case of a Pod spawned by a ReplicaSet, the `.metadata.ownerreferences` field will contain information on the owning ReplicaSet

```
╰─ k explain pod.metadata.ownerReferences                                                                                                                                      ─╯
KIND:     Pod
VERSION:  v1

RESOURCE: ownerReferences <[]Object>

DESCRIPTION:
     List of objects depended by this object. If ALL objects in the list have
     been deleted, this object will be garbage collected. If this object is
     managed by a controller, then an entry in this list will point to this
     controller, with the controller field set to true. There cannot be more
     than one managing controller.

     OwnerReference contains enough information to let you identify an owning
     object. An owning object must be in the same namespace as the dependent, or
     be cluster-scoped, so there is no namespace field.
```

Finding the ReplicaSet that has created a given Pod is thus a matter of retrieving the value for that annotation from the Pod's cluster representation: 

```
╰─ k -n awesome-namespace get pod hello-service-nts8r -o jsonpath='{.metadata.ownerReferences}' | jq                                                                           ─╯
[
  {
    "apiVersion": "apps/v1",
    "blockOwnerDeletion": true,
    "controller": true,
    "kind": "ReplicaSet",
    "name": "hello-service",
    "uid": "19708b70-e73d-4f40-8ce8-bf67e2e949f2"
  }
]
```

### Given the ReplicaSet, find the Pods it manages

Do label search. Example: 

```
╰─ k -n awesome-namespace get pod -l app=hello-service                                                                                                                         ─╯
NAME                  READY   STATUS    RESTARTS   AGE
hello-service-nts8r   1/1     Running   0          21m
```

## ReplicaSet auto-scaling

* Challenge: Running _enough_ ReplicaSets at all time is sufficient, but the definition of _enough_ varies from application to application
* Solution: Scale in reponse to custom application metrics
* Mechanism in Kubernetes that makes use of this: _Horizontal Pod Autoscaling_ (_HPA_)
* HPA requires the _heapster_ Pod be present in the cluster -- it is shipped with most Kubernetes installations and usually sits in the `kube-system` namespace

```
# Example: Scale based on CPU load
$ k autoscale rs hello-service --min=1 --max=7 --cpu-percent=80
```

* More elegent approach compared to applying autoscaling imperatively: scaling declaratively using a manifest
* Kubernetes API resource for those manifests is called `HorizontalPodAutoscalers`:

```
$ k get horizontalpodautoscalers.autoscaling
$ k get hpa
```









