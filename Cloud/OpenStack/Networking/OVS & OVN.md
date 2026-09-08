---
tags:
  - openstack
  - neutron
  - ovs
  - ovn
  - networking
---

# OVS & OVN

## Open vSwitch (OVS)

OVS là **software switch** chạy trong Linux kernel, làm dataplane cho Neutron.

### Kiến trúc OVS

```
ovs-vswitchd (userspace daemon)
      │
ovsdb-server (configuration database)
      │
OVS kernel module (fast path)
```

### Cấu trúc bridge trong OpenStack

```bash
# Xem toàn bộ OVS config
ovs-vsctl show

# Output thường thấy:
Bridge br-int          # Integration bridge
    Port patch-tun     # Kết nối với br-tun
    Port tap-vm1       # VM's tap interface
    Port tap-vm2

Bridge br-tun          # Tunnel bridge
    Port patch-int     # Kết nối với br-int
    Port vxlan-10.0.0.2  # VXLAN tunnel đến Compute 2

Bridge br-ex           # External bridge (provider network)
    Port eth1          # Physical NIC
    Port patch-int-ex
```

### OVS Commands

```bash
# Bridge management
ovs-vsctl add-br br-test
ovs-vsctl del-br br-test
ovs-vsctl list-br

# Port management
ovs-vsctl add-port br-int eth0
ovs-vsctl del-port br-int eth0
ovs-vsctl list-ports br-int

# Flow rules
ovs-ofctl dump-flows br-int
ovs-ofctl dump-flows br-int table=0
ovs-ofctl add-flow br-int "priority=100,in_port=1,actions=output:2"
ovs-ofctl del-flows br-int "in_port=1"

# Statistics
ovs-ofctl dump-ports br-int

# OVSDB
ovs-vsctl list Interface
ovs-vsctl list Port
ovs-vsctl list Bridge
ovs-vsctl get Interface eth0 ofport  # OpenFlow port number

# Packet tracing (debug)
ovs-appctl ofproto/trace br-int in_port=1,tcp,nw_dst=10.0.0.1,tcp_dst=80
```

### Connection Tracking (Conntrack)

OVS dùng Linux conntrack cho Security Groups:

```bash
# Xem conntrack table
conntrack -L | grep "10.0.1.10"

# Flow rules dùng conntrack
# ct_state=+new+trk → packet mới, chưa có trong table
# ct_state=+est+trk → established connection
ovs-ofctl dump-flows br-int | grep ct_state
```

## OVN (Open Virtual Network)

OVN là **control plane** phía trên OVS, cung cấp L2/L3 networking cho cloud:

```
Neutron (OVN driver)
      │
OVN Northbound DB (logical topology)
  ├── Logical Switches
  ├── Logical Routers
  ├── Load Balancers
  └── ACLs
      │
ovn-northd (northd daemon — translate NB→SB)
      │
OVN Southbound DB (physical bindings)
  ├── Datapath bindings
  ├── Port bindings
  └── Flow pipeline
      │
ovn-controller (trên mỗi host)
      │
OVS (actual dataplane)
```

### OVN vs OVS Agent (legacy)

| Tiêu chí | OVS Agent + L3 Agent | OVN |
|---------|---------------------|-----|
| L2 | OVS agent | ovn-controller |
| L3 (routing) | neutron-l3-agent | OVN native |
| DHCP | neutron-dhcp-agent | OVN native |
| Security Groups | iptables/conntrack | OVN ACL |
| Scalability | Kém hơn | Tốt hơn |
| Debugging | Phức tạp | Có tools tốt hơn |

### OVN Commands

```bash
# Northbound (logical view)
ovn-nbctl show
ovn-nbctl ls-list                    # Logical switches (networks)
ovn-nbctl lr-list                    # Logical routers
ovn-nbctl lsp-list <switch>          # Logical switch ports
ovn-nbctl acl-list <switch>          # ACL rules (security groups)
ovn-nbctl lb-list                    # Load balancers

# Southbound (physical view)
ovn-sbctl show
ovn-sbctl list chassis               # Physical hosts
ovn-sbctl lflow-list                 # Logical flows (từ northd)
ovn-sbctl dump-flows                 # Actual OVS flows

# Trace packet
ovn-trace <datapath> '<packet>'
ovn-trace neutron-<network-id> \
  'inport=="<port-id>",eth.dst==<mac>,ip4.dst==10.0.0.1'
```

### OVN Debugging workflow

```bash
# 1. Tìm logical port của VM
ovn-nbctl show | grep <vm-ip>

# 2. Xem ACL rules cho network
SWITCH=$(ovn-nbctl ls-list | grep <network-id>)
ovn-nbctl acl-list $SWITCH

# 3. Trace packet qua OVN
ovn-trace $SWITCH "inport==\"<port-id>\",ip4,ip4.src=10.0.1.10,ip4.dst=8.8.8.8,tcp,tcp.dst=80"

# 4. Xem physical flows trên host cụ thể
ovn-sbctl lflow-list | grep <datapath>
```

## So sánh và chọn backend

| Scenario | Recommendation |
|---------|----------------|
| Cluster mới, OpenStack 2023+ | **OVN** |
| Cluster cũ đang chạy | Giữ **OVS Agent** |
| Cần DVR | OVS DVR hoặc OVN (native distributed) |
| Cần BGP routing | OVN với OVN-BGP agent |
| Debug quen | OVS Agent (dễ debug iptables hơn) |

> [!warning] Migration OVS → OVN
> Migration từ OVS Agent sang OVN yêu cầu downtime ngắn và cần test kỹ trong lab trước.

---
*Xem thêm: [[Neutron Architecture]] | [[Provider & Tenant Networks]] | [[Security Groups & Floating IP]]*
