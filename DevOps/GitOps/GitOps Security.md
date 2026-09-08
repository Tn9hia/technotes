---
title: GitOps Security
tags:
  - gitops
  - security
  - secrets
  - deep-dive
date: 2026-04-26
---

# GitOps Security

## Threat Model — Ai tấn công vào đâu?

```
                    ┌──────────────────┐
  Developer  ──────►│   App Code Repo  │
                    └────────┬─────────┘
                             │ CI: build image
                    ┌────────▼─────────┐
  Attacker ────────►│  Image Registry  │◄── image poisoning
                    └────────┬─────────┘
                             │ update tag
  Attacker ────────►┌────────▼─────────┐
                    │  Manifest Repo   │◄── malicious commit
                    └────────┬─────────┘
                             │ ArgoCD watches
  Attacker ────────►┌────────▼─────────┐
                    │     ArgoCD       │◄── API abuse, RBAC bypass
                    └────────┬─────────┘
                             │ sync
                    ┌────────▼─────────┐
                    │ K8s Cluster      │◄── escalation, escape
                    └──────────────────┘
```

**Attack vectors chính:**
1. **Malicious code/image** — supply chain attack
2. **Plaintext secrets trong Git** — secret leakage
3. **Manifest repo compromise** — ai đó push malicious manifest
4. **ArgoCD API abuse** — unauthorized sync, RBAC bypass
5. **Cluster privilege escalation** — qua ArgoCD deploy malicious workload

---

## 1. Secret Management — Đừng bao giờ commit secret plain text

### Option A: Sealed Secrets (Bitnami)

Encrypt secret bằng public key của controller trong cluster. Chỉ controller đó mới decrypt được.

```
Developer                    Cluster
    │                           │
    │  kubeseal encrypt         │
    │  (dùng public key)        │
    ▼                           │
SealedSecret YAML ─── Git ──►  │
(encrypted, safe to commit)    │
                                │ SealedSecrets controller
                                │ decrypt bằng private key
                                ▼
                           Kubernetes Secret
```

**Setup:**

```bash
# Cài controller vào cluster
helm install sealed-secrets sealed-secrets/sealed-secrets \
  -n kube-system

# Lấy public key
kubeseal --fetch-cert \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  > pub-cert.pem

# Encrypt secret
kubectl create secret generic db-creds \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml | \
  kubeseal --cert pub-cert.pem \
  --format yaml > db-creds-sealed.yaml

# Commit db-creds-sealed.yaml vào Git — an toàn
```

**SealedSecret YAML:**

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-creds
  namespace: myapp
spec:
  encryptedData:
    password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEQAx...
  template:
    metadata:
      name: db-creds
      namespace: myapp
    type: Opaque
```

**Scope của SealedSecret:**

```bash
# strict (default): chỉ decrypt được trong đúng namespace + name
kubeseal --scope strict

# namespace-wide: decrypt được với bất kỳ name nào trong namespace đó
kubeseal --scope namespace-wide

# cluster-wide: decrypt được ở bất kỳ namespace nào
kubeseal --scope cluster-wide
```

**Key rotation:**

```bash
# Controller tự rotate key mỗi 30 ngày mặc định
# Old keys vẫn giữ để decrypt SealedSecrets cũ
# Xem keys hiện tại:
kubectl get secrets -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key
```

---

### Option B: External Secrets Operator (ESO)

Pull secret từ external secret store (Vault, AWS Secrets Manager, GCP Secret Manager) vào K8s Secret.

```
┌──────────────────┐    ESO pulls     ┌─────────────────┐
│  HashiCorp Vault │ ◄──────────────  │  ESO Controller │
│  AWS SM          │    (in-cluster)  │                 │
│  GCP SM          │                  └────────┬────────┘
│  Azure Key Vault │                           │ creates
└──────────────────┘                           ▼
                                         K8s Secret
```

**CRDs:**

```yaml
# SecretStore — kết nối đến secret backend
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: myapp
spec:
  provider:
    vault:
      server: "https://vault.internal.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp-role"
          serviceAccountRef:
            name: myapp-sa
---
# ExternalSecret — chỉ định secret nào cần pull
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: myapp
spec:
  refreshInterval: 1h       # re-sync mỗi 1 giờ
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-creds           # tên K8s Secret tạo ra
    creationPolicy: Owner
  data:
    - secretKey: password    # key trong K8s Secret
      remoteRef:
        key: myapp/db        # path trong Vault
        property: password   # field trong Vault secret
```

**ClusterSecretStore** — dùng cho toàn cluster, không phải per-namespace:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-cluster-backend
spec:
  provider:
    vault:
      server: "https://vault.internal.com"
      # ...
```

---

### Option C: SOPS (Secrets OPerationS)

Encrypt file trực tiếp bằng GPG / age / AWS KMS / GCP KMS. File encrypted commit vào Git.

```bash
# Cài sops
brew install sops  # hoặc download binary

# Tạo age key
age-keygen -o age.key
# public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p

# Encrypt file
sops --encrypt \
  --age age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p \
  secrets.yaml > secrets.enc.yaml

# Decrypt
SOPS_AGE_KEY_FILE=age.key sops --decrypt secrets.enc.yaml

# Edit encrypted file trực tiếp
SOPS_AGE_KEY_FILE=age.key sops secrets.enc.yaml
```

**.sops.yaml — config mặc định:**

```yaml
creation_rules:
  - path_regex: .*/prod/.*\.yaml
    age: "age1prod..."        # prod key
  - path_regex: .*/staging/.*\.yaml
    age: "age1staging..."     # staging key
  - path_regex: .*\.yaml
    age: "age1dev..."         # dev key mặc định
```

**Kết hợp SOPS + ArgoCD** dùng [argocd-vault-plugin](https://argocd-vault-plugin.readthedocs.io/) hoặc Helm secrets plugin.

---

### So sánh 3 options

| | Sealed Secrets | ESO | SOPS |
|---|---|---|---|
| Secret lưu ở | Git (encrypted) | External store | Git (encrypted) |
| Dependency | Controller trong cluster | Controller + external store | age/GPG key management |
| Rotation | Thủ công re-seal | Auto (refreshInterval) | Thủ công re-encrypt |
| Audit | Git history | Vault audit log | Git history |
| Phù hợp | Self-contained, simple | Enterprise, đã có Vault/SM | Không muốn external dependency |
| Gotcha | Key backup quan trọng | ESO controller là SPOF | Key distribution |

---

## 2. Git Repository Security

### Branch Protection

```
main (production manifests)
  ├── Require PR review: ≥ 2 approvers
  ├── Require status checks (CI/lint)
  ├── No direct push — kể cả admin
  ├── Require signed commits (GPG/SSH)
  └── Dismiss stale reviews khi có push mới

staging
  ├── Require PR review: ≥ 1 approver
  └── Require status checks

dev
  └── Chỉ require status checks (không cần review)
```

### Commit Signing

```bash
# Setup GPG signing
git config --global user.signingkey <KEY-ID>
git config --global commit.gpgsign true

# Hoặc SSH signing (Git 2.34+)
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgsign true

# Verify commit signature
git log --show-signature
```

**GitHub/GitLab**: enforce "Require signed commits" trong branch protection.

### Deploy Keys vs Personal Access Tokens

ArgoCD cần read access vào manifest repo:

```bash
# Option 1: Deploy key (SSH) — per-repo, read-only
ssh-keygen -t ed25519 -C "argocd@cluster-prod" -f argocd-deploy-key
# Thêm public key vào repo Settings → Deploy Keys (read-only)
# Thêm private key vào ArgoCD

# Option 2: GitHub App — granular permissions, không expire
# Tạo GitHub App với Repository: Contents (read)
# ArgoCD hỗ trợ GitHub App authentication

# KHÔNG dùng Personal Access Token của người thật
# → khi người đó nghỉ việc, token revoke → ArgoCD mất access
```

---

## 3. ArgoCD Security Hardening

### Authentication

```yaml
# argocd-cm ConfigMap
data:
  # Tắt local admin user (dùng SSO thay thế)
  admin.enabled: "false"

  # OIDC với GitHub/Okta/Keycloak
  oidc.config: |
    name: Okta
    issuer: https://myorg.okta.com/oauth2/default
    clientID: $oidc.okta.clientID
    clientSecret: $oidc.okta.clientSecret
    requestedScopes:
      - openid
      - profile
      - email
      - groups
    requestedIDTokenClaims:
      groups:
        essential: true
```

### TLS & Network

```yaml
# argocd-server với TLS
# Luôn enable HTTPS, redirect HTTP → HTTPS
argocd-server --insecure=false

# Restrict argocd-server exposure
# Không expose ArgoCD UI/API ra internet trực tiếp
# Dùng internal LoadBalancer hoặc VPN-only Ingress

# Ingress với authentication middleware (nếu dùng)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/auth-url: "https://oauth2-proxy/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://oauth2-proxy/oauth2/sign_in"
```

### Webhook Secret

```bash
# Khi setup GitHub webhook → ArgoCD:
# Luôn set secret để verify request đến từ GitHub thật

# Trong GitHub webhook settings:
# Secret: <random 32+ char string>

# Trong ArgoCD (argocd-secret):
kubectl patch secret argocd-secret -n argocd \
  --type='json' \
  -p='[{"op":"add","path":"/data/webhook.github.secret","value":"'$(echo -n "mysecret" | base64)'"}]'
```

### Namespace Isolation

```yaml
# AppProject giới hạn namespace deploy được
spec:
  destinations:
    - namespace: "dev-*"        # chỉ namespace bắt đầu bằng dev-
      server: "https://kubernetes.default.svc"
    - namespace: "staging-*"
      server: "https://kubernetes.default.svc"
    # KHÔNG có prod-* → team dev không thể deploy lên prod
```

---

## 4. Supply Chain Security

### Image Signing với Cosign

```bash
# Ký image sau khi build (trong CI)
cosign sign --key cosign.key registry.internal/myapp:v1.2.3

# Verify trước khi deploy
cosign verify --key cosign.pub registry.internal/myapp:v1.2.3
```

**Enforce image signing trong K8s** với Kyverno hoặc Sigstore Policy Controller:

```yaml
# Kyverno policy: từ chối pod dùng image chưa ký
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-image-signature
      match:
        resources:
          kinds: [Pod]
      verifyImages:
        - imageReferences:
            - "registry.internal/*"
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      ...
                      -----END PUBLIC KEY-----
```

### SBOM (Software Bill of Materials)

```bash
# Generate SBOM
syft registry.internal/myapp:v1.2.3 -o spdx-json > sbom.json

# Attach SBOM vào image
cosign attach sbom --sbom sbom.json registry.internal/myapp:v1.2.3

# Scan SBOM cho CVE
grype sbom:./sbom.json
```

### Image scanning trong CI

```yaml
# GitHub Actions example
- name: Scan image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: registry.internal/myapp:v1.2.3
    severity: CRITICAL,HIGH
    exit-code: 1    # fail pipeline nếu có CRITICAL/HIGH CVE
```

---

## 5. Audit Trail

### Git như audit log

```bash
# Ai đã thay đổi gì, khi nào
git log --all --oneline --graph apps/myapp/

# Tìm commit đã deploy image tag cụ thể
git log -S "image: myapp:v1.2.3" --source --all

# Diff giữa 2 deployment
git diff v1.2.2..v1.2.3 -- apps/myapp/
```

### ArgoCD audit events

```bash
# ArgoCD ghi audit log trong Kubernetes events
kubectl get events -n argocd --sort-by='.lastTimestamp'

# Hoặc qua API
argocd app history myapp

# ArgoCD cũng support audit log qua plugin (Syslog, Splunk)
```

### Kubernetes audit logging

Bật K8s audit log để capture mọi API call từ ArgoCD:

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log tất cả actions từ argocd service accounts
  - level: RequestResponse
    users: ["system:serviceaccount:argocd:argocd-application-controller"]
    verbs: ["create", "update", "patch", "delete"]
```

---

## 6. Checklist Security tối thiểu

### Git Repository
- [ ] Branch protection trên `main`/`prod` — require PR review
- [ ] Require signed commits
- [ ] Deploy key read-only cho ArgoCD (không dùng PAT cá nhân)
- [ ] Không commit secret plain text — dùng Sealed Secrets / ESO / SOPS
- [ ] `.gitignore` block `*.key`, `*.pem`, `.env`

### ArgoCD
- [ ] Tắt local admin sau khi setup SSO
- [ ] RBAC: role:readonly là default, developer không được sync prod
- [ ] AppProject restrict namespace và source repo
- [ ] Webhook secret đã set
- [ ] ArgoCD UI không expose ra internet trực tiếp
- [ ] TLS enabled (không dùng `--insecure`)
- [ ] Regular backup argocd-secret (chứa private keys)

### Cluster
- [ ] ArgoCD ServiceAccount chỉ có quyền cần thiết (không phải cluster-admin)
- [ ] NetworkPolicy restrict traffic đến/từ ArgoCD pods
- [ ] Image signing policy enforce (Kyverno/Sigstore)
- [ ] Scan image CVE trong CI pipeline

### Secrets
- [ ] Sealed Secrets controller key được backup
- [ ] ESO/Vault: service account chỉ có quyền đọc secret cần thiết
- [ ] Secret rotation schedule đã được define
- [ ] Alert khi secret gần expire

---

## Gotchas Security

- **Sealed Secrets key loss = data loss**: nếu controller bị xoá mà không backup private key → không decrypt được SealedSecrets cũ → phải re-create toàn bộ secrets. **Backup key thường xuyên.**
- **ESO refreshInterval quá thấp**: refresh mỗi 1 phút × 1000 secrets = Vault bị DDoS bởi chính mình. Set `refreshInterval` hợp lý (1h là đủ cho hầu hết case).
- **ArgoCD Application trong default project**: default project không có restriction → có thể deploy lên bất kỳ namespace nào. Luôn tạo Project riêng cho workload production.
- **`cluster-admin` cho ArgoCD**: nhiều tutorial give ArgoCD `cluster-admin` cho tiện. Production không bao giờ làm vậy — scope role của ArgoCD đúng với namespace nó manage.
- **Git history = permanent**: khi xóa file secret, nó vẫn còn trong git history. Nếu commit nhầm secret → phải `git filter-repo` và rotate credential ngay lập tức.
