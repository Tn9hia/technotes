---
tags:
  - openstack
  - neutron
  - networking
  - vlan
  - vxlan
---

# Provider & Tenant Networks

## Provider Network

**Provider network** do admin tạo, ánh xạ trực tiếp với physical network.

### Flat Network

```bash
# Không có VLAN tag, tất cả traffic trên cùng L2
openstack network create \
  --provider-network-type flat \
  --provider-physical-network physnet1 \
  --external \
  flat-external
```

### VLAN Network

```bash
# Dùng VLAN tag 100
openstack network create \
  --provider-network-type vlan \
  --provider-physical-network physnet1 \
  --provider-segment 100 \
  --external \
  provider-vlan100

openstack subnet create \
  --network provider-vlan100 \
  --subnet-range 203.0.113.0/24 \
  --allocation-pool start=203.0.113.100,end=203.0.113.200 \
  --gateway 203.0.113.1 \
  --no-dhcp \
  provider-vlan100-subnet
```

## Tenant (Project) Network

User tự tạo, isolated. Thường dùng **overlay** (VXLAN/GRE).

### VXLAN Network

```bash
# VXLAN: UDP port 4789, VNI 24-bit (16M segments)
openstack network create \
  --provider-network-type vxlan \
  my-tenant-net

openstack subnet create \
  --network my-tenant-net \
  --subnet-range 192.168.1.0/24 \
  --dns-nameserver 8.8.8.8 \
  my-tenant-subnet
```

## Kết nối Provider ↔ Tenant

```bash
# Router kết nối tenant network ra provider
openstack router create my-router
openstack router set --external-gateway provider-vlan100 my-router
openstack router add subnet my-router my-tenant-subnet

# Bây giờ VMs trong tenant net có thể floating IP từ provider-vlan100
```

```
Internet
   │
Provider VLAN100 (203.0.113.0/24)
   │
Virtual Router (Floating IP NAT)
   │
Tenant VXLAN Network (192.168.1.0/24)
   │
VMs
```

## ML2 Physical Network Mapping

Cấu hình mapping giữa **physnet name** và **actual NIC/bridge**:

```ini
# ml2_conf.ini (OVS plugin)
[ovs]
bridge_mappings = physnet1:br-ex,physnet2:br-storage

# physnet1 map vào bridge br-ex (kết nối với external NIC)
# physnet2 map vào bridge br-storage
```

```ini
# ml2_conf.ini (OVN)
[ovn]
bridge_mappings = physnet1:br-ex
```

## VLAN Ranges

```ini
# ml2_conf.ini
[ml2_type_vlan]
network_vlan_ranges = physnet1:100:200,physnet2:300:400

# physnet1: dùng VLAN 100-200 cho tenant networks
# physnet2: dùng VLAN 300-400

[ml2_type_vxlan]
vni_ranges = 10:10000
# VNI từ 10 đến 10000 cho VXLAN tenant networks
```

## Network Topology ví dụ Production

```
Internet
    │
TOR Switch (VLAN 10: external, VLAN 20-100: provider VLANs)
    │
   br-ex (OVS bridge, trunk mode)
    │
   Neutron L3 agent (qrouter-xxx namespace)
    │  ← Floating IP NAT
   br-int (integration bridge)
    │
   br-tun (VXLAN tunnels)
    │
Physical NIC (overlay network, VLAN 200)
    │
TOR Switch → other compute nodes
```

## Shared Networks

```bash
# Network shared giữa các projects
openstack network create \
  --share \
  shared-internal-net

# Chỉ admin có thể tạo shared network
```

## External Networks và Floating IPs Pool

```bash
# Xem available floating IPs
openstack floating ip list --status DOWN  # DOWN = unassigned

# Xem network pool
openstack subnet show provider-vlan100-subnet | grep allocation
```

## Troubleshooting

```bash
# VM không lấy được DHCP
# 1. Kiểm tra DHCP agent
openstack network agent list | grep dhcp
# 2. Vào namespace kiểm tra
ip netns exec qdhcp-<net-id> ss -lnup | grep 67
# 3. Xem log dnsmasq
ip netns exec qdhcp-<net-id> journalctl | grep dnsmasq

# VM không thể ping ra ngoài
# 1. Kiểm tra router
openstack router show my-router
# 2. Kiểm tra iptables trong qrouter namespace
ip netns exec qrouter-<id> iptables -t nat -L -n -v | grep MASQUERADE
# 3. Kiểm tra security group có cho phép ICMP không
openstack security group rule list | grep icmp
```

---
*Xem thêm: [[Neutron Architecture]] | [[Security Groups & Floating IP]] | [[OVS & OVN]] | [[Neutron - Networking]]*
