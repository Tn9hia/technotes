---
tags:
  - openstack
  - ceph
  - storage
  - rbd
---

# Ceph Integration với OpenStack

Ceph là **distributed storage system** phổ biến nhất cho OpenStack production. Một Ceph cluster có thể thay thế Cinder LVM, Swift, và cung cấp storage cho Nova.

## Kiến trúc Ceph

```
┌──────────────────────────────────────────────────────┐
│                    Ceph Cluster                      │
│                                                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│  │  MON 1  │  │  MON 2  │  │  MON 3  │  (quorum)   │
│  └─────────┘  └─────────┘  └─────────┘             │
│                                                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│  │  MGR 1  │  │  MGR 2  │  │   MDS   │ (CephFS)    │
│  └─────────┘  └─────────┘  └─────────┘             │
│                                                      │
│  OSD 0  OSD 1  OSD 2  OSD 3  ...  OSD N             │
│  (1 disk per OSD, distributed across nodes)          │
└──────────────────────────────────────────────────────┘
```

### Components

| Component | Số lượng tối thiểu | Chức năng |
|-----------|-------------------|-----------|
| MON (Monitor) | 3 (odd number) | Cluster state, quorum |
| MGR (Manager) | 2 | Metrics, REST API, modules |
| OSD (Object Storage Daemon) | 3 | Actual data storage |
| MDS (Metadata Server) | 1+ | CephFS only |
| RGW (RADOS Gateway) | 1+ | S3/Swift API |

## Ceph Pools cho OpenStack

```bash
# Tạo pools (trên Ceph cluster)
# Pool cho Glance images
ceph osd pool create images 128
# Pool cho Cinder volumes
ceph osd pool create volumes 128
# Pool cho Nova ephemeral
ceph osd pool create vms 128
# Pool cho Cinder backup
ceph osd pool create backups 64

# Initialize pools
rbd pool init images
rbd pool init volumes
rbd pool init vms
rbd pool init backups
```

## Tạo Ceph Users

```bash
# User cho Glance
ceph auth get-or-create client.glance \
  mon 'profile rbd' \
  osd 'profile rbd pool=images' \
  mgr 'profile rbd pool=images' \
  > /etc/ceph/ceph.client.glance.keyring

# User cho Cinder
ceph auth get-or-create client.cinder \
  mon 'profile rbd' \
  osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images' \
  mgr 'profile rbd pool=volumes, profile rbd pool=vms' \
  > /etc/ceph/ceph.client.cinder.keyring

# User cho Nova
ceph auth get-or-create client.nova \
  mon 'profile rbd' \
  osd 'profile rbd pool=vms, profile rbd-read-only pool=volumes, profile rbd-read-only pool=images' \
  mgr 'profile rbd pool=vms' \
  > /etc/ceph/ceph.client.nova.keyring
```

## Cấu hình Glance dùng Ceph

```ini
# glance-api.conf
[glance_store]
default_store = rbd
stores = rbd
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
```

## Cấu hình Cinder dùng Ceph

```ini
# cinder.conf
[DEFAULT]
enabled_backends = ceph

[ceph]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_flatten_volume_from_snapshot = false
rbd_max_clone_depth = 5
rbd_store_chunk_size = 4
rados_connect_timeout = -1
rbd_user = cinder
rbd_secret_uuid = <libvirt-secret-uuid>
volume_backend_name = ceph
```

## Cấu hình Nova dùng Ceph

```ini
# nova.conf
[libvirt]
images_type = rbd
images_rbd_pool = vms
images_rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = nova
rbd_secret_uuid = <libvirt-secret-uuid>
disk_cachemodes = network=writeback
```

> [!tip] COW Clone: Boot VM cực nhanh
> Khi Nova + Cinder + Glance đều dùng Ceph:
> 1. Glance image lưu trong pool `images`
> 2. Nova **clone** image (COW) vào pool `vms` — không copy data
> 3. Boot VM ngay lập tức từ clone
>
> Thay vì copy 10GB image, chỉ tốn < 1 giây!

## Libvirt Secret (Nova-Cinder auth)

```bash
# Tạo libvirt secret cho Cinder trên mỗi compute node
cat > secret.xml <<EOF
<secret ephemeral='no' private='no'>
  <uuid>$(uuidgen)</uuid>
  <usage type='ceph'>
    <name>client.cinder secret</name>
  </usage>
</secret>
EOF

virsh secret-define --file secret.xml
SECRET_UUID=$(virsh secret-list | grep cinder | awk '{print $1}')
virsh secret-set-value --secret $SECRET_UUID \
  --base64 $(ceph auth get-key client.cinder)
```

## Ceph Commands

```bash
# Cluster health
ceph status
ceph health detail
ceph -w  # watch mode

# OSD
ceph osd tree
ceph osd stat
ceph osd df           # disk usage per OSD
ceph osd perf         # I/O latency

# PG (Placement Groups)
ceph pg stat
ceph pg dump | head   # chi tiết từng PG

# Pool
ceph df               # usage per pool
ceph osd pool stats   # I/O per pool
rbd ls -p volumes     # list images trong pool

# I/O performance test
rados bench -p volumes 30 write --no-cleanup
rados bench -p volumes 30 seq
rados bench -p volumes 30 rand

# RBD specific
rbd info volumes/<vol-id>
rbd disk-usage -p volumes
rbd snap ls volumes/<vol-id>
```

## Replication vs Erasure Coding

| | 3x Replication | Erasure Coding 4+2 |
|--|-----|-----|
| Raw → usable | 33% | 66% |
| Overhead | 3x | 1.5x |
| Performance | Better | Worse (CPU intensive) |
| Min OSDs | 3 | 6 |
| Recovery | Fast | Slower |

```bash
# Tạo EC pool
ceph osd erasure-code-profile set ec-profile \
  k=4 m=2 plugin=jerasure technique=reed_sol_van

ceph osd pool create ec-pool 128 erasure ec-profile
```

## Ceph Monitoring

```bash
# Ceph Exporter cho Prometheus
# Port 9283 (trên MGR node)
ceph mgr module enable prometheus

# Dashboard
ceph mgr module enable dashboard
ceph dashboard create-self-signed-cert
ceph dashboard set-login-credentials admin password
```

---
*Xem thêm: [[Cloud/OpenStack/Storage/Storage Overview]] | [[Cinder - Block Storage]] | [[Glance - Image]] | [[Nova - Compute]]*
