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

