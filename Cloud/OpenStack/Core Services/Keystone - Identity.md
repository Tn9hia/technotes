---
tags:
  - openstack
  - keystone
  - identity
  - authentication
---

# Keystone — Identity Service

Keystone là **trung tâm xác thực** của OpenStack. Mọi service đều phải validate token qua Keystone trước khi xử lý request.

> [!abstract] Vai trò của Keystone
> 1. **Authentication** — Xác minh danh tính (user/service)
> 2. **Authorization** — Phân quyền (role-based)
> 3. **Service Catalog** — Cung cấp endpoint list cho tất cả services

## Kiến trúc

```
User/Service
    │
    ├─► POST /v3/auth/tokens ─► Keystone
    │         (credentials)          │
    │                                ├── Validate credentials
    │                                ├── Generate token
    │                                └── Return token + catalog
    │
    ├─► X-Auth-Token: <token> ─► Nova/Neutron/etc
                                       │
                                       └─► Validate token ──► Keystone
```

## Concepts chính

### Domain
- **Top-level container** cho users, groups, projects
- Default domain: `Default`
- Dùng để phân chia tenant giữa các tổ chức lớn

### Project (Tenant)
- **Đơn vị cô lập tài nguyên** (VMs, networks, volumes)
- Mỗi resource thuộc về 1 project
- User cần được assign role vào project để dùng

### User
- Có thể thuộc về nhiều projects với roles khác nhau
- Xác thực bằng password hoặc token

### Role
- Quyền được gán cho user trong một project/domain
- Default roles: `admin`, `member`, `reader`
- Roles được định nghĩa trong `policy.yaml`

### Group
- Tập hợp users, có thể assign role cho cả group

### Service & Endpoint
- Mỗi OpenStack service đăng ký với Keystone
- Có 3 loại endpoint: `internal`, `public`, `admin`

### Token
- **Fernet token** (hiện tại): signed, không cần lưu DB
  - Lightweight, stateless
  - Cần sync Fernet key giữa các Keystone nodes
- **UUID token** (cũ): lưu trong DB, phải validate online

## Token Flow

```bash
# 1. Get token
curl -s -X POST http://keystone:5000/v3/auth/tokens \
  -H "Content-Type: application/json" \
  -d '{
    "auth": {
      "identity": {
        "methods": ["password"],
        "password": {
          "user": {
            "name": "admin",
            "domain": {"name": "Default"},
            "password": "secret"
          }
        }
      },
      "scope": {
        "project": {
          "name": "admin",
          "domain": {"name": "Default"}
        }
      }
    }
  }'

# Token nằm trong header: X-Subject-Token
# Response body chứa: catalog, roles, project info
```

## Keystone API Endpoints

| Endpoint | Port | Dùng cho |
|----------|------|---------|
| Public | 5000 | Users, services bên ngoài |
| Admin | 5000 (v3) | Admin operations (cũ: 35357) |
| Internal | 5000 | Internal service communication |

> [!info] Port 35357 deprecated
> Từ OpenStack Rocky trở đi, admin endpoint dùng chung port 5000. Port 35357 không còn được dùng.

## Cấu hình quan trọng

```ini
# /etc/keystone/keystone.conf

[DEFAULT]
admin_token = (disable in production)

[database]
connection = mysql+pymysql://keystone:password@controller/keystone

[token]
provider = fernet
expiration = 3600  # 1 hour

[fernet_tokens]
key_repository = /etc/keystone/fernet-keys/
max_active_keys = 3

[cache]
enabled = true
backend = dogpile.cache.memcached
memcache_servers = controller1:11211,controller2:11211,controller3:11211
```

## CLI Commands

```bash
# List resources
openstack domain list
openstack project list
openstack user list
openstack role list
openstack service list
openstack endpoint list

# Tạo project và user mới
openstack project create --domain Default myproject
openstack user create --domain Default --password secret myuser
openstack role add --project myproject --user myuser member

# Kiểm tra token
openstack token issue
openstack token revoke <token-id>

# Catalog
openstack catalog list
openstack catalog show compute
```

## Fernet Key Rotation

```bash
# Rotate keys (cần chạy trên TẤT CẢ Keystone nodes)
keystone-manage fernet_rotate --keystone-user keystone --keystone-group keystone

# Sync keys giữa các nodes (cần cấu hình cron job)
```

> [!warning] Fernet key sync trong HA
> Trong HA deployment, Fernet keys phải được sync giữa tất cả Keystone instances. Kolla-Ansible tự động handle việc này qua shared storage hoặc ansible sync.

---
*Deep dive: [[Keystone Deep Dive]] | [[Projects, Domains & Users]] | [[RBAC & Policies]]*
*Xem thêm: [[OpenStack]]*
