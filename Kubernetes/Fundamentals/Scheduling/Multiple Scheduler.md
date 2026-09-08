Kubernetes ships with a default scheduler that is described [here](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/). If the default scheduler does not suit your needs you can implement your own scheduler. Moreover, you can even run multiple schedulers simultaneously alongside the default scheduler and instruct Kubernetes what scheduler to use for each of your pods. Let's learn how to run multiple schedulers in Kubernetes with an example.

A detailed description of how to implement a scheduler is outside the scope of this document. Please refer to the kube-scheduler implementation in [pkg/scheduler](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler) in the Kubernetes source directory for a canonical example.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotation-second-scheduler
  labels:
    name: multischeduler-example
spec:
  schedulerName: my-scheduler
  containers:
  - name: pod-with-second-annotation-container
    image: registry.k8s.io/pause:3.8

```

To view which scheduler  schedule pod:
```shell
kubectl get events -o wide
```

View scheduler log
```shell
kubectl logs <scheduler-name>
```