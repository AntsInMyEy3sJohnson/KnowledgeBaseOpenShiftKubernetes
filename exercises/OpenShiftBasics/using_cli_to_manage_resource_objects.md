# Using the CLI to manage resource objects

## Deploying an application

```
$ oc login -u developer -p developer
$ oc new-project my-project
$ oc new-app openshiftroadshow/parksmap-katacoda:1.2.0 --name parksmap
$ oc rollout status deployment/parksmap
$ oc svc expose svc/parksmap
```

## Resource object types

```
$ oc get all
$ oc get all -o name
$ oc get routes
# To get all the different types of resource objects:
$ oc api-resources
# Many resource types abbreviations ("configmap" --> "cm")
$ oc get cm
```

## Querying resource objects

```
# 'describe' will return descriptions in human-readable way, to retrieve output easier
# to parse for machines, use 'get -o json' or 'get -o yaml'
$ oc describe routes/parksmap
$ oc get routes/parksmap -o yaml
# To see description of purpose of certain field, provide the 'explain' command with 
# path specifier for resource in question
$ oc explain route.spec.host
```

## Editing resource objects

```
$ oc edit routes/parksmap -json
```

## Creating resource objects

```
$ oc create -f parksmap-fqdn.json
$ oc get routes
$ oc create route edge parksmap-fqdn \
  --service parksmap
  --insecure-policy Allow \
  --hostname www.example.com
```

## Replacing resource objects

```
$ oc replace -f parksmap-fqdn.json
$ oc path routes/parksmap.json \
  --patch '{"spec": {"tls": {"insecureEdgeTerminationPolicy": "Allow"}}}'
```

## Labeling resource objects

```
# Assign label
$ oc label service/parksmap web=true
# Use assigned label
$ oc get all -o name --selector web=true
# Remove label
$ oc label service/parksmap web-
```

## Deleting resource objects

```
$ oc delete route/parksmap-fqdn
$ oc delete route --selector app=parksmap
$ oc delete svc,route --selector app=parksmap
$ oc delete all --selector app=parksmap
# Danger zone! This will delete everything -- hence explicit confirmation 
# using '--all' required
$ oc delete all --all
```