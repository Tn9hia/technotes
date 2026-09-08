---
tags:
  - openstack
  - glance
  - image
---

# Glance — Image Service

Glance cung cấp dịch vụ **lưu trữ và quản lý disk images** cho VMs. Khi boot VM, Nova lấy image từ Glance.

## Kiến trúc

```
User/Nova
   │
glance-api ──► glance-registry (metadata, deprecated từ Queens)
   │
Backend Store
├── File (local filesystem)
├── Ceph RBD (production recommended)
├── Swift
├── S3
└── HTTP (read-only, từ URL)
```

## Image Formats

### Disk Format

| Format | Mô tả | Dùng khi |
|--------|-------|---------|
| `qcow2` | QEMU Copy-On-Write | KVM, hỗ trợ snapshot, compression |
| `raw` | Raw disk image | Hiệu năng cao nhất (không overhead) |
| `vmdk` | VMware | Migrate từ VMware |
| `vhd` | Hyper-V | Migrate từ Hyper-V |
| `iso` | CD-ROM image | Boot từ ISO |

### Container Format

| Format | Mô tả |
|--------|-------|
| `bare` | Chỉ có disk image (phổ biến nhất) |
| `ovf` | OVF package |
| `aki/ari/ami` | Amazon format |

## Upload Image

```bash
# Upload image từ file
openstack image create "Ubuntu 22.04" \
  --file ubuntu-22.04-server-cloudimg-amd64.img \
  --disk-format qcow2 \
  --container-format bare \
  --public

# Upload từ URL (Glance download trực tiếp)
openstack image create "CirrOS" \
  --copy-from http://download.cirros-cloud.net/0.6.1/cirros-0.6.1-x86_64-disk.img \
  --disk-format qcow2 --container-format bare --public

# Convert từ vmdk sang qcow2 trước khi upload
qemu-img convert -f vmdk -O qcow2 source.vmdk output.qcow2
qemu-img info output.qcow2
```

## Image Properties

```bash
# Set properties (hints cho Nova scheduler)
openstack image set \
  --property hw_disk_bus=virtio \
  --property hw_vif_model=virtio \
  --property hw_scsi_model=virtio-scsi \
  --property os_distro=ubuntu \
  --property os_version=22.04 \
  ubuntu-22.04

# Xem properties
openstack image show ubuntu-22.04
```

### Quan trọng properties

| Property | Giá trị | Tác dụng |
|----------|---------|---------|
| `hw_disk_bus` | `virtio`/`scsi`/`ide` | Disk controller type |
| `hw_vif_model` | `virtio`/`e1000` | NIC model |
| `hw_rng_model` | `virtio` | Random number generator |
| `os_distro` | `ubuntu`/`centos` | OS detection |
| `img_config_drive` | `mandatory` | Yêu cầu config drive |

## Backend: Ceph RBD (Production)

```ini
# glance-api.conf
[glance_store]
default_store = rbd
stores = rbd

[glance_store]
rbd_store_pool = images
rbd_store_user = glance
rbd_store_ceph_conf = /etc/ceph/ceph.conf
rbd_store_chunk_size = 8
```

> [!tip] Copy-on-write với Ceph + Nova
> Khi Glance và Nova đều dùng Ceph, việc tạo VM từ image sẽ dùng **Ceph COW clone** → không cần copy toàn bộ image → boot nhanh hơn nhiều.

## Image Visibility

```bash
# Public: tất cả projects nhìn thấy
# Private: chỉ owner
# Shared: chia sẻ với specific projects
# Community: tất cả nhìn thấy nhưng không maintained

openstack image set --public <image-id>
openstack image set --private <image-id>

# Share với project khác
openstack image add project <image-id> <project-id>
openstack image set --accept <image-id>  # project đích phải accept
```

## Image Download và Inspect

```bash
# Download image
openstack image save --file output.qcow2 <image-id>

# Inspect disk image
qemu-img info image.qcow2
virt-filesystems --long -h --all -a image.qcow2
```

## Cloud Images phổ biến

| OS | URL download |
|----|-------------|
| Ubuntu 22.04 | `cloud-images.ubuntu.com` |
| CentOS Stream 9 | `cloud.centos.org` |
| Debian 12 | `cloud.debian.org` |
| Rocky Linux 9 | `dl.rockylinux.org` |
| CirrOS (test) | `download.cirros-cloud.net` |

> [!note] Cloud images vs ISO
> Cloud images đã được cấu hình sẵn `cloud-init`, không có mật khẩu root, và dùng SSH key để login. Đây là format đúng cho OpenStack.

---
*Xem thêm: [[Nova - Compute]] | [[Cloud/AWS/SAA/Storage/Storage Overview]] | [[Ceph Integration]] | [[OpenStack]]*
