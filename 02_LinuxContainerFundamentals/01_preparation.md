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


