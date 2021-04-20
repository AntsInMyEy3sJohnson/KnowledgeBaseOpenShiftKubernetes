# Getting Started With OpenShift Pipelines

## Installing the Pipelines Operator

```
# Create subscription object to subscribe namespace to the OpenShift Pipelines Operator
$ cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator
  namespace: openshift-operators
spec:
  # Channel: Name of the channel we want to subscribe to
  channel: ocp-4.6
  # Name of the operator
  name: openshift-pipelines-operator-rh
  # CatalogSource providing the Operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Check and verify installation
until oc api-resources --api-group=tekton.dev | grep tekton.dev &> /dev/null
do
  echo "Operator installation still in progress..."
  sleep 5
done

echo "Operator ready to rumble!"
```

## Create sample Task

* A Task defines a series of steps that run in a defined order
* Each step -- thus, each Task -- completes a certain set of build work
* Every Task runs in its own Pod on the cluster, and each Task Pod contains the step it represents as a running container

```
# Example: simple Task that prints "Hello, world"
$ cat <<EOF | oc apply -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello-world-task
spec:
  steps:
  - name: greet-the-world
    image: registry.access.redhat.com/rhel7/rhel-tools
    command:
    - /bin/bash
    args: ['-c', 'echo Hello, World!']
EOF

# Use Tekton CLI to start the task
$ tkn task start --showlog hello-world-task
  TaskRun started: hello-world-task-run-njl7v
  Waiting for logs to be available...
  [greet-the-world] Hello, World!
```

## Task Resource definitions

* Tasks can be parameterized by passing flags into it
* Those so-called `params` are an essential tool to make Tasks more generic and, thus, more reusable across pipelines

```
# Task that uses a parameter to apply some manifest files stored in a directory
$ cat <<EOF | oc apply -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: apply-manifests
spec:
  workspaces:
  - name: source
  params:
    - name: manifest_dir
      description: The directory in source that contains yaml manifests
      type: string
      default: "k8s"
  steps:
  - name: apply
    image: quay.io/openshift/origin-cli:latest
    workingDir: /workspace/source
    command: ["/bin/bash", "-c"]
    args:
    - |-
      echo "Applying manifests in ${inputs.params.manifest_dir} directory"
      oc apply -f $(inputs.params.manifest_dir)
      echo "-------------------------------------------------------------"
EOF

# Task to update a deployment:
$ cat <<EOF | oc apply -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: update-deployment
spec:
  params:
    - name: deployment
      description: The name of the deployment patch the image
      type: string
    - name: IMAGE
      description: Location of image to be patched with
      type: string
  steps:
    - name: patch
      image: quay.io/openshift/origin-cli:latest
      command: ["/bin/bash", "-c"]
      args:
        - |-
          oc patch deployment $(inputs.params.deployment) --patch='{"spec":{"template":{"spec":{
            "containers":[{
              "name": "$(inputs.params.deployment)",
              "image":"$(inputs.params.IMAGE)"
            }]
EOF

# Create Persistent Volume Claim to provide filesystem for pipeline execution:
$ cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: source-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 512Mi
EOF

# View created tasks:
tkn task ls
  NAME                DESCRIPTION   AGE
  apply-manifests                   5 minutes ago
  hello-world-task                  22 minutes ago
  update-deployment                 1 minute ago
```

## Pipeline creation

* Pipeline: Construct that defines an ordered series of Tasks
* Tasks should be designed so they perform exactly one simple, atomic step, with the goal of being able to reuse them across Pipelines (or even within a single pipeline)

```
# Sample pipeline
apiVersion: tekton.dev/v1beta1
metadata:
  name: my-awesome-build-and-deploy-pipeline
spec:
  workspaces:
  - name: shared-workspace
  params:
  - name: deployment-name
    type: string
    description: "Name of the deployment to be patched"
  - name: git-url
    type: string
    description: "URL of the Git repo containing the deployment code"
  - name: git-revision
    type: string
    description: "Revision to be fetched from the Git repository"
    default: "master"
  - name: IMAGE
    type: string
    description: "Image to be built from the code contained in the Git repository"
  tasks:
  - name: fetch-repository
    taskRef:
      name: git-clone
      kind: ClusterTask
    workspaces:
    - name: output
      workspace: shared-workspace
    params:
    - name: url
      value: $(params.git-url)
    - name: subdirectory
      value: ""
    - name: deleteExisting
      value: "true"
    - name: revision
      value: $(params.git-revision)
  - name: build-image
    taskRef:
      name: buildah
      kind: ClusterTask
    params:
    - name: TLSVERIFY
      value: "false"
    - name: IMAGE
      value: $(params.IMAGE)
    workspaces:
    - name: source
      workspace: shared-workspace
    runAfter:
    - fetch-repository
  - name: apply-manifests
    taskRef:
      name: apply-manifests
    workspaces:
    - name: source:
      workspace: shared-workspace
    runAfter:
    - build-image
  - name: update-deployment
    taskRef:
      name: update-deployment
    workspaces:
      name: source
      workspace: shared-workspace
    params:
    - name: deployment
      value: $(params.deployment-name)
    - name: IMAGE
      value: $(params.IMAGE)
    runAfter:
    - apply-manifests
```

## Trigger pipeline

* Pipeline trigger by using Tekton CLI to create a new `PipelineRun`
* All parameters declared within the pipeline definition can be passed to the `tkn` call

```
# Pipeline invocation to build backend application
$ tkn pipeline start my-awesome-build-and-deploy-pipeline \ 
  -w name=shared-workspace,claimName=source-pvc \
  -p deployment-name=vote-api \
  -p git-url=https://github.com/openshift/pipelines-vote-api.git \
  -p IMAGE=image-registry.openshift-image-registry.svc:5000/pipelines-tutorial/vote-api \
  --showlog

# Pipeline invocatoin to build frontend application
$ tkn pipeline start my-awesome-build-and-deploy-pipeline \
  -w name=shared-workspace,claimName=source-pvc \
  -p deployment-name=vote-ui \
  -p git-url=https://github.com/openshift-pipelines/vote-ui.git \
  -p IMAGE=image-registry.openshift-image-registry.svc:5000/pipelines-tutorial/vote-ui \
  --showlog
```

* Pipeline invocation will create the Pods corresponding to the individual Tasks defined in the Pipeline
* List of Pipelines (both running and inactive) can be viewed using `tkn pipeline ls`
* List of PipelineRuns can be viewed using `tkn pipelinerun ls`



