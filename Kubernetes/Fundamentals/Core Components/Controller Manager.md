The Kubernetes controller manager is a daemon that embeds the core control loops shipped with Kubernetes.  In Kubernetes, a controller is a control loop that **watches** the shared state of the cluster through the apiserver and **makes changes** attempting to move the current state towards the desired state.

Config file:
```
cat /etc/kubernetes/manifests/kube-controller-manager.yaml 
```