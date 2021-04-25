# Kubernetes API fundamentals

## Kubernetes manifests

```
# Create new project:
$ oc new-project mysupercoolproject

# Create pod manifest:
$ cat > pod-multi-container.yaml <<EOF
> apiVersion: v1
> kind: Pod
> metadata:
>   name: my-two-container-pod
>   namespace: myproject
>   labels:
>     environment: dev
> spec:
>   containers:
>     - name: server
>       image: nginx:1.13-alpine
>       ports:
>         - containerPort: 80
>           protocol: TCP
>     - name: sidecar
>       image: alpine:latest
>       command: ["/usr/bin/tail", "-f", "/dev/null"]
>   restartPolicy: Never
> EOF
# Start pod:
$ oc create -f pod-multi-container.yaml

# Start shell session inside "server" container:
$ oc exec -it my-two-container-pod -c server -- /bin/sh
# Examine a couple of things:
$ ip address
$ netstat -ntlp
$ hostname
$ ps

# Doing the same thing within the sidecar container would reveal 
# that each container runs within its own cgroup, but the containers
# share the same IPC, network, and UTC (hostname) namespaces. This 
# is because both containers run in the same Pod.
```

## Basic operations with the Kubernetes API

```
# Examine currently available APIs and their versions:
$ oc api-versions
# Specifying high verbosity level enables us to see requests made towards
# Kubernetes API, as well as the responses it sent back:
$ oc get po --v=8

# "oc proxy" can be used to proxy local requests made against the given port 
# to the Kubernetes API:
$ oc proxy -port=8001

# In new terminal, send request to that port on localhost:
$ curl -XGET http://localhost:8001
# This call will be forwarded to the Kubernetes API, and the API will respond with a list
# of all endpoints it offers
# Request to see all API details:
$ curl localhost:8001/openapi/v2

# We can also send a request to the API asking it to list all pods in the environment:
$ curl localhost:8001/api/v1/pods | jq .items[].metadata.name
# Decrease request scope by asking API only for pods within given namespace:
$ curl localhost:8001/api/v1/namespaces/myproject/pods
# Query API for details about specific pod:
$ curl localhost:8001/api/v1/namespaces/myproject/pods/my-two-container-pod

# Export pod manifest:
$ oc get po my-two-container-pod --export -o json > podmanifest.json
# Replace version 1.13 of Alpine image used by "sidecar" container with version 1.14:
$ sed -i 's|nginx:1.13-alpine|nginx:1.14-alpine|g' podmanifest.json
# Send updated manifest back to Kubernetes API to replace old manifest and old pods that 
# were created based on the old manifest:
$ curl -XPUT localhost:8001/api/v1/namespaces/myproject/pods/my-two-container-pod \
    -H "Content-type: application/json" \
    -d @podmanifest.json

# Delete pod:
$ curl -XDELETE localhost:8001/api/v1/namespaces/myproject/pods/my-two-container-pod
# Attempt to query pod will now yield a JSON respone message informing us 
# the pod could not be found:
$ curl -XGET localhost:8001/api/v1/namespaces/myproject/pods/my-two-container-pod
  {
    "kind": "Status",
    "apiVersion": "v1",
    "metadata": {
    
    },
    "status": "Failure",
    "message": "pods \"my-two-container-pod\" not found",
    "reason": "NotFound",
    "details": {
      "name": "my-two-container-pod",
      "kind": "pods"
    },
    "code": 404
}
```

## Replica sets

```
# List all pods within current namespace:
$ oc get po -n myproject

# Create manifest file for ReplicaSet object:
$ cat > my-replica-set.yaml <<EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myfirstreplicaset
  namespace: myproject
spec:
  selector:
    matchLabels:
      app: myfirstapp
  replicas: 3
  template:
    metadata:
      labels:
        app: myfirstapp
    spec:
      containers:
        - name: nodejs
          containers: openshiftkatacoda/blog-django-py
EOF

# Apply manifest file to create ReplicaSet:
$ oc create -f my-first-replica-set.yaml

# View pods:
$ oc get po -l app=myfirstapp --show-labels -w
  NAME                      READY   STATUS    RESTARTS   AGE   LABELS
  myfirstreplicaset-4bvfd   1/1     Running   0          61s   app=myfirstapp
  myfirstreplicaset-d6dvt   1/1     Running   0          61s   app=myfirstapp
  myfirstreplicaset-g8cn7   1/1     Running   0          61s   app=myfirstapp

# The ReplicaSet will make sure the number of pods will match the number 
# specified in the manifest. In this case, 'replicas' was set to 3, so the 
# ReplicaSet object will make sure there's always exactly 3 pods running.
# One can verify this by deleting some pods and watching new ones spawn:
$ oc delete po -l app=myfirstapp

# Rescale ReplicaSet (i. e., increase or decrease number of running pods):
$ oc scale rs/myfirstreplicaset --replicas=6
$ oc scale rs/myfirstreplicaset --replicas=3

# Behind the curtains, the 'oc scale' command interacts with the '/scale' 
# endpoint on behalf of the user. Therefore, we can also use that endpoint to 
# to interact with the ReplicaSet (for the commands against 'localhost:8001' to 
# reach the Kubernetes API, the 'oc proxy' command needs to be running):
$ curl -XGET localhost:8001/apis/apps/v1/namespaces/myproject/replicasets/myfirstreplicaset/scale
  {
    "kind": "Scale",
    "apiVersion": "autoscaling/v1",
    "metadata": {
      "name": "myfirstreplicaset",
      "namespace": "myproject",
      "selfLink": "/apis/apps/v1/namespaces/myproject/replicasets/myfirstreplicaset/scale",
      "uid": "ebc7f036-5d35-43dd-b4ff-7933304c821a",
      "resourceVersion": "310991",
      "creationTimestamp": "2021-03-30T18:46:28Z"
    },
    "spec": {
      "replicas": 2
    },
    "status": {
      "replicas": 2,
      "selector": "app=myfirstapp"
    }
  }

# PUT can be used to rescale the ReplicaSet:
curl  -X PUT localhost:8001/apis/apps/v1/namespaces/myproject/replicasets/myfirstreplicaset/scale \
  -H "Content-type: application/json" -d '{"kind":"Scale","apiVersion":"autoscaling/v1","metadata":{"name":"myfirstreplicaset","namespace":"myproject"},"spec":{"replicas":5}}'

# View status of ReplicaSet:
$ curl -XGET localhost:8001/apis/apps/v1/namespaces/myproject/replicasets/myfirstreplicaset/status
```

## Custom Resource Definitions

```
# Create manifest file for Postgres Custom Resource Definition:
cat > postgres-custom-resource-definition.yaml <<EOF
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: postgres.rd.example.com
spec:
  group: rd.example.com
  names:
    kind: Postgres
    listKind: PostgresList
    singular: postgres
    plural: postgreses
    shortNames:
      - pg
  scope: Namespaced
  versions:
    - name: v1alpha1
      schema:
        openAPIV3Schema:
          type: object
          x-kubernetes-preserve-unknown-fields: true
      served: true
      storage: true
EOF

# Apply manifest file to create Custom Resource Definition (CRD) object:
$ oc create -a postgres-custom-resource-definition.yaml

# Result: The Kubernetes API will now list a new API group called 'rd.example.com':
$ curl -XGET localhost:8001/apis | jq .groups[].name | grep example
  "rd.example.com"
# (Also reflected in the 'oc api-versions' command)

# Using the Kubernetes API, we can explore the API versions within the new API group. 
# Unsurprisingly, the 'postgres' resource will be displayed.
$ curl -XGET localhost:8001/apis/rd.example.com/v1alpha1 | jq

# With the new 'postgres' resource type in place, the oc client recognizes it as such:
$ oc get postgres
$ oc get pg

# Make use of new Custom Resource Definition and create Postgres manifest file:
$ cat > my-wordpress-database.yaml <<EOF
apiVersion: "rd.example.com/v1alpha1"
kind: Postgres
metadata:
  name: wordpressdb
spec:
  user: postgres
  password: postgres
  database: mytestdb
  nodes: 5
EOF

# This manifest file can now be used to create a Postgres object:
$ oc create -f my-wordpress-database.yaml
$ oc get postgres
$ oc get pg wordpressdb -o yaml
```




