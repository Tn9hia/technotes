---
tags:
  - openstack
  - nova
  - compute
---

# Nova — Compute Service

Nova là service quản lý **vòng đời máy ảo (VM lifecycle)**. Đây là core service quan trọng nhất của OpenStack.

## Kiến trúc

```
                    nova-api
                       │
              nova-conductor (DB access layer)
             /           \
    nova-scheduler    nova-compute (trên từng Compute node)
                           │
                        libvirt/KVM
                           │
                          VMs
```

### Các components

| Component | Chạy trên | Chức năng |
|-----------|-----------|-----------|
| `nova-api` | Controller | Tiếp nhận REST API requests |
| `nova-scheduler` | Controller | Chọn compute node để đặt VM |
| `nova-conductor` | Controller | DB proxy (compute không direct access DB) |
| `nova-compute` | Compute | Gọi libvirt để tạo/manage VM |
| `nova-novncproxy` | Controller | VNC console access |
| `nova-placement` | Controller (tách riêng) | Track resource inventory |

> [!info] nova-placement → Placement API
> Từ OpenStack Stein, `nova-placement` trở thành service độc lập: **Placement API**. Nó track tài nguyên của hypervisors (CPU, RAM, disk, GPU...).

## VM Lifecycle

```
           boot request
               │
           nova-api
               │ validate, quota check
           nova-scheduler ──► Chọn host (dựa trên filters + weights)
               │
           nova-conductor ──► Lưu DB
               │
           nova-compute (trên host được chọn)
               │
        ┌──────┼───────────┐
        │      │           │
     Glance  Neutron    Cinder
    (image) (network)  (volume)
        │      │           │
        └──────┼───────────┘
               │
            libvirt/KVM
               │
              VM created ✓
```

## Scheduler

### Filters (loại bỏ host không phù hợp)
- `AvailabilityZoneFilter` — chỉ host trong AZ được chọn
- `ComputeFilter` — host phải đang up
- `RamFilter` — đủ RAM (với overcommit ratio)
- `DiskFilter` — đủ disk
- `ComputeCapabilitiesFilter` — match extra specs
- `ImagePropertiesFilter` — CPU architecture, hypervisor type
- `ServerGroupAntiAffinityFilter` — anti-affinity rules
- `AggregateInstanceExtraSpecsFilter` — host aggregate metadata

### Weights (sắp xếp ưu tiên)
- `RAMWeigher` — ưu tiên host nhiều RAM nhất (mặc định, spread strategy)
- `DiskWeigher`
- Custom weighers

```ini
# nova.conf
[filter_scheduler]
available_filters = nova.scheduler.filters.all_filters
enabled_filters = ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter
```

## Flavor

Định nghĩa **kích thước VM** (vCPU, RAM, disk):

```bash
# List flavors
openstack flavor list

# Tạo flavor
openstack flavor create --vcpus 4 --ram 8192 --disk 50 m1.large

# Extra specs (cho scheduler filters)
openstack flavor set m1.large \
  --property hw:cpu_policy=dedicated \
  --property hw:mem_page_size=large
```

### Extra Specs quan trọng

| Extra Spec | Giá trị | Mục đích |
|-----------|---------|---------|
| `hw:cpu_policy` | `dedicated`/`shared` | CPU pinning |
| `hw:mem_page_size` | `large`/`small`/`1GB` | Huge pages (NUMA) |
| `hw:numa_nodes` | `1`/`2` | NUMA topology |
| `hw:vif_multiqueue_enabled` | `true` | Network multiqueue |
| `aggregate_instance_extra_specs:ssd` | `true` | Host aggregate filter |

## Live Migration

```bash
# Live migrate (shared storage hoặc block migration)
openstack server migrate --live-migration --host compute02 <vm-id>

# Block live migration (không cần shared storage, copy disk)
openstack server migrate --live-migration --block-migration --host compute02 <vm-id>

# Kiểm tra trạng thái
openstack server show <vm-id> | grep -i migrat
```

> [!tip] Shared storage giúp live migration nhanh hơn
> Nếu dùng Ceph làm ephemeral storage (nova + ceph), live migration chỉ cần chuyển memory state, không cần copy disk.

## Server Groups (Anti-Affinity)

```bash
# Tạo anti-affinity group (VMs sẽ đặt trên các host khác nhau)
openstack server group create --policy anti-affinity ha-group

# Boot VM vào group
openstack server create \
  --hint group=<group-id> \
  --flavor m1.medium \
  --image ubuntu-22.04 \
  web-01
```

## Console Access

```bash
# VNC console URL
openstack console url show --novnc <vm-id>

# Serial console
openstack console url show --serial <vm-id>

# Console log
openstack console log show <vm-id>
```

## Resize VM

```bash
# Resize (thay đổi flavor)
openstack server resize --flavor m1.xlarge <vm-id>

# Confirm resize
openstack server resize --confirm <vm-id>

# Revert nếu không hài lòng
openstack server resize --revert <vm-id>
```

## Nova trên Compute node

```bash
# Trên compute node, kiểm tra libvirt
virsh list --all
virsh dominfo <instance-name>  # instance-xxxxxxxx

# Nova instance directory
ls /var/lib/nova/instances/
ls /var/lib/nova/instances/<instance-id>/

# Logs
tail -f /var/log/nova/nova-compute.log
```

## Quota Management

```bash
# Xem quota của project
openstack quota show <project-id>

# Set quota
openstack quota set --instances 50 --cores 200 --ram 512000 <project-id>
```

---
*Xem thêm: [[Glance - Image]] | [[Neutron - Networking]] | [[Cinder - Block Storage]] | [[OpenStack]]*
