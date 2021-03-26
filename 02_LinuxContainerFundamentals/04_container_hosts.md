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
$ cat /var/lib/containers/storage/overlay-containers/$(podman ps \
  -l -q --no-trunc)/userdata/config.json | jq .
# -> Podman hasn't actually started the container yet -- it just created 
# the config.json file and exited (that's because the given container image, 
# 'on-off-container', only starts a process within the container if
# the '/mnt/on' # file exists, see section below)
# Container now in state "EXITED":
$ podman ps -a
```

### Invoke container runtime

Now given: storage and spec file (in the form of _config.json_). Therefore, next step can be to create containerized process by invoking container runtime (here: _runc_).

Given container image (`on-off-container`) will only start process if file `/mnt/on` exists. 

```
# Create file
$ touch /mnt/on
# Launch container
$ podman start on-off-container
# This time, container will actually keep running after being started.
# Container status is now 'Up':
$ podman ps -a
# Shell into running container and examine file contents of test file created earlier:
$ podman exec -it on-off-container bash
$ ls -lisah /mnt
  [root@5f5d3692fa73 /]# ls -lisah /mnt
  total 8.0K
  1703937 4.0K drwxr-xr-x. 2 root root 4.0K Mar 26 13:10 .
   477233 4.0K drwxr-xr-x. 1 root root 4.0K Mar 26 13:00 ..
  1706405    0 -rw-r--r--. 1 root root    0 Mar 26 13:10 on
# Perform clean-up:
$ exit
$ podman kill -a
$ podman rm -a
```

## Generate contexts dynamically to protect containers

Involved: SELinux and sVirt

```
# Generate some running containers:
$ podman run -dt registry.access.redhat.com/ubi7/ubi sleep 10
  32247171e51ffcfc739587791f27ea4573d330fe6a2d8a7cd908571954e2d892
$ podman run -dt registry.access.redhat.com/ubi7/ubi sleep 10
  44f826724cb4291db2d5843fa6e377175ea33b0ad5c4edc0cd186aeeed9be1a2
$ ps -efZ | grep container_t | grep sleep
  system_u:system_r:container_t:s0:c257,c298 root 8440 8427  7 13:18 pts/0 00:00:00 sleep 10
  system_u:system_r:container_t:s0:c138,c506 root 8537 8524 14 13:18 pts/0 00:00:00 sleep 10
```

Takeway of the above process table output: Two running containers have dynamically generated, thus differing _MLS_ (_Multi Level Security_) labels (container 1: _c257,c298_; container 2: _c138,c506_), meaning they are prevented from accessing each other's memory, files, etc.

SELinux labels both the processes and the files accessed by the process:

```
# Create directory
$ mkdir /tmp/selinux-test
$ ls -alhZ /tmp/selinux-test/
  drwxr-xr-x. root root unconfined_u:object_r:user_tmp_t:s0 .
  drwxrwxrwt. root root system_u:object_r:tmp_t:s0       ..
# -> Type is set to 'user_tmp_t', but no MLS labels set just yet
# Running a couple of containers will make SELinux set new
# MLS labels every time:
$ podman run -t -v /tmp/selinux-test:/tmp/selinux-test:Z \
  registry.access.redhat.com/ubi7/ubi ls -alhZ /tmp/selinux-test
    drwxr-xr-x. root root system_u:object_r:container_file_t:s0:c530,c695 .
    drwxrwxrwt. root root system_u:object_r:container_file_t:s0:c530,c695 ..
# On the host system, the directory will carry the labels of the 
# last container that was run
$ ls -alhZ /tmp/selinux-test/
```

## Cgroups


* _Control groups_
* Cgroups get created dynamiclly with each container instantiation
* Means for containers to prevent accessing each other's resources
* Cgroups are a feature of the Linux kernel that limits, accounts for, and isolates usage of resources like CPU, RAM, network, and so on for each process
* Container engines make use of cgroupts to run containerized processes

## SECCOMP

* Tool to limit how a containerized process can interact with the kernel
* SECCOMP can be understood as a kind of firewall that can be used to block certain system calls
* Not active by default, but can be very powerful tool to block misbehaving containers when activated and properly configured

```
# Sample SECCOMP configuration:
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "syscalls": [
    {
      "name": "fchmodat",
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}
# Use configuration when starting a container will prevent chmod to successfully 
# run in the container:
$ podman run -it --security-opt seccomp=./labs/lab3-step5/chmod.json \
  registry.access.redhat.com/ubi7/ubi chmod 777 /etc/hosts
    chmod: changing permissions of '/etc/hosts': Operation not permitted
```

