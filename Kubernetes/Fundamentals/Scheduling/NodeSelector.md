Assign pod to spectify node
## Command
### Label node
```shell
kubectl labels node <node-name> <label-key>:<label-value>

ex: kubectl labels node node1 size:large 
```
### Assign pod to node
```yaml
apiVerison: v1
kind: Pod
metadata:
	name: my-app
spec: 
	containers:
	-	name: nginx
		image: nginx
	nodeSelector
		size: large
```