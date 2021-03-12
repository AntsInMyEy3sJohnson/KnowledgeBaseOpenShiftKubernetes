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

## Transferring files in and out of containers

### Creating an initial project
```
$ oc login -u developer -p developer
$ oc new-project my-project
# Only works if an image stream, an image, or a template with the 
# given name exists
$ oc new-app my-new-app 
$ kubectl create deployment hello-node \
  --image=gcr.io/hello-minikube-zero-install/hello-node
$ oc new-app openshiftkatacoda/blog-django-py --name blog
$ oc expose svc/blog
$ oc rollout status deployment blog # $ oc rollout status dc/blog
$ oc get pods --selector deployment=blog
# To extract name of running pod:
$ oc get pods --selector deployment=blog \
  -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'
# To establish remote shell into pod (and potentially run some commands):
$ oc rsh <pod> <commands to run in pod>
```

### Downloading files from a container

```
# To copy files from container inside pod to local machine:
# Form: $ oc rsync <pod>:/dir/in/pod ./local/dir
# Use '--exclude' and '--include' to specify patterns for exluding or including 
# files and folders
# Use '--container' to specify container within pod if multiple containers run inside pod
$ oc rsync $POD:/opt/app-root/src/db.sqlite3 .
# Can also be used to copy entire folders:
$ oc rsync $POD:/opt/app-root/src/media .

```
### Uploading files to a container

```
# Form: oc rsync ./local/dir <pod>:/remote/dir
# Caution: No form to upload only single file, use '--exclude' and '--include' instead
# Example: Upload 'robots.txt' file
$ cat > robots.txt << !
> User-agent: *
> Disallow: /
> !
# To upload file:
$ oc rsync . $POD:/opt/app-root/src/htdocs \
    --exclude=* \
    --include=robots.txt \
    --no-perms
```

### Synchronizing files with a container

```
$ git clone <...>
# To make 'rsync' watch for file changes, use '--watch' 
# (works both local->remote and vice-versa)
$ oc rsync blog-django-py/. $POD:/opt/app-root/src \
    --no-perms \
    --watch &
$ jobs
$ oc rsh $POD kill -HUP 1
$ oc set env deployment blog MOD_WSGI_RELOAD_ON_CHANGES=1
$ kill -9 %1
```

### Copying files to a persistent volume

```
# (Not actually supposed to be accessed, just used to keep pod alive)
$ oc run dummy --image centos/httpd-24-centos7
$ oc get all --selector run=dummy
$ oc set volume dc/dummy --add \
  --name=tmp-mount \
  --claim-name=data \
  --type pvc \
  --claim-size=1G \
  --mount-path /mnt
$ oc get pvc
$ oc rsync ./ $POD:/mnt --strategy=tar
$ oc rsh $POD ls -las /mnt
# To merely mount existing persistent volume without creating new one:
$ oc set volume dc/dummy --add \
  --name=tmp-mount \
  --claim-name=data \
  --mnt-path /mnt
$ oc delete all --selector run=dummy
oc get all --selector run=dummy -o name
```

