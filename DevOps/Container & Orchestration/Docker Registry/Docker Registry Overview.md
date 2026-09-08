---
title: Docker Registry — Harbor
tags:
  - docker
  - harbor
  - registry
  - deep-dive
date: 2026-04-26
---

# Docker Registry — Harbor

## What is Harbor?

Harbor là enterprise-grade private container registry — CNCF graduated project. Ngoài lưu trữ images còn có: vulnerability scanning, image signing, replication, RBAC, garbage collection.

```
Developers → docker push → Harbor ─┬─ Scan (Trivy)
                                    ├─ Sign (Cosign/Notation)
                                    └─ Replicate → Other registries

CI/CD pipeline → docker pull → Harbor → Deploy

Air-gap mirror: Internet → Harbor ← Docker Hub/GCR/ECR replication
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Harbor                                │
│                                                              │
│  ┌───────────┐  ┌───────────┐  ┌──────────────────────┐    │
│  │  Portal   │  │  Core     │  │  Registry (v2)       │    │
│  │  (Nginx   │  │  (REST API│  │  (actual image store)│    │
│  │   proxy)  │  │  + auth)  │  │                      │    │
│  └─────┬─────┘  └─────┬─────┘  └──────────┬───────────┘   │
│        │              │                    │               │
│  ┌─────▼──────────────▼────────────────────▼──────────┐    │
│  │                   Database (PostgreSQL)              │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐               │
│  │ Jobservice│  │  Trivy    │  │  Redis    │               │
│  │(async jobs│  │ (scanner) │  │  (cache,  │               │
│  │ replication│ │           │  │   queue)  │               │
│  │  GC, scan)│  └───────────┘  └───────────┘               │
│  └───────────┘                                              │
└─────────────────────────────────────────────────────────────┘
         │ Storage backend
         ├── Filesystem (local)
         ├── S3 / MinIO
         └── Azure Blob / GCS
```

**Components:**
- **Core**: REST API server, authentication, authorization, policy
- **Registry**: Distribution v2 — actual OCI image storage
- **Jobservice**: Async job runner (replication, GC, scanning, webhook)
- **Trivy**: Vulnerability scanner — scan on push hoặc on-demand
- **Portal**: Web UI (Nginx proxy + Angular SPA)
- **PostgreSQL**: Metadata store (users, projects, policies, artifacts)
- **Redis**: Job queue, session cache, rate limiting

---

## Installation

### Docker Compose (recommended cho single node)

```bash
# Download installer
wget https://github.com/goharbor/harbor/releases/download/v2.11.0/harbor-offline-installer-v2.11.0.tgz
tar xvf harbor-offline-installer-v2.11.0.tgz
cd harbor

# Configure
cp harbor.yml.tmpl harbor.yml
```

```yaml
# harbor.yml (key configs)
hostname: registry.internal          # FQDN — phải resolve

http:
  port: 80

https:
  port: 443
  certificate: /etc/harbor/certs/registry.crt
  private_key: /etc/harbor/certs/registry.key

harbor_admin_password: Harbor12345   # đổi ngay sau install

database:
  password: root123                  # PostgreSQL password

data_volume: /data/harbor            # lưu images

trivy:
  ignore_unfixed: false
  skip_update: false
  offline_scan: false                # true cho air-gap

jobservice:
  max_job_workers: 10

log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200m
    location: /var/log/harbor

# External database (production)
# external_database:
#   host: postgres.internal
#   port: 5432
#   username: harbor
#   password: harbor_db_pass
#   sslmode: require

# External Redis
# external_redis:
#   host: redis.internal:6379
#   password: redis_pass
#   registry_db_index: 1
#   jobservice_db_index: 2
#   trivy_db_index: 5
#   idle_timeout_seconds: 30
```

```bash
# Install
sudo ./install.sh --with-trivy

# Start / Stop
docker compose -f /data/harbor/docker-compose.yml up -d
docker compose -f /data/harbor/docker-compose.yml down

# Reconfigure sau khi thay đổi harbor.yml
sudo ./prepare
docker compose -f /data/harbor/docker-compose.yml up -d
```

### Helm (Kubernetes)

```bash
helm repo add harbor https://helm.goharbor.io
helm repo update

helm install harbor harbor/harbor \
  --namespace harbor \
  --create-namespace \
  --set expose.type=ingress \
  --set expose.ingress.hosts.core=registry.internal \
  --set externalURL=https://registry.internal \
  --set harborAdminPassword=Harbor12345 \
  --set persistence.enabled=true \
  --set persistence.persistentVolumeClaim.registry.size=100Gi
```

---

## Projects & Repositories

Harbor tổ chức theo **Project** (tương tự namespace):

```
Harbor
├── library/              ← Public project (mặc định)
│   └── nginx:1.26
├── platform/             ← Private project
│   ├── myapp:1.2.3
│   └── myapp:latest
└── base-images/          ← Shared base images
    ├── ubuntu:22.04-custom
    └── node:20-alpine-custom
```

```bash
# Login
docker login registry.internal
# Username: admin / Password: Harbor12345

# Push image
docker tag myapp:1.2.3 registry.internal/platform/myapp:1.2.3
docker push registry.internal/platform/myapp:1.2.3

# Pull image
docker pull registry.internal/platform/myapp:1.2.3

# Untag / delete tag (qua API)
curl -X DELETE -u admin:password \
  "https://registry.internal/api/v2.0/projects/platform/repositories/myapp/artifacts/1.2.3"
```

**Robot accounts (CI/CD):**
```bash
# Tạo qua UI: Projects → <project> → Robot Accounts → New Robot Account
# Hoặc qua API:
curl -X POST -u admin:password \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ci-robot",
    "duration": 365,
    "permissions": [{
      "kind": "project",
      "namespace": "platform",
      "access": [
        {"resource": "repository", "action": "pull"},
        {"resource": "repository", "action": "push"}
      ]
    }]
  }' \
  "https://registry.internal/api/v2.0/robots"
```

---

## User Management — LDAP/OIDC

### LDAP Integration

```
Administration → Configuration → Authentication → LDAP

LDAP URL: ldap://ldap.internal:389
LDAP Search DN: cn=admin,dc=company,dc=com
LDAP Search Password: ****
LDAP Base DN: dc=company,dc=com
LDAP Filter: (objectClass=person)
LDAP UID attribute: sAMAccountName
LDAP Scope: Subtree
LDAP Group Base DN: ou=Groups,dc=company,dc=com
LDAP Group Filter: (objectClass=groupOfNames)
LDAP Group GID attribute: cn
LDAP Group Admin DN: cn=harbor-admins,ou=Groups,dc=company,dc=com
```

### OIDC Integration (Keycloak/Okta/Azure AD)

```
Administration → Configuration → Authentication → OIDC

OIDC Provider Name: Keycloak
OIDC Endpoint: https://keycloak.internal/realms/company
OIDC Client ID: harbor
OIDC Client Secret: <secret>
OIDC Scope: openid,profile,email,groups
Verify Certificate: true
Auto onboard: true    # tự tạo user khi login lần đầu
Username claim: preferred_username
Group claim name: groups
```

---

## RBAC

Harbor RBAC theo Project + Role:

| Role | Pull | Push | Delete | Manage |
|---|---|---|---|---|
| **Guest** | ✓ | ✗ | ✗ | ✗ |
| **Developer** | ✓ | ✓ | ✗ | ✗ |
| **Maintainer** | ✓ | ✓ | ✓ | ✓ |
| **Project Admin** | ✓ | ✓ | ✓ | ✓ + manage members |
| **Limited Guest** | ✓ (public only) | ✗ | ✗ | ✗ |

```bash
# Assign user to project qua API
curl -X POST -u admin:password \
  -H "Content-Type: application/json" \
  -d '{"role_id": 2, "member_user": {"username": "john"}}' \
  "https://registry.internal/api/v2.0/projects/platform/members"

# Role IDs: 1=Admin, 2=Developer, 3=Guest, 4=Maintainer, 5=LimitedGuest
```

---

## Replication

Replicate images từ/tới external registries.

### Push replication (Harbor → target)

```
Configuration → Registries → New Endpoint
  Provider: Docker Hub / GCR / ECR / Harbor
  Name: docker-hub-mirror
  Endpoint URL: https://hub.docker.com
  Access ID: username
  Access Secret: password

Replications → New Replication Rule
  Name: sync-to-dr
  Replication mode: Push-based
  Source: filter images (e.g., platform/**)
  Destination: harbor-dr
  Trigger: Event-based (on push) hoặc Scheduled
  Override: true
  Bandwidth: 10 Mbps (throttle)
```

### Pull replication (Harbor ← source) — Air-gap mirror

```
Replication mode: Pull-based
Source: Docker Hub
  Filter: library/nginx, library/postgres, library/redis
Destination namespace: mirror
Trigger: Scheduled (0 2 * * *) — mỗi đêm 2am

# Kết quả: registry.internal/mirror/library/nginx:latest
# → Pull từ Docker Hub mỗi đêm, cache locally
```

```bash
# Trigger replication manually
curl -X POST -u admin:password \
  "https://registry.internal/api/v2.0/replication/executions" \
  -H "Content-Type: application/json" \
  -d '{"policy_id": 1}'

# Check status
curl -u admin:password \
  "https://registry.internal/api/v2.0/replication/executions?policy_id=1" | jq '.[] | {id, status, start_time}'
```

---

## Garbage Collection

Xoá unreferenced blobs sau khi tags bị xoá.

```
Administration → Garbage Collection

Schedule: Daily (0 2 * * *)
Delete Untagged Artifacts: true    # xoá artifacts không có tag
Workers: 1
```

```bash
# Trigger GC manually
curl -X POST -u admin:password \
  "https://registry.internal/api/v2.0/system/gc/schedule" \
  -H "Content-Type: application/json" \
  -d '{"schedule": {"type": "Manual"}}'

# Check GC history
curl -u admin:password \
  "https://registry.internal/api/v2.0/system/gc" | jq '.[0]'
```

**Chú ý:** GC cần Harbor dừng nhận write requests để tránh data inconsistency. Harbor v2.x có online GC (không cần downtime) nhưng cần cẩn thận khi có concurrent pushes.

---

## Image Scanning — Trivy Integration

```
Projects → <project> → Configuration
  Automatically scan images on push: ✓
  Prevent vulnerable images from running: ✓
  Vulnerability severity threshold: Critical

# Scan on-demand
curl -X POST -u admin:password \
  "https://registry.internal/api/v2.0/projects/platform/repositories/myapp/artifacts/1.2.3/scan"

# Get scan result
curl -u admin:password \
  "https://registry.internal/api/v2.0/projects/platform/repositories/myapp/artifacts/1.2.3/additions/vulnerabilities" | jq .
```

**Scan policy — block deploy nếu có Critical:**
```
Projects → <project> → Configuration
  Prevent vulnerable images from running → Critical
```

→ `docker pull` sẽ fail với 412 Precondition Failed nếu image có Critical CVE.

---

## Image Signing — Cosign + Notation

### Cosign

```bash
# Sign sau khi push
cosign sign --key cosign.key registry.internal/platform/myapp:1.2.3

# Verify (trong CD pipeline hoặc Kyverno policy)
cosign verify --key cosign.pub registry.internal/platform/myapp:1.2.3
```

### Notation (CNCF standard)

```bash
# Install notation
brew install notation

# Generate key
notation cert generate-test --default "harbor-signer"

# Sign
notation sign registry.internal/platform/myapp:1.2.3

# Verify
notation verify registry.internal/platform/myapp:1.2.3

# Add trust policy
notation policy import trust-policy.json
```

---

## Webhook

Trigger CI/CD events khi có image push/scan complete.

```
Projects → <project> → Webhooks → New Webhook
  Notify Type: HTTP
  Endpoint URL: https://jenkins.internal/webhook/harbor
  Auth Header: Authorization: Bearer <token>
  Events: Push Artifact, Scanning Finished, Replication
```

**Webhook payload (Push Artifact):**
```json
{
  "type": "PUSH_ARTIFACT",
  "occur_at": 1714089600,
  "operator": "robot$ci-robot",
  "event_data": {
    "resources": [{
      "resource_url": "registry.internal/platform/myapp:1.2.3",
      "tag": "1.2.3",
      "digest": "sha256:abc123..."
    }],
    "repository": {
      "name": "myapp",
      "namespace": "platform",
      "full_name": "platform/myapp",
      "type": "private"
    }
  }
}
```

---

## Retention Policy

Tự động xoá old tags để tiết kiệm storage.

```
Projects → <project> → Tag Retention → Add Rule

Repositories matching: **                  (tất cả)
Tags matching: **
Retain: 10 most recently pushed tags       (giữ 10 tag mới nhất)
Exclude: latest, stable, v*.*.*            (không xoá tagged versions)

# Rule 2: Retain tags pushed in last 30 days
Retain: pushed within the last 30 days
```

---

## Ops Runbook

```bash
# Health check
curl https://registry.internal/api/v2.0/health | jq .

# Harbor logs
docker compose -f /data/harbor/docker-compose.yml logs -f core
docker compose -f /data/harbor/docker-compose.yml logs -f jobservice
docker compose -f /data/harbor/docker-compose.yml logs -f registry

# Disk usage
curl -u admin:password \
  "https://registry.internal/api/v2.0/statistics" | jq .

# List repositories trong project
curl -u admin:password \
  "https://registry.internal/api/v2.0/projects/platform/repositories?page_size=100" | jq '.[].name'

# List tags của repository
curl -u admin:password \
  "https://registry.internal/api/v2.0/projects/platform/repositories/myapp/artifacts?with_tag=true" | jq '.[].tags[].name'

# Xoá artifact cụ thể
curl -X DELETE -u admin:password \
  "https://registry.internal/api/v2.0/projects/platform/repositories/myapp/artifacts/sha256:abc123"

# Backup harbor database
docker exec harbor-db pg_dump -U postgres registry > harbor-db-backup.sql

# Restore
cat harbor-db-backup.sql | docker exec -i harbor-db psql -U postgres registry
```

**Upgrade Harbor:**
```bash
# 1. Backup database
docker exec harbor-db pg_dumpall -U postgres > backup-$(date +%Y%m%d).sql

# 2. Stop Harbor
docker compose -f /data/harbor/docker-compose.yml down

# 3. Download new version
wget https://github.com/goharbor/harbor/releases/download/v2.12.0/harbor-offline-installer-v2.12.0.tgz
tar xvf harbor-offline-installer-v2.12.0.tgz
cd harbor

# 4. Copy harbor.yml từ old version, update nếu cần
cp /old/harbor/harbor.yml harbor.yml

# 5. Run migration
docker run -it --rm \
  -v /data/harbor/database:/var/lib/postgresql/data \
  goharbor/harbor-migrator:v2.12.0 migrate

# 6. Install mới
sudo ./prepare --with-trivy
docker compose -f /data/harbor/docker-compose.yml up -d
```

---

## Gotchas

- **GC và data loss**: xoá tag không ngay lập tức xoá blob (layer data). GC cần chạy riêng để reclaim space. Nếu GC fail giữa chừng → có thể orphaned blobs. Check GC logs sau mỗi run.
- **Replication và rate limiting**: pull từ Docker Hub bị rate limit (100 pulls/6h với anonymous, 200/6h với free account). Dùng authenticated replication endpoint.
- **Robot account token expire**: robot accounts có expiry (default 365 ngày). CI/CD break khi expire mà không có alert. Set reminder hoặc dùng no-expiry robots (cần Harbor admin).
- **LDAP group mapping**: group membership trong Harbor chỉ sync khi user login. Nếu user được thêm vào LDAP group → phải login lại để nhận permissions mới.
- **Scan on push và performance**: Trivy scan blocking push nếu policy yêu cầu. Large images (1GB+) có thể timeout. Điều chỉnh jobservice worker timeout trong harbor.yml.
- **Storage backend migration**: chuyển từ filesystem sang S3 cần dùng Harbor migration tool + downtime. Plan sớm khi cài đặt ban đầu.
- **Webhook không retry đủ**: Harbor webhook retry có giới hạn. Nếu CI endpoint down lâu → miss events. Implement idempotent CI trigger với polling backup.
