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
* See sample manifest file for usage examples

## Secrets

* Passwords, private keys, access tokens, TLS certificates and the like should not linger around anywhere in plain text!
* Hence, using ConfigMaps to inject such values into a Pod is bad (mkay)
* Instead, to store and inject such values, _Secrets_ should be used

### Secrets creation

See YAML for sample manifest

### Making use of Secrets

* Similar to ConfigMaps, except the type of volume is a `secret` rather than a `configMap` volume
* Caution: The base64-encoded string contained in the Secret is available to the Pod in decoded form
* Caution, the second: The maximum size for both ConfigMaps and Secrets is 1 MB

### Private Docker registries

* Common requirement: Pull images from private registry requirsing some sort of authentication
* Means to address this requirement in Kubernetes: Image Pull Secrets
* Image Pull Secrets are stored just like regular Secrets, but they are consumed in a different manner -- instead of mounting them using a `secret` volume, the Pod's `spec.imagePullSecrets` field is used
* Example (embedded in Deployment manifest):

```
# ...
kind: Deployment
# ...
spec:
  # ...
  template:
    spec:
      containers:
      - name: sample-container
      # ...
    imagePullSecrets:
    - name: some-image-pull-secret
    volumes:
    # ...
```

* If the same Image Pull Secret is specified over and over again to pull from the same registry, the Secret can also be added to the Service Account associated with the Pods

## Managing ConfigMaps and Secrets

* The usual `create`, `delete`, `get`, and `describe` commands are applicable to ConfigMaps and Secrets, too
* Some interesting options are available for creating ConfigMaps and Secrets imperatively via the command line, though:
    * `--from-file=<somefile>`: Load data from the given file. The file name will be the key name in the created ConfigMap or Secret, and its contents will be the value associated to the key in the ConfigMap or Secret.
    * `--from-file=<key>=<somefile>`: Load data from the given file with the name of the key explicitly defined.
    * `--from-file=<somedir>`: Load from the given directory all files whose name corresponds to a valid file name.
    * `--from-literal=<somekey>=<someval>`: Put specified key-value pair directly into ConfigMap or Secret.
* Updating ConfigMaps and Secrets is possible in three ways:
    * Update from file
    * Recreate and update
        * If ConfigMap or Secret inputs are stored somewhere separate on disk, then a new manifest has to be created first, and then this manifest has to be applied to update the corresponding ConfigMap or Secret that already exists on the cluster
        * Sample command:
        ```
        k create secret generic my-secret --from-file=my-file --dry-run -o yaml | k replace -f -
        ```
        * Useful because base64-encoding is handled for us -- we don't have to perform this step manually
    * Editing the current version
* Once a ConfigMap or Secret was updated, the updated version will automatically be pushed to all volumes using the ConfigMap or Secret in question
* There is, however, currently no way to automatically restart the Pod once a ConfigMap or Secret that it makes use of has been updated, so if this behavior is desired, it has to be provided in some other way









