# Ingress

* Use case: Route traffic from and to applications running on the cluser
* Explored so far: Service object with `type: NodePort` and `type: LoadBalancer` 
* NodePorts and LoadBalancers don't solve the problem well enough:
    * NodePort: Application outside cluster (_calling_ application) needs to know a specific port the application on the cluster to be invoked listens on
    * LoadBalancer: Leads to allocation of scarce and often expensive resources for each service
* Popular approach in more traditional application platform environments: virtual hosting
    * Many HTTP addresses are hosted on the same IP address
    * Implemented by means of load balancer or reverse proxy
    * Load balancer or reverse proxy examines incoming traffic and forwards ("proxies") the call to some other program
* HTTP load balancing system in Kubernetes referred to as _Ingress_ and functions in the same way
* Drawback of virtual hosting: operator has to manage load balancer configuration file (which, in a dynamic environment, can become very complex as virtual hosts come and go)
* Kubernetes standardizes the load balancer configuration file, puts it into a standard Kubernetes API object that can be acted upon like the others, and combines multiple ingress objects into a single file for the load balancer to consume. Thus, the task of managing the load balancer configuration file is vastly simplified
* Usually, Kubernetes' ingress system works like the following:
    * Ingress controller reachable from outside cluster because it's exposed using `type: LoadBalancer`
    * Ingress controller examines incoming requests and forwards them to upstream "servers" based on some criteria
    * Configuration for how requests are forwarded and to whom is result of ingress controller reading and monitoring ingress objects

## Ingress controllers and ingress specifications

* Important difference between ingress object and all other Kubernetes API objects: Split into controller implementation and a resource specification that the controller makes use of
* Kubernetes itself does not provide an off-the-shelf controller implementation
    * Ingress objects can be acted upon (created, altered) like any other type of object in the Kubernetes API, but...
    * ... out-of-the-box, there is no piece of logic to act on those objects
    * Instead, users have to install and manage an outside controller themselves
* The ingress controller being pluggable is due to the fact that there is no single HTTP load balancer "to rule them all" (address all requirements of all clients in all scenarios), so users must be able to choose one that suits their needs best

## Contour and Envoy

* Contour: ingress controller designed to work with Envoy
* Envoy: open-source load balancer built to be dynamically configured using an API
* Contour translates Kubernetes Ingress objects so Envoy can understand and work with them

### Installation

To install Contour, the following command can be used:

```
$ kubectl apply -f https://j.hept.io/contour-deployment-rbac
```

The above does the following:

```
namespace/projectcontour created
serviceaccount/contour created
serviceaccount/envoy created
configmap/contour created
customresourcedefinition.apiextensions.k8s.io/extensionservices.projectcontour.io created
customresourcedefinition.apiextensions.k8s.io/httpproxies.projectcontour.io created
customresourcedefinition.apiextensions.k8s.io/tlscertificatedelegations.projectcontour.io created
serviceaccount/contour-certgen created
rolebinding.rbac.authorization.k8s.io/contour created
role.rbac.authorization.k8s.io/contour-certgen created
job.batch/contour-certgen-v1.14.1 created
clusterrolebinding.rbac.authorization.k8s.io/contour created
clusterrole.rbac.authorization.k8s.io/contour created
service/contour created
service/envoy created
deployment.apps/contour created
daemonset.apps/envoy created
```

```
$ k -n projectcontour get service envoy -o wide
```

Ingress controller now configured -- create some DNS entries for external IP, then launch upstream ("backend") services:

```
$ k run backend1 \
    --image=antsinmyey3sjohnson/hello-container-service:1.0 \
    --replicas=1 \
    --port=8081
# Create an accompanying service object
$ k expose deployment backend1ba

$ k run backend2 \
    --image=antsinmyey3sjohnson/hello-container-service:1.0 \
    --replicas=1 \
    --port=8081
$ k expose deployment backend2

$ k get svc -o wide

# Caution: The Ingress below will only work with port types 'NodePort' or 'LoadBalancer', 
# 'ClusterIP' is not meant to be accessed directly via Ingress on GCloud:
# https://stackoverflow.com/questions/51572249/why-does-google-cloud-show-an-error-when-using-clusterip

# --> Edit services to have a named port of type 'NodePort'. Example after editing:
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2021-04-17T07:07:16Z"
  labels:
    run: backend1
  name: backend1
  namespace: default
  resourceVersion: "78919881"
  selfLink: /api/v1/namespaces/default/services/backend1
  uid: de9e99d2-ec90-4d98-9574-82ca2b8d0a46
spec:
  clusterIP: 10.35.248.66
  externalTrafficPolicy: Cluster
  ports:
  - name: backend1-http
    nodePort: 32358
    port: 8081
    protocol: TCP
    targetPort: 8081
  selector:
    run: backend1
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

### Simple usage

No request examination to determine which upstream service to route the request to -- just blindly forward request to one particular service:

```
$ cat <<EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-simple-ingress
  labels:
      name: my-simple-ingress
spec:
  backend:
    serviceName: backend1
    servicePort: 8081

$ k get ingress 
$ k describe my-simple-ingress
# ^ This will contain an error message on GCP if the port type 
# of the specified service is 'ClusterIP'
```

### Using hostnames

```
$ cat <<EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-host-ingress
  labels:
      name: my-host-ingress
spec:
  rules:
  - host: alpaca.example.org
    http:
      paths:
      - backend:
          serviceName: backend1
          servicePort: 8081
```

### Using paths

```
$ cat <<EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-path-ingress
  labels:
    name: my-path-ingress
spec:
  rules:
  - host: bandicoot.example.org
    http:
      paths:
      - path: "/"
        backend: 
          serviceName: backend2
          servicePort: 8081
      - path: "/a/"
        backend:
          serviceName: backend1
          servicePort: 8081
```

* Pattern shown above -- using multiple paths on the same domain -- can be used to direct traffic to different upstream services on different paths of that domain
* Caution: A request containing, for example, `/a`, will show up to the upstream server precisely like this, so the application this requests is directed to based on the Ingress rule needs to be able to serve a request on this particular subpath








