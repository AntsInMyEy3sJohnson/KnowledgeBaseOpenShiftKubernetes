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

## Singleton-based storage

* Abstractions in Kubernetes for horizontal scaling like ReplicaSets assume the Pods they manage or stateless (or nearly stateless) and, thus, easily replaceable
* Most storage solutions do not meet this assumption
* One approach to tackle this is simply not to replicate storage -- the challenge of how to manage multliple Pods that are, in fact, not stateless and thus not replaceable vanishes
* This means the database or storage solution will run only in one Pod; there is no replication
* In terms of reliability, this is roughly the equivalent of running that storage solution on one VM sonewhere outside the cluster, so it's not any less reliable than how many components are run outside the cluster
* Drawbacks:
  * No horizontal scaling means to load-balancing across service instances
  * Downtime inevitable in case of machine failure or upgrades
* Nonetheless, running storage singletons is sufficient for many smaller-scale applications that may not be mission critical

### Example: MySQL storage singleton

* Three basic objects required:
  * A PersistentVolume to manage the storage representation independently from the running application (i. e., the Pod) for it to survive the Pod being restarted or rescheduled
  * A Pod to run the MySQL application itself
  * A Service to make the MySQL Pod available for other applications in the cluster
* Recap:
  * Idea: Pods are ephemeral and can be rescheduled or restarted anytime, so coupling persistence to the lifecycle of the Pod is not a good idea if persisted state is supposed to survive the Pod being restarted or the Pod being rescheduled and appearing on another machine
  * Solution: PersistentVolumes represent storage locations whose lifecycle is decoupled from any Pod or container
* Volume for the MySQL example:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-database-persistent-volume
  # Caution: PVs are not namespaced; they are cluster-wide resources!
  labels:
    volume: some-volume
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 5Gi
  nfs:
    server: 172.15.0.1
    path: /exports
```

* The PersistentVolume is a cluster-level resource and since it represents storage that can be seen on one level with compute resources (CPU and RAM), it's usually the cluster administrator's job to provision it
* If an application developer wants to consume the storage represented by a PersistentVolume (or a part of it), then he needs to "claim" it by means of a PersistentVolumeClaim
* PersistentVolumeClaim for the MySQL example:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-database-persistent-volume-claim
  # PVCs are namespaced resources
  namespace: awesome-namespace
spec:
  selector:
    matchLabels:
      volume: some-volume
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  
```

* While it might seem like a bit too over-engineered to pull in the PersistentVolumeClaim as an additional layer of abstraction between the Pod and the PersistentVolume, the additional layer serves the purpose of decoupling the Pod (application-level) definition from the storage definition
* With the PersistentVolume and the PersistentVolumeClaim in place, the next step on our way to the reliable storage singleton is to define a ReplicaSet to manage the singleton Pod:

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: mysql-replicaset
  namespace: awesome-namespace
labels:
  app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql-container
        image: mysql
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1
            memory: 2Gi
        env:
        # ...
        livenessProbe:
          tcpSocket:
            port: 3306
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: database
          mountPath: /var/lib/mysql
      volumeMounts:
      - name: database
        persistentVolumeClaim:
          # Has to match the name supplied on the PVC
          claimName: my-database-persistent-volume-claim
```

* Last step in example is creation of a Service object to make the MySQL storage singleton consumable by other applications on the cluster:

```
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: awesome-namespace
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    protocol: TCP
```

### StorageClasses

* It's easy to imagine the number of PersistentVolumes that have to be provisioned by cluster administrators grows over time as the number of applications and components requiring persistence increases
* Thus, with a large number of persistence-requiring components, managing PersistentVolumes will turn into a time-consuming and tedious task
* Idea: Include a third persistence-related abstraction layer (in addition to PersistentVolumes and PersistentVolumeClaims, that is) that enables the Kubernetes control plane to create PersistentVolumes dynamically based on a submitted PersistentVolumeClaim
* This third abstraction layer is called a _StorageClass_
* Instead of creating PersistentVolumes, cluster opators create StorageClasses and Kubernetes will use them to dyncamically generate PersistentVolumes based on the information contained in the PersistentVolumeClaim on one hand and the StorageClass on the other
* Sample StorageClass object offered by default on a GKE cluster:

```
$ k get sc standard -o yaml
# Some fields omitted for brevity
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  # Just like PVs, StorageClasses, too, are not namespaced, but cluster-wide
  annotations: 
    storageclass.kubernetes.io/is-default-class: "true"
  labels:
    addonmanager.kubernetes.io/mode: EnsureExists
provisioner:
  kubernetes.io/gce-pd
parameters:
  type: pd-standard
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

* Sample PersistentVolumeClaim that makes use of this default StorageClass:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-persistent-volume-claim
  namespace: awesome-namespace
spec:
  # No 'spec.storageClassName' specified -- if a default 
  # storage class is available on the cluster, the
  # 'DefaultStorageClass' admission controller will 
  # automatically add 'spec.storageClassName' pointing
  # to the default storage class
  accessModes:
  - ReadWriteOnce
  resources:
    requests: 
      storage: 2Gi
```

* One pitfall to be aware of when dynamically provisioning volumes using StorageClasses: The lifespan of the PersistentVolume is determined by the `reclaimPolicy` field (which a PersistentVolume inherits from its StorageClass when dynamically provisionied)
* Default for `storageclass.reclaimPolicy` is `Delete`, which means the lifecycle of the resulting PersistentVOlume is bound to the Pod issuing the PersistentVolumeClaim to use it
* Once the Pod is gone, the Volume will be deleted as well
* If this behavior is not desired, use `Retain` instead (`Recycle` available, too, but deprecated)







  



    