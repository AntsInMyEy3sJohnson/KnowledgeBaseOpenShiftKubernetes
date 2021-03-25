# Container Hosts

## Container engines and the Linux kernel

```
$ docker run -td registry.access.redhat.com/ubi7/ubi top
$ docker run -td registry.access.redhat.com/ubi7/ubi top
$ podman run -td registry-access.redhat.com/ubi7/ubi top
# Inspect process table of host:
$ ps -efZ | grep -v grep | grep " top"
  system_u:system_r:container_t:s0:c322,c619 root 5195 5176  1 20:23 pts/2 00:00:00 top
  system_u:system_r:container_t:s0:c589,c798 root 5299 5280  1 20:23 pts/3 00:00:00 top
  system_u:system_r:container_t:s0:c247,c778 root 5565 5552  2 20:23 pts/0 00:00:00 top
# Takeaway: The `top` command was started within three containers, but the command still shows up 
# as a regular process in the process table. That's becaue container processes are regular Linux
# processes which are isolated using kernel technologies like namespaces, selinux and cgroups (sometimes 
# referred to as _sandboxing_ or _isolation_).
# => The docker daemon itself also runs alongside these processes!
```

## Creasting a container step by step

General workflow is mostly the same for all OCI-compliant container engines:

1. Pull/expand/mount image
1. Create OCI-compliant specification file 
1. Call `runc` (or another OCI-compliant container runtime), handing it the spec file

### Pull/expand/mount image

```
# Create container using Podman:
$ podman create -dt --name on-off-container \
    -v /mnt:/mnt:Z \
    quay.io/fatherlinux/on-off-container
# Container is in CREATED status -- not running yet:
$ podman ps -a

# Won't show our mount just yet
$ mount | grep -v docker | grep merged
$ podman mount on-off-container
  /var/lib/containers/storage/overlay/67c13b34f1860e1bdbb87e56e8154571befed229c7f6f5ad081042f7d5dbd901/merged
# -> System-level mount point into the overlay filesystem used by the container. All contents on the container's file 
# system can be changed here.

# Will now contain the mount for the Docker container
$ mount | grep -v docker | grep merged
$ ls $(podman mount on-off-container)
 bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
$ touch $(podman mount on-off-container)/test.txt
$ ls $(podman mount on-off-container)
 bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  test.txt  tmp  usr  var
```

### Create specification file

```
# Trigger creation of spec file by container engine
$ podman start on-off-container
  on-off-container
$ podman ps -l -q --no-trunc
  26865e7fd5f11bd8cd084191e5cae2205d02e7ce46d15bd58d6c1df24f55374e
# View newly created spec file
```