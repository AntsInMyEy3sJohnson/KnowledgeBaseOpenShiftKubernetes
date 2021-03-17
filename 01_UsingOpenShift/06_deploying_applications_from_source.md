# Deploying applications from source

## Deploying on the command line

```
$ oc new-app python:3.6-ubi8~https://github.com/openshift-katacoda/blog-django-py
# 'buildconfig'
$ oc logs bc/blog-django-py --follow
$ oc expose service/blog-django-py
```

## Triggering a new build

```
# Use case: Source has changed, changes need to be reflected 
# in application running on cluster
$ oc start-build blog-django-py
$ oc get builds --watch
$ oc describe bc/blog-django-py
# Use "binary build" to deploy from local source in order to iterate more quickly 
$ git clone https://gibthub.com/openshift-katacoda/blog-django-py
$ cd blog-django-py && echo 'BLOG_BANNER_COLOR=blue' >> .s2i/environment
$ oc start-build blog-django-py --from-dir=. --wait
# Switch to building from remote code repository again
$ oc start-build blog-django-py
$ oc cancel-build blog-django-py-<build number>
```
