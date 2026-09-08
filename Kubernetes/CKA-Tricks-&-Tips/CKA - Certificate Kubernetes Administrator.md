**Storage** - 10%
- Implement storage classes and dynamic volume provisioning
- Configure volume types, access modes and reclaim policies
- Manage persistent volumes and persistent volume claims

**Workloads & Scheduling** - 15%
- Understand application deployments and how to perform rolling update and rollbacks
- Use ConfigMaps and Secrets to configure applications
- Configure workload autoscaling
- Understand the primitives used to create robust, self-healing, application deployments
- Configure Pod admission and scheduling (limits, node affinity, etc.)

**Services & Networking** - 20%
- Understand connectivity between Pods
- Define and enforce Network Policies
- Use ClusterIP, NodePort, LoadBalancer service types and endpoints
- Use the Gateway API to manage Ingress traffic
- Know how to use Ingress controllers and Ingress resources
- Understand and use CoreDNS

**Cluster Architecture, Installation & Configuration** - 25%
- Manage role based access control (RBAC)
- Prepare underlying infrastructure for installing a Kubernetes cluster
- Create and manage Kubernetes clusters using kubeadm
- Manage the lifecycle of Kubernetes clusters
- Implement and configure a highly-available control plane
- Use Helm and Kustomize to install cluster components
- Understand extension interfaces (CNI, CSI, CRI, etc.)
- Understand CRDs, install and configure operators

**Troubleshooting** - 30%
- Troubleshoot clusters and nodes
- Troubleshoot cluster components
- Monitor cluster and application resource usage
- Manage and evaluate container output streams
- Troubleshoot services and networking

## Dump test
- https://officialdumps.com/exam/cka
## Important Note
### CNI
In the CKA exam, for a question that requires you to deploy a network add-on, unless specifically directed, you may use any of the solutions described in the link above.

**_However,_** the documentation currently does not contain a direct reference to the exact command to be used to deploy a third-party network add-on.

The links above redirect to third-party/vendor sites or GitHub repositories, which cannot be used in the exam. This has been intentionally done to keep the content in the Kubernetes documentation vendor-neutral.

**NOTE:** In the official exam, all essential CNI deployment details will be provided.

**Reference**
- [Installing Addons - Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/addons/)
- [Implementing the Kubernetes Networking Model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model)


