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
 


