# Multi-Cluster GitOps With OpenShift

## Exploring the web environment

```
$ argocd cluster add pre
$ argocd cluster add pro
$ argocd cluster list
  SERVER                          NAME  STATUS      MESSAGE
  https://172.17.0.13:8443        pro   Successful
  https://172.17.0.22:8443        pre   Successful
  https://kubernetes.default.svc        Successful
```


## Deploying a simple app to multiple clusters using the ArgoCD CLI

```
# Tell ArgoCD about the Git repository
$ argocd repo add http://gogs.2886795282-80-kota01.environments.katacoda.com/student/gitops-lab.git
$ argocd repo list

# Define application on 'pre' cluster initialized in ArgoCD earlier on
$ argocd app create --project default --name pre-reverse-words-app \
 --repo http://gogs.2886795282-80-kota01.environments.katacoda.com/student/gitops-lab.git \
 --path reversewords_app/base \
 --dest-server $(argocd cluster list | grep pre | awk '{print $1}') \
 --dest-namespace reverse-words \
 --revision pre \
 --sync-policy automated

# Define application on 'pro' cluster
$ argocd app create --project default --name pro-reverse-words-app \
 --repo http://gogs.2886795282-80-kota01.environments.katacoda.com/student/gitops-lab.git \
 --path reversewords_app/base \
 --dest-server $(argocd cluster list | grep pro | awk '{print $1}') \
 --dest-namespace reverse-words \
 --revision pro \
 --sync-policy automated

# Query application status
$ argocd app get pre-reverse-words-app
$ argocd app get pro-reverse-words-app

# Make sure application components have been created on OpenShift clusters
$ oc --context pre get ns reverse-words
$ oc --context pre -n reverse-words get deployment
$ oc --context pre -n reverse-words get svc

$ oc --context pro get ns reverse-words
$ oc --context pro -n reverse-words get deployment
$ oc --context pro -n reverse-words get svc

# Expose route
$ oc --context pre -n reverse-words expose svc reverse-words --name=reverse-words-pre
$ oc --context pro -n reverse-words expose svc reverse-words --name=reverse-words-pro

# Query applications
$ curl -X POST http://$(oc --context pre -n reverse-words get route reverse-words-pre \ 
-o jsonpath='{.spec.host}') -d '{"word":"PALC"}'
$ curl -X POST http://$(oc --context pro -n reverse-words get route reverse-words-pro \ 
-o jsonpath='{.spec.host}') -d '{"word":"PALC"}'
```
