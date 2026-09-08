## Label
_Labels_ are key/value pairs that are attached to [objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/#kubernetes-objects) such as Pods. 

- Use selector to filter resource

```shell
kubectl get pod --selector env=prod,app=frontend
```
## Field selectors
_Field selectors_ let you select Kubernetes [objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/#kubernetes-objects) based on the value of one or more resource fields.
- Field Selectors
```shell
kubectl get pods --field-selector status.phase=Running
```

### List of supported fields

| Kind                      | Fields                                                                                                                                                                                                                                                                              |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Pod                       | `spec.nodeName`  <br>`spec.restartPolicy`  <br>`spec.schedulerName`  <br>`spec.serviceAccountName`  <br>`spec.hostNetwork`  <br>`status.phase`  <br>`status.podIP`  <br>`status.podIPs`  <br>`status.nominatedNodeName`                                                             |
| Event                     | `involvedObject.kind`  <br>`involvedObject.namespace`  <br>`involvedObject.name`  <br>`involvedObject.uid`  <br>`involvedObject.apiVersion`  <br>`involvedObject.resourceVersion`  <br>`involvedObject.fieldPath`  <br>`reason`  <br>`reportingComponent`  <br>`source`  <br>`type` |
| Secret                    | `type`                                                                                                                                                                                                                                                                              |
| Namespace                 | `status.phase`                                                                                                                                                                                                                                                                      |
| ReplicaSet                | `status.replicas`                                                                                                                                                                                                                                                                   |
| ReplicationController     | `status.replicas`                                                                                                                                                                                                                                                                   |
| Job                       | `status.successful`                                                                                                                                                                                                                                                                 |
| Node                      | `spec.unschedulable`                                                                                                                                                                                                                                                                |
| CertificateSigningRequest | `spec.signerName`                                                                                                                                                                                                                                                                   |
### Supported operators
_Field selectors_ let you select Kubernetes [objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/#kubernetes-objects) based on the value of one or more resource fields. Here are some examples of field selector queries:

You can use the `=`, `==`, and `!=` operators with field selectors (`=` and `==` mean the same thing).

```shell
# operator of field selector
kubectl get services  --all-namespaces --field-selector metadata.namespace!=default

# Chain selectors
kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always
```