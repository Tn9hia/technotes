---
title: Vault Auth Methods & Policies
tags:
  - vault
  - auth
  - rbac
  - policies
date: 2026-04-27
---

# Vault Auth Methods & Policies

Auth Methods xác thực identity của client. Sau khi authenticate, Vault issue một Token với policies attached.

```bash
# List enabled auth methods
vault auth list

# Enable auth method
vault auth enable kubernetes
vault auth enable approle
vault auth enable oidc
```

---

## Policies — HCL

Policies định nghĩa **what** một token có thể làm, sau khi đã authenticate.

### Policy Syntax

```hcl
# /etc/vault/policies/myapp-production.hcl

# KV v2: đọc secrets
path "secret/data/production/myapp/*" {
  capabilities = ["read", "list"]
}

# KV v2: đọc metadata
path "secret/metadata/production/myapp/*" {
  capabilities = ["read", "list"]
}

# Database: request credentials
path "database/creds/myapp-readonly" {
  capabilities = ["read"]
}

# PKI: issue certificates
path "pki_int/issue/internal-services" {
  capabilities = ["create", "update"]
}

# Transit: encrypt/decrypt specific key
path "transit/encrypt/myapp-data" {
  capabilities = ["update"]
}
path "transit/decrypt/myapp-data" {
  capabilities = ["update"]
}

# Deny sensitive paths explicitly
path "secret/data/production/admin/*" {
  capabilities = ["deny"]
}
```

### Capabilities

| Capability | HTTP Method | Nghĩa |
|---|---|---|
| `create` | POST | Tạo mới (path chưa tồn tại) |
| `read` | GET | Đọc |
| `update` | POST/PUT | Update (path đã tồn tại) |
| `delete` | DELETE | Xóa |
| `list` | LIST | List keys |
| `sudo` | * | Admin operations (e.g., mount engines) |
| `deny` | * | Deny, overrides tất cả capabilities khác |

```hcl
# Wildcard paths
path "secret/data/production/*" {
  capabilities = ["read", "list"]
}

# Glob: match nhiều path segments
path "secret/data/+/myapp/*" {
  # + match một segment
  # → secret/data/production/myapp/config ✓
  # → secret/data/staging/myapp/config ✓
  # → secret/data/prod/staging/myapp/config ✗ (+ chỉ một segment)
  capabilities = ["read"]
}
```

### Policy Templating

```hcl
# Dùng identity thông tin trong policy
# entity.name: Vault identity entity name
# entity.aliases.<mount>.name: alias name từ specific mount

# Mỗi user chỉ đọc namespace của chính họ
path "secret/data/users/{{identity.entity.name}}/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# Dùng metadata
path "secret/data/teams/{{identity.entity.metadata.team}}/*" {
  capabilities = ["read", "list"]
}
```

```bash
# Create policy
vault policy write myapp-production /etc/vault/policies/myapp-production.hcl

# Read policy
vault policy read myapp-production

# List policies
vault policy list

# Delete policy
vault policy delete myapp-production
```

---

## Kubernetes Auth Method

Kubernetes auth cho phép pods authenticate với Vault dựa trên Service Account JWT token.

```
K8s Pod
  → có /var/run/secrets/kubernetes.io/serviceaccount/token (JWT)
  → gửi JWT tới Vault
  → Vault verify JWT với K8s API server
  → Vault check: namespace, service account, pod labels match role?
  → Issue Vault token với configured policies
```

### Setup

```bash
# Enable
vault auth enable kubernetes

# Configure: cho Vault biết K8s API server để verify JWTs
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token
  # token_reviewer_jwt: Vault dùng token này để call K8s TokenReview API

# Tạo role: bind K8s identity → Vault policies
vault write auth/kubernetes/role/myapp-production \
  bound_service_account_names="myapp" \
  bound_service_account_namespaces="production" \
  policies="myapp-production" \
  ttl=1h \
  max_ttl=24h

# Multiple service accounts / namespaces
vault write auth/kubernetes/role/monitoring \
  bound_service_account_names="prometheus,grafana" \
  bound_service_account_namespaces="monitoring" \
  policies="readonly-metrics" \
  ttl=24h
```

### Test từ trong Pod

```bash
# Pod có service account token
JWT=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# Authenticate
curl -s --request POST \
  --data "{\"jwt\": \"$JWT\", \"role\": \"myapp-production\"}" \
  https://vault.internal:8200/v1/auth/kubernetes/login \
  | jq '.auth.client_token'
```

---

## AppRole Auth Method

AppRole dành cho applications và CI/CD pipelines — không có K8s identity.

```
Concept:
  Role ID   = username (public, không sensitive)
  Secret ID = password (sensitive, short-lived)
  → Vault token

Separation of duties:
  Ops team: biết Role ID, tạo Secret ID, inject vào app
  App: dùng Role ID + Secret ID để authenticate
  → Không có người nào biết cả hai trừ app runtime
```

```bash
# Enable
vault auth enable approle

# Tạo role
vault write auth/approle/role/cicd-pipeline \
  secret_id_ttl=30m \        # Secret ID tự expire sau 30 phút
  secret_id_num_uses=1 \     # Secret ID chỉ dùng 1 lần
  token_policies="cicd-policy" \
  token_ttl=1h \
  token_max_ttl=4h \
  token_num_uses=0           # token có thể dùng unlimited times trong TTL

# Lấy Role ID (public, có thể share)
vault read auth/approle/role/cicd-pipeline/role-id
# → role_id: aBcDeF12-3456-7890-aBcD-eFgHiJkLmNoP

# Generate Secret ID (sensitive, dùng 1 lần)
vault write -f auth/approle/role/cicd-pipeline/secret-id
# → secret_id: xYzAbC12-3456-7890-xYzA-bCdEfGhIjKlM
# → secret_id_accessor: accessor để revoke nếu cần

# Authenticate
vault write auth/approle/login \
  role_id="aBcDeF12..." \
  secret_id="xYzAbC12..."
# → client_token: s.AbCdEfGhIjKl...

# Response wrapping: Secret ID được wrapped (extra security cho CI/CD)
vault write -wrap-ttl=30s -f auth/approle/role/cicd-pipeline/secret-id
# → wrapping_token: s.wrapping... (chỉ dùng được 1 lần trong 30s)
# CI/CD job unwrap này để lấy actual Secret ID
vault unwrap s.wrapping...
```

### CI/CD Pattern với AppRole

```yaml
# GitHub Actions — inject Secret ID lúc runtime
- name: Vault Login
  run: |
    ROLE_ID="${{ vars.VAULT_ROLE_ID }}"    # public, store in GitHub vars
    # Secret ID generated dynamically — bởi Vault-actions hoặc custom step

    # Option 1: dùng hashicorp/vault-action
    - uses: hashicorp/vault-action@v2
      with:
        url: https://vault.internal:8200
        method: approle
        roleId: ${{ vars.VAULT_ROLE_ID }}
        secretId: ${{ secrets.VAULT_SECRET_ID }}   # rotate định kỳ
        secrets: |
          secret/data/cicd/docker-registry registry_password | REGISTRY_PASSWORD;
          secret/data/cicd/deploy-key key | DEPLOY_KEY
```

---

## OIDC / JWT Auth Method

Dùng cho human users qua SSO (Keycloak, Okta, Google, GitHub Actions OIDC).

```bash
vault auth enable oidc

vault write auth/oidc/config \
  oidc_discovery_url="https://keycloak.internal/realms/myrealm" \
  oidc_client_id="vault" \
  oidc_client_secret="client-secret-from-keycloak" \
  default_role="developer"

vault write auth/oidc/role/developer \
  user_claim="sub" \
  groups_claim="groups" \        # map Keycloak groups → Vault groups
  allowed_redirect_uris="https://vault.internal:8200/ui/vault/auth/oidc/oidc/callback" \
  policies="developer-policy" \
  ttl=8h

# Login (opens browser)
vault login -method=oidc role=developer
```

### GitHub Actions OIDC (Keyless CI)

```bash
vault auth enable jwt

vault write auth/jwt/config \
  oidc_discovery_url="https://token.actions.githubusercontent.com" \
  bound_issuer="https://token.actions.githubusercontent.com"

vault write auth/jwt/role/github-actions-deploy \
  role_type="jwt" \
  user_claim="sub" \
  bound_claims='{
    "repository": "myorg/myapp",
    "ref": "refs/heads/main"
  }' \
  policies="deploy-policy" \
  ttl=15m
```

```yaml
# GitHub Actions workflow
- name: Vault Login (OIDC — no secret needed!)
  uses: hashicorp/vault-action@v2
  with:
    url: https://vault.internal:8200
    method: jwt
    role: github-actions-deploy
    jwtGithubAudience: "https://vault.internal"
    secrets: |
      secret/data/production/deploy token | DEPLOY_TOKEN
```

---

## Token Auth

Token auth là base — tất cả auth methods cuối cùng đều issue tokens.

```bash
# Tạo token manually
vault token create \
  -policy=myapp-production \
  -ttl=1h \
  -display-name="myapp-server-01"

# Token với use limit
vault token create \
  -policy=readonly \
  -use-limit=10 \      # chỉ dùng được 10 lần
  -ttl=24h

# Tạo orphan token (không bị revoke khi parent revoke)
vault token create -orphan -policy=long-running-service

# Renew token
vault token renew

# Revoke token
vault token revoke s.AbCdEfGhIjKl...

# Lookup token info
vault token lookup s.AbCdEfGhIjKl...
```

---

## Identity — Entity & Group

Vault Identity consolidates multiple auth backends thành một logical identity.

```bash
# Entity: đại diện cho một user/service
vault write identity/entity \
  name="john-doe" \
  metadata="team=backend" \
  metadata="role=developer"

# Alias: link auth backend identity → entity
vault write identity/entity-alias \
  name="john.doe@company.com" \    # OIDC email claim
  canonical_id="<entity-id>" \
  mount_accessor="<oidc-mount-accessor>"

# John authenticate qua OIDC → entity "john-doe" → policies từ entity

# Group
vault write identity/group \
  name="backend-team" \
  policies="backend-policy" \
  member_entity_ids="<entity-id-1>,<entity-id-2>"

# External group (sync từ OIDC groups claim)
vault write identity/group \
  name="backend-devs" \
  type="external" \
  policies="developer-policy"

vault write identity/group-alias \
  name="backend" \              # Keycloak group name
  canonical_id="<group-id>" \
  mount_accessor="<oidc-mount-accessor>"
```

---

## Gotchas

- **Policy deny order**: `deny` overrides tất cả. Nếu một policy deny và policy khác allow cùng path → deny thắng. Cẩn thận với wildcard policies.
- **KV v2 metadata vs data path**: Policy cho `secret/data/*` chỉ allow đọc values. Để list keys cần `secret/metadata/*`. Hay quên dẫn đến "permission denied" khi list.
- **AppRole Secret ID single-use**: `secret_id_num_uses=1` là good practice nhưng nếu app retry login (network error) → Secret ID đã consumed → fail. Cần generate new Secret ID. Tune `secret_id_num_uses=3` cho apps với retry logic.
- **Kubernetes auth và projected tokens**: K8s 1.21+ dùng projected service account tokens (bound tokens, auto-rotate). Vault cần `disable_iss_validation=true` nếu cluster không expose OIDC discovery endpoint. Check Vault K8s auth config carefully.
- **OIDC redirect URI**: URI trong Vault role config phải match **exactly** với URI trong OIDC provider (Keycloak). Extra slash, HTTP vs HTTPS → auth fail với cryptic error.
- **Token renewal và Vault Agent**: Vault Agent renew tokens tự động. Nhưng nếu Vault Agent restart và old token đã expire → Agent phải re-authenticate. Đảm bảo auth method (AppRole/K8s) vẫn hoạt động khi Agent restarts.
