---
tags:
  - openstack
  - ha
  - high-availability
  - architecture
---

# HA Architecture

## Overview

```
            ┌──────────────┐
            │   HAProxy    │  ← VIP (keepalived)
            │  10.0.0.10   │
            └──────┬───────┘
                   │
       ┌───────────┼───────────┐
       │           │           │
┌──────▼──┐  ┌────▼───┐  ┌────▼──┐
│ Ctrl-1  │  │ Ctrl-2 │  │Ctrl-3 │  ← Active/Active
│         │  │        │  │       │
│ - API   │  │ - API  │  │ - API │
│ - Sched │  │ - Sched│  │ -Sched│
│ - MySQL │◄─┤ - MySQL├─►│ -MySQL│  ← Galera Cluster
│ - MQ    │◄─┤ - MQ   ├─►│ - MQ  │  ← RabbitMQ Quorum
│ - Keystone│ │        │  │       │
└─────────┘  └────────┘  └───────┘
       │           │           │
       └───────────┼───────────┘
                   │
       ┌───────────┼───────────┐
       │           │           │
┌──────▼──┐  ┌────▼───┐  ┌────▼──┐
│Compute-1│  │Compute2│  │Compu-N│  ← Scale out
└─────────┘  └────────┘  └───────┘
       │           │           │
       └───────────┼───────────┘
                   │
             Ceph Cluster (3+ nodes)
```

## HAProxy

HAProxy làm **reverse proxy và load balancer** cho OpenStack APIs:

```
frontend keystone_public
    bind *:5000
    default_backend keystone_back

backend keystone_back
    balance leastconn
    option httpchk GET /v3
    server ctrl1 10.0.0.11:5000 check inter 2000
    server ctrl2 10.0.0.12:5000 check inter 2000
    server ctrl3 10.0.0.13:5000 check inter 2000
```

```bash
# Xem HAProxy stats
curl http://controller:1936/stats
# hoặc web UI
```

## Keepalived (VIP Management)

```ini
# /etc/keepalived/keepalived.conf (trên 1 controller)
vrrp_instance VI_1 {
    state MASTER       # MASTER trên ctrl1, BACKUP trên ctrl2, ctrl3
    interface bond0
    virtual_router_id 51
    priority 150       # Cao nhất trên ctrl1
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass openstack
    }
    virtual_ipaddress {
        10.0.0.10/24   # VIP
    }
}
```

## MariaDB Galera Cluster

3 MariaDB nodes chạy **synchronous multi-master replication**:

```ini
# /etc/mysql/mariadb.conf.d/galera.cnf
[mysqld]
binlog_format = ROW
default-storage-engine = innodb
innodb_autoinc_lock_mode = 2
bind-address = 0.0.0.0

# Galera Provider Configuration
wsrep_on = ON
wsrep_provider = /usr/lib/galera/libgalera_smm.so

# Galera Cluster Configuration
wsrep_cluster_name = "openstack_cluster"
wsrep_cluster_address = "gcomm://ctrl1,ctrl2,ctrl3"

wsrep_sst_method = rsync
wsrep_node_address = "10.0.0.11"  # IP của node này
wsrep_node_name = "ctrl1"
```

```bash
# Khởi tạo cluster (chỉ chạy lần đầu trên node đầu tiên)
galera_new_cluster

# Các node khác join
systemctl start mariadb

# Kiểm tra cluster status
mysql -e "SHOW STATUS LIKE 'wsrep%'"
mysql -e "SHOW GLOBAL STATUS LIKE 'wsrep_cluster_size'"  # phải là 3
```

> [!warning] Galera: không split-brain
> Với 3 nodes, nếu 2 nodes mất kết nối với 1 node, 2 nodes tiếp tục hoạt động. Node bị cô lập sẽ tự dừng write. Đây là quorum mechanism.

## RabbitMQ Quorum Queues

```bash
# Tạo cluster RabbitMQ
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@ctrl1
rabbitmqctl start_app

# Kiểm tra cluster
rabbitmqctl cluster_status

# Enable quorum queues (khuyến nghị thay mirror queues)
rabbitmqctl set_policy ha-all "^" \
  '{"ha-mode":"all","ha-sync-mode":"automatic"}' \
  --apply-to queues
```

## Compute HA (Nova)

### Evacuation (VM HA)

Khi compute node fail, admin phải **evacuate** VMs sang node khác:

```bash
# Kiểm tra compute services
openstack compute service list

# Disable failed host
openstack compute service set --disable ctrl1 nova-compute

# Evacuate tất cả VMs từ host fail
openstack server list --host failed-compute --all-projects
openstack host evacuate failed-compute

# Evacuate thủ công từng VM
openstack server evacuate <vm-id> --host healthy-compute
```

### Masakari — Automated VM HA

Masakari tự động evacuate khi compute node fail:

```bash
# Tạo failover segment
openstack segment create \
  --recovery-method auto \
  --name compute-segment

# Đăng ký hosts vào segment
openstack segment host create \
  --segment compute-segment \
  --type COMPUTE \
  --control-attributes name=compute-01 \
  compute-01
```

## Cinder HA

Cinder-volume cần **Active/Passive** HA (không active/active):

```bash
# Với Pacemaker (legacy)
pcs resource create cinder-volume systemd:openstack-cinder-volume \
  op monitor interval=60s

# Với Kolla: chạy cinder-volume chỉ trên 1 node, dùng VIP
```

## Neutron L3 HA (VRRP)

```bash
# Tạo HA router
openstack router create --ha my-ha-router

# L3 agent tạo VRRP giữa các L3 agents
# Một agent là MASTER, còn lại BACKUP
```

## Checklist HA

| Component | HA Method | Min Nodes |
|-----------|-----------|-----------|
| Keystone API | HAProxy + Active/Active | 3 |
| Nova API | HAProxy + Active/Active | 3 |
| Neutron API | HAProxy + Active/Active | 3 |
| MariaDB | Galera multi-master | 3 |
| RabbitMQ | Quorum queues | 3 |
| Memcached | Multiple instances | 3 |
| HAProxy | Keepalived VRRP | 2 |
| Cinder-volume | Active/Passive | 2 |
| Neutron L3 | VRRP per router | 2+ L3 agents |
| Ceph | Replication (3x) | 3 OSD nodes |

---
*Xem thêm: [[Pacemaker & Corosync]] | [[Database HA - Galera]] | [[Deployment Models]] | [[OpenStack]]*
