The Kubernetes API server validates and configures data for the api objects which include **pods, services, replicationcontrollers, and others**. The API Server services REST operations and provides the frontend to the cluster's shared state through which all other components interact.

Reponsible for: 
- Authentication user
- Validate request
- Retrieve data
- Update ETCD
- Scheduler perform update 
- Kubelet do the task
 To view api-server option:
```
cat /etc/kubernetes/manifests/kube-apiserver.yaml 
```