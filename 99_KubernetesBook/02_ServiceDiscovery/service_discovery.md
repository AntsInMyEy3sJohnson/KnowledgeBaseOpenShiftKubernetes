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
* Caution: A service object with is cluster IP doesn't yet expose pods to outside of the cluster -- it only makes them reachable from within the cluster

```
$ kubectl run my-deployment --image=some/image --port=8080 -- labels="some=labels"
$ kubectl expose deployment my-deployment
$ kubectl get svc -o wide
```


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

## External access

### NodePort

* If the `spec.type` field of a service object is set to `NodePort`, Kubernetes will choose a random port (or use a specific one if a port has been defined by the user) to open on all of the cluster nodes
* All nodes then direct traffic sent to that port to the pods identified by the service object's label selector
* A NodePort works in addition to a cluster IP, so defining a NodePort won't overwrite or replace the cluster IP
* The NodePort can be set when creating the service object from a YML definition by means of the `spec.type` or when creating it on the fly using `kubectl expose` with `--type NodePort` specified

### LoadBalancer

* Extension of NodePort: Additionally configures the environment the cluster is running in (some kind of cloud, in most cases) to create a load balancer pointed at the cluser nodes and the given NodePort
* To ask the cloud environment for a publicly accessible load balancer to reach the pods identified by the service object's label selectors, set the `spec.type` field to `LoadBalancer`
* In case the cloud environment uses DNS-based load balancers, the load balancer will be accessible via a public name, otherwise, a public IP address will be assigned

## Advanced stuff

### Endpoints

* Use case: Call service without cluster IP
* Kubernetes creates an accompanying object for every service object -- the _endpoint_ object
* The endpoints object encapsulates all IP addresses for that service
* An application may therefore also query the Kubernetes API directly to look up endpoints and call them

```
$ kubectl get endpoints my-deployment --watch
$ kubectl describe endpoints my-deployment
```

* Endpoint objects great tool for new applications built to run on Kubernetes

### Manual service discovery

* Service objects are essentially named label selectors -- their labels identify a set of pods, and it is their task to load-balance traffic to those pods
* Thus, one could easly query the Kubernetes API oneself to query the pods in question and then implement one's own service discovery

```
$ kubectl -n my-namespace get pod -o wide --show-labels
$ kubectl -n my-namespace get pod -o wide --selector key1=value1,key=value2
```

* Challenge: Keep in sync the labels to be used -- service object helps solve this problem

### kube-proxy

* Functionality like cluster IPs is performed by _kube-proxy_, a Kubernetes component running on every node in cluster
* For example:
    * Kube-proxy employs the Kubernetes API server to listen for new services in cluster
    * If a new service emerges, the kube-proxy will rewrite the set of _iptables_ on that node host so they point at one of the endpoints for that service
    * Iptables will be rewritten whenever set of endpoints for service changes (because of pods appearing and disappearing, for example)