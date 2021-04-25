# Deploying Applications From Source

## Creating an initial project

```
$ oc login -u developer -p developer
$ oc new-project myproject
```

## Deploying using the command line

```
$ oc new-app python:latest~https://github.com/openshift-katacoda/blog-django-py
$ oc status
$ oc logs buildconfig blog-django-py --follow
$ oc logs bc blog-django-py --follow
$ oc expose service blog-django-py
$ oc get route blog-django-py
```

## Triggering a new build

```
$ oc start-build blog-django-py
$ oc get builds --watch
$ oc describe bc blog-django-py
$ oc get builds
```

