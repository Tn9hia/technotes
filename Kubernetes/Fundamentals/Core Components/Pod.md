_Pods_ are the smallest deployable units of computing that you can create and manage in Kubernetes.
A _Pod_  is a group of one or more containers, with shared storage and network resources, and a specification for how to run the containers.
Pods in a Kubernetes cluster are used in two main ways:
- Pods that run a single container.
- Pods that run multiple containers that need to work together. (1 main container and 1 or more helper container that share the same resource)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    # This is the pod template
    spec:
      containers:
      - name: hello
        image: busybox:1.28
        command: ['sh', '-c', 'echo "Hello, Kubernetes!" && sleep 3600']
      restartPolicy: OnFailure
```
**apiVersion**: Version of api used to create object
**kind**: type of object
**metadata:** metadata of object (can have any key-value pair) in dictionary type
**spec**: information for the object 

Here are some examples of workload resources that manage one or more Pods:
- Deployment
- StatefulSet
- DaemonSet
## Pod templates
**PodTemplate is used to manage pod in k8s**
Modifying the pod template or switching to a new pod template has no direct effect on the Pods that already exist. If you change the pod template for a workload resource, that resource needs to create replacement Pods that use the updated template.