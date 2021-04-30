# Storage Integrations

* Nearly all applications beyond the scope of small PoCs will sooner or later have to persist state somewhere
* In an environment like Kubernetes that encourages declarative state descriptions of decoupled, immutable components this poses a special challenge since most storage solutions cannot be purely configured in a declarative way, but require some imperative commands to be run
* There are a variety of approaches to tackle the challenge of how to enable state persistence for components running on Kubernetes:
    * `StatefulSets`
    * Integration of existing storage solutions
    * Running state-encapsulating singletons

## Integrating existing storage solutions

* Scenario: Existing storage solution (like a database on some server) running outside the cluster
* Challenge: Make this server accessible (i. e., consumuable) for applications running on Kubernetes
* Solution: Use a regular `Service` object, but since there are no Pods reachable for that service on the cluster itself, supply it with the `externalName` attribute rather than a label selector that would usually identify a set of Pods
* Example:

```
apiVersion: v1
kind: Service
metatdata:
  name: some-database
  namespace: some-namespace
spec: 
  type: ExternalName
  externalName: legacy-database.company.internal
```

* The approach outlined here, of course, works for any kind of cluster-external component that is resolvable by name, so not just for databases
* With the external component being integrated like this, applications on the Kubernetes cluster can consume it by referring to the service name (here: `some-database.some-namespace`) as usual
* For this to work, the name supplied for `externalName` needs to be resolvable via DNS
* In case the external component to be integrated is only offered via a IP address, the integration process becomes a little more complex -- the `Service` object has to created first without any `spec` so no endpoints are resolved, and then the corresponding `Endpoints` object has to be added manually
* Example:

```
# Service
apiVersion: v1
kind: Service
metadata:
  name: a-nameless-database
  namespace: some-namespace

### Endpoints
apiVersion: v1
kind: Endpoints
metadata:
  # This name must match with the Service's name!
  name: a-nameless-database
  namespace: some-namespace
subsets:
# This is an array, so it can hold multiple IP addresses in case 
# the service to be reached runs on multiple addresses for redundancy
- addresses:
  - ip: 10.37.8.1
  ports:
  - port: 6007
```

* While the integration concepts shown above work, they have one important caveat: no health checking
* Thus, it is up to the cluster administrator to make sure the services hosted outside Kubernetes are actually up and running


    