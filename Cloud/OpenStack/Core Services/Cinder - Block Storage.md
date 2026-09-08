---
tags:
  - openstack
  - cinder
  - storage
  - block-storage
---

# Cinder — Block Storage Service

Cinder cung cấp **persistent block storage** cho VMs — tương đương EBS của AWS. Volume tồn tại độc lập với VM lifecycle.

## Kiến trúc

```
cinder-api (Controller)
      │
cinder-scheduler ──► Chọn storage backend
      │
cinder-volume (Storage node)
      │
   Backend Driver
   ├── LVM (mặc định, local)
   ├── Ceph RBD (khuyến nghị production)
   ├── NFS
   ├── NetApp
   └── Pure Storage, EMC, etc.
```

### Components

| Component | Nơi chạy | Chức năng |
|-----------|----------|-----------|
| `cinder-api` | Controller | REST API |
| `cinder-scheduler` | Controller | Chọn backend cho volume |
| `cinder-volume` | Storage/Controller | Quản lý volume trên backend |
| `cinder-backup` | Storage | Backup volume |

## Volume Types

```bash
# List volume types
openstack volume type list

# Tạo volume type với backend cụ thể
openstack volume type create ceph-ssd \
  --property volume_backend_name=ceph-ssd

openstack volume type create lvm-hdd \
  --property volume_backend_name=lvm-hdd

# Tạo volume
openstack volume create --size 50 --type ceph-ssd my-volume

# Attach vào VM
openstack server add volume <vm-id> <volume-id>
```

## Backend: LVM (Default)

```ini
# cinder.conf
[DEFAULT]
enabled_backends = lvm

[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = tgtadm
volume_backend_name = lvm
```

```bash
# Kiểm tra LVM volume group trên storage node
vgs cinder-volumes
lvs | grep volume
```

## Backend: Ceph RBD (Production Recommended)

```ini
# cinder.conf
[DEFAULT]
enabled_backends = ceph-ssd,ceph-hdd

[ceph-ssd]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes-ssd
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
volume_backend_name = ceph-ssd

[ceph-hdd]
volume_driver = cinder.volume.drivers.rbd.RBDDriver
rbd_pool = volumes-hdd
rbd_ceph_conf = /etc/ceph/ceph.conf
rbd_user = cinder
volume_backend_name = ceph-hdd
```

> [!tip] Ceph cho phép live migration dễ dàng
> Khi dùng Ceph RBD, volume không gắn với bất kỳ node nào → live migration VM không cần copy data.

## Snapshot và Backup

```bash
# Snapshot volume
openstack volume snapshot create --volume <vol-id> my-snapshot

# Tạo volume từ snapshot
openstack volume create --snapshot <snap-id> --size 50 restored-vol

# Backup volume (lưu vào Swift hoặc NFS)
openstack volume backup create --name my-backup <vol-id>

# Restore backup
openstack volume backup restore <backup-id> <vol-id>
```

## Volume Migration

```bash
# Migrate volume giữa backends
openstack volume migrate --host <cinder-host@backend#pool> <vol-id>
```

## Multi-attach

Từ OpenStack Train, hỗ trợ attach cùng 1 volume vào nhiều VMs (cần dùng shared filesystem protocol như RWX).

```bash
# Tạo volume type hỗ trợ multi-attach
openstack volume type create multiattach \
  --property multiattach="<is> True"

openstack volume create --type multiattach --size 10 shared-vol
```

## Quotas

```bash
# Xem quota volumes
openstack quota show --volume <project-id>

# Set quota
openstack quota set --volumes 100 --gigabytes 5000 <project-id>
```

## Troubleshooting

```bash
# Xem log
tail -f /var/log/cinder/cinder-volume.log

# Kiểm tra volume bị stuck
openstack volume list --status error
openstack volume list --status available

# Reset trạng thái volume
openstack volume set --state available <vol-id>

# Trên storage node: kiểm tra iSCSI targets (nếu dùng LVM)
tgtadm --mode target --op show

# Kiểm tra Ceph RBD pools
rbd ls -p volumes
rbd info volumes/<volume-id>
```

---
*Xem thêm: [[Cloud/AWS/SAA/Storage/Storage Overview]] | [[Ceph Integration]] | [[Swift - Object Storage]] | [[OpenStack]]*
