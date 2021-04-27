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
    * Cluster-scoped roles and role bindings: `ClusterRole` + `ClusterRoleBinding`
