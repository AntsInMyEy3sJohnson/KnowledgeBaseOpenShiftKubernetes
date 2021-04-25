# Jobs

* Long-running processes like web applications, message queues, and databases make up the majority of workloads run on Kubernetes, but there is sometimes also a need to run short-lived tasks
* Kubernetes addresses these kinds of Tasks by means of the _Job_ object
* A Deployment, for example, will restart Pods once they have terminated, regardless of their exit code
* The Job object, on the other hand, spawns Pods that run until they successfully terminate (exit status zero)
* Therefore, Jobs are the right tool for things that must run successfully exactly once, like batch jobs or data migrations

## The Job object

* A Job object's responsibility is to create Pods defined by means of the Job specification's template, which then run until successful completion, in which case they are not restarted
* The Job object can coordinate a number of Pods in parallel
* The Job controller will spawn a new Pod based on its template whenever a Pod terminates unsuccessfully
* Caution: A Job's Pods have to be scheduled like any other, so if the required resources are not available in the cluster, the Job won't be able to launch the Pod
* Due to Kubernetes being an inherently distributed system, there is also a chance that a Job will launch duplicate Pods in some failure scenarios

## Job patterns

* Jobs are best suited for batch-like workloads where individual work items are taken on by one or more Pods
* A Job's behavior is defined by the two primary attributes of number of job completions and number of Pods to be run in parallel -- the default is _1_ for both, meaning a Job will spawn one Pod and let it run until it has completed successfully once (_run once until completion_ pattern)
* Often-used Job patterns:
    * __Completions = 1; parallelism = 1__: "One-shot" jobs -- one Pod runs until it has successfully terminated once, often used for data migrations and the like
    * __Completions >= 1; parallelism >= 1__: Multiple Pods process a workload in parallel -- one or more Pods run one or more times until a pre-defined completion count is reached
    * __Completions = 1; parallelism >= 2__: Multiple Pods work on the load of a centralized queue -- one or more Pods running successfuly once

### One-Shot jobs

Sample manifest see YAML.

```
$ k apply -f 99_KubernetesBook/07_Jobs/simple_one_shot_job.yaml
namespace/job-exploring-namespace unchanged
job.batch/calculate-pi-job created

$ k -n job-exploring-namespace get jobs
NAME               COMPLETIONS   DURATION   AGE
calculate-pi-job   1/1           45s        114s

$ k -n job-exploring-namespace describe job calculate-pi-job
Name:           calculate-pi-job
Namespace:      job-exploring-namespace
Selector:       controller-uid=13005368-88f8-4832-9442-545e5be0bfeb
Labels:         controller-uid=13005368-88f8-4832-9442-545e5be0bfeb
                job-name=calculate-pi-job
Annotations:    <none>
Parallelism:    1
Completions:    1
Start Time:     Sun, 25 Apr 2021 13:26:46 +0200
Completed At:   Sun, 25 Apr 2021 13:27:31 +0200
Duration:       45s
Pods Statuses:  0 Running / 1 Succeeded / 0 Failed
Pod Template:
  Labels:  controller-uid=13005368-88f8-4832-9442-545e5be0bfeb
           job-name=calculate-pi-job
  Containers:
   pi-calculator:
    Image:      perl
    Port:       <none>
    Host Port:  <none>
    Command:
      perl
      -Mbignum=bpi
      -wle
      print bpi(2000)
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From            Message
  ----    ------            ----   ----            -------
  Normal  SuccessfulCreate  2m35s  job-controller  Created pod: calculate-pi-job-7r4fm
  Normal  Completed         111s   job-controller  Job completed
```

* Note: No labels given in manifest as the Job object automatically picks a unique label it wants to use to identify the Pods it is responsible for
* In the sample manifest file for the simple one-shot job, the `restartPolicy` was set to `Never`, but this is misleading -- this restart policy will just tell the node's `kubelet` not to restart the failed Pod, but the Job object managing the Pod will notice the Pod errored out and spawn a new one anyway
* Therefore, it's more advisable to specify `restartPolicy=OnFailure` so the behavior of the two actors -- kubelet and Job object -- is consistent
* Use liveness probes to address cases where Pods just "hang" without crashing -- this way, if the liveness probe determines the Pod is dead, the Job object will automatically replace it

### Work queues

* Common scenario for the Job object to be used in
    * Some task populates a queue with a set of work items
    * A worker job can then fetch one or more work items from the queue
    * The entirety of worker jobs will run until the work queue is empty
* For work queue mode, the `spec.completions` parameter has to be left unset -- only specify `spec.parallelism`
* This means _n_ Pods (as defined by the `spec.parallelism` parameter) will be spawned, and they have to coordinate among themselves what piece of work weach Pod should work on
* As soon as _any_ of the Pods finishes successfully, no new Pods will be created, and no other Pod should keep working (this coordination work has to be done by the worker application inside the Pods!)
* Once at least one Pod has terminated with success and all Pods have terminated, the Job will be marked as successfully completed

## CronJob

* The _CronJob_ object in Kubernetes is very handy to address use cases where a task needs to be run at a certain interval
* Sample manifest:

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: awesome-cron-job
spec:
  # Will make the task run every fifth hour
  schedule: "0 */5 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: my-batch-job
            image: my-batch-job-image
        restartPolicy: OnFailure
```
