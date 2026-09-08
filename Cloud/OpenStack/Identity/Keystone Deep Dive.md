---
tags:
  - openstack
  - keystone
  - identity
  - authentication
---

# Keystone Deep Dive

## Authentication Backends

Keystone hỗ trợ nhiều backend:

| Backend | Mô tả | Dùng khi |
|---------|-------|---------|
| **SQL** (default) | MariaDB | Development, small deployments |
| **LDAP** | Active Directory, OpenLDAP | Enterprise với existing AD |
| **Federated** | SAML2, OpenID Connect | SSO, multi-cloud |

### LDAP Integration

```ini
# keystone.conf
[identity]
driver = ldap

[ldap]
url = ldap://ldap.company.com
user = cn=keystone,dc=company,dc=com
password = secret
suffix = dc=company,dc=com
user_tree_dn = ou=Users,dc=company,dc=com
group_tree_dn = ou=Groups,dc=company,dc=com
user_id_attribute = sAMAccountName
user_name_attribute = sAMAccountName
user_mail_attribute = mail
```

## Token Providers

### Fernet Token (Current standard)

```
Header.Payload.Signature
  │       │        │
  │   Encrypted    HMAC
  │   (AES-256)
  │
Fernet version
```

- **Stateless**: không lưu DB
- **Cần sync key** giữa các Keystone nodes
- **Key rotation** cần periodic

```bash
# Xem Fernet keys
ls /etc/keystone/fernet-keys/
# 0  ← staged key (chờ active)
# 1  ← primary key (ký token mới)
# 2  ← secondary key (verify token cũ)

# Rotate key (chạy trên tất cả Keystone nodes)
keystone-manage fernet_rotate \
  --keystone-user keystone \
  --keystone-group keystone

# Validate token
openstack token issue
openstack token introspect <token>
```

### JWT Token (Experimental, newer)

Thay Fernet bằng JWT (JSON Web Token) — dễ inspect hơn.

## Token Scope

Token phải có **scope** để hoạt động với services:

```bash
# Unscoped token (chỉ để list projects)
openstack token issue

# Project-scoped token
openstack token issue --os-project-name myproject

# Domain-scoped token (admin operations)
openstack token issue --os-domain-name Default

# System-scoped token (global admin)
openstack token issue --os-system-all
```

## Service Catalog

Service catalog chứa **endpoint URLs** của tất cả services:

```bash
# Xem catalog
openstack catalog list

# Output:
# +----------+----------------+-----------------------------------------------------+
# | Name     | Type           | Endpoints                                           |
# +----------+----------------+-----------------------------------------------------+
# | nova     | compute        | RegionOne                                           |
# |          |                |   public: http://controller:8774/v2.1               |
# |          |                |   internal: http://controller:8774/v2.1             |
# |          |                |   admin: http://controller:8774/v2.1                |
# +----------+----------------+-----------------------------------------------------+

# Xem endpoint cụ thể
openstack endpoint list --service compute
openstack endpoint show <endpoint-id>
```

### Regions và Availability Zones

```bash
# Tạo region
openstack region create Region1
openstack region create --parent-region Region1 AZ1

# Tạo endpoint trong region cụ thể
openstack endpoint create \
  --region Region1 \
  compute public http://nova.region1.example.com:8774/v2.1
```

## Middleware: Keystonemiddleware

Tất cả OpenStack services dùng `keystonemiddleware.auth_token` để validate tokens:

```ini
# Trong config file của mỗi service (nova.conf, neutron.conf...)
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova_password

# Cache token validation results (tránh gọi Keystone mỗi request)
memcache_security_strategy = ENCRYPT
memcache_secret_key = my_secret_key
```

## Application Credentials

Cho phép apps lấy token mà không cần user credentials:

```bash
# Tạo application credential
openstack application credential create \
  --secret mysecret \
  --role member \
  my-app-cred

# Dùng để auth
export OS_AUTH_TYPE=v3applicationcredential
export OS_APPLICATION_CREDENTIAL_ID=<id>
export OS_APPLICATION_CREDENTIAL_SECRET=mysecret
openstack token issue
```

## Federated Identity (SAML/OIDC)

```ini
# keystone.conf
[auth]
methods = password,token,saml2,mapped,application_credential

[federation]
trusted_dashboard = https://horizon.example.com/auth/websso/
```

## Keystone Caching

```ini
# keystone.conf
[cache]
enabled = true
backend = dogpile.cache.memcached
backend_argument = url:controller1:11211,controller2:11211

[token]
caching = true
cache_time = 300  # 5 minutes

[revoke]
backend = sql
```

## Troubleshooting

```bash
# Kiểm tra Keystone service
openstack service list

# Xem log
tail -f /var/log/keystone/keystone.log

# Test authentication
curl -si http://controller:5000/v3 | head -5

# Kiểm tra endpoint connectivity
openstack endpoint list --format json | python3 -m json.tool

# Fernet key sync check (tất cả nodes phải có cùng keys)
md5sum /etc/keystone/fernet-keys/*

# Xem token expiry
openstack token issue -f json | python3 -c "import json,sys; t=json.load(sys.stdin); print(t['expires'])"
```

---
*Xem thêm: [[Keystone - Identity]] | [[Projects, Domains & Users]] | [[RBAC & Policies]] | [[OpenStack]]*
