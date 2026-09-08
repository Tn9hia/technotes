---
tags:
  - openstack
  - prerequisites
---

# Prerequisites — Kiến thức cần có trước OpenStack

> [!abstract] Tổng quan
> OpenStack là hệ thống phức tạp, kết hợp nhiều công nghệ. Để vận hành tốt, cần nắm vững các nền tảng dưới đây trước khi đi sâu vào từng service.

## Linux & System

### Must-have
- **Linux administration**: systemd, journald, cron, user/group, file permissions
- **Process management**: ps, top, kill, lsof, strace cơ bản
- **Network tools**: ip, ss, tcpdump, iptables, nftables
- **Package management**: apt (Ubuntu/Debian), dnf (RHEL/CentOS)
- **Shell scripting**: bash cơ bản để viết automation scripts

### Storage
- LVM (Logical Volume Manager) — Cinder sử dụng LVM driver mặc định
- NFS, iSCSI cơ bản
- Filesystem: ext4, xfs

### Virtualization
- **KVM/QEMU**: hypervisor mà Nova sử dụng
- **Libvirt**: API quản lý VM
- **virsh** commands: `virsh list`, `virsh console`, `virsh dumpxml`
- **CPU virtualization**: VT-x/AMD-V, nested virtualization

```bash
# Kiểm tra hỗ trợ virtualization
egrep -c '(vmx|svm)' /proc/cpuinfo
# Kiểm tra kvm module
lsmod | grep kvm
```

## Networking

### L2 Fundamentals
- VLAN, 802.1Q trunking
- Linux bridge, bond, team
- **Open vSwitch (OVS)**: ovs-vsctl, ovs-ofctl — Neutron dùng OVS làm dataplane

### L3 Fundamentals
- IP routing, static route, default gateway
- NAT (SNAT, DNAT) — Floating IP dùng DNAT
- **iptables/nftables**: security groups được implement qua iptables
- Network namespaces — Neutron dùng network namespace cho mỗi router/DHCP

### Overlay Networking
- **VXLAN**: UDP-based tunnel, VNI, VTEP — Neutron tenant network
- **GRE**: IP tunnel — phổ biến hơn trong cấu hình cũ
- BGP cơ bản (nếu dùng Calico/BGP route)

### DNS & DHCP
- dnsmasq — DHCP agent của Neutron dùng dnsmasq
- bind9 cơ bản

## Middleware & Infrastructure

### Message Queue
- **RabbitMQ**: tất cả OpenStack services dùng AMQP để giao tiếp async
  - Exchange, Queue, Binding, Routing Key
  - `rabbitmqctl` commands cơ bản

### Database
- **MariaDB/MySQL**: hầu hết services lưu state vào MySQL
  - SQL queries cơ bản (SELECT, JOIN, EXPLAIN)
  - Replication, backup với mysqldump
- **Galera Cluster**: multi-master replication cho HA

### Caching
- **Memcached**: Keystone token caching, session caching

### HTTP & API
- REST API concepts (GET, POST, PUT, DELETE, status codes)
- **curl** để test API
- JSON format
- TLS/SSL cơ bản

## Container & Automation

### Docker/Podman
- Cần thiết nếu deploy bằng **Kolla-Ansible** (container-based deployment)

### Ansible
- **Kolla-Ansible** là phương pháp deploy phổ biến nhất
- Cần biết: inventory, playbook, role, variable, vault

### Python
- OpenStack viết bằng Python 3
- Cần để đọc config, debug, viết script tương tác với SDK

## Monitoring Stack

- **Prometheus + Grafana**: monitoring cơ bản
- **Elasticsearch/OpenSearch**: log aggregation (ELK stack)
- Log format: biết đọc OpenStack log với request-id tracking

## Roadmap học

```
Linux fundamentals
    ↓
Networking (L2/L3/Overlay)
    ↓
KVM/Libvirt virtualization
    ↓
RabbitMQ + MariaDB basics
    ↓
OpenStack components → [[OpenStack]]
    ↓
[[Deployment/Deployment Models]]
    ↓
[[Operations/CLI Commands]]
    ↓
[[Operations/Troubleshooting]]
```

---
*Xem thêm: [[OpenStack]]*
