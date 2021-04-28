# RBAC On Kubernetes

* Mechanism for restricting access to and actions on Kubernetes APIs
* Goals: 
    * Security: Make sure only appropriate (by means of their roles) users have access to a set of APIs in the cluster
    * Reduce "blast radius" of actions executed accidentally
* RBAC is not the only means to harden a cluster towards malicious actions, but it is an important tool nonetheless

## Introduction to authentication and authorization in general

### Authentication
* Provides identity of caller issuing request
* Every request made to the Kubernetes control plane needs to pass an authentication step -- even if that step determines the request can proceed without authentication
* No identity store integrated in Kubernetes itself -- instead focus on making integrations with third-party providers as straightforward as possible
* If authentication fails, the client will receive an HTTP 401 error

### Authorization
* Determines whether a given user is allowed to perform the requested action
* Thus, authorization=user identity + requested resource + requested action (HTTP verb)
* Requests can only proceed to the authorization phase once successfully authenticated
* Client will receive HTTP 403 if not authorized to perform the requested action on the resource contained in the request

## Role-Based Access Control

### Kubernetes and the notion of identity

* All requests are linked to some identity -- even unauthenticated ones are associated to the `system:unauthenticated` group
* Two different kinds of identities in Kubernetes: user identities vs. Service Account identities
* Service Accounts are created and managed by Kubernetes itself and are usually linked to cluster-internal components
* User accounts, on the other hand, represent either actual, human users of the cluster or cluster-external services

### Roles and Role Bindings

* Kubernetes employs combination of Roles and Role Bindings to determine whether a given request is authorized for the previously established identity
* _Role_: Encapsulates set of abstract capabilities
* _RoleBinding_: Assignment of a Role to one or more identities (represented by users or Service Accounts)
* Example: 
    * Role `awesomedeploymentrole` allows users to deploy stuff
    * Assigning that Role to the user identity _Dave_ by means of a Role Binding grants the right to deploy things to that user
* Depending on the scope of the role and its bindings, there are two different types of objects in Kubernetes to represent them:
    * Namespaced roles and role bindings: `Role` + `RoleBinding`
        * This type of role encapsulation represents capabilities within one single namespace (meaning the resources mentioned in the Role are namespcaed resources)
        * Therefore, a Role cannot be used to encapsulate the capability of accessing, in any way, a cluster-wide resource, such as a CustomResourceDefinition
        * Associating a Role and a RoleBinding can only serve to provide authorization within the namespace containing both the Role and the RoleBinding
        * Example manifest for Role: see YAML
    * Equilavent of Role-RoleBinding combination, but on larger scope: `ClusterRole` + `ClusterRoleBinding`

### Role verbs

* A Role is a set of abstract capabilities
* Each capability is defined in terms of a resource plus a verb (or a list of verbs) that define the action (or actions) that can be performed on the given resource or resources
* Roughly, those verbs correspond to HTTP verbs, but they are more numerous and with subtler differences

### Built-in roles

* The built-in identities internal to Kubernetes that it needs to perform its functions need defined sets of capabilities, too
* Thus, Kubernetes comes with a lot of pre-defined cluster roles that can be viewed, used as templates, or directly used
* In fact, there are four pre-defined ClusterRole objects that are designed for generic end users:
    * `cluster-admin`: Grants all rights across the entire cluster
    * `admin`: Like its cluster-wide counterpart, but scoped to a single namespace
    * `edit`: Enables its holders to modify resources within a single namespace
    * `view`: Read-only access to a namespace
* To view all ClusterRoles: `k get clusterrole`
* To view all ClusterRoleBindings: `k get clusterrolebinding`
* __Caution__: 
    * The Kubernetes API server installs a couple of ClusterRoles upon starting
    * Among them is `system:unauthenticated`, which allows unauthenticated users read-only access to the API server's API discoery endpoint
    * This is bad (mkay) in any cluster exposed to the Internet, thus the API server should be prevented to install this ClusterRole in such cases via the `--anonymous-auth=false` flag

## Managing RBAC

* Unfortunately, managing RBAC for a Kubernetes cluster can be a frustrating and cumbersome act
* Even more unfortunately -- and more concerning -- is the fact that misconfigured RBAC can lead to serious security vulnerabilites on the affected cluster
* Thus, to make the task of managing RBAC less of a burden and more of a joy, one can make use of a couple of helpers

### _can-i_ to test authorization

* As its name suggests, the tool checks if a given user can perform a specific action
* Thus, the tool is quite helpful to validate RBAC configurations in scope of initial cluster configuration
* Simple usage example:
    * Capability defined by resource plus verb
    * So, simplest usage to verify if a user is granted a certain capability includes a resource and a verb
    * `k auth can-i create persistentvolumeclaim`
* _can-i_ is also able to verify whether a certain action can be performed on a subresource of a resource: `k auth can-i get deployments --subresource=events`

### RBAC and source control

* The fact that RBAC Kubernetes resources -- just like all other Kubernetes resources -- can be described in simple, text-based formats like YAML or JSON means they can be easily subjected to version control
* In all scenarios with a need for good audit, accountability and rollback capabilities, RBAB resources in particular should be kept in version control
* The `kubectl apply` command cannot be, ahem, _applied_ to RBAC-related resources, though -- instead, `reconcile` is used
* `reconcile` will reconcile a text-based representation of an RBAC resource, like a ClusterRole or a ClusterRoleBinding, with the current state of the cluster
* `reconcile` is a sub-command of `kubectl auth`, so: `k auth reconcile -f my-rbac-resource.yaml`
* The `--dry-run` flag works on `reconcile`, too -- use it to view the contents that would have been sent to the API server 

## Going beyond the basics

* Managing RBAC with a small number or roles and identities is one thing -- doing it at scale is another
* Kubernetes provides some additional, advanced capabilities that support cluster operators with managing RBAC at scale

### ClusterRole aggregation

* Use case: Create a ClusterRole as combination of other ClusterRoles
* Simple method: Create a new ClusterRole and put in it all the rules from the desired other ClusterRoles
* Not good -- not only tedious, but also prone to errors since changes in the "source" roles don't get automatically reflected in the newly created role
* So-called _aggregation rules_ can be used to combine the capabilities -- rules -- of a set of ClusterRoles into one new ClusterRole
* Aggregation rules work by pulling together all desired constituent ClusterRoles using label selectors
* The built-in Kubernetes `edit` ClusterRole is a great example for how to make good use of aggregration rules and ClusterRoles: The latter are used to define very fine-grained capabilities, and the former then aggregate them into higher-level capability sets

```
# Excerpt from the `edit` ClusterRole's manifest
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
name: edit
aggregationRule:
clusterRoleSelector:
- matchLabels:
    # The 'edit' role thus aggregates all ClusterRole carrying 
    # this particular label with the given value, and...
    rbac.authorization.kubernetes.io/aggregate-to-edit: "true"
labels:
    kubernetes.io/bootstrapping: rbac-defaults
    # ... is itself a constituent of the 'admin' ClusterRole
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
# ...
```

### Using groups for bindings

* In environments with many identities, _groups_ can be pulled in as an additional layer of abstraction between those identities and the Role they are supposed to be assigned to
* This means the Role, via a RoleBinding, will be assigned to a group, and every identity who is a member of that group then gains the set of capabilites defined in the Role
* The concept of a group being linked to the capabilites this group is allowed to have also maps better to most organizations -- their smallest atomic unit that can be given some capabilites is usually the team, not the individual identity
* To bind a ClusterRole to a group, the `Group` kind is used in the `subject` field:
  ```
  # ...
  subjects:
    # Caution: No '/v1' here!
  - apiGroup: rbac.authorization.k8s.io
    kind: Group
    name: awesome-group
  # ...
  ```
* In Kubernetes, such groups are supplied by authentication providers





