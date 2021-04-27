# Introduction to containers

## Container images

```
$ podman pull quay.io/fedora/fedora:34-x86_64
$ podman run -t quay.io/fedora/fedora:34-x86_64 cat /etc/redhat-release
$ podman run -t registry.access.redhat.com/ubi7/ubi cat /etc/redhat-release
```

## Container registries

```
$ podman pull quay.io/fatherlinux/linux-container-fundamentals-2-0-introduction
$ podman run -d -p 3306:3306 quay.io/fatherlinux/linux-container-fundamentals-2-0-introduction
$ curl localhost:3306
```

Sample build file:

```
FROM registry.access.redhat.com/ubi7/ubi-minimal

RUN microdnf -y install nmap-ncat && \
    echo "Hello, container world!" > /srv/hello-container-world.txt

ENTRYPOINT bash -c 'while true; do /usr/bin/nc -l -p 3306 < /srv/hello-container-world.txt; done'
```

## Container hosts

Keep in mind: Container s are simply regular Linux processes that were forked as child processes of a container runtime like _runc_.
```
$ podman ps -ls
$ podman top -l %C etime tty time
```

## Container orchestration

```
$ curl https://raw.githubusercontent.com/fatherlinux/two-pizza-team/master/two-pizza-team-ubi.yaml
# Will create two services, two replication controllers, and a route
$ oc create -f https://raw.githubusercontent.com/fatherlinux/two-pizza-team/master/two-pizza-team-ubi.yaml
$ for i in {1..5}; do oc get po; sleep 3; done
$ oc get svc/pepperoni-pizza -o yaml | grep ip | awk '{print $3}'
$ curl $(oc get svc/pepperoni-pizza -o yaml | grep ip | awk '{print $3}')
$ curl $(oc get svc/cheese-pizza -o yaml | grep clusterIP | awk '{print $2}'):3306
```

