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

### Making use of tools outside the container

```
# Create new container, mount image, and get copy-on-write layer:
$ WORKING_MOUNT=$(buildah mount $(buildah from scratch))
# Wait, that was fantastic! We started, quite literally, "from scratch"... below, 
# a couple of very basic tools will be added (only bash and coreutils), resulting
# in a very minimal image. The same approach can be used to create optimallzy sized images for 
# specific use cases.
$ echo $WORKING_MOUNT
# Verify directory is empty:
$ ls -alh $WORKING_MOUNT
# Installation of basic tools:
$ yum install --installroot $WORKING_MOUNT bash coreutils \ 
    --releasever 8 --setopt install_weak_deps=false -y
$ yum clean all -y --installroot $WORKING_MOUNT --releasever 8
# Root directory of copy-on-write layer should now contain 
# a lot of files and directories:
$ ls -alh $WORKING_DIRECTORY
# Commit copy-on-write layer as new image layer:
$ buildah commit working-container minimal
# Run the "minimal" image and verify bash is there:
$ podman run -it minimal bash
  bash-4.4# echo "Hello, minimal image!"
  Hello, minimal image!
# Perform clean-up:
$ exit
$ buildah delete -a
```

### Pulling in external data during the build

```
# Create a dummy file (representing some kind of large dependency that needs to 
# be pulled in during image build time):
$ dd if=/dev/zero of=~/data/test.bin bs=1MB count=100
  100+0 records in
  100+0 records out
  100000000 bytes (100 MB, 95 MiB) copied, 0.0451312 s, 2.2 GB/s
$ ls -alh ~/data/test.bin 
  -rw-r--r--. 1 root root 96M Mar 28 17:12 /home/rhel/data/test.bin
# Create working container:
$ buildah from ubi8
$ buildah mount ubi8-working-container
# Simulate consuming data inside the container:
$ buildah run -v ~/data:/data:Z ubi8-working-container \
  dd if=/data/test.bin of=/etc/small-test.bin bs=100 count=2
# (In the above command, the ":Z" option is used to relabel the data for 
# SELinux, and the "dd" command simply represents consunimg the dummy file)
# Commit copy-on-write layer and perform clean-up:
$ buildah commit ubi8-working-container ubi8-data
$ buildah delete -a
```

## Skopeo: moving and sharing images

Skopeo: Very handy comamnd-line utility able to perform various actions on container images and remote registries

### Inspect images remotely

Remember: Caching an image locally means extracting it to a rootfs, which has always been a root operation. Therefore, one might want to remotely inspect the image before making the local container engine pull it down... and this is where Skopeo comes to the rescue!

```
# Remotey inspect an image
$ skopeo inspect docker://registry.fedoraproject.org/fedora
# This reveals some handy meta data about the given image: Besides architecture and OS 
# information, the command also shows us the image labels consumed by most container 
# engines to pass them to the container runtime for them to be constructed as 
# environment variables.
# Comparison: See meta data on running container using Podman
$ podman run --name meta-data-container -id registry.fedoraproject.org/fedora bash
$ podman inspect meta-data-container
```

### Pulling images

```
# Copy image to local container storage:
$ skopeo copy docker://registry.fedoraproject.org/fedora containers-storage:fedora
# Copy and extract image into local directory:
$ skopeo copy docker://registry.fedoraproject.org/fedora dir:$HOME/fedora
# Advantage: Not mapped into container storage, so image contents can be 
# easily examined 
```

### Moving images between local container storages

```
# Copy image from Podman to Docker container storage:
$ skopeo copy \
  containers-storage:registry.fedoraproject.org/fedora \
  docker-daemon:registry.fedoraproject.org/fedora:latest
```

### Moving images between container registries

```
$ skopeo copy \
  docker://registry.fedoraproject.org/fedora \
  docker://quay.io/fatherlinux/fedora \
  --dest-creds fatherlinux+fedora:5R4YX2LHHVB682OX232TMFSBGFT350IV70SBLDKU46LAFIY6HEGN4OYGJ2SCD4HI
```









