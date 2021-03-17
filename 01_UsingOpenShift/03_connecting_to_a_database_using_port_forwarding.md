# Connecting to a database using port forwarding

## Creating an initial project

```
$ oc login -u developer -p developer
$ oc new-project my-project
```

## Deploying a PostgreSQL database

```
$ oc new-app postgresql-ephemeral --name database \
  --param DATABASE_SERVICE_NAME=database \
  --param POSTGRESQL_DATABASE=sampledb \
  --param POSTGRESQL_USER=username \
  --param POSTGRESQL_PASSWORD=password
$ oc rollout status dc/database
```

## Starting an interactive shell

```
$ oc get pods --selector name=database
$ POD=`oc get pods --selector name=database -o custom-columns=NAME:.metadata.name --no-headers`; echo $POD
```

## Creating a remote connection

```
# 'oc port-forward'
# General form: oc port-forward <pod-name> <local-port>:<remote-port>
$ oc port-forward $POD 15432:5432 &
$ psql sampledb username --host=127.0.0.1 --port=15432
$ \q
$ jobs
$ kill %1
```