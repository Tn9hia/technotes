---
tags:
  - openstack
  - storage
  - cinder
  - swift
  - ceph
---

# Storage Overview

## 3 loại storage trong OpenStack

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenStack Storage                        │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Block Storage│  │Object Storage│  │  Shared File     │  │
│  │   (Cinder)   │  │   (Swift)    │  │  System (Manila) │  │
│  │              │  │              │  │                  │  │
│  │ - Persistent │  │ - Unstructured│ │ - NFS/CIFS share │  │
│  │ - Mounted vào│  │ - REST API   │  │ - Multiple VMs   │  │
│  │   1 VM       │  │ - Like S3    │  │ - shared access  │  │
│  │ - Like EBS   │  │              │  │ - Like EFS       │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Ephemeral vs Persistent Storage

### Ephemeral (Tạm thời)
- Lưu trên **Compute node** local disk
- **Mất khi VM bị deleted**
- Nhanh (local), không shared
- `nova.conf: default_ephemeral_format = ext4`

### Persistent (Cinder Volume)
- Lưu trên **Storage backend** (Ceph, LVM, NetApp...)
- **Tồn tại sau khi VM bị deleted**
- Có thể detach và reattach
- Hỗ trợ snapshot

```bash
# Boot VM từ volume (persistent boot disk)
openstack server create \
  --boot-from-volume 50 \
  --image ubuntu-22.04 \
  --flavor m1.large \
  persistent-vm
```

## Storage Backends So sánh

| Backend | Type | Performance | Production? | Notes |
|---------|------|-------------|-------------|-------|
| **LVM** | Block | OK | ⚠️ Single node | Default, không HA |
| **Ceph RBD** | Block | Good | ✅ | **Khuyến nghị** |
| **NFS** | File-based Block | Moderate | ⚠️ | Đơn giản nhưng scaling kém |
| **iSCSI (SAN)** | Block | Excellent | ✅ | Hardware SAN đắt |
| **NetApp** | Block/File | Excellent | ✅ | Enterprise |
| **Pure Storage** | Block | Excellent | ✅ | All-flash |
| **Ceph RadosGW** | Object | Good | ✅ | S3 compatible |
| **Swift** | Object | Good | ✅ | OpenStack native |

## Ceph — One Backend to Rule Them All

Ceph có thể cung cấp **tất cả loại storage**:

```
Ceph Cluster
├── RBD (RADOS Block Device) → Cinder volumes + Nova ephemeral
├── RADOS Gateway (RadosGW/RGW) → Object storage (S3/Swift API)
└── CephFS → Manila shared filesystem
```

## Data Path

### Read path (Cinder + Ceph)

```
VM ──► virtio-blk driver ──► QEMU/KVM ──► librbd ──► Ceph OSD
```

### Write path

```
VM ──► write data ──► librbd (client) ──► Primary OSD ──► 2 Replica OSDs
                                              (ack sau khi N replicas written)
```

## Storage Network

> [!important] Luôn có dedicated Storage Network
> Ceph replication traffic rất lớn. Phải có VLAN/NIC riêng.

```
Server
├── eth0/eth1 → bond0 → Management (VLAN 10)
├── eth2/eth3 → bond1 → Compute/Tunnel (VLAN 20)
└── eth4/eth5 → bond2 → Storage (VLAN 30)
                              │
                        Ceph public network   ← Clients (Nova, Cinder)
                        Ceph cluster network  ← OSD replication
```

## Manila — Shared File System (Optional)

Manila cung cấp **NFS/CIFS shares** cho nhiều VMs:

```bash
# Tạo share
openstack share create \
  --name my-share \
  --share-type default \
  NFS 100

# Lấy export location
openstack share show my-share | grep export_locations

# Mount vào VM
mount -t nfs <share-ip>:<path> /mnt/shared
```

## Storage Troubleshooting

```bash
# Cinder volume stuck in "creating"
openstack volume show <vol-id>
# Xem log trên storage node
tail -f /var/log/cinder/cinder-volume.log

# Ceph health
ceph status
ceph health detail
ceph df

# Kiểm tra Ceph OSD
ceph osd tree
ceph osd stat
ceph pg stat  # placement groups

# Xem RBD volumes
rbd ls -p volumes
rbd info volumes/<volume-id>
rbd disk-usage -p volumes
```

---
*Xem thêm: [[Ceph Integration]] | [[Cinder - Block Storage]] | [[Swift - Object Storage]] | [[OpenStack]]*
