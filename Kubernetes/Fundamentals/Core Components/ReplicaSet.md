A ReplicaSet's purpose is to maintain a stable set of replica Pods running at any given time. Usually, you define a Deployment and let that Deployment manage ReplicaSets automatically.


## Replication Controller
rc-definition.yml
```yaml
apiVersion: v1
kind: ReplicationController
metadata: 
	name: myapp-rc
	labels:
		app: myapp
		type: frontend
spec:
	template:
		metadata:
			name: myapp-pod
			labels:
				app: myapp
				type: frontend-pod
			specs:
				containers:
				-	name: nginx-container
					image: nginx
	replicas: 3
```
### Create replicas controller
```bash
kubectl create -f rc-definition.yml
```
### Get replicas controller
```bash
kubectl get replicationcontroller
```

## Replica set
replicaset-definition.yaml
```yaml
apiVersion: apps/V1
kind: ReplicaSet
metadata:
	name: myapp-rc
	labels:
		app: myapp
		type: frontend
spec:
	template:
		metadata:
			name: myapp-pod
			labels:
				app: myapp
				type: frontend-pod
			specs:
				containers:
				-	name: nginx-container
					image: nginx
	replicas: 3
	selector:
		matchLabels:
			type: front-end
```

> [!IMPORTANT] 
> - With ReplicaSet, **selector** is required because ReplicaSet can manage Pods that is not created by ReplicaSet
> - Labels and Selector is very import for Replicaset to indentify which Pod should be monitor
****

### Create replicas controller
```bash
kubectl create -f rc-definition.yml
```

### Get replicas controller
```bash
kubectl get replicaset
```

### Update ReplicaSet
To update ReplicaSet, update the definition file and run the following command to update ReplicaSet
```bash
kubectl replace -f replicaset-definition.yaml
```

Or use command:
```bash
kubectl scale --replicas=6 -f replicaset-definition.yaml
```

```bash
kubectl scale --replicas=6 -f replicaset myap-rc
```