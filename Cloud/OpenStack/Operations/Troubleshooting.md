---
tags:
  - openstack
  - troubleshooting
  - operations
  - debug
---

# Troubleshooting

## Log Files Reference

| Service | Log Path |
|---------|---------|
| Keystone | `/var/log/keystone/keystone.log` |
| Nova API | `/var/log/nova/nova-api.log` |
| Nova Compute | `/var/log/nova/nova-compute.log` |
| Nova Scheduler | `/var/log/nova/nova-scheduler.log` |
| Neutron Server | `/var/log/neutron/neutron-server.log` |
| Neutron OVS Agent | `/var/log/neutron/openvswitch-agent.log` |
| Neutron L3 Agent | `/var/log/neutron/l3-agent.log` |
| Neutron DHCP Agent | `/var/log/neutron/dhcp-agent.log` |
| Glance | `/var/log/glance/api.log` |
| Cinder API | `/var/log/cinder/cinder-api.log` |
| Cinder Volume | `/var/log/cinder/cinder-volume.log` |
| Heat | `/var/log/heat/heat-api.log`, `heat-engine.log` |

> [!tip] Kolla container logs
> Với Kolla-Ansible, dùng `docker logs <container-name>` hoặc `/var/log/<service>/`:
> ```bash
> docker logs nova_compute --tail 100 -f
> docker logs neutron_server --tail 100 -f
> ```

## Tìm request theo ID

Mỗi OpenStack API request có unique `req-<uuid>`. Dùng để trace qua nhiều services:

```bash
# Lấy request ID từ CLI
openstack --debug server create ... 2>&1 | grep "req-"

# Tìm request trong log
grep "req-abc123" /var/log/nova/nova-api.log
grep "req-abc123" /var/log/nova/nova-compute.log
```

## VM bị stuck "Building" hoặc không boot

```bash
# 1. Xem VM status và fault
openstack server show <vm-id>
openstack server show <vm-id> | grep -i fault

# 2. Xem log của nova-compute trên host được schedule
openstack server show <vm-id> | grep "OS-EXT-SRV-ATTR:host"
ssh compute-01
tail -200 /var/log/nova/nova-compute.log | grep <vm-id>

# 3. Xem virsh (libvirt)
virsh list --all | grep instance
virsh dominfo instance-<id>
virsh console instance-<id>

# 4. Xem console log của VM
openstack console log show <vm-id>

# 5. Xem scheduler log (tại sao schedule vào host nào)
grep <vm-id> /var/log/nova/nova-scheduler.log
```

## VM Error State

```bash
# Xem lý do error
openstack server show <vm-id> -f json | jq '.fault'

# Hard reboot
openstack server reboot --hard <vm-id>

# Nếu vẫn error, reset state
openstack server set --state active <vm-id>

# Rebuild VM (giữ volume, thay image)
openstack server rebuild --image ubuntu-22.04 <vm-id>
```

## Network Issues

### VM không lấy được IP

```bash
# 1. Kiểm tra DHCP agent
openstack network agent list | grep dhcp
# Status phải là UP

# 2. Xem DHCP namespace
ip netns | grep qdhcp
ip netns exec qdhcp-<net-id> ss -lnup | grep :67

# 3. Xem lease file
ip netns exec qdhcp-<net-id> cat /var/lib/neutron/dhcp/<net-id>/leases

# 4. Xem dnsmasq log
ip netns exec qdhcp-<net-id> journalctl | grep dnsmasq
```

### VM không ping ra ngoài

```bash
# 1. Kiểm tra security group có allow ICMP không
openstack security group rule list <sg-name> | grep icmp

# 2. Kiểm tra router
openstack router show my-router | grep external_gateway

# 3. Vào router namespace test
ip netns exec qrouter-<id> ping 8.8.8.8

# 4. Xem iptables NAT
ip netns exec qrouter-<id> iptables -t nat -L -n -v

# 5. Xem L3 agent log
tail -50 /var/log/neutron/l3-agent.log
```

### Floating IP không hoạt động

```bash
# 1. Kiểm tra association
openstack floating ip show <fip>

# 2. Xem iptables DNAT trong qrouter
ip netns exec qrouter-<id> iptables -t nat -L -n -v | grep <private-ip>

# 3. Kiểm tra ARP trên provider network
ip netns exec qrouter-<id> arping -c 3 -I <interface> <gateway-ip>
```

## Storage Issues

### Volume stuck "creating"

```bash
# 1. Xem Cinder volume log
tail -100 /var/log/cinder/cinder-volume.log | grep <vol-id>

# 2. Xem Ceph nếu dùng Ceph backend
ceph health detail
rbd ls -p volumes | grep <vol-id>

# 3. Reset state
openstack volume set --state error <vol-id>
openstack volume delete <vol-id>
```

### Volume attach fail

```bash
# 1. Kiểm tra multipath trên compute node
multipath -ll

# 2. Kiểm tra libvirt có kết nối Ceph không
virsh secret-list
rbd ls -p volumes  # chạy với user nova có quyền không?

# 3. Xem nova-compute log
grep "attach" /var/log/nova/nova-compute.log | tail -30
```

## Identity Issues

### 401 Unauthorized

```bash
# 1. Kiểm tra token còn hạn không
openstack token issue

# 2. Kiểm tra endpoint URL đúng không
openstack endpoint list --service identity

# 3. Sync Fernet keys
ls /etc/keystone/fernet-keys/

# 4. Kiểm tra memcached
echo "stats" | nc controller 11211 | head

# 5. Xem Keystone log
tail -50 /var/log/keystone/keystone.log
```

## RabbitMQ Issues

```bash
# Kiểm tra RabbitMQ cluster
rabbitmqctl cluster_status

# Xem queues
rabbitmqctl list_queues name messages consumers

# Xem connections
rabbitmqctl list_connections user vhost

# Purge stuck queue
rabbitmqctl purge_queue compute.ctrl1

# Restart RabbitMQ (Kolla)
docker restart rabbitmq
```

## MariaDB Issues

```bash
# Kiểm tra Galera
mysql -e "SHOW GLOBAL STATUS LIKE 'wsrep%';"

# Kiểm tra connections
mysql -e "SHOW PROCESSLIST;"
mysql -e "SHOW STATUS LIKE 'Threads_connected';"

# Xem slow queries
mysql -e "SHOW GLOBAL STATUS LIKE 'Slow_queries';"

# Kiểm tra DB size
mysql -e "SELECT table_schema, ROUND(SUM(data_length+index_length)/1024/1024,2) AS 'DB Size (MB)' FROM information_schema.tables GROUP BY table_schema;"
```

## Common Error Messages

| Error | Nguyên nhân | Fix |
|-------|-------------|-----|
| `No valid host was found` | Scheduler không tìm được host phù hợp | Xem filter, kiểm tra resource availability |
| `Quota exceeded` | Vượt quota | Tăng quota hoặc giải phóng resources |
| `Instance in wrong power state` | VM đang ở state không phù hợp | `openstack server set --state active` |
| `Authentication required` | Token hết hạn hoặc sai credential | Re-source openrc file |
| `Endpoint not found` | Service không đăng ký endpoint | `openstack endpoint list` kiểm tra |
| `Build of instance aborted` | Lỗi khi khởi tạo VM | Xem nova-compute log |

---
*Xem thêm: [[CLI Commands]] | [[Monitoring & Alerting]] | [[Day 2 Operations]] | [[OpenStack]]*
