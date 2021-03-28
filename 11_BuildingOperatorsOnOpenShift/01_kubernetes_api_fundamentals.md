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

