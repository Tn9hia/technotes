---
title: Vault — Kubernetes Integration
tags:
  - vault
  - kubernetes
  - agent-injector
  - csi
  - eso
date: 2026-04-27
---

# Vault — Kubernetes Integration

## So Sánh 3 Approaches

```
┌─────────────────────────────────────────────────────────────────────┐
│ Approach          │ How                    │ Best for               │
├───────────────────┼────────────────────────┼────────────────────────┤
│ Vault Agent       │ Sidecar inject secrets  │ Existing apps, file-  │
│ Injector          │ vào file/env trong pod  │ based config           │
├───────────────────┼────────────────────────┼────────────────────────┤
│ Vault CSI         │ Secrets mount như       │ Apps expect filesystem │
│ Provider          │ volume, no sidecar      │ volume, không muốn    │
│                   │                         │ sidecar overhead       │
├───────────────────┼────────────────────────┼────────────────────────┤
│ ESO + Vault       │ Vault → K8s Secret      │ Apps dùng K8s Secrets, │
│                   │ (sync via operator)     │ GitOps friendly        │
└───────────────────┴────────────────────────┴────────────────────────┘
```

---

## Vault Agent Injector

Vault Agent Injector là mutating webhook — tự động inject Vault Agent sidecar vào pods có annotations.

### Install

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

helm install vault hashicorp/vault \
  -n vault \
  --create-namespace \
  --set "server.ha.enabled=true" \
  --set "server.ha.replicas=3" \
  --set "server.ha.raft.enabled=true" \
  --set "server.ha.raft.setNodeId=true" \
  --set "injector.enabled=true" \
  --set "injector.replicas=2" \     # 2 injector replicas cho HA
  --set "server.ingress.enabled=true" \
  --set "server.ingress.hosts[0].host=vault.internal"
```

### Basic Injection

```yaml
# Pod annotations → Vault Agent Injector tự inject sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    metadata:
      annotations:
        # Enable injection
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp-production"
        vault.hashicorp.com/tls-skip-verify: "false"   # verify Vault TLS

        # Inject secret vào file /vault/secrets/config
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/production/myapp/config"

    spec:
      serviceAccountName: myapp    # phải match K8s auth role
      containers:
        - name: app
          image: myapp:v1
          # Secret available tại /vault/secrets/config
```

**Secret file mặc định format:**
```
key=value
db_password=supersecret
api_key=abc123
```

### Custom Template

```yaml
# Dùng Go template để format secret file
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "myapp-production"

  # Secret 1: Database connection string
  vault.hashicorp.com/agent-inject-secret-db.env: "secret/data/production/myapp/database"
  vault.hashicorp.com/agent-inject-template-db.env: |
    {{- with secret "secret/data/production/myapp/database" -}}
    export DB_HOST="{{ .Data.data.host }}"
    export DB_USER="{{ .Data.data.username }}"
    export DB_PASS="{{ .Data.data.password }}"
    export DB_NAME="{{ .Data.data.dbname }}"
    {{- end }}

  # Secret 2: API keys file
  vault.hashicorp.com/agent-inject-secret-api-keys.json: "secret/data/production/myapp/api"
  vault.hashicorp.com/agent-inject-template-api-keys.json: |
    {{- with secret "secret/data/production/myapp/api" -}}
    {
      "stripe_key": "{{ .Data.data.stripe_key }}",
      "sendgrid_key": "{{ .Data.data.sendgrid_key }}"
    }
    {{- end }}

  # Dynamic DB credentials
  vault.hashicorp.com/agent-inject-secret-db-creds.txt: "database/creds/myapp-readonly"
  vault.hashicorp.com/agent-inject-template-db-creds.txt: |
    {{- with secret "database/creds/myapp-readonly" -}}
    postgres://{{ .Data.username }}:{{ .Data.password }}@postgres:5432/myapp
    {{- end }}
```

```bash
# Dùng trong app — source env file hoặc đọc từ file
command: ["sh", "-c", "source /vault/secrets/db.env && ./myapp"]

# Hoặc app đọc file trực tiếp
# /vault/secrets/db-creds.txt: postgres://v-k8s-abc:pass@postgres:5432/myapp
```

### Vault Agent Annotations — Full Reference

```yaml
annotations:
  # ─── Basic ───
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "myapp-production"
  vault.hashicorp.com/agent-pre-populate-only: "false"   # true: only init container, no sidecar

  # ─── Vault connection ───
  vault.hashicorp.com/agent-configmap: ""                # use custom configmap
  vault.hashicorp.com/tls-secret: "vault-tls"            # TLS cert secret cho verify

  # ─── Secret volume ───
  vault.hashicorp.com/secret-volume-path: "/vault/secrets"   # mount point

  # ─── Resource limits ───
  vault.hashicorp.com/agent-limits-cpu: "500m"
  vault.hashicorp.com/agent-limits-mem: "128Mi"
  vault.hashicorp.com/agent-requests-cpu: "100m"
  vault.hashicorp.com/agent-requests-mem: "64Mi"

  # ─── Init vs sidecar ───
  # Init container: populate secrets trước khi app container start
  # Sidecar: keep running để renew dynamic secrets (DB creds, etc.)
  vault.hashicorp.com/agent-inject-containers: "app"    # chỉ inject vào specific container

  # ─── Log level ───
  vault.hashicorp.com/log-level: "info"
```

### Vault Agent Config via ConfigMap

```yaml
# Cho complex configs — thay vì nhiều annotations
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-vault-config
  namespace: production
data:
  config.hcl: |
    vault {
      address = "https://vault.internal:8200"
    }

    auto_auth {
      method "kubernetes" {
        mount_path = "auth/kubernetes"
        config = {
          role = "myapp-production"
        }
      }

      sink "file" {
        config = {
          path = "/vault/secrets/.token"
        }
      }
    }

    template {
      source      = "/vault/templates/db.tmpl"
      destination = "/vault/secrets/db.env"
      perms       = "0640"
      command     = "pkill -HUP myapp"   # signal app để reload (optional)
    }

    template {
      source      = "/vault/templates/app.tmpl"
      destination = "/vault/secrets/app.env"
    }
```

---

## Vault CSI Provider

Vault CSI Provider mount secrets trực tiếp như volume — không có sidecar. Dùng Secrets Store CSI Driver.

```bash
# Install Secrets Store CSI Driver
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  -n kube-system \
  --set syncSecret.enabled=true   # sync CSI secrets → K8s Secret (optional)

# Install Vault CSI Provider
helm install vault hashicorp/vault \
  -n vault \
  --set "server.enabled=false" \    # chỉ install CSI provider (server đã có)
  --set "csi.enabled=true"
```

```yaml
# SecretProviderClass — định nghĩa secrets cần mount
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: myapp-secrets
  namespace: production
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.internal:8200"
    roleName: "myapp-production"
    objects: |
      - objectName: "db-password"
        secretPath: "secret/data/production/myapp/database"
        secretKey: "password"
      - objectName: "api-key"
        secretPath: "secret/data/production/myapp/api"
        secretKey: "stripe_key"

  # Sync to K8s Secret (optional)
  secretObjects:
    - secretName: myapp-synced-secrets
      type: Opaque
      data:
        - objectName: db-password
          key: DB_PASSWORD
        - objectName: api-key
          key: STRIPE_KEY

---
# Pod sử dụng CSI volume
spec:
  serviceAccountName: myapp
  containers:
    - name: app
      image: myapp:v1
      volumeMounts:
        - name: vault-secrets
          mountPath: "/mnt/secrets"
          readOnly: true
      env:
        # Nếu dùng secretObjects sync
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-synced-secrets
              key: DB_PASSWORD

  volumes:
    - name: vault-secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: myapp-secrets
```

---

## External Secrets Operator + Vault

ESO sync Vault secrets → K8s Secrets. Đã cover trong K8s Secrets Management, đây là Vault-specific deep dive.

Cross-reference: [[Secrets Management]]

```yaml
# ClusterSecretStore với Vault
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-cluster-store
spec:
  provider:
    vault:
      server: "https://vault.internal:8200"
      path: "secret"       # KV v2 mount path
      version: "v2"
      caBundle: <base64-ca>
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "eso-production"
          serviceAccountRef:
            name: eso-service-account
            namespace: external-secrets
```

```yaml
# ExternalSecret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
  namespace: production
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: vault-cluster-store
    kind: ClusterSecretStore
  target:
    name: myapp-secrets    # K8s Secret name
    creationPolicy: Owner
    template:
      type: Opaque
      data:
        # Transform: tạo connection string từ multiple fields
        DATABASE_URL: "postgresql://{{ .username }}:{{ .password }}@{{ .host }}/myapp"
  dataFrom:
    - extract:
        key: production/myapp/database   # Vault path (không cần "secret/data/" prefix)
```

---

## Vault Namespace per Team (On-Premise Pattern)

Không có Vault Enterprise namespaces, dùng path-based isolation với strict policies:

```
Vault paths layout:
  secret/
    production/
      myapp/
      payments/
    staging/
      myapp/
  database/
    production/
      myapp-pg
    staging/
      myapp-pg

Auth roles:
  myapp-prod: bound to namespace=production SA=myapp → policies=[prod-myapp]
  myapp-stage: bound to namespace=staging SA=myapp → policies=[stage-myapp]

Policies:
  prod-myapp:   path "secret/data/production/myapp/*" { capabilities = ["read"] }
  stage-myapp:  path "secret/data/staging/myapp/*" { capabilities = ["read","create","update"] }
```

---

## Vault + Ansible

```yaml
# Ansible dùng community.hashi_vault collection
- name: Get database password from Vault
  community.hashi_vault.vault_kv2_get:
    url: https://vault.internal:8200
    auth_method: approle
    role_id: "{{ vault_role_id }}"
    secret_id: "{{ vault_secret_id }}"
    path: production/myapp/database
    mount_point: secret
  register: vault_secret

- name: Use the secret
  ansible.builtin.debug:
    msg: "DB host: {{ vault_secret.secret.host }}"
```

```bash
# Vault CLI trong Ansible (alternative)
- name: Read Vault secret via CLI
  ansible.builtin.command: >
    vault kv get -field=password secret/production/myapp/database
  environment:
    VAULT_ADDR: https://vault.internal:8200
    VAULT_TOKEN: "{{ vault_token }}"
  register: db_password
  no_log: true    # QUAN TRỌNG: không log sensitive output
```

---

## Vault + Terraform

```hcl
# provider.tf
terraform {
  required_providers {
    vault = {
      source  = "hashicorp/vault"
      version = "~> 4.0"
    }
  }
}

provider "vault" {
  address = "https://vault.internal:8200"
  # Auth via VAULT_TOKEN env var hoặc:
  auth_login {
    path = "auth/approle/login"
    parameters = {
      role_id   = var.vault_role_id
      secret_id = var.vault_secret_id
    }
  }
}

# Read secret
data "vault_kv_secret_v2" "db_credentials" {
  mount = "secret"
  name  = "production/database"
}

# Dùng trong resource
resource "kubernetes_secret" "db_secret" {
  data = {
    password = data.vault_kv_secret_v2.db_credentials.data["password"]
  }
}

# Write secret (Terraform managing secrets — cẩn thận với state!)
resource "vault_kv_secret_v2" "app_config" {
  mount = "secret"
  name  = "production/myapp/config"
  data_json = jsonencode({
    endpoint = "https://api.internal"
    timeout  = "30s"
  })
}
# LƯU Ý: sensitive values trong Terraform state → cần remote state + encryption
```

---

## Gotchas

- **Injector init container ordering**: Vault Agent init container chạy trước app container. Nếu Vault unreachable → init fail → pod stuck in Init:0/1. Cần Vault HA và proper network policies.
- **CSI và secret refresh**: CSI provider mount secrets tại pod start time. Secrets không tự update khi Vault value thay đổi (không như Agent sidecar). Cần pod restart hoặc dùng `autoRotation` feature (beta).
- **ESO và Vault token TTL**: ESO dùng Vault token để authenticate. Token có TTL — ESO tự renew, nhưng nếu ESO pod restart và token expired → phải re-authenticate. Cần đảm bảo auth method (K8s SA) vẫn valid.
- **`no_log: true` trong Ansible**: Khi đọc secrets từ Vault trong Ansible tasks, luôn thêm `no_log: true`. Thiếu → secret value xuất hiện trong Ansible logs/callbacks → exposed.
- **Terraform state và secrets**: Nếu Vault provider write secrets vào Vault, Vault provider resource state trong tfstate **không** chứa secret values (chỉ chứa path). Nhưng nếu dùng `vault_kv_secret` để read và pass vào other resources → value có thể appear in tfstate. Dùng `sensitive = true` và remote state với encryption.
- **Injector và namespace restrictions**: Vault injector webhook có thể configure để chỉ watch specific namespaces. Check `injector.namespaceSelector` trong Helm values — nếu không config → inject vào tất cả namespaces kể cả kube-system (dangerous).
