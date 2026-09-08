---
tags:
  - openstack
  - ironic
  - bare-metal
---

# Ironic — Bare Metal Provisioning Service

Ironic cho phép OpenStack **provision và quản lý máy chủ vật lý** giống như VMs — Bare Metal as a Service.

## Kiến trúc

```
Nova (nếu tích hợp) hoặc Ironic API trực tiếp
           │
     ironic-api
           │
     ironic-conductor ──► Drivers
                               ├── IPMI (iDRAC, iLO, BMC)
                               ├── Redfish
                               ├── iDRAC
                               └── SNMP
                          │
                     ironic-inspector (hardware discovery)
```

## Provisioning Flow

```
1. Enroll node (đăng ký BMC thông tin)
2. Inspect (tự động detect hardware specs)
3. Provide (đưa node vào available pool)
4. Deploy (PXE boot → deploy image → configure)
5. Active (node ready to use)
6. Clean (wipe disk khi release)
```

## Node States

```
enroll → manageable → available → active
                          ↑           │
                          └── clean ──┘
```

## Cấu hình và Enrollment

```bash
# Enroll một bare metal node
openstack baremetal node create \
  --name compute-01 \
  --driver ipmi \
  --driver-info ipmi_address=10.0.0.100 \
  --driver-info ipmi_username=admin \
  --driver-info ipmi_password=secret \
  --property cpus=32 \
  --property memory_mb=131072 \
  --property local_gb=480 \
  --property cpu_arch=x86_64

# Set deployment interface
openstack baremetal node set compute-01 \
  --deploy-interface direct

# Set boot interface
openstack baremetal node set compute-01 \
  --boot-interface pxe

# Manage node (từ enroll → manageable)
openstack baremetal node manage compute-01

# Inspect hardware
openstack baremetal node inspect compute-01

# Make available
openstack baremetal node provide compute-01

# List nodes và trạng thái
openstack baremetal node list
```

## PXE Boot Setup

```
┌─────────────┐    PXE Request    ┌──────────────────┐
│ Bare Metal  │ ────────────────► │ ironic-conductor  │
│   Server    │                   │ (DHCP + TFTP)     │
│             │ ◄──────────────── │                   │
└─────────────┘  Boot IPA (agent) └──────────────────┘
                                         │
                                   IPA (Ironic Python Agent)
                                   chạy trên server
                                         │
                                   Deploy image ──► Disk
```

## Ironic Python Agent (IPA)

IPA là ramdisk chạy trên server được provision:
- Nhận lệnh từ ironic-conductor
- Ghi image vào disk
- Cấu hình network interface
- Report hardware info

## Tích hợp với Nova

Khi tích hợp Nova + Ironic:
```bash
# Tạo flavor cho bare metal
openstack flavor create baremetal.large \
  --vcpus 32 --ram 131072 --disk 480 \
  --property baremetal=true

# Nova sẽ dùng Ironic để provision thay vì libvirt
openstack server create \
  --flavor baremetal.large \
  --image ubuntu-22.04 \
  my-bare-metal-vm
```

## Cleaning (disk wipe khi release)

```ini
# ironic.conf
[conductor]
automated_clean = true
clean_callback_timeout = 1800

# Steps làm sạch
[deploy]
erase_devices_priority = 10
erase_devices_metadata_priority = 20
```

## Use Cases

- **HPC Cluster**: cần raw hardware performance
- **NFV (Network Function Virtualization)**: SR-IOV, DPDK
- **Database servers**: I/O intensive, không muốn virtualization overhead
- **GPU servers**: ML training workloads

---
*Xem thêm: [[Hardware Requirements]] | [[Nova - Compute]] | [[OpenStack]]*
