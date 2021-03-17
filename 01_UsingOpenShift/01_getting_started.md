# Using OpenShift

## Logging in to an OpenShift cluster

### Logging in to an OpenShift cluster via the command line

```
$ oc login
$ oc whoami
$ oc whoami --show-server
$ oc get projects 
```

### Collaborating with other users
```
$ oc login --username user1 --password user1
$ oc new-project mysecrets
$ oc adm policy add-role-to-user view developer -n mysecrets
```

### Switching between accounts
```
$ oc login --username developer
$ oc whoami
$ oc whoami --show-context
$ oc config get-contexts
```
