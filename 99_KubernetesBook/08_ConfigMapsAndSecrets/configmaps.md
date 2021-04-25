# ConfigMaps And Secrets

* Goal: Keep images themselves as generic as possible so they can be reused across stages and in different use cases
* Question: How, then, to inject specialized data, such as configuration artifacts, into an image so the image is fully functional in a specific stage or use case?
* Solution: _ConfigMaps_ and _Secrets_ as means to specialize use of image at runtime to particular use case or stage
    * ConfigMaps: Means to encapsulate and then provide configuration artifacts to workloads, which could be fine-grained information like individual key-value configuration properties or entire files in the form of large strings
    * Secrets: Same as ConfigMaps, but for sensitive configuration artifacts like passwords and TLS certificates

## ConfigMaps

* ConfigMaps have to exist in the cluster prior to the Pods requiring them being started as a ConfigMap is combined with the Pod right before the Pod is started
* Consequently, both the container image(-s) and the Pod definition can remain generic and thus be reused across apps, and for each specific scenario the image-pod combination is employed in, it's just the ConfigMap that changes

### Creating ConfigMaps

Sample manifest see YAML file.

### Using ConfigMaps

* Three options to use ConfigMap:
    * __Filesystem__: ConfigMap can be mounted into Pod, and one file will be created for each key encapsulated in the ConfigMap's `data` section. Each file's contents will mirror the value of the key as given in the ConfigMap.
    * __Env variable__ (see sample manifest file)
    * __Command-line argument__: Dynamically create container's command line using value from a ConfigMap