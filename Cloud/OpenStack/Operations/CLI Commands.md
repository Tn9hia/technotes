---
tags:
  - openstack
  - cli
  - operations
  - commands
---

# CLI Commands

## Setup

```bash
# Cài đặt openstack CLI
pip install python-openstackclient

# Source credentials
source /etc/kolla/admin-openrc.sh

# Hoặc set env vars thủ công
export OS_AUTH_URL=http://controller:5000/v3
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=secret
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2

# Kiểm tra
openstack token issue
openstack service list
```

## Nova — Compute

```bash
# Server (VM) management
openstack server list
openstack server list --all-projects  # admin only
openstack server list --host compute-01  # filter by host
openstack server show <vm-id>

# Tạo VM
openstack server create \
  --image ubuntu-22.04 \
  --flavor m1.medium \
  --network internal-net \
  --key-name mykey \
  --security-group web-sg \
  web-01

# Start/Stop/Reboot
openstack server start <vm-id>
openstack server stop <vm-id>
openstack server reboot <vm-id>
openstack server reboot --hard <vm-id>  # power cycle

# Delete
openstack server delete <vm-id>

# Resize
openstack server resize --flavor m1.large <vm-id>
openstack server resize --confirm <vm-id>

# Snapshot
openstack server image create --name my-snapshot <vm-id>

# Console
openstack console url show --novnc <vm-id>
openstack console log show <vm-id>
openstack console log show --lines 50 <vm-id>

# Metadata
openstack server set --property env=production <vm-id>

# Hypervisor
openstack hypervisor list
openstack hypervisor show compute-01
openstack hypervisor stats show

# Compute services
openstack compute service list
openstack compute service set --disable compute-01 nova-compute
openstack compute service set --enable compute-01 nova-compute

# Host aggregates
openstack aggregate list
openstack aggregate create --zone nova-zone ssd-hosts
openstack aggregate add host ssd-hosts compute-01
```

## Neutron — Networking

```bash
# Networks
openstack network list
openstack network show my-net
openstack network create my-net
openstack network delete my-net
openstack network set --disable my-net

# Subnets
openstack subnet list
openstack subnet create \
  --network my-net \
  --subnet-range 10.0.1.0/24 \
  --dns-nameserver 8.8.8.8 \
  --gateway 10.0.1.1 \
  my-subnet

# Routers
openstack router list
openstack router create my-router
openstack router set --external-gateway provider my-router
openstack router add subnet my-router my-subnet
openstack router remove subnet my-router my-subnet
openstack router show my-router

# Floating IPs
openstack floating ip list
openstack floating ip create provider-net
openstack server add floating ip <vm-id> <fip>
openstack server remove floating ip <vm-id> <fip>
openstack floating ip delete <fip>

# Security Groups
openstack security group list
openstack security group create web-sg
openstack security group rule list web-sg
openstack security group rule create \
  --protocol tcp --dst-port 22 web-sg
openstack server add security group <vm-id> web-sg

# Ports
openstack port list --server <vm-id>
openstack port show <port-id>
openstack port set --fixed-ip ip-address=10.0.1.100 <port-id>

# Network Agents
openstack network agent list
openstack network agent show <agent-id>
```

## Glance — Images

```bash
openstack image list
openstack image list --public
openstack image show ubuntu-22.04
openstack image create --file image.qcow2 --disk-format qcow2 --container-format bare --public ubuntu-22.04
openstack image set --property hw_disk_bus=virtio ubuntu-22.04
openstack image delete <image-id>
openstack image save --file output.img <image-id>
```

## Cinder — Block Storage

```bash
# Volumes
openstack volume list
openstack volume show <vol-id>
openstack volume create --size 50 --type ceph my-volume
openstack server add volume <vm-id> <vol-id>
openstack server remove volume <vm-id> <vol-id>
openstack volume delete <vol-id>
openstack volume set --state available <vol-id>  # reset state

# Snapshots
openstack volume snapshot create --volume <vol-id> my-snap
openstack volume snapshot list
openstack volume snapshot delete <snap-id>

# Volume types
openstack volume type list
openstack volume type create --property volume_backend_name=ceph ceph-type

# Backup
openstack volume backup create --name my-backup <vol-id>
openstack volume backup list
openstack volume backup restore <backup-id> <vol-id>

# Volume services
openstack volume service list
```

## Keystone — Identity

```bash
# Projects
openstack project list
openstack project create --domain Default myproject
openstack project set --disable myproject

# Users
openstack user list
openstack user create --password secret myuser
openstack user set --password newsecret myuser

# Roles
openstack role list
openstack role add --project myproject --user myuser member
openstack role remove --project myproject --user myuser member
openstack role assignment list --project myproject

# Endpoints
openstack endpoint list
openstack service list
openstack catalog list
```

## Heat — Orchestration

```bash
openstack stack list
openstack stack create -t template.yaml my-stack
openstack stack update -t template-v2.yaml my-stack
openstack stack delete my-stack
openstack stack show my-stack
openstack stack resource list my-stack
openstack stack event list my-stack
openstack orchestration template validate -t template.yaml
```

## Octavia — Load Balancer

```bash
openstack loadbalancer list
openstack loadbalancer show my-lb
openstack loadbalancer listener list
openstack loadbalancer pool list
openstack loadbalancer member list my-pool
openstack loadbalancer healthmonitor list
```

## Useful Patterns

```bash
# Tìm VM theo IP
openstack server list --all-projects | grep 10.0.1.100

# Xem tất cả VMs trên một compute host
openstack server list --all-projects --host compute-01

# Đếm VMs theo status
openstack server list --all-projects -f value -c Status | sort | uniq -c

# Xem usage của project
openstack usage show --project myproject

# Format output
openstack server list -f json | jq '.[] | {name, status, host: .["OS-EXT-SRV-ATTR:host"]}'
openstack server list -f csv --quote none
openstack server list -f table

# Debug: xem request ID
openstack --debug server list 2>&1 | grep "req-"
```

---
*Xem thêm: [[Troubleshooting]] | [[Day 2 Operations]] | [[Monitoring & Alerting]] | [[OpenStack]]*
