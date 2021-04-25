# Container Images

## Image layers and repositories: inspecting base images, layers, and image history

```
# Examine history of full ubi7 base image:
$ podman history registry.access.redhat.com/ubi7/ubi:latest
# Considerably smaller, minimal base image:
$ podman history registry.access.redhat.com/ubi7/ubi-minimal:latest
# Build new, multi-layered image using Dockerfile:
$ podman build -t ubi7-change -f ~/labs/lab2-step1/Dockerfile
$ podman images
# Viewing image's history is useful to explore how image was built:
$ podman history ubi7-change
```

A container repository is really just a collection of layers -- it is only for simplicity and convenience that we tend to refer to a layer as a _container image_.

## Image URLs: mapping business requirements to the URL, namespace, repository, and tag

```
# Inspect repository:
$ podman inspect ubi7/ubi
# Same result with full URL and tag of the repository on the registry server:
$ podman inspect registry.access.redhat.com/ubi7/ubi:latest
# Build new image with tag other than "latest":
$ podman build -t ubi7:test -f ~/labs/lab2-step1/Dockerfile
$ podman images
# The following won't work!
$ podman inspect ubi7
# This will:
$ podman inspect ubi7:test
# Generally: Always use full URI providing server, namespace, repository, and tag
```

## Image internals: inspecting libraries, interpreters, and operating system components whithin a container image

```
# Command "ldd" resolves libraries the given binary is linked against
$ podman run -it \
    registry.access.redhat.com/jboss-eap-7/eap70-openshift \
    ldd -v -r /usr/lib/jvm/java-1.8.0-openjdk/jre/bin/java
$ podman run -it \
    registry.access.redhat.com/ubi7/ubi \
    ldd /usr/bin/python
# RHEL tools container -- very convenient! Bundles many tools useful for debugging and troubleshooting in containerized environments
$ podman run -it \
    registry.access.redhat.com/rhel7/rhel-tools bash
$ ldd /usr/bin/curl
$ exit
# Whenever there is a security vulnerability within any of the libraries any of the binaries 
# inside an image is linked against, the image needs to be rebuilt to update the libraries in question
```
