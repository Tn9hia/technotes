**Static Pods** are managed directly by the kubelet daemon on a specific node, without the API server observing them. Unlike Pods that are managed by the control plane (for example, a Deployment); instead, the kubelet watches each static Pod (and restarts it if it fails).

Static Pods are always bound to one Kubelet on a specific node.

> [!note]
> If you are running clustered Kubernetes and are using static Pods to run a Pod on every node, you should probably be using a DaemonSet instead.
> 

> [!note]
> The spec of a static Pod cannot refer to other API objects (e.g., ServiceAccount, ConfigMap, Secret, etc).

To create a static pod, place definition file in a defined directory
Configure the kubelet on that node to set a `staticPodPath` value in the [kubelet configuration file](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).