---
tags:
  - openstack
  - horizon
  - dashboard
  - web-ui
---

# Horizon — Dashboard

Horizon là **Web UI** của OpenStack, cung cấp giao diện đồ họa để quản lý resources.

## Công nghệ

- Python **Django** framework
- Kết nối với các OpenStack APIs
- Hỗ trợ multi-domain, multi-project

## Cấu hình

```python
# /etc/openstack-dashboard/local_settings.py

OPENSTACK_HOST = "controller"
OPENSTACK_KEYSTONE_URL = "http://controller:5000/v3"

# Default domain
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "member"

# Enable APIs
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 3,
}

# Session
SESSION_ENGINE = "django.contrib.sessions.backends.cache"
CACHES = {
    "default": {
        "BACKEND": "django.core.cache.backends.memcached.PyMemcacheCache",
        "LOCATION": "controller:11211",
    }
}

# Time zone
TIME_ZONE = "Asia/Ho_Chi_Minh"
```

## Tính năng chính

### Compute
- Tạo, start, stop, delete VMs
- Console access (VNC, SPICE)
- Resize, snapshot VM
- Quản lý keypairs, security groups

### Network
- Tạo network, subnet, router
- Quản lý floating IPs
- Network Topology visualization

### Storage
- Tạo volumes, snapshots
- Attach/detach volumes

### Identity
- Quản lý projects, users (nếu là admin)

## Limitations

> [!warning] Horizon không phải production tool
> Horizon chủ yếu dùng cho **demo và quick management**. Production operations nên dùng:
> - **CLI** (`openstack` command) cho manual tasks
> - **Ansible/Terraform** cho automation
> - **Heat** cho infrastructure as code

## Thay thế Horizon

| Tool | Mô tả |
|------|-------|
| **Skyline** | Horizon thay thế, UI hiện đại hơn |
| **OpenStack CLI** | Command line, scriptable |
| **Terraform** | IaC cho OpenStack |
| **Ansible** | Automation |

---
*Xem thêm: [[OpenStack]] | [[CLI Commands]]*
