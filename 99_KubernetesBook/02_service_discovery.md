# Service Discovery

* Kubernetes not only supports, but was in fact built to encourage high levels of dynamism
* High level of dynamism offered by Kubernetes not supported by traditional network infrastructure. Example: DNS
    * DNS as great solution for the internet with services being relatively static, but ...
    * ... not so much in a Kubernetes environment where services may appear and disappear very quickly
* Service discovery: Find out which processes are listening at which addresses for which services

## Service Object

* The starting point for service discovery in Kubernetes is the _service object_
* In essence, a service object is a way to create a named label selector
* API command to create a service: `kubectl expose` (where `kubectl run` was an easy and quick means to create a deployment)
* The `kubectl expose` command will read label selectors as well as all relevant ports from the deployment object (if it's a deployment the service object was created for)
* A service object is assigned a cluster IP (virtual IP)
* Kubernetes will load-balance requests made to that IP across all pods identified by the service object's label selectors
* The cluster IP can be given a DNS address because it is stable. Thus, clients can simply refer to the DNS entry pointing to the cluster IP, and which pod they talk to is handled for them by Kubernetes
* Within the same namespace, the service name can be used to call the pods identified by the service object's label selectors

### DNS service
* The DNS service enabling this kind of behavior is enabled by a service itself being managed by Kubernetes
* DNS service provides DNS names for cluster IPs

Example of a full DNS name for a service:
```
my-application.demo-namespace.svc.cluster.local
```
In here:
* _some-application_: The service's name
* _demo-namespace_: Namespace the service is located in
* _svc_: Indicates this DNS name refers to a service object (enables support for other objects to be put behind cluster IPs in the future)
* _cluster.local_: Base domain name for the cluster (default that will be set in most clusters, altough it may be overridden by cluster administrator)

### Readiness checks

* Service object uses readiness checks to find out which pods are ready to handle requests
* Readiness checks can, for example, be defined on the deployment object under `spec.template.spec.containers[some-container].readinessprobe`

Example for a simple readiness probe:

```
...
    readinessProbe:
      httpGet:
        path: /some-readiness-endpoint-on-application
        port: 8443
    periodSeconds: 2
    initialDelaySeconds: 10
    failureThreshold: 5
    successThreshold: 1
```

* Pods create by deployment definition set up like the above will be checked for readiness by means of an HTTP GET request on `/some-readiness-endpoint-on-application` using port 8443
* Checks will be executed every two seconds until one check succeeds, and the first check will be conducted ten seconds after the pod came up
* Neat: Service object will load-balance only to pods whose readiness check succeeded
* This way, the service object along with a readiness check can also be used to implement graceful shutdowns


