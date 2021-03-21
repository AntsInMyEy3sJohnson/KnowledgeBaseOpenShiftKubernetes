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
    * Ingress objects can be actud upon (created, altered) like any other type of object in the Kubernetes API, but...
    * ... out-of-the-box, there is no piece of logic to act on those objects
    * Instead, users have to install and manage an outside controller themselves
* The ingress controller being pluggable is due to the fact that no single HTTP load balancer exists that fits all requirements of all clients in all scenarios, so users must be able to choose one that suits their needs best