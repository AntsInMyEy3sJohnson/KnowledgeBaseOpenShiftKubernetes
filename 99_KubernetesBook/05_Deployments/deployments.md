# Deployments

* Challenge with ReplicaSets and Pods: Both are meant to be linked to containers whose image doesn't change
* Therefore: One abstraction layer on top of ReplicaSets and Pods necessary -- one that is able to address use cases such as the release of new image versions
* In Kubernetes, this additional abstraction layer is the _Deployment_
* Deployments manage update rollouts very carefully:
    * It is possible to wait for user-specified time between the upgrade of individual Pods
    * Employs health checks to verify new application version is running correctly
    * Roll-out is stopped if too many failures occur
* A component called the _deployment controller_ running in Kubernetes' control plane is responsible for running the actual logic required to perform such sophisticated and sensible roll-outs
* Because the roll-out of a new application version is performed server-side with the Deployment object, roll-outs in places with poor internet connectivity are not a problem

## The first deployment

For manifest, see file `simple_deployment.yaml`.

* ReplicaSets manage Pods -- Deployments manage ReplicaSets, where one ReplicaSet is used to cover multiple `spec.template.spec.containers` items defined in the deployment manifest
* Deployments and ReplicaSets, too, are loosely coupled merely by labels -- the latter carry some labels, and the former define a selector matching those labels

## Creating deployments

* Spec of Deployment object very similar to spec in RepliaSet -- for example, both contain a Pod template defining the number of containers spawned for each replica managed by the deployment
* On top of that, the Deployment also defines a strategy in its `spec` field.

```
# Example for a rolling update spec field:
# ...
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
# ...
```


* Available rollout strategies: 'Recreate' and 'RollingUpdate'

## Managing Deployments

### OldReplicaSet, NewReplicaSet

```
$ k - awesome-namespace describe deployment awesome-deployment
```

* Important pieces of output from the above: `OldReplicaSet` and `NewReplicaSet` fields
* These fields contain a reference to the ReplicaSet this deployment is currently managing
* While a rollout is in progress, both fields will be populated, and once the rollout has completed, the `OldReplicaSet` field will carry a value of `<none>`

### The "rollout" command

* Command available for deployments: `kubectl rollout`
* Use `kubectl rollout history deployment <deployment>` to view history of rollouts associated to given Deployment object 

```
$ k -n awesome-namespace rollout history deployment awesome-deployment
  deployment.apps/awesome-deployment
  REVISION  CHANGE-CAUSE
  1         <none> 
  2         <none>
```

* If a rollout is currently in progress, its status can be viewed using `kubectl rollout status`. Example: `k -n awesome-namespace rollout status deployment awesome-deployment`

## Deployment updates

* The Deployment object is designed to describe a deployed application
* Two most common actions performed on a deployed application and thus actions to be supported by the Deployment objects: updating the application, and scaling the application up or down (in or out, more precisely speaking, as the mode to scale is horizontal scaling)

### Application scaling

Scaling up and down is as simple as increasing (or decreasing) the `spec.replicas` number in the Deployment manifest:

```
# ...
spec:
  replicas: 3 -> 5
# ...
```

### Application updates

* Updating the application means rolling out a new version of its image
* This is done on the container level, i. e. in the `spec.template.spec.containers` section
* Additionally, a change cause can be recorded using the `kubernetes.io/change-cause` annotation in the Pod template, i. e. in the `spec.template.metadata.annotations` section

```
# In the Deployment's manifest file
# ...
spec:
  template:
    metadata:
      annotations:
        kubernetes.io/change-cause: "Meaningful information about the change cause goes here"
# ...

$ k -n awesome-namespace rollout history deployment awesome-deployment
deployment.apps/awesome-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         Meaningful information about the change cause goes here
```

* Caution: Any annotation change will trigger a new rollout, so use with care!
* Both the old and the new RepliaSet are available once a deployment has finished, and each ReplicaSet maintains the images used for its containers
* This way, the previous (pre-update) versions of those ReplicaSets can easily be restored
* Rollouts can be paused using the `rollout pause` command: `k -n awesome-namespace rollout pause deployment awesome-deployment`
* Similarly, a rollout can be resumed using the `rollout resume` command

### Rollout history 

* Kubernetes control plane maintains history of all rollouts
* View entire history using `rollout history` command: `k -n awesome-namespace rollout history deployment awesome-deployment`
* Detail view of a particular rollout can be summoned with the help of the `--revision` flag

```
$ k -n awesome-namespace rollout history deployment awesome-deployment --revision=1
deployment.apps/awesome-deployment with revision #1
Pod Template:
  Labels:	app=hello-service
	pod-template-hash=7bb4dc794d
  Annotations:	example.org/deployment-version: 1.0
	kubernetes.io/change-cause: Meaningful information about the change cause goes here
  Containers:
   hello-service-container:
    Image:	antsinmyey3sjohnson/hello-container-service:1.0
    Port:	8081/TCP
    Host Port:	0/TCP
    Limits:
      cpu:	300m
      memory:	512Mi
    Requests:
      cpu:	100m
      memory:	256Mi
    Environment:	<none>
    Mounts:	<none>
   tools:
    Image:	registry.access.redhat.com/rhel7/rhel-tools:latest
    Port:	<none>
    Host Port:	<none>
    Command:
      /bin/sh
      -c
    Args:
      tail -f /dev/null
    Limits:
      cpu:	200m
      memory:	256Mi
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```

### Deployment rollback

```
$ k -n awesome-namespace rollout undo deployments awesome-deployment
deployment.apps/awesome-deployment rolled back

# Revision 1 now gone -- revision 3 = revision 1 due to rev. 2 rollout back to 1
# --> The Dewployment object reuses the old template and renumbers it
$ k -n awesome-namespace rollout history deployment awesome-deployment
deployment.apps/awesome-deployment
REVISION  CHANGE-CAUSE
2         Meaningful information about the change cause goes here
3         Meaningful information about the change cause goes here

# Roll back to specific revision:
$ k -n awesome-namespace rollout undo deployments awesome-deployment --to-revision=2
```

* By default, the completed rollout history is kept with the Deployment object
* The Deployment's `spec.revisionHistoryLimit` enables operators to specify the maximum number of revisions to be retained to prevent the rollout history from growing too large

## Strategies for performing rollouts

### The "Recreate" strategy

* Less sophisticated compared to `RollingUpdate`
* `Recreate` updates the ReplicaSet under management to use the new image and deletes all Pods currently associated with the Deployment
* The ReplicaSet then notices zero out of the _n_ desired replicas are avaiable, and re-create all Pods with the new image
* Because the `Recreate` strategy always entails at least a short service downtime, it should not be used in production-grade settings, but only in scenarios where service downtime is appropriate

### The "RollingUpdate" strategy

* Slower than `Recreate`, but still preferable in most scenarios because it is able to upgrade a service without incuring downtime
* The strategy works by updating a few Pods at a time (depending on how the strategy is configured in the Deployment object) until all Pods have been updated and serve the new application version




























