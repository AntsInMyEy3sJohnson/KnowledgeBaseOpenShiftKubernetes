# Deploying applications from images

## Deleting the application

```
$ oc get all -o name
$ oc describe route/blog-django-py
$ oc get all --selector app=blog-django-py
$ oc delete all --selector app=blog-django-py
```

## Deploying an application using the comman dline

```
# To verify that a given image is valid:
# Form: oc new-app --search <image>
$ oc new-app --search openshiftkatacoda/blog-django-py
$ oc new-app openshiftkatacoda/blog-django-py
$ oc expose service/blog-django-py
$ oc get route/blog-django-py
$ oc get route/blog-django-py -o custom-columns=HOST:.spec.host
```