**Taints and tolerations** work together to ensure that **pods are not scheduled onto inappropriate nodes**. One or more taints are applied to a node; this marks that the node should not accept any pods that do not tolerate the taints.
## Command
### Tain a node
You add a taint to a node
```shell
kubectl taint nodes node-name key1=value1:taint-effect
```
There are 3 kind of taint effect:
- **NoSchedule**: pod will not schedule on the node
- **PreferNoSchedule**: System'll try to avoid schedule on this node
- **NoExecute**: no new pod will be schedule on this node, exit pod will be evicted
### Toleration a pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "example-key"
    operator: "Exists"
    effect: "NoSchedule"
```

