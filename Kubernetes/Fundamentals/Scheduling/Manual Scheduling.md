By default pod is schedule automatically
To specify which node that pod should be assign to, define it in pod definition file
```yaml
apiVersion: v1
kind: Pod
metadata:
	name: nginx
spec:
	containers:
	-   name: nginx
		image: nginx
	nodeName: node02
```