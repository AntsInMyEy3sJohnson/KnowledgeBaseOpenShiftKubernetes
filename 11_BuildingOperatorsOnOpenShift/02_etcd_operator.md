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
```
