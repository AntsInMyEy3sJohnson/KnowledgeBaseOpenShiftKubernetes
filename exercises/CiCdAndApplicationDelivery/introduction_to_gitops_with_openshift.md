# Introduction To GitOps With OpenShift

## Introduction to ArgoCD

* Declarative, GitOps continuous delivery tool for Kubernetes
* _GitOps_: A Git repository is the source of truth for the definition of applications
* ArgoCD is, under the hood, implemented as a Kubernetes controller running the usual "current vs. desired state" reconciliation loop -- as soon as the current state deviates from the desired state (which is defined in a Git repository), ArgoCD will align the current state towards the desired state

## Exploring the web environment

```
$ argocd --help

# Log in to ArgoCD server from CLI
$ argocd --insecure --grpc-web login \
argocd-server-argocd.2886795272-80-hazel05.environments.katacoda.com:443 \
--username admin --password <argocd-server-pod-name>

# Update ArgoCD password from CLI
$ argocd --insecure --grpc-web \
--server argocd-server-argocd.2886795272-80-hazel05.environments.katacoda.com:443 \
account update-password --current-password <argocd-server-pod-name> \
--new-password student

# List available clusters
$ argocd cluster list
  SERVER                          NAME  STATUS      MESSAGE
  https://kubernetes.default.svc        Successful  
```

## Deploying a simple app using the CLI

```
# Add Git repository containing application files
$ argocd repo add http://gogs.2886795272-80-hazel05.environments.katacoda.com/student/gitops-lab.git
$ argocd repo list

# Create project for application in ArgoCD
$ argocd app create --project default --name reverse-words-app \
 --repo http://gogs.2886795272-80-hazel05.environments.katacoda.com/student/gitops-lab.git \
 --path simple-app/reversewords_app/ \
 --dest-server https://kubernetes.default.svc \
 --dest-namespace reverse-words \
 --revision master \
 --sync-policy automated
$ argocd app list

# Retrieve application status
$ argocd app get reverse-words-app
Name:               reverse-words-app
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          reverse-words
URL:                https://argocd-server-argocd.2886795272-80-hazel05.environments.katacoda.com/applications/reverse-words-app
Repo:               http://gogs.2886795272-80-hazel05.environments.katacoda.com/student/gitops-lab.git
Target:             master
Path:               simple-app/reversewords_app/
Sync Policy:        Automated
Sync Status:        Synced to master (6c56700)
Health Status:      Healthy

GROUP  KIND        NAMESPACE      NAME           STATUS   HEALTH   HOOK  MESSAGE
       Namespace   reverse-words  reverse-words  Running  Synced         namespace/reverse-words created
       Service     reverse-words  reverse-words  Synced   Healthy        service/reverse-words created
apps   Deployment  reverse-words  reverse-words  Synced   Healthy        deployment.apps/reverse-words created
       Namespace                  reverse-words  Synced   Unknown

# Verify application is running
$ oc get namespace reverse-words
  NAME            STATUS   AGE
  reverse-words   Active   5m32s
$ oc -n reverse-words get deployment
  NAME            READY   UP-TO-DATE   AVAILABLE   AGE
  reverse-words   1/1     1            1           6m6s

$ oc -n reverse-words get svc
  NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
  reverse-words   ClusterIP   172.25.251.117   <none>        8080/TCP   6m24s

$ oc -n reverse-words expose svc reverse-words
  route.route.openshift.io/reverse-words exposed

$ curl -X POST http://$(oc -n reverse-words get route reverse-words \
  -o jsonpath='{.spec.host}') -d '{"word":"PALC"}'
  {"reverse_word":"CLAP"}

# Check state recovery
# Delete deployment so live state doesn't match desired state anymore
$ oc -n reverse-words delete deployment reverse-words
$ argocd app list
# --> Will report the app is out-of-sync

# An attempt to sync the app will re-sync everything from the Git repository (and
# thus re-create the missing deployment), but it will also report one error
$ argocd app syns reverse-words-app
# Output will report that the previously created route is out-of-sync -- which is
# correct, since there is no accompanying manifest in the application's Git repository
```

