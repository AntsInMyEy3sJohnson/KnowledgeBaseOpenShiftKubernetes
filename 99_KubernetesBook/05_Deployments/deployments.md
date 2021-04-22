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

* ReplicaSets manage Pods -- Deployments manage ReplicaSets
* Deployments and ReplicaSets, too, are loosely coupled merely by labels -- the latter carry some labels, and the former define a selector matching those labels


