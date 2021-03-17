# Deploying Prometheus and Grafana on OpenShift

## Deploying Prometheus to collect exposed metrics
```
$ oc project pad-monitoring
```

---
Excursion: The _ConfigMap_ API object
* Applications often rely on external configuration (command-line arguments, environment variables, configuration files, ...)
* Problem: Containerized application not portable if such configuration artifacts are stored with container image itself
* Solution: _ConfigMap_
    * Stores fine-grained information like individual properties as well as coarse-grained information like entire configuration files or JSON blobs
    * Provides mechanisms to inject configuration artifacts into containers
* ConfigMap holds key-value pairs of information
* Comparable to _Secret_, but designed with convenience of managing and accessing configuration data in mind rather than protecting sensitive data
---

```
$ oc process -f ./prometheus-configmap.yml | oc apply -f -
# "dc": deploymentconfig
$ oc rollout latest dc/prometheus-demo -n pad-monitoring
$ oc status --suggest
$ oc rollout status dc/prometheus-demo
```

## Deploying Grafana to visualize Prometheus metrics

```
$ oc new-app grafana/grafana:6.6.1 -n pad-monitoring
$ oc expose svc/grafana -n pad-monitoring
$ oc describe route/grafana
$ oc describe route/prometheus-demo-route
# Datasource can be added and configured in Grafana UI
```

