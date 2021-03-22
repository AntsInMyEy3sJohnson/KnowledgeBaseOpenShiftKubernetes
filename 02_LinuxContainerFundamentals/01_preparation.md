# Linux container building blocks

* Container image
* Registry server
* Container orchestration
* Container host

## Container images

### What is it?
* Open-source code/libraries within a Linux distribution in a tarball
* Images made up of layers (even the base images)
    * Libraries (glibc, libssl, ...)
    * Binaries (httpd, ...)
    * Packages
    * Dependcy management (apt, yum, dnf)
    * Repositories
    * ...

### Static linking: dependency compilation
* Use case: Programs rely on libraries in order not having to re-implement often-used functionality (like SSL encryption/decryption)
* In general, everything a binary needs must be linked into it (static linking)
* Libraries can be compiled into binaries, which is referred to as _static linking_
* For example: C code + glibc + gcc makes up the program

### Dynamic linking: dependency sharing
* Use case: Binaries should be able to share the same libraries they rely on
* To implement this, a dynamic linker is required and the host's operating system needs to have access to this linker at runtime

### Packaging dependencies
* Use case: A component called a _resolver_ should be able to resolve a binary's dependencies at install time
* Those resolvers are more often called _dependency managers_: _yum_, _dnf_, _apt_, ...
* In the context of a container image build, those dependency managers have to resolve the binary's dependency tree at the time of building the image

### Parts of a container image
* Governed by OCI image specification standard
* Payload media types:
    * Image index/_manifest.json_: Provides index of image layers
    * Image layers encapsulate/describe a set of changes (a "delta") made to the underlying layer (which could be the base image)
    * _config.json_: Specifies env variables, command-line arguments and some meta information about the image
* Although an image in and of itself is used as an atomic unit, it shouldn't be understood and thought about us such because an image is really a description of layers, and each layer is, in turn, a description of changes

### Image layer as set of changes
* Each layer represents a set of changes
* Each change set can update or delete files or add new ones
* Resolving dependencies through package management
* All contained binaries still rely on dynamic linking at runtime

### Container image configuration
* The image as an executable can be run by the container engine
* To run an image, the container engine requires the _config.json_ file
    * Image variables
    * User options
    * Runtime defaults
* Image plus config.json handed to operating system kernel becomes the running container

## Container registries
 Container registries provide standardized means to find, run, share, pull, and introspect images, build new images, ...

### Flexibility vs. stability on traditional hosts vs. on container hosts
* Components to look at: OS dependencies, kernel space plus application to be run
* On traditional host
    * Kernel space and OS dependencies optimized for stability, application optimized for flexibility
    * Application relies on OS-level dependencies
    * Thus, application and updates of OS dependencies are tightly coupled -- conflict between optimization for stability of OS dependencies and optimization for flexibility of application
* On container host
    * Kernel space still optimized for stability, but both OS dependencies and application now encapsulated in container image
    * Container image is optimized for flexiblity -- thus, the OS dependencies and the application it contains are, too
    * Coupling between application and its dependencies on one hand and infrastructure on the other has thus been eliminated

### Pulling an image from a registry

```
$ docker pull registryserver/namespace/repository:tag
```
 
## Container hosts

### Introduction
* Containers are Linux (or Windows) processes, so containers don't run _on_ Docker
* As one of the many user space tools available, the Docker daemon interacts with the operating system's kernel to set up containers
* From the kernel's perspective, there is no distinction between a running container and other processes -- the container is also just a process

### Container engines
* Podman -> runc
* CRI-O -> runc
* dockerd -> containerd -> runc

### The container host 
* It's actually the host that is the "engine" for running containers
* The entire stack of components responsible for running containers communicates by means of the host's kernel
* Stack components (from lowest to highest in terms of abstraction layers):
    * operating system with its kernel
    * container runtime (_runc_)
    * container engine (e. g. Docker)
    * orchestration node (Kubelet)
* Kernel, container runtime, container engine, and things on top of the container engine must revision together and prevent regressions together -> Stack with many moving parts challenging to maintain

## Container engine

### Creating regular Linux processes
* A regular Linux process is created, destroyed, and managed with system-level calls into the kernel
* Examples:
    * `fork()` (e. g. Apache)
    * `exec()` (e. g. `ps`)
    * `exit()`
    * `kill()`
    * `open()`
    * `close()`
    * `system()`

### The "containerized" process
* From the kernel's perspective, a container is a regular process
* However, in contrast to many other processes, the process for a container is created by invoking `clone()`
* `clone()` creates namespaces for kernel resources like mounts, users, and network resources
* Thus, a process running inside a container process is the namespaced equivalent of the process running outside a container

### Container runtime
* Standardizes the way in which the user space communicates with the kernel
* Needs an OCI manifest (JSON files containing various directives) and a directory in the filesystem storing the extracted contents of a container image
* Example for a container runtime: _runc_ (popular container runtime today)

### Container engine
* Exposes API for clients (humans and other programs)
* Handles management of container images and storage
* Prepares all configuration and hands it to _runc_
    * Combination of image-, user-, and engine-provided defaults
    * Hierarchy of default configuration artifacts from highest to lowest: user -> image -> engine
    * Things work out-of-the-box because engine provides sensible defaults
* Examples for container engines: Docker (_dockerd_ + _container_), Podman


