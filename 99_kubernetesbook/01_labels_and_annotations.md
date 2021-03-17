# Labels

* Key-value pairs that can be attached to Kubernetes objects such as Pods and ReplicaSets
* Can be arbitrary
* Useful for attaching identifying information to Kubernetes objects
* Provide the foundation for grouping objects
* A label consists of two parts: prefix (optional) and name plus the value

## Applying labels

```
$ kubectl create ns kuar
$ kubectl run demo1-prd \
  --image=gcr.io/kuar-demo/kuard-amd64:blue \
  --labels="version=1,app=demo1,env=prd" \
  --namespace=kuar
$ kubectl -n kuar get pod --selector="app=demo1"
$ kubectl run demo1-tst \
  --image=gcr.io/kuar-demo/kuard-amd64:green \
  --labels="version=2,app=demo1,env=tst \
  --namespace=kuar
$ kubectl -n kuar get pod --show-labels
```

## Modifying labels

```
# Set label
$ kubectl -n kuar label pod demo1-tst "canary=true"
# Will display a dedicated column for the given label, each row containing the 
# label's value for this API object
$ kubectl -n kuar get pod -L canary
# Unset label
$ kubectl -n kuar label pod demo1-tst "canary-"
```

## Label selectors

```
$ kubectl -n kuar get pod --selector="version=2"
# Chain multiple labels to form a logical AND query involving those labels
# alonog with the given values
$ kubectl -n kuar get pod --selector="version=2,app=demo1"
$ kubectl -n kuar get pod --selector="version in (1, 3)"
$ kubectl -n kuar get pod --selector="version"
```

| Selector operator          | Meaning                                  |
|----------------------------|------------------------------------------|
| key=value                  | "key" is set to "value"                  |
| key!=value                 | "key" is not set to "value"              |
| key in (value1, value2)    | "key" is one of "value1" or "value2"     |
| key notin (value1, value2) | "key" is not one of "value1" or "value2" |
| key                        | "key" is set                             |
| !key                       | "key" is not set                         |

```
$ kubectl -n kuar get pod --selector="!env"
$ kubectl -n kuar get pod --selector
# Single quotes -- if the list of selectors contains an exclamation mark, 
# certain shells (Zsh, for example) will interpret the next word as an event
$ kubectl -n kuar get pod -l 'version=1,!somelabel'
$ kubectl -n kuar get pod -l 'version in (1, 3)' \
  -o custom-columns=NAME:.metadata.name
```

# Annotations

* Provide place to attach additional metadata to Kubernetes objects
* Attached metadata often used by tools and libraries 
* Important use case for annotations: rolling deployments
* Labels vs. annotations: Labels used to group and identify Kubernetes objects, annotations used to enrich them with additional information
* "When to use what?" --> Start by adding information as an annotation and turn it into a label if the piece of information turns out to be a useful identifier (to be used in selector queries, for example)
* Caution: Annotations should not be used as general-purpose information store. "Smell" for inappropriate usage: Annotations used on object without having a well-defined meaning or purpose for that particular object, i. e. the annotations are not semantically bound to that specific object
* Annotations are key-value pairs like labels, but due to them often being used for information interchange by different tools and libraries, the prefix or "namespace" part of the key is more important
* Value is free-form string field
  * Makes them very flexible, but... 
  * ... the value being free-form prevents any kind of data validation, so if an annotation is used to store data, there is no way for Kubernetes to guarantee the data is correct --> Can increase complexity of debugging errors

