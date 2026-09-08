---
tags:
  - openstack
  - deployment
---

# Deployment Models

[[OpenStack]] hỗ trợ nhiều mô hình triển khai tùy theo quy mô và yêu cầu.

## 1. All-in-One (AIO)

Tất cả services chạy trên **1 node duy nhất**.

```
┌─────────────────────────────────┐
│           AIO Node              │
│  Controller + Compute + Storage │
│                                 │
│  Keystone, Nova, Neutron,       │
│  Glance, Cinder, Horizon...     │
└─────────────────────────────────┘
```

**Dùng cho**: Dev/test, POC, lab environment
**Không dùng cho**: Production
**Tool**: Devstack, Packstack (deprecated)

## 2. Multi-Node (Standard)

Tách biệt các vai trò thành nhiều node.

```
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│  Controller Node  │   │  Compute Node(s) │   │  Storage Node(s) │
│                  │   │                  │   │                  │
│ Keystone         │   │ Nova-compute     │   │ Cinder-volume    │
│ Nova-API         │   │ Neutron-agent    │   │ Swift            │
│ Glance           │   │ Libvirt/KVM      │   │ Ceph OSD         │
│ Neutron-server   │   │                  │   │                  │
│ Cinder-API       │   │                  │   │                  │
│ Horizon          │   │                  │   │                  │
│ RabbitMQ         │   │                  │   │                  │
│ MariaDB          │   │                  │   │                  │
└──────────────────┘   └──────────────────┘   └──────────────────┘
```

**Tối thiểu**: 1 Controller + 1 Compute + 1 Storage
**Thực tế**: 1 Controller + 3+ Compute + 3+ Storage

## 3. High Availability (HA) — Production Standard

Controller được replicate để tránh SPOF (Single Point of Failure).

```
        ┌─────────────┐
        │ Load Balancer│  (HAProxy)
        │ VIP: 10.0.0.1│
        └──────┬──────┘
               │
    ┌──────────┼──────────┐
    │          │          │
┌───▼───┐  ┌──▼───┐  ┌───▼──┐
│ Ctrl1 │  │ Ctrl2│  │ Ctrl3│  ← Pacemaker/Corosync cluster
└───────┘  └──────┘  └──────┘
    │          │          │
    └──────────┼──────────┘
               │
    ┌──────────┼──────────┐
    │          │          │
┌───▼───┐  ┌──▼───┐  ┌───▼──┐
│Compute│  │Compute│  │Compute│  ← Scale out theo nhu cầu
└───────┘  └───────┘  └───────┘
    │          │          │
    └──────────┼──────────┘
               │
    ┌──────────┼──────────┐
    │          │          │
┌───▼───┐  ┌──▼───┐  ┌───▼──┐
│ Ceph  │  │ Ceph │  │ Ceph │  ← Minimum 3 OSD nodes
│ OSD1  │  │ OSD2 │  │ OSD3 │
└───────┘  └──────┘  └───────┘
```

### HA Components

| Component | HA Method |
|-----------|-----------|
| API Services | HAProxy + Active/Active |
| MariaDB | Galera multi-master cluster |
| RabbitMQ | Mirrored queues / Quorum queues |
| Memcached | Multiple instances |
| Neutron agents | VRRP (keepalived) cho L3 agent |
| Cinder | Active/Passive (Pacemaker) |

## 4. Hyperconverged (HCI)

Compute và Storage chạy cùng node. Phổ biến với Ceph.

```
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│   HCI Node 1        │   │   HCI Node 2        │   │   HCI Node 3        │
│                     │   │                     │   │                     │
│ Nova-compute (KVM)  │   │ Nova-compute (KVM)  │   │ Nova-compute (KVM)  │
│ Ceph OSD            │   │ Ceph OSD            │   │ Ceph OSD            │
│ Ceph MON            │   │ Ceph MON            │   │ Ceph MON            │
└─────────────────────┘   └─────────────────────┘   └─────────────────────┘
```

**Ưu điểm**: Giảm số server, đơn giản hóa cabling
**Nhược điểm**: Storage và Compute cạnh tranh tài nguyên

## 5. Edge / Distributed

OpenStack chạy ở edge locations kết nối về central cloud.

```
    Central Cloud
    ┌──────────────┐
    │  Full OpenStack│
    └──────┬───────┘
           │
    ┌──────┴───────┐
    │              │
┌───▼───┐      ┌───▼───┐
│ Edge1 │      │ Edge2 │
│ Starlingx│   │ Starlingx│
└───────┘      └───────┘
```

**Use case**: Telecom (NFV), IoT, Low-latency applications

## Network Topology cho Deployment

### Các loại network cần thiết

| Network | Mục đích | VLAN ví dụ |
|---------|----------|-----------|
| Management/API | OpenStack API, SSH | VLAN 10 |
| Internal/Tunnel | VM-to-VM overlay (VXLAN) | VLAN 20 |
| Storage | Ceph replication, iSCSI | VLAN 30 |
| External/Provider | Floating IP, external access | VLAN 40/trunk |
| IPMI/BMC | Out-of-band management | VLAN 50 |

> [!warning] Tách storage network
> Luôn tách **Storage network** riêng. Ceph replication traffic cực kỳ lớn và sẽ ảnh hưởng các traffic khác nếu chung network.

---
*Xem thêm: [[Hardware Requirements]] | [[Installation Methods]] | [[HA Architecture]] | [[OpenStack]]*
