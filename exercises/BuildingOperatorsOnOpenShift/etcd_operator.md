# Creating An Etcd Operator

## Writing the Custom Resource Definition

```
# Create new project:
$ oc new-project myproject

# Create manifest file for Custom Resource Definition (CRD):
$ cat > etcd-operator-custom-resource-definition.yaml <<EOF
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: etcdclusters.etcd.database.coreos.com
spec:
  group: etcd.database.coreos.com
  names:
    kind: EtcdCluster
    listKind: EtcdClusterList
    singular: etcdcluster
    plural: etcdclusters
    shortNames:
      - etcdcl
  scope: Namespaced
  versions:
    - name: v1beta2
      schema: 
        openAPIV3Schema:
          type: object
          x-kubernetes-preserve-unknown-fields: true
      served: true
      storage: true
EOF
$ oc create -f etcd-operator-custom-resource-definition.yaml
$ oc get crd etcdclusters.etcd.database.coreos.com
```

## ServiceAccount, Role, and RoleBinding

```
# We need a dedicated Service Account for running the Etcd operator:
$ cat > etcd-operator-service-account.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: etcd-operator-service-account
EOF
$ oc create -f etcd-operator-service-account.yaml
$ oc get sa

# The 'etcd-operator-service-account' will be assigned a dedicated role 
# for authorization to perform actions against the Kubernetes API:
$ cat > etcd-operator-role.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: 
  name: etcd-operator-role
rules:
- apiGroups:
  - etcd.database.coreos.com
  resources:
  - etcdclusters
  - etcdbackups
  - etcdrestores
  verbs:
  - '*'
- apiGroups:
  - ""
  resources:
  - pods
  - services
  - endpoints
  - persistentvolumeclaims
  - events
  verbs:
  - '*'
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - '*'
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
EOF
$ oc create -f etcd-operator-role.yaml
$ oc get roles

# The new Role now needs to be assigned to the previously created Service Account. 
# This is achieved by means of a RoleBinding:
$ cat > etcd-operator-role-binding.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: etcd-operator-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: etcd-operator-role
subjects:
- kind: ServiceAccount
  name: etcd-operator-service-account
  namespace: myproject
EOF
$ oc create -f etcd-operator-role-binding.yaml
$ oc get rolebindings
```

## Etcd Operator Deployment

```
# Using the following manifest file, we can create the Deployment object 
# containing the container image for the Etcd Operator:
$ cat > etcd-operator-deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    name: etcdoperator
  name: etcd-operator
spec:
  replicas: 1
  selector:
    matchLabels:
      name: etcd-operator
  template:
    metadata:
      labels:
        name: etcd-operator
    spec:
      containers:
      - command:
        - etcd-operator
        - --create-crd=false
        env:
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        image: quay.io/coreos/etcd-operator@sha256:c0301e4686c3ed4206e370b42de5a3bd2229b9fb4906cf85f3f30650424abec2
        imagePullPolicy: IfNotPresent
        name: etcd-operator
      serviceAccountName: etcd-operator-service-account
EOF
$ oc create -f etcd-operator-deployment.yaml 
  deployment.apps/etcd-operator created
$ oc get deploy
  NAME            READY   UP-TO-DATE   AVAILABLE   AGE
  etcd-operator   1/1     1            1           9s
$ oc get po
  NAME                             READY   STATUS    RESTARTS   AGE
  etcd-operator-76d5868c78-2fmcv   1/1     Running   0          14s

# Start process to follow operator logs:
$ export ETCD_OPERATOR_POD=$(oc get po -l name=etcd-operator \
    -o jsonpath='{.items[0].metadata.name}')
$ oc logs $ETCD_OPERATOR_POD -f
$ oc get endpoints etcd-operator -o yaml
```

## Etcd Cluster Custom Resource

Etcd operator now rolled out, now make use of previously (in the Custom Resource Definition) defined Custom Resource (`EtcdCluster`) to create an Etcd cluster:

```
$ oc project myproject
$ cat > etcd-operator-custom-resource.yaml <<EOF
apiVersion: etcd.database.coreos.com/v1beta2
kind: EtcdCluster
metadata:
  name: example-etcd-cluster
spec: 
  size: 3
  version: 3.1.10
EOF
$ oc create -f etcd-operator-custom-resource.yaml

# Remember: We've set this up as a custom resource, so the Kubernetes API knows about this 
# resource type. This means we can query it like any other resource type.
$ oc get etcdcluster
# Watch pods get created:
oc get po -l etcd_cluster=example-etcd-cluster -w

# Verify cluster has been exposed via a ClusterIP service:
$ oc get services -l etcd_cluster=example-etcd-cluster
  NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
  example-etcd-cluster          ClusterIP   None            <none>        2379/TCP,2380/TCP   2m30s
  example-etcd-cluster-client   ClusterIP   172.25.95.243   <none>        2379/TCP            2m31s
```

## Interacting with a live Etcd cluster

Let's try to interact with the Etcd cluster from within a Pod that has the Etcd client installed.

```
$ oc run myetcdclient --image=busybox busybox --restart=Never -- /usr/bin/tail -f /dev/null
$ oc get po -w
$ oc rsh myetcdclient
# Install Etcd client
$ wget https://github.com/coreos/etcd/releases/download/v3.1.4/etcd-v3.1.4-linux-amd64.tar.gz
$ tar -xvf etcd-v3.1.4-linux-amd64.tar.gz
$ cp etcd-v3.1.4-linux-amd64/etcdctl .

# Set environment variables for Etcd version and cluster endpoint:
$ export ETCDCTL_API=3
$ export ETCDCTL_ENDPOINTS=example-etcd-cluster-client:2379

# Write key-value pair to Etcd cluster:
$ ./etcdctl put operator sdk
$ ./etcdctl get operator

# Leave client Pod
$ exit
```

## Making use of the operator: scaling the cluster, updating Etcd version

The Etcd operator installed previously is able to detected many events describing changes made to the `EtcdCluster` Custom Resource. Among those events are scaling the cluster and modifying the Etcd version used.

```
# Scale cluster up:
$ oc patch etcdcluster example-etcd-cluster --type='json' -p '[{"op": "replace", "path": "/spec/size", "value":5}]'
  etcdcluster.etcd.database.coreos.com/example-etcd-cluster patched
# Here, the Etcd operator will detect the 'spec.size' field change in the Etcd cluster custom resource 
# and modify the number of Pods accordingly.

# Change version of Etcd cluser:
$ oc patch etcdcluster example-etcd-cluster --type='json' -p \
  '[{"op": "replace", "path": "/spec/version", "value":3.2.13}]'
  etcdcluster.etcd.database.coreos.com/example-etcd-cluster patched
# The Etcd operator will detect the modification made to the 'spec.size' field and 
# create a new cluster with the newly defined image.
```

## Clean-up work

```
$ oc delete etcdcluster example-etcd-cluster
$ oc delete deployment etcd-operator
$ oc delete crd ectdclusters.etcd.database.coreos.com
```


