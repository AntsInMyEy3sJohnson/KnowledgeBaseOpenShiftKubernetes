# Container orchestration

## Demo two-tier application: WordPress on MySQL

```
$ oc create -f ~/labs/wordpress-demo/persistent-volumes.yaml
# Persistent volumes now defined and waiting to be consumed:
$ oc get pv
  NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
  pv0001    1Gi        RWO            Recycle          Available                                      53s
  pv0002    1Gi        RWO            Recycle          Available                                      53s
  pv0003    2Gi        RWO            Recycle          Available                                      53s
  pv0004    2Gi        RWO            Recycle          Available                                      53s
# Those persistent volumes can now be consumed by applications.
# Create demo application (standard two-tier web application, namely, Wordpress frontend plus MySQL backend):
$ oc create -f ~/labs/wordpress-demo/wordpress-objects.yaml
$ oc describe po wordpress-
$ oc describe po mysql-
# Step by step, OpenShift will align the current state with the desired state. 
# Example: Persistent volume claims made by pods of the applications will eventually 
# become "Bound":
$ oc get pvc
  NAME          STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
  claim-mysql   Bound     pv0003    2Gi        RWO                           3m
  claim-wp      Bound     pv0001    1Gi        RWO                           3m
# Corresponding to this, the persistent volumes available on the cluster will 
# become "bound", too:
$ oc get pv
  NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                 STORAGECLASS   REASON    AGE
  pv0001    1Gi        RWO            Recycle          Bound       default/claim-wp                               9m
  pv0002    1Gi        RWO            Recycle          Available                                                  9m
  pv0003    2Gi        RWO            Recycle          Bound       default/claim-mysql                            9m
  pv0004    2Gi        RWO            Recycle          Available                                                  9m
# Check application state:
$ oc get po --selector name=wordpress
  NAME              READY     STATUS    RESTARTS   AGE
  wordpress-45hjn   1/1       Running   1          6m
$ oc get po --selector name=mysql
  NAME          READY     STATUS    RESTARTS   AGE
  mysql-njhp5   1/1       Running   0          6m
# View application logs:
$ oc logs $(oc get pods | grep mysql- | awk '{print $1}')
$ oc logs $(oc get pods | grep wordpress- | awk '{print $1}')
# Invoke web frontend:
$ curl http://$(oc get svc | grep wpfrontend | awk '{print $3}')/wp-admin/install.php
```

## Horizontal scaling

* Tool _Apache Bench_ (_ab_) used to scale and load-test a distributed system of Apache installations
* Tool is capable of revealing how many requests per second the Apache installation is able to handle

```
# Export service IP address for using it later on:
$ export SITE="http://$(oc get svc | grep wpfrontend | awk '{print $3}')/wp-admin/install.php"
# Get baseline of following metrics before scaling out: Time taken for tests, requests per second, 
# percentage of time served within a certain time (ms)
$ ab -n10 -c 3 -k -H "Accept-Encoding: gzip, deflate" $SITE
# Scale out:
$ oc scale --replicas=5 rc/wordpress
  replicationcontroller/wordpress scaled
# Get metrics again:
$ ab -n10 -c 3 -k -H "Accept-Encoding: gzip, deflate" $SITE
```
