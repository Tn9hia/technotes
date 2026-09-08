---
title: GitOps - Overview
tags:
  - gitops
  - argocd
  - infra
  - GlobalTechJSC
date: 2026-04-26
status: in-progress
---

# GitOps — Overview

Tags: #gitops #infra #cicd
Last updated: 2026-04-26

---

## 1. What — GitOps là gì?

**GitOps** là một operational model trong đó **Git repository là nguồn sự thật duy nhất (single source of truth)** cho toàn bộ trạng thái của hệ thống — infrastructure lẫn application.

4 nguyên tắc cốt lõi (theo OpenGitOps):

| Nguyên tắc | Ý nghĩa thực tế |
|---|---|
| **Declarative** | Describe *what* you want, không phải *how* to get there. YAML manifest, không phải shell script |
| **Versioned & Immutable** | Mọi thay đổi đều qua Git commit — có history, có rollback, có audit trail |
| **Pulled automatically** | Agent trong cluster tự pull state từ Git, không phải CI/CD push vào cluster |
| **Continuously reconciled** | Agent liên tục so sánh desired state (Git) vs actual state (cluster) và tự fix drift |

> GitOps không phải tool — là **methodology**. ArgoCD, Flux là các tool implement GitOps.

---

## 2. Why — Tại sao cần GitOps?

### Vấn đề của CI/CD truyền thống (Push-based)

```
Developer push code
    │
    ▼
CI Pipeline (build, test, push image)
    │
    ▼  kubectl apply / helm upgrade  ← CI có credentials vào cluster
    ▼
Kubernetes Cluster
```

**Vấn đề:**
- CI pipeline có quyền write vào production cluster → **attack surface lớn**
- Không có audit trail rõ ràng "ai deploy cái gì lúc mấy giờ"
- Cluster state có thể drift khỏi Git — ai đó `kubectl apply` thủ công → không ai biết
- Rollback = chạy lại pipeline cũ → chậm, error-prone
- Không có "desired state" document — cluster state là implicit knowledge

### GitOps giải quyết

```
Developer push code
    │
    ▼
CI Pipeline (build, test, push image)
    │  chỉ update image tag trong manifest repo
    ▼
Git Manifest Repo (desired state)
    │
    ▼  ArgoCD/Flux tự pull (không cần credentials từ ngoài vào cluster)
    ▼
Kubernetes Cluster
```

- **CI không cần credentials vào cluster** → giảm attack surface
- **Audit trail**: mọi thay đổi là Git commit — ai, khi nào, tại sao (PR description)
- **Drift detection**: ArgoCD phát hiện và alert/auto-fix khi cluster lệch khỏi Git
- **Rollback = `git revert`** → đơn giản, predictable
- **Declarative desired state** luôn có trong Git → onboarding dễ hơn

---

## 3. Pull-based vs Push-based

```
PUSH-BASED (Traditional CI/CD)
┌─────────┐    credentials    ┌───────────┐
│ CI/CD   │ ────────────────► │  Cluster  │
│ Pipeline│   kubectl/helm    │           │
└─────────┘                   └───────────┘
✗ CI có quyền vào cluster
✗ Credentials phải store trong CI system
✗ Không detect drift
✓ Đơn giản, ít moving parts

PULL-BASED (GitOps)
┌──────────┐              ┌─────────────────────────┐
│   Git    │ ◄──── pull ──│  ArgoCD / Flux (agent)  │
│   Repo   │              │  (chạy trong cluster)   │
└──────────┘              └─────────────────────────┘
✓ Cluster tự pull — không expose credentials ra ngoài
✓ Continuous reconciliation
✓ Audit trail đầy đủ
✗ Phức tạp hơn, cần học thêm tool
```

---

## 4. Architecture — Tổng thể GitOps flow

```
┌─────────────────────────────────────────────────────────────┐
│                        App Developer                        │
│  push code → app repo                                       │
└────────────────────────┬────────────────────────────────────┘
                         │
                    CI Pipeline
                    (build + test)
                         │
                    Push Docker image
                    to Registry (Harbor)
                         │
                    Update image tag
                    in manifest repo
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   Git Manifest Repo                         │
│  apps/                                                      │
│  ├── dev/myapp/values.yaml   ← image: myapp:v1.2.3          │
│  ├── staging/myapp/values.yaml                              │
│  └── prod/myapp/values.yaml                                 │
└────────────────────────┬────────────────────────────────────┘
                         │ ArgoCD watches (poll/webhook)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                     ArgoCD                                  │
│  Detects diff → sync → apply manifests to cluster          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
              Kubernetes Cluster
```

---

## 5. How — Ecosystem & Tooling

### GitOps Controllers

| Tool | Đặc điểm | Dùng khi |
|------|-----------|---------|
| **ArgoCD** | UI đẹp, nhiều tính năng, RBAC mạnh, App of Apps | Recommended — phổ biến nhất |
| **Flux v2** | Lightweight, GitOps toolkit, moduler | K8s-native, không cần UI |
| **Spinnaker** | Multi-cloud, complex pipeline | Large enterprise |
| **Jenkins X** | Jenkins + GitOps | Đội đang dùng Jenkins |

→ Deep dive: [[ArgoCD]]

### Secret Management

| Tool | Approach |
|------|---------|
| **Sealed Secrets** | Encrypt secret trong Git, decrypt trong cluster |
| **External Secrets Operator (ESO)** | Pull secret từ Vault/AWS SM/GCP SM vào K8s |
| **SOPS** | Encrypt file với GPG/age/KMS, store encrypted trong Git |
| **Vault Agent Injector** | Inject secret vào pod từ HashiCorp Vault |

→ Deep dive: [[GitOps Security]]

### Image Update Automation

| Tool | Vai trò |
|------|---------|
| **Flux Image Reflector** | Poll registry, tự update manifest khi có image mới |
| **Kargo** | Pipeline-based promotion across environments |
| **ArgoCD Image Updater** | Update image tag trong Git khi registry có tag mới |

### Repo Structure

Hai pattern chính:

**Monorepo** — tất cả trong 1 repo:
```
gitops-repo/
├── apps/
│   ├── myapp/
│   └── otherapp/
└── infra/
    ├── namespaces/
    └── rbac/
```

**Multi-repo** — app code và config tách biệt:
```
app-repo/          ← developer owned, CI builds image
config-repo/       ← ops owned, ArgoCD watches
```

→ Chi tiết: [[GitOps Patterns & Best Practices]]

---

## 6. Key Concepts cần nắm

### Desired State vs Actual State

```
Desired State (Git):  replicas: 3, image: myapp:v1.2.3
Actual State (cluster): replicas: 2, image: myapp:v1.2.2
                              ↑
                         DRIFT detected
                              │
                    ArgoCD: OutOfSync → Sync
```

### Reconciliation Loop

```
while true:
    desired = git.getManifests()
    actual  = cluster.getState()
    if desired != actual:
        cluster.apply(desired)
    sleep(reconcileInterval)
```

### Sync vs Refresh

- **Refresh**: ArgoCD re-read Git repo để update desired state (không apply)
- **Sync**: Apply desired state vào cluster (có thể trigger thủ công hoặc auto)

### App of Apps

ArgoCD Application quản lý các ArgoCD Application khác — bootstrap toàn bộ cluster từ 1 entry point duy nhất.

→ Chi tiết: [[ArgoCD#App of Apps]]

---

## 7. Security Considerations

- **Git repo là attack surface**: ai có write access vào manifest repo → có thể deploy bất kỳ thứ gì lên cluster
- **Secret không được store plain text trong Git** — dùng Sealed Secrets / ESO / SOPS
- **RBAC trong ArgoCD**: không phải ai cũng được sync production
- **Image signing**: verify image chưa bị tamper trước khi deploy (Cosign)
- **Branch protection**: production manifest branch cần require PR review

→ Chi tiết: [[GitOps Security]]

---

## 8. Ops Runbook

```bash
# Xem trạng thái tất cả applications
argocd app list

# Xem diff giữa Git và cluster
argocd app diff myapp

# Manual sync
argocd app sync myapp

# Rollback về revision trước
argocd app rollback myapp <revision-id>

# Xem history
argocd app history myapp

# Hard refresh (bypass cache, re-read Git)
argocd app get myapp --hard-refresh
```

---

## 9. Gotchas & Lessons Learned

> Điền thêm khi có kinh nghiệm thực tế.

- **"GitOps không có nghĩa là không cần CI"**: CI vẫn cần để build/test. GitOps chỉ thay đổi CD phase.
- **Config drift giữa environments**: dev/staging/prod dùng chung chart nhưng khác values → dễ bị out-of-sync nếu không có process rõ ràng
- **ArgoCD self-manage**: ArgoCD có thể quản lý chính nó qua GitOps — nhưng chicken-and-egg problem khi bootstrap lần đầu
- **Webhook vs polling**: Polling mặc định có delay (3 phút). Production nên setup webhook từ Git → ArgoCD để sync ngay lập tức

---

## 10. Resources

- [OpenGitOps Principles](https://opengitops.dev/)
- [ArgoCD Docs](https://argo-cd.readthedocs.io/)
- [Flux Docs](https://fluxcd.io/docs/)
- [GitOps Cookbook (O'Reilly)](https://www.oreilly.com/library/view/gitops-cookbook/9781492097464/)
- [CNCF GitOps Working Group](https://github.com/cncf/tag-app-delivery/tree/main/gitops-wg)
- https://codefresh.io/blog/stop-using-branches-deploying-different-gitops-environments/
- https://codefresh.io/blog/how-to-model-your-gitops-environments-and-promote-releases-between-them/