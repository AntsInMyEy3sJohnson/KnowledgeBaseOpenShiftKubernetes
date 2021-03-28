# Container standards

## OCI image specification

```
# Store image locally
$ mkdir rhel-tools-image && cd rhel-tools-image
$ podman pull registry.access.redhat.com/rhel7/rhel-tools:latest
# Save image as tar and unzip it:
$ podman save -o rhel-tools.tar rhel-tools
$ tar xfv rhel-tools.tar
  -r--r--r--. 1 root root 162928640 Jan  1  1970 0046fa34d5d69b95c3696fd7d15ae263f901b772823b525f55a7a42a8e8e050d.tar
  drwxr-xr-x. 2 root root      4096 Mar 27 13:54 2aa6eb75111d8da5d926353708163d13e833c891ba542697ebeaff9b4f0c7c9a
  drwxr-xr-x. 2 root root      4096 Mar 27 13:54 314857d594617f367d92c84a465b4bb01e32d1372d6875398a99bf15e1d08bb0
  drwxr-xr-x. 2 root root      4096 Mar 27 13:54 456075466664a20258abd1c7464b5f0ea59b6f7f936d17da59ba944f0e0b2d95
  -r--r--r--. 1 root root 215644160 Jan  1  1970 456075466664a20258abd1c7464b5f0ea59b6f7f936d17da59ba944f0e0b2d95.tar
  -r--r--r--. 1 root root     10240 Jan  1  1970 4620ae326a808e08d55b2f406893294ec7035b641be7aa2bb63d5bbf50aa1221.tar
  -r--r--r--. 1 root root      5746 Jan  1  1970 484e1a4e24564ce59e507f1b29781e7d8e31ad50e5493fc6a11da0acc4b6cab2.json
  -r--r--r--. 1 root root       359 Jan  1  1970 manifest.json
  -r--r--r--. 1 root root       110 Jan  1  1970 repositories
```

This reveals the most important image contents:

* Manifest file (_manifest.json)
* Config file  (here: _484e1a4e24564ce59e507f1b29781e7d8e31ad50e5493fc6a11da0acc4b6cab2.json_)
* More image layers, typically gzipped
* The meta data given in the config file resembles command-line options in both Docker and Podman
* Once all tar files are extracted, they can be mounted into a container's mount namespace

## OCI runtime specification

* Governs the format of the file handed to the container runtime (typically _runc_, although all OCI-compliant runtimes can work with the file format)
* The file handed to the container runtime is usually created by a container engine like Docker, but could, of course, also be created manually by a human user
* Three inputs flow into creation of the spec file:
    * Container image itself provides some input in the form of the _config.json_ file -- typically, the information attached to an image at build time contains instructions on how the image should be run
    * Default values provided by container engine (some configured in the container engine (like SECCOMP policies), some created dynamically by container engine (e. g. SELinux namespaces and bind mounts), and some hardcoded into the engien)
    * Command-line options provided by user of the container engine (which could also be a robot like Kubernetes)
* _runc_ is reference implementation for runtime specification

```
# Create simple spec file using runc:
$ runc spec
$ cat config.json | jq

# Slightly more advanced file using Podman (Podman is able to create a spec file without starting the container):
$ podman create --name rhel-tools -dt rhel-tools bash
  de6a915387ead129cc6d3387d40d3c93f27b2d63540086a3497387589370a93e
$ podman init rhel-tools
  de6a915387ead129cc6d3387d40d3c93f27b2d63540086a3497387589370a93e
# (Here, 'podman init' is the command that created the config file.)
# View generated file:
$ cat $(find /var/lib/containers/ | grep  $(podman ps --no-trunc -q | tail -n 1)/userdata/config.json) | jq
```

## OCI runtime reference implementation

### Setup

Two main things required to make OCI-compliant container runtime start a container:
* Filesystem to mount (often referred to as the _rootfs_)
* _config.json_ file

Note: A rootfs is, in essence, a Linux distribution extracted into a directory

```
# Start container, get its ID, mount it, then use rsync to synchronize the contents of its filesystem into a directory:
$ rsync -av $(podman mount $(podman create rhel-tools bash))/ /root/rhel-tools/rootfs/
# So, we now have a root file system we can use:
$ ls -alh /root/rhel-tools/rootfs
# Create simple spec file: 
$ runc spec -b /root/rhel-tools/
$ sed -i 's/"terminal": true/"terminal": false/' /root/rhel-tools/config.json
# All the container runtime needs to start a container now available:
$ ls -alh /root/rhel-tools
```

### Experiments

```
# Create empty container (user-space definition of container without spawning any processes):
$ runc create -b /root/rhel-tools/ rhel-tools
$ runc list

# Execute bash process in container:
$ runc exec --tty rhel-tools bash
# This looks just like a "normal" container, such as one created by Docker or Kubernetes or...
# ... that's because it is just like a regular container. So all it takes to create one 
# is really the rootfs and an OCI config file.
```

