---
title: HashiCorp Vault — Architecture & Operations
tags:
  - vault
  - secrets
  - security
  - operations
date: 2026-04-27
---

# Vault Architecture & Operations

## Core Concepts

```
┌─────────────────────────────────────────────────────────────┐
│  HashiCorp Vault                                             │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Auth Methods │  │Secret Engines│  │    Audit Devices │  │
│  │ - Kubernetes │  │ - KV v2      │  │ - File           │  │
│  │ - AppRole    │  │ - Database   │  │ - Syslog         │  │
│  │ - OIDC       │  │ - PKI        │  │ - Socket         │  │
│  │ - AWS IAM    │  │ - Transit    │  │                  │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────────────┘  │
│         │ token            │                                 │
│         ▼                  ▼                                 │
│  ┌─────────────────────────────────┐                        │
│  │           Policy Engine         │                        │
│  │  HCL policies: path + capabilities                       │
│  └────────────────┬────────────────┘                        │
│                   │                                         │
│  ┌────────────────▼────────────────┐                        │
│  │         Storage Backend         │                        │
│  │  - Integrated (Raft) — prod     │                        │
│  │  - Consul, etcd, S3 (legacy)    │                        │
│  └─────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

### Luồng cơ bản

```
Client (app / K8s pod / CI job)
  │
  ├─ 1. Authenticate với Auth Method
  │       → nhận Vault Token (có TTL, policies attached)
  │
  ├─ 2. Request secret/operation bằng Token
  │       → Vault check token's policies
  │
  └─ 3. Receive secret hoặc operation result
```

### Vault Token

```
Token là unit of identity trong Vault.
  - TTL: tokens tự expire
  - Use-limit: token có thể limit số lần dùng
  - Policies: list of HCL policies attached
  - Renewable: có thể renew trước khi expire
  - Orphan vs child tokens: child token bị revoke khi parent bị revoke
```

---

## Storage Backend — Integrated Raft

Raft (Integrated Storage) là recommended production storage backend từ Vault 1.4+. Không cần external Consul/etcd.

```
3-node Vault cluster với Raft:

  vault-1 (leader) ─── Raft consensus ───┐
  vault-2 (follower) ─────────────────────┤
  vault-3 (follower) ─────────────────────┘

  Leader handles all reads/writes
  Followers: passive replication
  Quorum = (n/2) + 1 — tolerate floor(n/2) failures
```

---

## Installation & Configuration

### Docker Compose (Dev/Lab)

```yaml
# docker-compose.yml
services:
  vault:
    image: hashicorp/vault:1.17
    cap_add:
      - IPC_LOCK          # prevent swap (mlock)
    ports:
      - "8200:8200"
    environment:
      VAULT_ADDR: http://0.0.0.0:8200
      VAULT_LOCAL_CONFIG: |
        ui = true
        listener "tcp" {
          address     = "0.0.0.0:8200"
          tls_disable = 1   # dev only
        }
        storage "file" {
          path = "/vault/data"
        }
    volumes:
      - vault-data:/vault/data
    command: server
```

### Production Config (Raft HA)

```hcl
# /etc/vault/vault.hcl — node 1 của 3-node cluster

ui = true

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/etc/vault/tls/vault.crt"
  tls_key_file  = "/etc/vault/tls/vault.key"
  # mutual TLS (optional)
  # tls_client_ca_file = "/etc/vault/tls/ca.crt"
  # tls_require_and_verify_client_cert = true
}

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-1"               # unique per node

  retry_join {
    leader_api_addr = "https://vault-1.internal:8200"
  }
  retry_join {
    leader_api_addr = "https://vault-2.internal:8200"
  }
  retry_join {
    leader_api_addr = "https://vault-3.internal:8200"
  }
}

api_addr     = "https://vault-1.internal:8200"    # this node's address
cluster_addr = "https://vault-1.internal:8201"    # Raft cluster communication

# Telemetry for Prometheus
telemetry {
  prometheus_retention_time = "30s"
  disable_hostname          = true
}

# Auto-unseal with AWS KMS
seal "awskms" {
  region     = "ap-southeast-1"
  kms_key_id = "arn:aws:kms:ap-southeast-1:123456789:key/abc123"
}
```

---

## Initialization & Unseal

### Manual Init (first time)

```bash
export VAULT_ADDR=https://vault-1.internal:8200

# Initialize Vault (chỉ làm 1 lần cho toàn cluster)
vault operator init \
  -key-shares=5 \          # 5 unseal key shards (Shamir's secret sharing)
  -key-threshold=3         # cần 3 trong 5 shards để unseal

# Output:
# Unseal Key 1: aBcDeFgHiJkLmN...
# Unseal Key 2: oPqRsTuVwXyZaB...
# Unseal Key 3: cDeFgHiJkLmNoP...
# Unseal Key 4: qRsTuVwXyZaBcD...
# Unseal Key 5: eFgHiJkLmNoPqR...
# Initial Root Token: s.XXXXXXXXXXXXXXXXXXXXXXXX

# QUAN TRỌNG: Lưu root token và unseal keys vào secure location
# Phân phát keys cho các people khác nhau (mỗi người giữ 1-2 keys)
# Root token: chỉ dùng cho initial setup, sau đó revoke!
```

### Manual Unseal (sau restart — không có auto-unseal)

```bash
# Cần chạy 3 lần với 3 keys khác nhau
vault operator unseal aBcDeFgHiJkLmN...
vault operator unseal oPqRsTuVwXyZaB...
vault operator unseal cDeFgHiJkLmNoP...

# Xem status
vault status
# Sealed: false
# HA Enabled: true
# HA Mode: active
```

### Auto-Unseal (Production — recommended)

Auto-unseal dùng cloud KMS để decrypt master key — không cần manual unseal sau restart.

```hcl
# Config đã có trong vault.hcl (ở trên)
seal "awskms" {
  region     = "ap-southeast-1"
  kms_key_id = "arn:aws:kms:ap-southeast-1:123456789:key/abc123"
}

# Với GCP KMS
seal "gcpckms" {
  project    = "my-project"
  region     = "asia-southeast1"
  key_ring   = "vault-key-ring"
  crypto_key = "vault-unseal-key"
}

# Với PKCS11 (on-premise HSM)
seal "pkcs11" {
  lib            = "/usr/lib/softhsm/libsofthsm2.so"
  slot           = "0"
  pin            = "1234"
  key_label      = "vault-hsm-key"
  hmac_key_label = "vault-hsm-hmac"
}
```

```bash
# Với auto-unseal, init dùng recovery keys (thay vì unseal keys)
vault operator init \
  -recovery-shares=5 \
  -recovery-threshold=3
# Recovery keys chỉ cần để emergency: nếu KMS mất access
```

---

## Cluster Operations

### Join Raft Cluster

```bash
# Node đầu tiên tự init
vault operator init ...

# Nodes còn lại join cluster (thay vì init)
vault operator raft join https://vault-1.internal:8200

# Xem cluster peers
vault operator raft list-peers
# Node ID     Address                  State     Voter
# vault-1     vault-1.internal:8201    leader    true
# vault-2     vault-2.internal:8201    follower  true
# vault-3     vault-3.internal:8201    follower  true
```

### Snapshot (Backup)

```bash
# Raft snapshot = full cluster state backup
vault operator raft snapshot save /backup/vault-$(date +%Y%m%d).snap

# Automated backup
cat > /etc/cron.daily/vault-backup <<'EOF'
#!/bin/bash
export VAULT_ADDR=https://vault.internal:8200
export VAULT_TOKEN=$(cat /etc/vault/backup-token)

vault operator raft snapshot save /backup/vault-$(date +%Y%m%d-%H%M).snap
aws s3 cp /backup/vault-$(date +%Y%m%d-%H%M).snap s3://my-vault-backup/
find /backup -name "vault-*.snap" -mtime +7 -delete
EOF
chmod +x /etc/cron.daily/vault-backup

# Restore từ snapshot
vault operator raft snapshot restore /backup/vault-20260427.snap
```

---

## Namespaces (Enterprise)

Namespaces tạo isolated Vault environments trong cùng cluster. CE edition không có namespaces.

```bash
# Tạo namespace cho team/project
vault namespace create dev-team
vault namespace create production

# Operations trong namespace
export VAULT_NAMESPACE=production
vault kv put secret/myapp password=supersecret
# hoặc
vault kv put -namespace=production secret/myapp password=supersecret
```

---

## Audit Logging

```bash
# Enable file audit (quan trọng — log tất cả operations)
vault audit enable file file_path=/var/log/vault/audit.log

# Enable syslog (để ship tới SIEM)
vault audit enable syslog

# Format: JSON, mỗi dòng một request/response
# Sensitive values bị hashed (HMAC-SHA256) trong audit log
# → không thấy raw passwords/tokens, nhưng có thể verify nếu có hash key

# Check audit devices
vault audit list -detailed

# QUAN TRỌNG: nếu tất cả audit devices fail → Vault từ chối tất cả operations
# Đây là design choice (fail secure)
```

---

## Monitoring với Prometheus

```bash
# Vault expose metrics tại /v1/sys/metrics
# Hoặc /metrics endpoint nếu prometheus_retention_time configured

# Prometheus scrape config
- job_name: vault
  metrics_path: /v1/sys/metrics
  params:
    format: [prometheus]
  bearer_token: <monitoring-token>
  static_configs:
    - targets: [vault.internal:8200]
  scheme: https
```

```yaml
# Key alerts
- alert: VaultSealed
  expr: vault_core_unsealed == 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Vault is sealed on {{ $labels.instance }}"

- alert: VaultLeaderElection
  expr: changes(vault_core_active[5m]) > 0
  labels:
    severity: warning
  annotations:
    summary: "Vault leader changed (possible failover)"

- alert: VaultTokenTTLExpiring
  expr: vault_token_count_by_ttl{creation_ttl="1h"} > 100
  labels:
    severity: info
```

---

## Best Practices

```
1. Root token:
   Chỉ dùng lúc initial setup. Sau đó tạo admin token với policies cụ thể.
   Revoke root token khi không cần: vault token revoke <root-token>
   Root token có thể regenerate khi cần: vault operator generate-root

2. Unseal keys / Recovery keys:
   Encrypt mỗi key với GPG key của người giữ
   Lưu ở nhiều locations khác nhau (safe, vault manager, HSM)
   Test recovery procedure định kỳ

3. Token TTL:
   Service tokens: 1h-24h với auto-renewal
   CI/CD tokens: short TTL (15m-1h), no renewal
   Human tokens: 8h-24h với MFA

4. Lease renewal:
   Secrets có lease (TTL). Client phải renew trước khi expire.
   Vault Agent tự handle renewal — không cần code application

5. Namespace isolation:
   Mỗi team/project có namespace riêng với policies riêng
   Cross-namespace: dùng identity aliases
```

---

## Gotchas

- **Mlock và swap**: Vault dùng `mlock` để prevent secrets từ được swap ra disk. Cần `IPC_LOCK` capability hoặc `sudo setcap cap_ipc_lock=+ep vault`. Nếu không có mlock → secrets có thể trong swap file.
- **Auto-unseal và KMS availability**: Nếu AWS KMS unreachable (VPC endpoint down, IAM role expired) → Vault không thể unseal sau restart. Cần monitor KMS connectivity và có emergency plan.
- **Raft performance**: Raft leader handle tất cả writes. Write-heavy workloads → leader bottleneck. Scale horizontally = thêm follower nodes giúp reads nhưng không giúp writes.
- **Audit log và disk**: Vault từ chối operations nếu audit log không ghi được. Nếu disk đầy → Vault bị lock. Monitor `/var/log/vault/` disk usage.
- **Token renewal race condition**: Client phải renew token trước khi expire. Nếu renew quá gần expire (network delay, retry) → token already expired → connection fail. Vault Agent renew ở 2/3 của TTL lifecycle.
