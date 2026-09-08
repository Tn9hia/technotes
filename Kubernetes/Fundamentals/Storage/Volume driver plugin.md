## Storage driver (aka container runtime storage driver)
- Think of it like a file system plugin for your container runtime. Examples: `overlay2`, `aufs`, `devicemapper`, `btrfs`

## Volume driver (Kubernetes storage plugin)
- Volume drivers tell Kubernetes how to talk to actual storage backends (local disk, NFS, Ceph, AWS EBS, Azure Disk, CSI plugins, etc.).
- In the old days, these were “in-tree” (built into Kubernetes core). Now most are **CSI drivers** (_Container Storage Interface_), which are separate installable plugins.

