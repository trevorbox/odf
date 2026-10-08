# Create auto-import-secret for primary and secondary clusters

```sh
oc config use primary

PRIMARY_CLUSTER=primary-cluster
PRIMARY_API=$(oc whoami --show-server)
PRIMARY_TOKEN=$(oc whoami -t)

oc config use secondary

SECONDARY_CLUSTER=secondary-cluster
SECONDARY_API=$(oc whoami --show-server)
SECONDARY_TOKEN=$(oc whoami -t)

oc config use hub

oc create secret generic auto-import-secret \
  --from-literal=autoImportRetry=2 \
  --from-literal=token=${PRIMARY_TOKEN} \
  --from-literal=server=${PRIMARY_API} \
  --namespace=${PRIMARY_CLUSTER}

oc create secret generic auto-import-secret \
  --from-literal=autoImportRetry=2 \
  --from-literal=token=${SECONDARY_TOKEN} \
  --from-literal=server=${SECONDARY_API} \
  --namespace=${SECONDARY_CLUSTER}
```
