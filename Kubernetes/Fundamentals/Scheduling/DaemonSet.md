A _DaemonSet_ ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the Pods it created.
**Example**
```yaml
apiVerison: apps/v1
kind: DaemonSet
metadata: 
	name: monitor-daemon
spec:
	selector:
		matchLabels:
			app: monitor-agent
	template:
		metadata:
			labels:
				app: monitor-agent
		spec:
			containers:
			-	name: monitor-agent
				image: monitor-agent
```
**To view DaemonSet**
```shell
kubectl get daemonset
```