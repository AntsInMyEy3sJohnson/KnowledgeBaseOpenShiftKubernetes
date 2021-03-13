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

## Connecting to a database using port forwarding

### Creating an initial project

```
$ oc login -u developer -p developer
$ oc new-project my-project
```

### Deploying a PostgreSQL database

```
$ oc new-app postgresql-ephemeral --name database \
  --param DATABASE_SERVICE_NAME=database \
  --param POSTGRESQL_DATABASE=sampledb \
  --param POSTGRESQL_USER=username \
  --param POSTGRESQL_PASSWORD=password
$ oc rollout status dc/database
```

### Starting an interactive shell

```
$ oc get pods --selector name=database
$ POD=`oc get pods --selector name=database -o custom-columns=NAME:.metadata.name --no-headers`; echo $POD
```

### Creating a remote connection

```
# 'oc port-forward'
# General form: oc port-forward <pod-name> <local-port>:<remote-port>
$ oc port-forward $POD 15432:5432 &
$ psql sampledb username --host=127.0.0.1 --port=15432
$ \q
$ jobs
$ kill %1
```

## Using the CLI to manage resource objects

### Deploying an application

```
$ oc login -u developer -p developer
$ oc new-project my-project
$ oc new-app openshiftroadshow/parksmap-katacoda:1.2.0 --name parksmap
$ oc rollout status deployment/parksmap
$ oc svc expose svc/parksmap
```
### Resource object types

```
$ oc get all
$ oc get all -o name
$ oc get routes
# To get all the different types of resource objects:
$ oc api-resources
# Many resource types abbreviations ("configmap" --> "cm")
$ oc get cm
```

### Querying resource objects

```
# 'describe' will return descriptions in human-readable way, to retrieve output easier
# to parse for machines, use 'get -o json' or 'get -o yaml'
$ oc describe routes/parksmap
$ oc get routes/parksmap -o yaml
# To see description of purpose of certain field, provide the 'explain' command with 
# path specifier for resource in question
$ oc explain route.spec.host
```

### Editing resource objects

```
$ oc edit routes/parksmap -json
```

### Creating resource objects

```
$ oc create -f parksmap-fqdn.json
$ oc get routes
$ oc create route edge parksmap-fqdn \
  --service parksmap
  --insecure-policy Allow \
  --hostname www.example.com
```

### Replacing resource objects

```
$ oc replace -f parksmap-fqdn.json
$ oc path routes/parksmap.json \
  --patch '{"spec": {"tls": {"insecureEdgeTerminationPolicy": "Allow"}}}'
```

### Labeling resource objects

```
# Assign label
$ oc label service/parksmap web=true
# Use assigned label
$ oc get all -o name --selector web=true
# Remove label
$ oc label service/parksmap web-
```

### Deleting resource objects

```
$ oc delete route/parksmap-fqdn
$ oc delete route --selector app=parksmap
$ oc delete svc,route --selector app=parksmap
$ oc delete all --selector app=parksmap
# Danger zone! This will delete everything -- hence explicit confirmation 
# using '--all' required
$ oc delete all --all
```

