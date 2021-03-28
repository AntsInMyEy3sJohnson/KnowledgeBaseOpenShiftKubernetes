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

## Checkpointing and restoring with CRIU

Podman can use CRIU to checkpoint and restore containers on the same host (comes in handy for containers that have a long start-up time or possess a cache of some sort that needs to be populated ("cache warming"), etc.)

Lifecycle of a container from start to finish using Podman:

1. `podman pull` -- Pull container image
1. `podman create` -- Make Podman create tracking meta data to `/var/lib/containers` or `.local/share/containers`
1. `podman mount` -- Create copy-on-write layer and mount container image with read/write layer above it
1. `podman init` -- Make Podman create `config.json` file
1. `podman start` -- Run workload by handing `config.json` file and rootfs to container runtime (typically `runc`)
1. Workload runs either as daemon or as batch process
1. `podman kill` -- Kills process(-es) within container
1. `podman rm` -- Copy-on-write layer gets unmounted and deleted
1. `podman rmi` -- Remove image from `/var/lib/containers` or `/.local/share/containers`

Step 7 is where CRIU comes in. It enables users to break down process shutdown into a couple of fine-grained steps (steps 1 through 6 are identical to the above):

7. `podman checkpoint` -- Make Podman dump memory contents to disk and kill process or processes
7. Memory dumped to disk, workloads no longer running
7. `podman restore` -- Restores previously dumped memory content into new processes
7. Workload (either batch job or daemon) up and running again
7. `podman kill` -- See above
7. `podman rmi` -- See above
7. `podman rmi` -- See above


```
# Start container that increments some numbers:
$ podman run -d --name looper ubi8 /bin/sh -c \
  'i=0; while true; do echo $i; i=$(expr $i + 1); sleep 1; done'
# Verify numbers are generated:
$ podman logs -l
  1
  2
  3
  4
  5
  ...
# ("-l", "--latest": "Act on latest container Podman is aware of")
# Dump memory contents and kill container processes:
$ podman container checkpoint -l
  8f9312bec0373d4bbc6215cf1084fa55496d181fdd3266005f1f9b9f1ae6f8d1
$ podman ps -a
  CONTAINER ID  IMAGE                                             COMMAND               CREATED         STATUS                     PORTS                 NAMES
  8f9312bec037  registry.access.redhat.com/ubi8:latest            /bin/sh -c i=0; w...  3 minutes ago   Exited (0) 13 seconds ago                        looper
# Note: Container is in "Exited" state, meaning it stopped running, but its
# copy-on-write layer hasn't been deleted yet

# Now, restore container with previously checkpointed memory contents:
$ podman container restore -l
# Numbers are now incrementing again, but not from 0, but from the latest number
# the previous container was stopped at:
$ podman logs -l
# Kill container:
$ podman kill -a
# (Will kill process(-es), delete all contents in copy-on-write-layer, 
# and remove all meta data for all containers)
```

## Custom SELinux policies made easy with Udica

* Use case: SELinux blocks a container's process, so the container doesn't start. 
* Problem: How to run container anyway without (a) simply disabling SELinux, or (b) becoming an SELinux expert?
* Solution: Udica!

```
# Example for container that will not run out of the box with SELinux enabled:
$ podman run --name home-test -v /home/:/home:ro -it ubi8 ls -al /home
# This command tries to mount the "/home" directory as read-only into the container, 
# which SELinux, by default, does not allow
# Udica can be used to set up a custom SELinux rule quickly and easily

# Extract container metadata:
$ podman inspect home-test > home-test.json

# Udica can read this file and create a custom SELinux policy for us:
udica -j home-test.json home_test
# Load new policy into SELinux:
$ semodule -i home_test.cil /usr/share/udica/templates/{base_container.cil,home_container.cil}

# To make SELinux allow the read-only mount of the "/home" directory, we have use an 
# option telling it to label the container process for it to use the newly created policy:
$ podman run --name home-test-2 --security-opt \
  label=type:home_test.process -v /home/:/home:ro \
  -id ubi8 bash
# Execute "ls" command home directory:
$ podman exec -it home-test-2 ls /home
# Viewing the process table, we can see the process runs with the "home_test.process" 
# SELinux policy:
$ ps -efZ | grep home_test
# Verify there is, in fact, a new rule in this SELinux policy:
$ sesearch -A -s home_test.process -t home_root_t -c dir -p read
```













