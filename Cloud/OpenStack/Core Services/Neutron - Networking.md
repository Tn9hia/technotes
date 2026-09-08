---
tags:
  - openstack
  - neutron
  - networking
  - sdn
---

# Neutron — Networking Service

Neutron cung cấp **Network-as-a-Service** cho OpenStack. Tạo và quản lý virtual networks, subnets, routers, security groups, và floating IPs.

> [!abstract] Neutron vs Nova-network
> Trước đây OpenStack dùng nova-network (flat, simple). Neutron thay thế hoàn toàn, cung cấp multi-tenant SDN với full L2/L3 control.

## Kiến trúc tổng thể

```
                neutron-server (Controller)
                       │
              neutron-plugin (ML2)
             /           \
    L2 Agent              L3 Agent
    (OVS/OVN/LinuxBridge) (iptables/OVN)
    (trên mỗi Compute node) (trên Network node)
            │                    │
         DHCP Agent         Metadata Agent
```

### Components

| Component | Nơi chạy | Chức năng |
|-----------|----------|-----------|
| `neutron-server` | Controller | API server, plugin handler |
| `neutron-openvswitch-agent` | Compute/Network | L2 dataplane (OVS) |
| `neutron-l3-agent` | Network/Controller | Router, Floating IP, NAT |
| `neutron-dhcp-agent` | Network/Controller | DHCP cho VMs (dnsmasq) |
| `neutron-metadata-agent` | Network/Controller | Instance metadata proxy |

## ML2 Plugin (Modular Layer 2)

ML2 là plugin chính của Neutron, cho phép kết hợp nhiều loại backend.

```
ML2 Plugin
├── Type Drivers (loại network)
│   ├── flat
│   ├── vlan
│   ├── vxlan
│   └── gre
└── Mechanism Drivers (backend implementation)
    ├── openvswitch (OVS)
    ├── ovn (OVN - Open Virtual Network)
    ├── linuxbridge
    └── sriovnicswitch (SR-IOV)
```

## Network Types

### Provider Network (External)
- Do admin tạo, map vào physical network
- VM trong provider network nhận IP từ external DHCP hoặc neutron DHCP
- Dùng cho public/external access

### Tenant/Project Network (Internal/Overlay)
- User tự tạo, isolated
- Thường dùng VXLAN/GRE để overlay
- Cần router để kết nối với provider network

```bash
# Tạo provider network (admin)
openstack network create \
  --provider-network-type vlan \
  --provider-physical-network physnet1 \
  --provider-segment 100 \
  --external \
  provider-vlan100

# Tạo tenant network (user)
openstack network create my-net
openstack subnet create \
  --network my-net \
  --subnet-range 10.0.1.0/24 \
  --dns-nameserver 8.8.8.8 \
  my-subnet

# Tạo router và kết nối
openstack router create my-router
openstack router set --external-gateway provider-vlan100 my-router
openstack router add subnet my-router my-subnet
```

## Floating IP

Floating IP là **DNAT/SNAT** giữa external IP và VM's private IP.

```bash
# Tạo floating IP (từ provider network pool)
openstack floating ip create provider-vlan100

# Gán floating IP cho VM
openstack server add floating ip <vm-id> <floating-ip>

# Xem mapping
openstack floating ip list

# Gỡ floating IP
openstack server remove floating ip <vm-id> <floating-ip>
```

```
Internet ──► Floating IP (203.0.113.10)
                    │  DNAT (L3 agent)
              VM Private IP (10.0.1.5)
```

## Security Groups

Hoạt động như **stateful firewall** tại VM level (iptables/OVS flow rules).

```bash
# Tạo security group
openstack security group create web-sg

# Cho phép SSH
openstack security group rule create \
  --protocol tcp --dst-port 22 \
  --remote-ip 0.0.0.0/0 web-sg

# Cho phép HTTP/HTTPS
openstack security group rule create \
  --protocol tcp --dst-port 80 web-sg
openstack security group rule create \
  --protocol tcp --dst-port 443 web-sg

# Cho phép ICMP
openstack security group rule create \
  --protocol icmp web-sg

# Gán security group cho VM
openstack server add security group <vm-id> web-sg
```

## DVR (Distributed Virtual Router)

DVR phân tán L3 routing về từng Compute node, giảm tải Network node.

```
Without DVR:
VM1 (Compute1) ──► Network node (L3 agent) ──► Internet

With DVR:
VM1 (Compute1) ──► L3 agent trên Compute1 ──► Internet
```

```ini
# neutron.conf
[DEFAULT]
router_distributed = True
```

## OVN (Open Virtual Network) — Modern Backend

OVN là backend hiện đại thay thế OVS agent + L3 agent:
- Tích hợp native L2/L3 trong OVN
- Không cần L3 agent riêng
- Hiệu năng tốt hơn, ít components hơn

```ini
# ML2 config cho OVN
[ml2]
mechanism_drivers = ovn

[ovn]
ovn_nb_connection = tcp:10.0.0.10:6641
ovn_sb_connection = tcp:10.0.0.10:6642
```

## Network Namespaces

Neutron dùng Linux network namespaces để isolate:

```bash
# Trên Network node, xem các namespace
ip netns list
# qdhcp-<network-id>    ← DHCP namespace
# qrouter-<router-id>   ← Router namespace
# fip-<floatip>         ← Floating IP namespace (nếu không dùng DVR)

# Vào namespace để debug
ip netns exec qrouter-<id> ip addr
ip netns exec qrouter-<id> iptables -t nat -L
ip netns exec qdhcp-<id> ss -lnup
```

## Troubleshooting Network

```bash
# Kiểm tra agents
openstack network agent list

# Xem log neutron server
tail -f /var/log/neutron/neutron-server.log

# Trên compute node: xem OVS bridges
ovs-vsctl show
ovs-ofctl dump-flows br-int | head -20

# Ping test từ trong namespace
ip netns exec qrouter-<id> ping 8.8.8.8
```

---
*Deep dive: [[Neutron Architecture]] | [[Provider & Tenant Networks]] | [[Security Groups & Floating IP]] | [[OVS & OVN]]*
*Xem thêm: [[Nova - Compute]] | [[OpenStack]]*
