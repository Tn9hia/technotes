## Deploy Management cluster
- vSphere Supervisor (new): integrate closely to vSphere => from vsphere 7
- Standalone Management Cluster(Old): before vsphere 6
## Workload cluster

Tanzu Kubernetes Grid hosts three different types of workload clusters:
**Class-based clusters**
- Are Kubernetes objects of type `Cluster`
- Are a new type of cluster introduced in TKG 2.x
- Have basic topology defined in a `spec.topology` block
    - For example, number and type of worker and control plane nodes
**TKC-based clusters (legacy)**
- Are Kubernetes objects of type `TanzuKubernetesCluster`, used by the vSphere Kubernetes Service (VKS - formerly known as TKG Service) in vSphere Supervisor 7.
- **Plan-based clusters (legacy)**
    - Are Kubernetes objects of type `Cluster`
    - Can be created by using a standalone TKG v2.x management cluster on vSphere 7 and 8
> [!important]
> Class-based clusters with `class: tanzukubernetescluster`, all lowercase, are different from TKC-based clusters, which have object type `TanzuKubernetesCluster`. The `TanzuKubernetesCluster` type of cluster is not described in the TKG documentation.

### Two ways to create workload cluster:
![[Pasted image 20250820144616.png]]

## Package
Installing a _package_ on a workload cluster created by Tanzu Kubernetes Grid adds a functionality to the cluster. This functionality typically provides services to the workloads that the cluster hosts. For example, the Antrea package provides the Antrea container network interface (CNI), the Contour package ingress control services, the Harbor package a private container registry, and so on.

### Types of Packages

Tanzu Kubernetes Grid includes the following types of packages:

- **Auto-managed packages.** These packages are installed and upgraded automatically by Tanzu Kubernetes Grid. See the [Auto-Managed Packages](https://techdocs.broadcom.com/us/en/vmware-tanzu/standalone-components/tanzu-kubernetes-grid/2-5/tkg/about-tkg-packages-index.html#auto) section below.
- **CLI-managed packages.** These packages are installed and upgraded explicitly by using the Tanzu CLI. Located in the `tanzu-standard` package repository or in other repositories that you add to your clusters. See the [CLI-Managed Packages](https://techdocs.broadcom.com/us/en/vmware-tanzu/standalone-components/tanzu-kubernetes-grid/2-5/tkg/about-tkg-packages-index.html#cli) section below.
## Tanzu Kubernetes Releases
To support running diverse applications efficiently and reliably, you can customize Tanzu Kubernetes Grid (TKG) clusters to run its worker nodes and other VMs on different Kubernetes versions, operating systems, and operating system (OS) versions. For supported Kubernetes versions, VMware publishes Tanzu Kubernetes releases (TKrs), which associate a specific patch version of Kubernetes with compatible versions of a base OS plus compatible versions of additional components required by cluster nodes.