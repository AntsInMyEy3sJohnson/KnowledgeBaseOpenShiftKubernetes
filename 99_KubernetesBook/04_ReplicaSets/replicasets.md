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
* 