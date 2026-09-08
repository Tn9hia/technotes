---
tags:
  - openstack
  - keystone
  - rbac
  - security
  - policy
---

# RBAC & Policies

## Policy.yaml

Mỗi OpenStack service có file `policy.yaml` (hoặc `policy.json`) định nghĩa quyền của từng API:

```
/etc/nova/policy.yaml
/etc/neutron/policy.yaml
/etc/cinder/policy.yaml
/etc/glance/policy.yaml
```

## Policy Format

```yaml
# policy.yaml
"context_is_admin": "role:admin"
"context_is_reader": "role:reader"

# Rule format: "<action>": "<condition>"
"os_compute_api:servers:create": "rule:context_is_admin or role:member"
"os_compute_api:servers:delete": "rule:context_is_admin"
"os_compute_api:servers:list": ""  # empty = allow all authenticated

# Conditions:
# ""                  → always allow
# "!"                 → always deny
# "role:admin"        → require role admin
# "project_id:%(project_id)s"  → must be same project
# "rule:context_is_admin or role:member"  → OR
# "role:admin and domain_id:%(domain_id)s"  → AND
```

## Default Policies

Từ OpenStack Victoria, dùng **Scope-based RBAC**:

```yaml
# Nova policy.yaml ví dụ
"context_is_admin": "role:admin and system_scope:all"
"os_compute_api:servers:create": "role:member"
"os_compute_api:servers:create:forced_host": "role:admin and system_scope:all"

# Chỉ admin ở system scope mới có thể:
"os_compute_api:os-hypervisors:list": "role:admin and system_scope:all"
"os_compute_api:os-aggregates:index": "role:admin and system_scope:all"
```

## Scope Types

```bash
# System scope: toàn bộ cloud
openstack --os-system-scope all server list --all-projects

# Domain scope: 1 domain
openstack --os-domain-name company-a project list

# Project scope: 1 project
openstack --os-project-name myproject server list
```

## Custom Policy Overrides

```yaml
# /etc/nova/policy.yaml — override mặc định

# Cho phép member tạo VM với host hint (mặc định chỉ admin)
"os_compute_api:servers:create:attach_network": "role:member"

# Chỉ admin mới xóa được network của người khác
"os_network_api:delete_network": "role:admin"

# Cho phép member list tất cả VMs trong project của họ
"os_compute_api:servers:index": "role:reader or role:member or role:admin"
```

## Xem Policy hiện tại

```bash
# List all policies
openstack policy list --service compute
openstack policy show compute <policy-rule>

# Với oslo.policy tool
oslopolicy-checker --namespace nova \
  --target '{"project_id": "abc123"}' \
  --rule "os_compute_api:servers:create"

# Generate sample policy file
oslopolicy-sample-generator --namespace nova > nova-policy-sample.yaml
```

## Security Best Practices

> [!warning] Không dùng "!" cho policy chính
> Dùng "!" sẽ block hoàn toàn, kể cả admin. Hãy test kỹ trước khi deploy.

### Phân quyền theo vai trò

```yaml
# Ví dụ phân quyền rõ ràng

# Chỉ admin tạo được flavor
"os_compute_api:flavors:create": "role:admin and system_scope:all"

# Member chỉ đọc flavor
"os_compute_api:flavors:show": "role:reader or role:member or role:admin"

# Admin provider network, user không tạo được
"create_network:provider:network_type": "rule:context_is_admin"
"create_network:provider:segmentation_id": "rule:context_is_admin"

# User chỉ có thể tạo tenant network
"create_network": "role:member"
```

## Audit Logging

```ini
# oslo_messaging và keystonemiddleware hỗ trợ audit logging

# nova.conf
[oslo_messaging_notifications]
driver = messagingv2
topics = notifications

# Audit log mọi API call
[audit]
api_audit_map = /etc/nova/api_audit_map.conf
```

```bash
# Xem audit events
openstack event list --long  # nếu có Ceilometer/Panko
```

## Trusted Roles (Heat Stack Owner)

```bash
# User cần role heat_stack_owner để tạo Heat stack
openstack role add \
  --project myproject \
  --user myuser \
  heat_stack_owner
```

---
*Xem thêm: [[Keystone - Identity]] | [[Keystone Deep Dive]] | [[Projects, Domains & Users]]*
