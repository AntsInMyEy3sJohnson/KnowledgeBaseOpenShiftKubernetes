# DaemonSets

* Goal of both Deployments and ReplicaSets: Create mulitple replicas of a service across the cluster for redundancy and distributing load
* _DaemonSets_ also create replicas of a service, but their motivation is to create one service replica on each node (or on a subset of nodes)
* This kind of behavior is typically desired when having to place agents or daemons on each node (such as log and metrics collectors or installer daemons)
* More generally speaking, DaemonSets are not meant for traditional applications, but for services that extend the set of capabilities and features of the cluster itself
* The DaemonSet, then, ensures one copy of a Pod is running on every node on the cluster (or a subset of nodes, depending on the DaemonSet's configuration)
* Thus, DaemonSets and ReplicaSets are very similar: Both are used to create long-running services, and both ensure that the observed state of the cluster (with regards to the service they are responsible for) matches the desired state
* Labels can be used to select only a subset of nodes for a DaemonSet's Pods to run on

## The DaemonSet scheduler

* Default behavior of a DaemonSet: Create copy of a Pod on every node in the cluster
* When a node selector is employed, the set of target nodes is limited to those nodes carrying a matching set of labels
* A DaemonSet determines the node to run a Pod on at Pod creation time by setting the `spec.nodeName` field in the Pod's `spec` section
* Consequently, the Kubernetes scheduler will ignore Pods under management by DaemonSets
* DaemonSets, too, make use of the reconciliation loop to compare desired and observed state 
* Example: If desired Pod is not yet present on node, then the DaemonSet will launch one there, and if a new node is added to the cluster, a Pod will be scheduled there, too, by the same logic

## DaemonSet creation

Example manifest see YAML file.

```
$ k apply -f 99_KubernetesBook/06_DaemonSets/simple_daemonset.yaml
$ k -n my-awesome-daemonset-testing-namespace describe ds my-awesome-daemonset
Name:           my-awesome-daemonset
Selector:       app=my-awesome-daemonset
Node-Selector:  <none>
Labels:         app=my-awesome-daemonset
Annotations:    deprecated.daemonset.template.generation: 2
Desired Number of Nodes Scheduled: 3
Current Number of Nodes Scheduled: 3
Number of Nodes Scheduled with Up-to-date Pods: 3
Number of Nodes Scheduled with Available Pods: 3
Number of Nodes Misscheduled: 0
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=my-awesome-daemonset
  Containers:
   my-logging-service:
    Image:      registry.access.redhat.com/rhel7/rhel-tools
    Port:       <none>
    Host Port:  <none>
    Command:
      /bin/bash
      -c
    Args:
      while true; do echo "I do very important logging things on this node!"; sleep 5; done
    Limits:
      cpu:     200m
      memory:  512Mi
    Requests:
      cpu:        100m
      memory:     256Mi
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From                  Message
  ----    ------            ----   ----                  -------
  Normal  SuccessfulCreate  4m35s  daemonset-controller  Created pod: my-awesome-daemonset-2g549
# ...
```

* The output of the above `describe` command indicates the DaemonSet has created three Pods -- this makes sense, as the given cluster has three nodes, and each node will -- in the absence of a node selector -- receive a Pod copy

## Limiting DaemonSet Pods to specific nodes

* Labels can be used on nodes to indicate the node is particularly suited -- or not suited -- to run a certain workload
* On the DaemonSet's side, those labels can be provided in the node selector, thus telling the DaemonSet to put Pods only onto a subset of nodes carrying those labels

### Node labels

* First step is to provide labels for nodes
* Command to use in `kubectl` for doing so: `label`
* Example: `k label nodes gke-awesome-k8s-default-pool-955b6b8f-0v5r gpu=true`
* A selector can then be used to retrieve only nodes matching the given label: `k get nodes -l gpu=true`

### Node selectors

* In the context of DaemonSets, node selectors can be used to limit the set of nodes the DaemonSet's Pods can be scheduled onto
* When creating a DaemonSet, the node selector can be defined as part of the Pod spec:

```
# ...
spec:
  template: 
    metadata:
      app: some-app
      gpu: true
    spec:
      nodeSelector:
        gpu: true
      containers:
      # ...
```

Caution: Once a DaemonSet has scheduled a Pod onto a node based on some labels in the Pod spec's node selector, removing those labels -- such that the node selector query would not return that node any longer -- will cause the DaemonSet to remove the Pod from that node.

## DaemonSet updates

Mechanism equivalent to the rollout mechanism on Deployment objects.

## DaemonSet deletions

```
$ k delete -f 99_KubernetesBook/06_DaemonSets/simple_daemonset.yaml
```



