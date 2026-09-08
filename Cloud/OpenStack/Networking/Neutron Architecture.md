---
tags:
  - openstack
  - neutron
  - networking
  - architecture
---

# Neutron Architecture (Deep Dive)

## Plugin Architecture

```
neutron-server
      │
   ML2 Plugin (Modular Layer 2)
      │
   ┌──┴──────────────┐
   │                 │
Type Drivers    Mechanism Drivers
(network type)  (implementation)
   ├── flat        ├── OVS (openvswitch)
   ├── vlan        ├── OVN
   ├── vxlan       ├── linuxbridge
   └── gre         ├── sriov
                   └── macvtap
```

## Data Flow: VM gửi packet

### Với OVS backend

```
VM (tap interface)
  │
  ▼
br-int (Integration Bridge) ← OVS internal bridge
  │   ← security group rules (OVS flow rules)
  ▼
br-tun (Tunnel Bridge) ← VXLAN/GRE encap/decap
  │
  ▼
Physical NIC (eth0)
  │
  ▼
Physical Network (VXLAN tunnel to other compute)
```

### Bridge layout trên Compute node

```bash
# Xem bridges
ovs-vsctl show

# Thường thấy:
# br-int    - tất cả VM ports kết nối vào đây
# br-tun    - VXLAN/GRE tunneling
# br-ex     - external/provider network (nếu có)

# Xem flow rules
ovs-ofctl dump-flows br-int
ovs-ofctl dump-flows br-tun
```

## OVS Flow Processing

```
OVS bridge nhận packet
  │
  ▼
Flow table 0 (input classification)
  │
  ▼
Flow table 1+ (L2/L3 processing)
  │
  ▼
Flow table N (output action: forward/drop/rewrite)
```

```bash
# Xem flow tables
ovs-ofctl dump-flows br-int table=0

# Xem port list và port numbers
ovs-ofctl show br-int
```

## DHCP Agent

Mỗi **tenant network subnet** với DHCP enabled → Neutron tạo:
- 1 network namespace: `qdhcp-<network-id>`
- 1 dnsmasq process trong namespace đó

```bash
# Xem DHCP namespaces
ip netns | grep qdhcp

# Xem dnsmasq process
ip netns exec qdhcp-<id> ps aux | grep dnsmasq

# Xem DHCP lease
ip netns exec qdhcp-<id> cat /var/lib/neutron/dhcp/<network-id>/leases
```

## L3 Agent (Router)

Mỗi **virtual router** → L3 agent tạo:
- 1 network namespace: `qrouter-<router-id>`
- iptables rules cho NAT (floating IP)
- keepalived cho VRRP (HA router)

```bash
# Xem router namespaces
ip netns | grep qrouter

# Xem routes trong router
ip netns exec qrouter-<id> ip route

# Xem iptables NAT rules (Floating IP mapping)
ip netns exec qrouter-<id> iptables -t nat -L -n -v

# Ping test từ router
ip netns exec qrouter-<id> ping 8.8.8.8
```

## Metadata Service

VM cần truy cập `169.254.169.254` để lấy user-data, SSH key từ Nova:

```
VM ──► 169.254.169.254:80
         │ (neutron intercept)
         ▼
    qrouter namespace
         │ (forward to metadata agent)
         ▼
    neutron-metadata-agent
         │ (add X-Forwarded-For, X-Instance-ID)
         ▼
    nova-api (metadata endpoint)
```

## HA Router (VRRP)

```bash
# Enable HA router
openstack router create --ha my-ha-router

# L3 agent sẽ tạo keepalived cho VRRP
# Active router giữ VIP, standby sẵn sàng take over
ip netns exec qrouter-<id> ip addr  # xem VIP
```

## Network Node vs DVR

### Legacy (Network Node)
- Tất cả router và DHCP chạy trên **Network node**
- Bottleneck: mọi east-west traffic phải qua network node

### DVR (Distributed Virtual Router)
- L3 agent **trên mỗi compute node** xử lý router cho VMs trên node đó
- North-south (floating IP) vẫn qua Network node (hoặc distributed với DVR SNAT)
- East-west **không** qua network node → tốt hơn nhiều

```ini
# neutron.conf (controller)
[DEFAULT]
router_distributed = True

# l3_agent.ini (compute nodes)
[DEFAULT]
agent_mode = dvr
```

## OVN (Open Virtual Network) — Thế hệ mới

OVN thay thế OVS agents + L3 agent + DHCP agent:

```
Neutron Server
     │
OVN ML2 driver
     │
OVN Northbound DB (logical network topology)
     │
ovn-northd (translation daemon)
     │
OVN Southbound DB (physical bindings)
     │
ovn-controller (trên mỗi host)
     │
OVS dataplane
```

```bash
# OVN commands
ovn-nbctl show
ovn-sbctl show
ovn-nbctl ls-list  # logical switches
ovn-nbctl lr-list  # logical routers
```

---
*Xem thêm: [[Neutron - Networking]] | [[Provider & Tenant Networks]] | [[OVS & OVN]] | [[Security Groups & Floating IP]]*
