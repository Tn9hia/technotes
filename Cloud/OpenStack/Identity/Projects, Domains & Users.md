---
tags:
  - openstack
  - keystone
  - multitenancy
  - identity
---

# Projects, Domains & Users

## Hierarchy

```
Cloud (OpenStack)
└── Domain A (công ty A)
    ├── Project A1 (dev team)
    │   ├── User 1 (role: member)
    │   └── User 2 (role: admin)
    ├── Project A2 (prod team)
    │   └── Group Ops (role: member)
    └── User 3 (domain-level: domain admin)
└── Domain B (công ty B)
    └── Project B1
        └── ...
└── Default Domain
    ├── Project admin
    └── User admin (role: admin)
```

## Domain

Domain là **top-level isolation boundary**:

```bash
# Tạo domain
openstack domain create --description "Company A" company-a

# List domains
openstack domain list

# Disable domain (vô hiệu hóa tất cả users/projects trong domain đó)
openstack domain set --disable company-a

# Delete domain (phải disable trước)
openstack domain set --disable company-a
openstack domain delete company-a
```

## Project (Tenant)

```bash
# Tạo project
openstack project create \
  --domain company-a \
  --description "Development Environment" \
  dev-project

# List projects
openstack project list --domain company-a

# Set quota cho project
openstack quota set \
  --instances 50 \
  --cores 200 \
  --ram 512000 \
  --volumes 100 \
  --gigabytes 5000 \
  --floating-ips 10 \
  dev-project

# Xem quota
openstack quota show dev-project

# Hierarchical projects (project con)
openstack project create \
  --parent dev-project \
  sub-project-frontend
```

## User Management

```bash
# Tạo user
openstack user create \
  --domain company-a \
  --password "P@ssw0rd123" \
  --email john@company-a.com \
  john

# Thay đổi password
openstack user set --password "NewP@ss" john

# Disable user (không xóa data)
openstack user set --disable john

# Xem user info
openstack user show john

# List users trong domain
openstack user list --domain company-a
```

## Role Assignment

```bash
# List available roles
openstack role list

# Gán role cho user trong project
openstack role add \
  --project dev-project \
  --user john \
  member

# Gán domain admin
openstack role add \
  --domain company-a \
  --user john \
  admin

# Gán cho group
openstack role add \
  --project dev-project \
  --group ops-team \
  member

# Xem role assignment
openstack role assignment list --project dev-project
openstack role assignment list --user john --names

# Revoke role
openstack role remove \
  --project dev-project \
  --user john \
  member
```

## Groups

```bash
# Tạo group
openstack group create \
  --domain company-a \
  ops-team

# Thêm user vào group
openstack group add user ops-team john
openstack group add user ops-team jane

# Gán role cho group (tất cả members trong group được role)
openstack role add \
  --project dev-project \
  --group ops-team \
  member

# List members
openstack group contains user ops-team john
openstack user list --group ops-team
```

## Default Roles (từ OpenStack Wallaby)

```
admin     > member    > reader
  │           │           │
  ▼           ▼           ▼
Full      Normal      Read-only
control   user        access
```

- **admin**: full control trong scope
- **member**: create/modify resources
- **reader**: view-only
- **heat_stack_owner**: tạo Heat stacks
- **load-balancer_member**: tạo LBaaS resources

## Service Projects

Mỗi OpenStack service có user/project riêng:

```bash
# Xem service users
openstack user list --project service

# nova, neutron, glance, cinder, heat, octavia...
# Đều có password riêng, được Keystone authenticate

openstack user show nova
```

## Multi-Domain Admin

```bash
# Cloud admin: quản lý tất cả domains
openstack role add --system all --user admin admin

# Domain admin: quản lý 1 domain
openstack role add --domain company-a --user domain-admin admin

# Project admin: quản lý 1 project
openstack role add --project dev-project --user proj-admin admin
```

---
*Xem thêm: [[Keystone - Identity]] | [[Keystone Deep Dive]] | [[RBAC & Policies]]*
