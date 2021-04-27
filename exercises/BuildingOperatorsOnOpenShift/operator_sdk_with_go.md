# Exploring The Operator SDK With Golang

## Creation of a new project

```
$ oc new-project myproject

# Create directory for Operator project to live in:
$ mkdir -p $HOME/projects/podset-operator
$ cd $HOME/projects/podset-operator

# Initialize new Go-based Operator SDK project:
$ operator-sdk init --domain=example.com --repo=github.com/redhat/podset-operator
```

## Creating API and controller

```
# The following will create a new Custom Resource Definition API 
# called 'PodSet' having a version of 'app.example.com/v1alpha1' and kind 'PodSet', 
# as well as some boilerplate controller logic and a couple of 
# 'Kustomize' configuration files.
$ operator-sdk create api --group=app --version=v1alpha1 --kind=PodSet --resource --controller
# --> New directories created for us: /api, /config, /controllers
```

## Defining spec and status

Kubernetes works by continuously and constantly comparing desired state (_spec_) with the actual cluster state (_status_). This is why almost every functional object (with some exceptions, _ConfigMaps_ being one) has a `spec` and a `status` field.


```
// 'spec' and 'status' attributeds in the 'api/v1alpha1/podset_types.go' file (unmodified)

// ... more stuff above
type PodSet struct {
        metav1.TypeMeta   `json:",inline"`
        metav1.ObjectMeta `json:"metadata,omitempty"`

        Spec   PodSetSpec   `json:"spec,omitempty"`
        Status PodSetStatus `json:"status,omitempty"`
}
// ... more stuff below

```

Both `PodSetSpec` and `PodSetStatus` are also defined as their own type structs. We can modify them so each contains a description of the desired fields that indicate the desired state (_spec_) and status of any `PodSet` resource (caution: keep the `+kubebuilder` strings!):

```
// PodSetSpec defines the desired state of PodSet
type PodSetSpec struct {
        // Replicas is the desired number of pods for the PodSet
        // +kubebuilder:validation:Minimum=1
        // +kubebuilder:validation:Maximum=10
        Replicas int32 `json:"replicas,omitempty"`
}

// PodSetStatus defines the current status of PodSet
type PodSetStatus struct {
        PodNames          []string `json:"podNames"`
        AvailableReplicas int32    `json:"availableReplicas"`
}
```

Whenever a modification was made to a `*_types.go` file, run `make generate` for the `zz_generated.deepcopy.go` file.

```
# With the updated 'zz_generated.deepcopy.go' file in place, we can 
# use 'make manifests' to generate customized CRD as well as additional object YAML files.
$ make manifests

# Verify CRD yaml file has been updated:
$ cat config/crd/bases/app.example.com_podsets.yaml
  ---
  apiVersion: apiextensions.k8s.io/v1beta1
  kind: CustomResourceDefinition
  metadata:
    annotations:
      controller-gen.kubebuilder.io/version: v0.3.0
    creationTimestamp: null
    name: podsets.app.example.com
  spec:
    additionalPrinterColumns:
    - JSONPath: .spec.replicas
      name: Desired
      type: string
    - JSONPath: .status.availableReplicas
      name: Available
      type: string
    group: app.example.com
    names:
      kind: PodSet
      listKind: PodSetList
      plural: podsets
      singular: podset
    scope: Namespaced
    # ...

# PodSet CRD can now be rolled out to OpenShift cluster:
$ oc apply -f config/crd/bases/app.example.com_podsets.yaml
$ oc get crd podsets.app.example.com -o yaml
```

## Customizing operator funcationality

* Default controller logic in `controllers/podset_controller.go`
* Goal: Provide all knowledge to perform reconcile loop whenever objects of `kind: PodSet` are added, updated, or deleted PLUS whenever those actions are performed on the Pods owned by a PodSet
* Update `SetupWithManager` and `Reconcile` operator methods with desired functionality

```
package controllers

import (
        "context"
        "reflect"

        "github.com/go-logr/logr"
        "k8s.io/apimachinery/pkg/runtime"
        ctrl "sigs.k8s.io/controller-runtime"
        "sigs.k8s.io/controller-runtime/pkg/client"

        appv1alpha1 "github.com/redhat/podset-operator/api/v1alpha1"

        "k8s.io/apimachinery/pkg/labels"
        "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
        "sigs.k8s.io/controller-runtime/pkg/reconcile"

        "k8s.io/apimachinery/pkg/api/errors"
        metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

        corev1 "k8s.io/api/core/v1"
)

// PodSetReconciler reconciles a PodSet object
type PodSetReconciler struct {
        client.Client
        Log    logr.Logger
        Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=app.example.com,resources=podsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=app.example.com,resources=podsets/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=v1,resources=pods,verbs=get;list;watch;create;update;patch;delete

func (r *PodSetReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
        _ = context.Background()
        _ = r.Log.WithValues("podset", req.NamespacedName)

        // Fetch the PodSet instance
        instance := &appv1alpha1.PodSet{}
        err := r.Get(context.TODO(), req.NamespacedName, instance)
        if err != nil {
                if errors.IsNotFound(err) {
                        // Request object not found, could have been deleted after reconcile request.
                        // Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
                        // Return and don't requeue
                        return reconcile.Result{}, nil
                }
                // Error reading the object - requeue the request.
                return reconcile.Result{}, err
        }
        // List all pods owned by this PodSet instance
        podSet := instance
        podList := &corev1.PodList{}
        lbs := map[string]string{
                "app":     podSet.Name,
                "version": "v0.1",
        }
        labelSelector := labels.SelectorFromSet(lbs)
        listOps := &client.ListOptions{Namespace: podSet.Namespace, LabelSelector: labelSelector}
        if err = r.List(context.TODO(), podList, listOps); err != nil {
                return reconcile.Result{}, err
        }

        // Count the pods that are pending or running as available
        var available []corev1.Pod
        for _, pod := range podList.Items {
                if pod.ObjectMeta.DeletionTimestamp != nil {
                        continue
                }
                if pod.Status.Phase == corev1.PodRunning || pod.Status.Phase == corev1.PodPending {
                        available = append(available, pod)
                }
        }
        numAvailable := int32(len(available))
        availableNames := []string{}
        for _, pod := range available {
                availableNames = append(availableNames, pod.ObjectMeta.Name)
        }

        // Update the status if necessary
        status := appv1alpha1.PodSetStatus{
                PodNames: availableNames,
                AvailableReplicas: numAvailable,
        }
        if !reflect.DeepEqual(podSet.Status, status) {
                podSet.Status = status
                err = r.Status().Update(context.TODO(), podSet)
                if err != nil {
                        r.Log.Error(err, "Failed to update PodSet status")
                        return reconcile.Result{}, err
                }
        }

        if numAvailable > podSet.Spec.Replicas {
                r.Log.Info("Scaling down pods", "Currently available", numAvailable, "Required replicas", podSet.Spec.Replicas)
                diff := numAvailable - podSet.Spec.Replicas
                dpods := available[:diff]
                for _, dpod := range dpods {
                        err = r.Delete(context.TODO(), &dpod)
                        if err != nil {
                                r.Log.Error(err, "Failed to delete pod", "pod.name", dpod.Name)
                                return reconcile.Result{}, err
                        }
                }
                return reconcile.Result{Requeue: true}, nil
        }

        if numAvailable < podSet.Spec.Replicas {
                r.Log.Info("Scaling up pods", "Currently available", numAvailable, "Required replicas", podSet.Spec.Replicas)
                // Define a new Pod object
                pod := newPodForCR(podSet)
                // Set PodSet instance as the owner and controller
                if err := controllerutil.SetControllerReference(podSet, pod, r.Scheme); err != nil {
                        return reconcile.Result{}, err
                }
                err = r.Create(context.TODO(), pod)
                if err != nil {
                        r.Log.Error(err, "Failed to create pod", "pod.name", pod.Name)
                        return reconcile.Result{}, err
                }
                return reconcile.Result{Requeue: true}, nil
        }

        return reconcile.Result{}, nil
}

// newPodForCR returns a busybox pod with the same name/namespace as the cr
func newPodForCR(cr *appv1alpha1.PodSet) *corev1.Pod {
        labels := map[string]string{
                "app":     cr.Name,
                "version": "v0.1",
        }
        return &corev1.Pod{
                ObjectMeta: metav1.ObjectMeta{
                        GenerateName: cr.Name + "-pod",
                        Namespace:    cr.Namespace,
                        Labels:       labels,
                },
                Spec: corev1.PodSpec{
                        Containers: []corev1.Container{
                                {
                                        Name:    "busybox",
                                        Image:   "busybox",
                                        Command: []string{"sleep", "3600"},
                                },
                        },
                },
        }
}

//SetupWithManager defines how the controller will watch for resources
func (r *PodSetReconciler) SetupWithManager(mgr ctrl.Manager) error {
        return ctrl.NewControllerManagedBy(mgr).
                For(&appv1alpha1.PodSet{}).
                Owns(&corev1.Pod{}).
                Complete(r)
}
```

## Running operator locally outside the cluster

Two ways of running operator after having registered the CRD:
* As Pod inside Kubernetes cluster (for productive scenarios)
* As Go program outside the cluster using the Operator SDK (for developing and testing)

```
# Run operator:
WATCH_NAMESPACE=myproject make run
```

## Creating PodSet custom resource

```
$ cd $HOME/projects/podset-operator
# Sample file for us to customize:
$ cat config/samples/app_v1alpha1_podset.yaml
  apiVersion: app.example.com/v1alpha1
  kind: PodSet
  metadata:
    name: podset-sample
  spec:
    # Add fields here
    foo: bar

# Update and apply file:
cat > config/samples/app_v1alpha1_podset.yaml <<EOF
apiVersion: app.example.com/v1alpha1
kind: PodSet
metadata:
  name: podset-sample
spec:
  replicas: 3
EOF
$ oc create -f config/samples/app_v1alpha1_podset.yaml
$ oc get podset
  NAME            DESIRED   AVAILABLE
  podset-sample   3         3

# Verify desired number of pods is up and running:
$ oc get po
  NAME                     READY   STATUS    RESTARTS   AGE
  podset-sample-podfxmnh   1/1     Running   0          81s
  podset-sample-podmdh5h   1/1     Running   0          81s
  podset-sample-podtrmrh   1/1     Running   0          81s

# Check that status shows names of currently owned pods:
$ oc get podset podset-sample -o custom-columns=PODS:.status.podNames

# Update desired number of replicas in PodSet:
$ oc patch podset podset-sample --type='json' \
    -p '[{"op": "replace", "path": "/spec/replicas", "value":5}]'
$ oc get po
```

## Clean-up: deleting the PodSet Custom Resource

```
$ oc get pods -o yaml | grep ownerReferences -A10
$ oc delete podset podset-sample
$ oc get po
```





