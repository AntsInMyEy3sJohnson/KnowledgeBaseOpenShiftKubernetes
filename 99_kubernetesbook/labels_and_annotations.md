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
# Single quotes -- if the list of selectors contains an exclamation work, 
# certain shells (Zsh, for example) will interpret the next word as an event
$ kubectl -n kuar get pod -l 'version=1,!somelabel'
$ kubectl -n kuar get pod -l 'version in (1, 3)' \
  -o custom-columns=NAME:.metadata.name
```
