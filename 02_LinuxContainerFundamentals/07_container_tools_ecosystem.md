# Ecosystem Of Container Tools

## Podman

```
# Pull image:
$ podman pull ubi8
# Short-hand equivalent of:
# podman pull registry.access.redhat.com/ubi8:latest

# List locally cached images:
$ podman images

# Run container and attach to bash process within the container:
$ podman run -it ubi8 bash
$ exit

# List running containers:
# podman ps -a
```

In contrast to Docker, Podman does not run as a daemon (it's "daemon-less"), but acts as an interactive command (more like bash, for example) and can therefore -- again in contrast to Docker -- be run without root privileges. Another consequence of being "daemonless" is the fact that Podman doesn't require a client-server architecture like Docker (where there's a Docker CLI and the Docker daemon it connects to).

```
# Start two containers:
$ podman run -it ubi8 bash
$ podman run -it ubi8 sleep 60
# Examine process table:
$ pstree -Slnc
        ├─conmon─┬─{conmon}
        │        └─bash(ipc,mnt,net,pid,uts)
        ...
        └─conmon─┬─{conmon}
                 └─sleep(ipc,mnt,net,pid,uts)
# -> No Podman process!
```

There is no Podman process because containers disconnect from Podman once started. To make commands like `podman ps` work, Podman keeps track of all container meta data. 

Execution chain with Podman (assuming the command run inside the container is `sleep 10`):

`bash -> podman -> conmon -> conmon -> runc -> sleep`

Once the first `conmon` process has invoked the second, it exits, disconnecting itself from the second `conmon` process as well as all of its children. Thus, the containerized processes are independent from Podman.

```
# Kill running containers and clean up exited containers:
$ podman kill --all
$ podman rm --all
# Delete all locally cached images in one command:
$ podman rmi --all
```

## Buildah: granularity and control for building images

Similar to Podman, Buildah also doesn't run as a daemon.

Basic decisions to be made when building a container image:
1. Start from scratch or build on existing base image? Base image is most common approch, but starting from scratch can be appropriate when only working with a small set of statically linked binaries
1. Execute build from inside or outside of container? Run commands to build next container image layer inside container vs. using tools on the host (i. e., external to the container) to build next layer? Building externally is new concept with Buildah -- using existing container engines, builds are always executed internally. Building externally can be useful when smaller image is required or when image is supposed to be read-only (i. e., never built upon). Common example for building internally: Building Java things since they typically need a running JVM; common example for building externally: adding RPMs
1. Required data available internally or needs to be pulled in from outside? All files and configuration needed to build image available inside image itself vs. needing to access data on the build host. For example, it might be convenient to mount a large RPM cache inside the container during the build process, but whatever was required to perform the build should not be part of the production image (unless it's also required in there)

### Preparation

```
# "Simulate root" for Buildah:
$ buildah unshare
```

### Basic build

```
# Specify base image to start from (will create reference to "working container" -- the "scratch pad" 
# we can use to attach mounts to and run commands on etc.):
$ buildah from ubi8
  ...
  Storing signatures
  ubi8-working-container
$ buildah containers
  CONTAINER ID  BUILDER  IMAGE ID     IMAGE NAME                       CONTAINER NAME
  3361256d660c     *     4199acc83c6a registry.access.redhat.com/ub... ubi8-working-container

# Mount our "scratch pad":
$ buildah mount ubi8-working-container

# Add file to container's file system:
$ echo "Hello, Buildah world!" > $(buildah mount ubi8-working-container)/etc/hello.conf
# Check contents of directory in copy-on-write layer:
$ ls -alh $(buildah mount ubi8-working-container)/etc | grep hello
  -rw-r--r--.  1 root root   22 Mar 27 15:56 hello.conf
# Check file contents:
$ cat $(buildah mount ubi8-working-container)/etc/hello.conf
  Hello, Buildah world!
# This copy-on-write layer can now be committed as a new image layer:
$ buildah commit ubi8-working-container ubi8-hello
  Getting image source signatures
  ...
  Writing manifest to image destination
  Storing signatures
  8ae95f068bcb097424d970bff37e8352a1864ba33a0fb2cb61e1e3dc6466db0e
# New image layer is now present in local cache:
$ buildah images
  REPOSITORY                        TAG      IMAGE ID       CREATED              SIZE
  localhost/ubi8-hello              latest   8ae95f068bcb   About a minute ago   213 MB
  registry.access.redhat.com/ubi8   latest   4199acc83c6a   6 weeks ago          213 MB
$ podman images
  REPOSITORY                       TAG     IMAGE ID      CREATED             SIZE
  localhost/ubi8-hello             latest  8ae95f068bcb  About a minute ago  213 MB
  registry.access.redhat.com/ubi8  latest  4199acc83c6a  6 weeks ago         213 MB
# Clean up building environment (remove reference to "scratch pad containers", mounts etc):
$ buildah delete -a
# Of course, the previously assembled and committed image layer is still present:
$ buildah images
```

### Using tools outside the container





