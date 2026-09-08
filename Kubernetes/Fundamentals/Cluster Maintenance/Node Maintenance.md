To mainternance a node, you should drain pod in the node first
```
kubectl drain <node-name>
```
To mark node unscheduling,
```
kubectl unconcordon <node-name>
```
To mark node is able to schedule
```
kubectl cordon <node-name>
```

> [!node]
> When a node offline for 5 minutes, it will be mark as down

