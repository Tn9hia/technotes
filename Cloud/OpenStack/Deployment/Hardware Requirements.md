---
tags:
  - openstack
  - deployment
  - hardware
---

# Hardware Requirements

## Cấu hình tối thiểu (Lab/POC)

### All-in-One

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 4 cores (VT-x/AMD-V) | 8+ cores |
| RAM | 16 GB | 32 GB |
| Storage | 100 GB SSD | 500 GB SSD |
| NIC | 1 x 1GbE | 2 x 10GbE |

### Multi-node (Minimum viable)

**Controller Node**

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 4 cores | 16 cores |
| RAM | 16 GB | 32 GB |
| Storage | 200 GB SSD (OS + MariaDB + logs) | 500 GB NVMe |
| NIC | 2 x 1GbE | 2 x 10GbE bonded |

**Compute Node**

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 8 cores (VT-x enabled) | 32-64 cores |
| RAM | 32 GB | 128-512 GB |
| Storage | 100 GB SSD (OS) | 200 GB NVMe |
| NIC | 2 x 1GbE | 2 x 25GbE bonded |

**Storage Node (Ceph)**

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 4 cores | 16 cores |
| RAM | 16 GB (min 1 GB/OSD) | 64 GB |
| OSD Disks | 3 x HDD | 6-12 x NVMe + WAL SSD |
| NIC | 2 x 10GbE | 2 x 25GbE bonded |

## Cấu hình Production

### Controller HA (3 nodes)

```
Controller Node (x3):
- CPU: 2 x Intel Xeon 16-core (hoặc AMD EPYC)
- RAM: 64-128 GB ECC
- Storage:
  - 2 x 480 GB SSD (RAID-1) cho OS
  - 2 x 1.6 TB NVMe (MariaDB, RabbitMQ)
- NIC: 2 x 25GbE (bonded/LACP)
- OOB: iDRAC/iLO/BMC
```

### Compute Node

```
Compute Node:
- CPU: 2 x AMD EPYC 7742 (128 cores total)
- RAM: 512 GB - 1 TB ECC
- Storage: 2 x 480 GB SSD (RAID-1) cho OS
- NIC: 2 x 25GbE + 2 x 10GbE (storage)
- OOB: iDRAC/iLO
```

> [!tip] CPU overcommit ratio
> Nova mặc định CPU overcommit ratio = **16:1**, RAM = **1.5:1**
> Ví dụ: 128 physical cores → 2048 vCPUs có thể cấp phát

### Ceph OSD Node

```
Ceph Node:
- CPU: 1 x 16-core (Ceph không cần nhiều CPU)
- RAM: 128 GB (2 GB/OSD minimum, 4-8 GB recommended)
- OSD: 12 x 4TB NVMe (raw) hoặc 24 x HDD + NVMe WAL
- NIC: 2 x 25GbE public + 2 x 25GbE cluster (replication)
```

## Network Interface Planning

### Physical NICs và bonding

```
┌─────────────────────────────────────────┐
│            Server                       │
│                                         │
│  eth0 ──┐                              │
│          ├── bond0 ── Management/API    │
│  eth1 ──┘                              │
│                                         │
│  eth2 ──┐                              │
│          ├── bond1 ── Storage (Ceph)    │
│  eth3 ──┘                              │
│                                         │
│  eth4 ──┐                              │
│          ├── bond2 ── Tunnel/Overlay    │
│  eth5 ──┘             + Provider VLAN  │
└─────────────────────────────────────────┘
```

### BIOS Settings quan trọng

| Setting | Value | Lý do |
|---------|-------|-------|
| Virtualization (VT-x/AMD-V) | **Enable** | KVM hypervisor |
| SR-IOV | Enable (nếu dùng) | High-perf networking |
| IOMMU | Enable | SR-IOV, GPU passthrough |
| Hyper-Threading | Tùy | Trade-off perf vs security |
| NUMA | Enable | Performance |
| Turbo Boost | Enable | Performance |
| Power Mode | Max Performance | Tránh CPU throttling |
| C-states | Disable hoặc C1 | Low latency workloads |

## Storage Sizing

### MariaDB (trên Controller)

Ước tính growth theo số VM:
- Mỗi VM/network/port tạo ra nhiều DB records
- 1000 VMs ≈ 50-100 GB database size
- Tăng trưởng log ≈ 10-20 GB/ngày (cần rotation)

### Glance Image Store

- Mỗi OS image: 1-20 GB
- Ước tính: 10-20 images × 10 GB = 100-200 GB
- Nên dùng Ceph RBD hoặc Swift làm backend thay vì local filesystem

### Nova Instance Storage

- Ephemeral disk: lưu trên Compute node
- Cinder volume: lưu trên Ceph (khuyến nghị)
- Live migration dễ hơn khi dùng shared storage (Ceph)

## IPMI / Out-of-Band Management

> [!important] Bắt buộc có IPMI/BMC
> Production OpenStack cần có OOB (iDRAC, iLO, IPMI) để:
> - Reboot server khi OS treo
> - Sử dụng Ironic cho bare metal provisioning
> - Remote console khi mất network

---
*Xem thêm: [[Deployment Models]] | [[Installation Methods]] | [[Cloud/AWS/SAA/Storage/Storage Overview]] | [[Ceph Integration]]*
