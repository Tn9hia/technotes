> [!note]
 >Kube-apiserver must have the hightest version
> Controller manager and kube scheduller can have lower 1 version
>Kubelet and kubeproxy can have lower 2 version
When upgrade version of k8s cluster, it should be upgrade one mirror version at a time
## Process to upgrade
### 1. Upgrade Control Plane (Master) Node(s)

- When the control plane is taken down for upgrade, **worker nodes continue running existing workloads**.
- Applications remain available, but **no new pods can be scheduled** (API server unavailable).
- Once the upgrade is complete, the control plane resumes normal scheduling and management.
### 2. Upgrade Worker Nodes

There are three main approaches:
- **Option 1: Upgrade all nodes at once**
    - Fastest method.
    - Causes full workload downtime.
    - Not recommended for production.
- **Option 2: Upgrade nodes one by one (Rolling Upgrade)**
    - Use `kubectl drain <node>` to safely evict workloads.
    - Upgrade the node → rejoin it to the cluster.
    - Repeat for each worker node.
    - Keeps the cluster serving workloads with minimal disruption.
- **Option 3: Add & Replace (Blue/Green style)**
    - Add new nodes running the upgraded version.
    - Migrate workloads to the new nodes.
    - Remove or upgrade the old nodes afterward.
    - Minimizes downtime, best practice for large production environments.

### Main Steps to Upgrade a K8s Cluster
1. **Plan & Backup**
    - Check compatibility (control plane version, kubelet, CNI, CSI, etc.).
    - Backup etcd and manifests.
    - Make sure workloads are healthy.
2. **Upgrade the Control Plane (Master Nodes)**
    - `kubectl drain` the control plane node.
    - Upgrade `kubeadm` package → run `kubeadm upgrade plan` → `kubeadm upgrade apply <version>`.
    - Upgrade `kubelet` + `kubectl` on that node, then restart.
    - `kubectl uncordon` when done.
    - Repeat for all control plane nodes (if HA).
3. **Upgrade Worker Nodes**
    - Option A: Upgrade one by one (safe, rolling).
    - Option B: Add new node(s) with new version, drain + remove old nodes (blue/green).
    - Steps per node: `kubectl drain` → upgrade `kubeadm/kubelet` → `kubeadm upgrade node` → restart kubelet → `kubectl uncordon`.
4. **Upgrade Add-ons**
    - Update CNI, CSI, CoreDNS, kube-proxy, Ingress controller, monitoring stack, etc.
    - Verify network & storage plugins support the new version.
5. **Validation**
    - Check `kubectl get nodes` → all versions aligned.
    - Run smoke tests on workloads.
    - Monitor logs & metrics for issues.
## Reference
- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/