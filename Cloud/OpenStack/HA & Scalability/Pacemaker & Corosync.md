---
tags:
  - openstack
  - ha
  - pacemaker
  - corosync
  - cluster
---

# Pacemaker & Corosync

## Overview

- **Corosync**: Cluster communication layer (membership, messaging)
- **Pacemaker**: Cluster resource manager (start/stop/monitor resources)

Dùng trong OpenStack HA để quản lý:
- VIP (Virtual IP) — thay thế / bổ sung keepalived
- Cinder-volume (active/passive)
- Một số services không support active/active

## Kiến trúc

```
Pacemaker (CRM)
    │
Corosync (messaging)
    │
Cluster nodes (ctrl1, ctrl2, ctrl3)
```

## Cài đặt

```bash
# Ubuntu
apt install pacemaker corosync pcs fence-agents

# CentOS/RHEL
dnf install pacemaker corosync pcs fence-agents-all

# Start pcsd (Pacemaker/Corosync daemon)
systemctl enable --now pcsd
passwd hacluster  # Đặt password cho user hacluster
```

## Cấu hình Cluster

```bash
# Authenticate nodes
pcs host auth ctrl1 ctrl2 ctrl3 -u hacluster

# Tạo cluster
pcs cluster setup openstack-cluster ctrl1 ctrl2 ctrl3

# Start cluster
pcs cluster start --all
pcs cluster enable --all

# Kiểm tra
pcs status
pcs status nodes
```

## Resources

### VIP Resource

```bash
# Tạo VIP resource
pcs resource create vip_public \
  ocf:heartbeat:IPaddr2 \
  ip=10.0.0.10 \
  cidr_netmask=24 \
  nic=bond0 \
  op monitor interval=2s

# Xem resources
pcs resource show
pcs resource show vip_public
```

### HAProxy Resource

```bash
pcs resource create haproxy \
  systemd:haproxy \
  op monitor interval=60s

# Constraint: HAProxy phải chạy cùng node với VIP
pcs constraint colocation add haproxy with vip_public score=INFINITY
pcs constraint order vip_public then haproxy
```

### Cinder-volume Resource (Active/Passive)

```bash
pcs resource create cinder-volume \
  systemd:openstack-cinder-volume \
  op monitor interval=30s timeout=60s

# Chỉ chạy trên 1 node (không clone)
# Failover tự động khi node fail
```

## STONITH (Shoot The Other Node In The Head)

STONITH fencing đảm bảo node fail được tắt thật sự trước khi failover, tránh split-brain và data corruption:

```bash
# Fencing bằng IPMI
pcs stonith create ipmi-ctrl1 \
  fence_ipmilan \
  ipaddr=10.0.0.101 \
  login=admin \
  passwd=password \
  lanplus=1 \
  pcmk_host_list=ctrl1

pcs stonith create ipmi-ctrl2 \
  fence_ipmilan \
  ipaddr=10.0.0.102 \
  login=admin \
  passwd=password \
  lanplus=1 \
  pcmk_host_list=ctrl2

# Kiểm tra fencing
pcs stonith show
stonith_admin -T ipmi-ctrl1  # test fencing
```

> [!important] STONITH là bắt buộc trong production
> Không có STONITH, Pacemaker sẽ từ chối chạy resources (hoặc phải disable STONITH - KHÔNG khuyến nghị).
> `pcs property set stonith-enabled=false` chỉ dùng cho lab.

## Maintenance Mode

```bash
# Đưa node vào maintenance (không failover)
pcs node standby ctrl1

# Thực hiện maintenance trên ctrl1...

# Đưa node trở lại
pcs node unstandby ctrl1

# Maintenance toàn cluster
pcs property set maintenance-mode=true
# ... maintenance ...
pcs property set maintenance-mode=false
```

## Constraints

```bash
# Colocation: resource A phải cùng node với B
pcs constraint colocation add haproxy with vip_public score=INFINITY

# Order: A phải start trước B
pcs constraint order vip_public then haproxy

# Location: resource ưu tiên/tránh node cụ thể
pcs constraint location haproxy prefers ctrl1=100  # prefer ctrl1
pcs constraint location cinder-volume avoids ctrl3  # avoid ctrl3

# Xem tất cả constraints
pcs constraint show
```

## Troubleshooting

```bash
# Xem cluster status chi tiết
pcs status --full

# Xem lịch sử failover
pcs status history

# Xem log
journalctl -u corosync -u pacemaker -f

# Kiểm tra quorum
corosync-quorumtool

# Reset resource bị fail
pcs resource cleanup <resource-name>

# Move resource thủ công
pcs resource move haproxy ctrl2
pcs resource unmove haproxy  # remove constraint sau khi move
```

## Corosync Quorum

```ini
# /etc/corosync/corosync.conf
quorum {
    provider: corosync_votequorum
    two_node: 0
    expected_votes: 3
    # Cluster hoạt động khi có đa số: ≥ 2/3 nodes
}
```

---
*Xem thêm: [[HA Architecture]] | [[Database HA - Galera]] | [[OpenStack]]*
