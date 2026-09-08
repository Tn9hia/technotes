---
title: HashiCorp Vault
tags:
  - vault
  - secrets
  - security
  - index
date: 2026-04-27
---

# HashiCorp Vault

Vault là secrets management platform — single source of truth cho tất cả credentials, certificates, và sensitive configuration trong hệ thống.

## Contents

- [[Architecture & Operations]] — Core concepts (Auth/Engines/Policies), Raft HA cluster, initialization, manual vs auto-unseal (AWS KMS/GCP KMS/HSM), snapshot backup, audit logging, monitoring
- [[Secret Engines]] — KV v2 (versioning, patch, metadata), Database dynamic credentials (PostgreSQL/MySQL), PKI (Root CA → Intermediate CA → leaf certs, cert-manager integration), Transit (encryption-as-a-service), SSH OTP
- [[Auth Methods & Policies]] — HCL policy syntax (capabilities, wildcards, templating), Kubernetes auth, AppRole (CI/CD pattern, response wrapping), OIDC/JWT (Keycloak, GitHub Actions keyless), Identity entities & groups
- [[Kubernetes Integration]] — Vault Agent Injector (annotations, Go templates, full annotation reference), Vault CSI Provider, ESO + Vault deep dive, Vault + Ansible, Vault + Terraform

---

## Quick Reference

### Vault CLI Essentials

```bash
export VAULT_ADDR=https://vault.internal:8200
export VAULT_TOKEN=s.AbCdEfGhIjKl...

# Status
vault status

# Login (với token)
vault login s.AbCdEfGhIjKl...

# KV operations
vault kv get secret/production/myapp
vault kv put secret/production/myapp key=value
vault kv patch secret/production/myapp key=new_value
vault kv delete secret/production/myapp

# Dynamic DB creds
vault read database/creds/myapp-readonly

# PKI cert
vault write pki_int/issue/internal-services \
  common_name="myapp.internal" ttl=720h

# Token info
vault token lookup
vault token renew

# Auth list
vault auth list
vault secrets list
vault policy list
```

### Setup Checklist (New Vault Cluster)

```
1. [ ] Install với Raft HA (3 nodes minimum)
2. [ ] Configure TLS (không dùng tls_disable trên production)
3. [ ] Initialize cluster: vault operator init -recovery-shares=5 -recovery-threshold=3
4. [ ] Configure auto-unseal (AWS KMS / GCP KMS)
5. [ ] Enable audit log: vault audit enable file file_path=/var/log/vault/audit.log
6. [ ] Enable auth methods: kubernetes, approle, oidc
7. [ ] Create admin policy (non-root): vault policy write admin admin.hcl
8. [ ] Revoke root token sau khi setup xong
9. [ ] Configure monitoring: prometheus metrics, alerts cho seal/leader-election
10. [ ] Test DR: snapshot + restore, auto-unseal recovery
```

### Policy Template Snippet

```hcl
# Minimal app policy
path "secret/data/production/{{app_name}}/*" {
  capabilities = ["read", "list"]
}
path "secret/metadata/production/{{app_name}}/*" {
  capabilities = ["read", "list"]
}
path "database/creds/{{app_name}}-readonly" {
  capabilities = ["read"]
}
```

---

## Architecture Overview

```
Developers / CI/CD         Kubernetes Pods           Ansible / Terraform
     │                          │                            │
     │ OIDC/AppRole              │ K8s Auth (JWT SA)          │ AppRole/Token
     ▼                          ▼                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                      HashiCorp Vault Cluster                      │
│                                                                  │
│  Auth Methods    Secret Engines        Policies                  │
│  - OIDC          - KV v2               - HCL path-based          │
│  - AppRole       - Database            - Identity groups         │
│  - Kubernetes    - PKI                 - Policy templating       │
│  - JWT           - Transit                                       │
│                                                                  │
│  Storage: Integrated Raft (3 nodes, auto-unseal via KMS)        │
└──────────────────────────────────────────────────────────────────┘
```

---

## Cross-References

- Vault Agent Injector trong K8s pods: [[Secrets Management]]
- ESO + Vault: [[Secrets Management]]
- Dynamic DB creds cho PostgreSQL: [[Operations & Security]]
- PKI + cert-manager: [[Network Security]]
- Vault trong DevSecOps pipeline: [[DevSecOps]]
