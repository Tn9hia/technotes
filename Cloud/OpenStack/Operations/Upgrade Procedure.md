---
tags:
  - openstack
  - upgrade
  - operations
---

# Upgrade Procedure

## OpenStack Release Cycle

```
N-1 release ──► N release ──► N+1 release
  (current)    (upgrade to)

Ví dụ: Zed → 2023.1 (Antelope) → 2023.2 (Bobcat) → 2024.1 (Caracal)
```

### SLURP (Skip-Level Upgrade Release Process)

```
2023.1 (Antelope) ──────────────────────────► 2024.1 (Caracal)
                   Skip 2023.2 (Bobcat)
```

Các release SLURP: 2023.1, 2024.1, 2025.1... (odd trong cycle)

> [!warning] Upgrade rule
> OpenStack chỉ hỗ trợ upgrade **từng phiên bản một** (N → N+1), trừ khi đích là SLURP release.

## Pre-Upgrade Checklist

```bash
# 1. Đọc Release Notes
# https://docs.openstack.org/releasenotes/

# 2. Kiểm tra upgrade matrix
# https://docs.openstack.org/kolla-ansible/latest/support_matrix.html

# 3. Test trong Lab/Staging trước

# 4. Backup
# - Database (MariaDB)
# - /etc/kolla/passwords.yml
# - /etc/kolla/globals.yml
# - Ceph config

# 5. Kiểm tra health trước upgrade
openstack compute service list
openstack network agent list
ceph status

# 6. Thông báo users
# Downtime window: các API services sẽ restart
```

## Upgrade với Kolla-Ansible

```bash
# 1. Update Kolla-Ansible
pip install kolla-ansible==<new-version>

# 2. Update globals.yml
vim /etc/kolla/globals.yml
# Thay openstack_release: "2023.1" → "2024.1"

# 3. Pull images mới
kolla-ansible -i inventory/multinode pull

# 4. Pre-upgrade checks
kolla-ansible -i inventory/multinode prechecks

# 5. Run upgrade
kolla-ansible -i inventory/multinode upgrade

# 6. Post-upgrade checks
source /etc/kolla/admin-openrc.sh
openstack compute service list
openstack network agent list
openstack volume service list
```

## Rolling Upgrade (Minimal Downtime)

OpenStack hỗ trợ **rolling upgrade** — upgrade từng component theo thứ tự:

```
1. Keystone
2. Glance
3. Cinder (API + Scheduler + Volume)
4. Nova (API + Scheduler → Conductor → Compute)
5. Neutron (Server → Agents)
6. Horizon
7. Heat, Octavia...
```

### Nova Rolling Upgrade

```bash
# Nova có explicit rolling upgrade support

# Trên controller:
nova-manage api_db sync
nova-manage db sync

# Upgrade nova-api, nova-scheduler, nova-conductor trước
# Upgrade nova-compute sau (từng host một)

# Sau khi upgrade xong tất cả compute:
nova-manage db online_data_migrations
```

### Kolla: Upgrade từng service

```bash
# Upgrade chỉ nova
kolla-ansible -i inventory/multinode upgrade --tags nova

# Upgrade chỉ neutron
kolla-ansible -i inventory/multinode upgrade --tags neutron

# Upgrade trên 1 host cụ thể
kolla-ansible -i inventory/multinode upgrade --limit compute-01
```

## Database Migrations

```bash
# Chạy DB migrations sau khi upgrade container/packages
# Nova
nova-manage api_db sync
nova-manage db sync
nova-manage db online_data_migrations

# Neutron
neutron-db-manage upgrade heads

# Keystone
keystone-manage db_sync

# Cinder
cinder-manage db sync

# Glance
glance-manage db_sync
```

## Rollback

> [!danger] Rollback rất khó
> OpenStack DB migrations không phải lúc nào cũng reversible. Cần có backup DB đầy đủ trước upgrade.

```bash
# Nếu upgrade fail, rollback:
# 1. Restore database từ backup
# 2. Rollback container images về version cũ

# Kolla: specify image version
vim /etc/kolla/globals.yml
# openstack_release: "2023.1"  ← rollback về version cũ

kolla-ansible -i inventory/multinode pull
kolla-ansible -i inventory/multinode deploy
```

## Upgrade Path ví dụ (Zed → Caracal)

```
Zed (2022.2)
    │ Upgrade
    ▼
Antelope (2023.1)  ← SLURP release
    │ Upgrade (có thể skip Bobcat nếu đích là Caracal)
    ▼
Caracal (2024.1)  ← SLURP release
```

```bash
# Bước 1: Zed → Antelope
vim globals.yml  # openstack_release: "2023.1"
kolla-ansible upgrade

# Verify
openstack compute service list

# Bước 2: Antelope → Caracal
vim globals.yml  # openstack_release: "2024.1"
kolla-ansible upgrade
```

## Post-Upgrade Validation

```bash
#!/bin/bash
# post_upgrade_check.sh

echo "=== Service Health ==="
openstack compute service list
openstack network agent list
openstack volume service list

echo "=== Test VM Creation ==="
openstack server create \
  --image cirros \
  --flavor m1.tiny \
  --network internal-net \
  --wait \
  post-upgrade-test-vm

openstack server show post-upgrade-test-vm | grep status
openstack server delete post-upgrade-test-vm

echo "=== Test Network ==="
openstack network create test-upgrade-net
openstack subnet create \
  --network test-upgrade-net \
  --subnet-range 192.168.200.0/24 \
  test-upgrade-subnet
openstack network delete test-upgrade-net

echo "=== API Versions ==="
openstack versions show
```

---
*Xem thêm: [[Day 2 Operations]] | [[Installation Methods]] | [[HA Architecture]] | [[OpenStack]]*
