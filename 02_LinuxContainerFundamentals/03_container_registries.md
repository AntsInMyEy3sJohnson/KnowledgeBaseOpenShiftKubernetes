# Container Registries

## The basics of trust: quality and provenance

* Quality: "You must download a trusted thing"
* Provenance: "You must download the thing from a trusted source"

### The trusted thing 

```
# Remotely inspect image before pulling it:
$ skopeo inspect docker://registry.fedoraproject.org/fedora
# This is not sufficient! To find out if the "thing" is trustworthy, we first have to establish 
# whether the source is.
```

### The trusted source

```
$ curl -I https://registry.fedoraproject.org
$ curl 2>&1 -kvv https://registry.fedoraproject.org | grep subject
```

In a real-world scenario, it's the container engine's job to check those certificates. Thus, system administrators have to take care of distributing the appropriate CA certificates into production.

## Evaluating the trust of images and registry servers

```
$ podman run -it docker.io/centos:7.0.1406 yum updateinfo
# -> No information on package updates -> Difficult to map CVEs to RPM packages -> Difficult to update individual packages affected by CVE

$ podman run -it registry.fedoraproject.org/fedora dnf updateinfo
# -> Decent metadata about pacakge updates, but no CVE mapping

$ podman run -it docker.io/ubuntu/trusty-20170330 \
    /bin/bash -c "apt-get update && apt list --upgradable"
# -> Similar information level to Fedora, so again no CVE mappings

$ podman run -it registry.access.redhat.com/ubi7/ubi:7.6-73 yum updateinfo security
# -> With an active subsciption, this would yield CVE mappings.
```

Even without an active RedHat subscription, the image's health rating is always available in the RedHat Container Image Catalog. Whenever possible, use the [RedHat Container Image Catalog](https://access.redhat.com/containers/).

Stuff on my Kubernetes cluster:

```
$ k create ns rhel-tools-test
$ k -n rhel-tools-test run rhel-tools -i -t \
    --image=registry.access.redhat.com/rhel7/rhel-tools:latest
$ k -n rhel-tools-test get po
$ k -n rhel-tools-test attach rhel-tools -c rhel-tools -i -t
$ k delete ns rhel-tools-test
```

## Analyzing storage and graph drivers

__Caution__: Whenever a container image is pulled, each layer is cached locally and thus mapped into a shared filesystem (the _root_ file system, typically _overlay2_ or _devicemapper_). This has two important implications:
1. Due to historical reasons, caching a container image locally is a _root_ operation.
1. All sensitive data committed to an image layer is accessible to everyone on the system. This means if, say, a password is committed to a layer, everyone on the system will be able to read it.

```
# Docker storage:
$ docker info 2>&1 | grep -E 'Storage | Root'
  Storage Driver: devicemapper
  Docker Root Dir: /var/lib/docker
$ tree /var/lib/docker

# Podman storage:
$ podman info | grep -A3 Graph
  GraphDriverName: overlay
  GraphOptions: {}
  GraphRoot: /var/lib/containers/storage
  GraphStatus:
    Backing Filesystem: extfs
    Native Overlay Diff: "true"
    Supports d_type: "true"

$ podman pull registry.access.redhat.com/ubi7/ubi
$ find /var/lib/containers/storage | grep redhat-release | tail -n 1
```

