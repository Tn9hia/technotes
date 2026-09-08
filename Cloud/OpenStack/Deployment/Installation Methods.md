---
tags:
  - openstack
  - deployment
  - installation
---

# Installation Methods

## So sánh các phương pháp

| Tool | Type | Complexity | Production? | Notes |
|------|------|-----------|-------------|-------|
| **Kolla-Ansible** | Container-based | Medium | ✅ Yes | **Phổ biến nhất** |
| **OpenStack Ansible** | Bare metal | High | ✅ Yes | Flexible, phức tạp hơn |
| **TripleO** | OpenStack-on-OpenStack | Very High | ✅ Yes | Red Hat ecosystem |
| **Devstack** | Script | Low | ❌ Dev only | Dev/test nhanh |
| **Packstack** | Puppet | Medium | ⚠️ Legacy | RHEL/CentOS, deprecated |
| **Sunbeam** | Snaps | Low | ⚠️ Experimental | Ubuntu, newest |
| **Manual** | Manual | Very High | ✅ Yes | Full control, painful |

## Kolla-Ansible (Khuyến nghị)

Triển khai OpenStack dưới dạng **Docker containers**.

### Ưu điểm
- Isolation: mỗi service trong container riêng
- Upgrade dễ hơn (update image, recreate container)
- Rollback dễ: giữ lại image cũ
- Community lớn, tài liệu tốt

### Kiến trúc

```
Deploy Node (Ansible)
    │
    ├── ansible-playbook deploy.yml
    │
    ├── Controller Nodes ──► Docker containers
    │     ├── kolla/keystone
    │     ├── kolla/nova-api
    │     ├── kolla/neutron-server
    │     ├── kolla/mariadb
    │     └── kolla/rabbitmq
    │
    └── Compute Nodes ──► Docker containers
          ├── kolla/nova-compute
          ├── kolla/neutron-openvswitch-agent
          └── kolla/libvirtd
```

### Cấu trúc thư mục

```
/etc/kolla/
├── globals.yml          ← Cấu hình chính (network, backends...)
├── passwords.yml        ← Auto-generated passwords
├── config/              ← Override config cho từng service
│   ├── nova/
│   │   └── nova.conf
│   └── neutron/
│       └── neutron.conf
└── inventory/
    ├── multinode        ← Production inventory
    └── all-in-one       ← AIO inventory
```

### Workflow triển khai

```bash
# 1. Cài đặt Kolla-Ansible
pip install kolla-ansible

# 2. Copy cấu hình mẫu
cp /etc/kolla/globals.yml.example /etc/kolla/globals.yml

# 3. Chỉnh sửa globals.yml
vim /etc/kolla/globals.yml
# Quan trọng:
# kolla_base_distro: "ubuntu"
# openstack_release: "2024.1"
# network_interface: "eth0"
# neutron_external_interface: "eth1"
# kolla_internal_vip_address: "10.0.0.10"

# 4. Generate passwords
kolla-genpwd

# 5. Kiểm tra connectivity
kolla-ansible -i inventory/multinode prechecks

# 6. Bootstrap servers
kolla-ansible -i inventory/multinode bootstrap-servers

# 7. Deploy
kolla-ansible -i inventory/multinode deploy

# 8. Generate admin credentials
kolla-ansible post-deploy
source /etc/kolla/admin-openrc.sh
```

### globals.yml quan trọng nhất

```yaml
# Network config
network_interface: "bond0"
neutron_external_interface: "bond1"
kolla_internal_vip_address: "10.0.0.10"
kolla_external_vip_address: "203.0.113.10"  # nếu có external VIP

# OpenStack release
openstack_release: "2024.1"
kolla_base_distro: "ubuntu"

# HA
enable_haproxy: "yes"
enable_keepalived: "yes"

# Neutron
neutron_plugin_agent: "openvswitch"  # hoặc "ovn"
enable_neutron_dvr: "yes"            # Distributed Virtual Router
enable_neutron_provider_networks: "yes"

# Storage backends
enable_cinder: "yes"
cinder_backend_ceph: "yes"
glance_backend_ceph: "yes"
nova_compute_virt_type: "kvm"

# Optional services
enable_heat: "yes"
enable_horizon: "yes"
enable_octavia: "yes"
enable_barbican: "yes"
```

## Devstack (Dev/Test only)

```bash
# Clone và chạy
git clone https://opendev.org/openstack/devstack
cd devstack
cat > local.conf <<EOF
[[local|localrc]]
ADMIN_PASSWORD=secret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
EOF

./stack.sh
```

> [!warning] Không dùng Devstack cho Production
> Devstack tự cài và cấu hình mọi thứ vào hệ thống, không có isolation, rất khó maintain và upgrade.

## OpenStack Ansible (OSA)

- Deploy trực tiếp lên bare metal (không dùng container)
- Phù hợp khi cần customize sâu hơn Kolla
- Repository: `openstack/openstack-ansible`

```bash
# Bootstrap
git clone https://opendev.org/openstack/openstack-ansible /opt/openstack-ansible
cd /opt/openstack-ansible
scripts/bootstrap-ansible.sh

# Configure inventory
cp -r etc/openstack_deploy /etc/openstack_deploy
vim /etc/openstack_deploy/openstack_user_config.yml

# Run
openstack-ansible setup-hosts.yml
openstack-ansible setup-infrastructure.yml
openstack-ansible setup-openstack.yml
```

## Post-installation Verification

```bash
# Source credentials
source /etc/kolla/admin-openrc.sh

# Kiểm tra services
openstack service list
openstack endpoint list

# Kiểm tra compute
openstack compute service list
openstack hypervisor list

# Kiểm tra network agents
openstack network agent list

# Kiểm tra storage
openstack volume service list

# Tạo thử network và VM
openstack network create test-net
openstack subnet create --network test-net --subnet-range 192.168.100.0/24 test-subnet
openstack server create --image cirros --flavor m1.tiny --network test-net test-vm
```

---
*Xem thêm: [[Deployment Models]] | [[Hardware Requirements]] | [[OpenStack]]*
