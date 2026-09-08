---
title: CI/CD Concepts & Overview
tags:
  - cicd
  - devops
  - overview
date: 2026-04-26
---

# CI/CD Concepts & Overview

## CI/CD là gì?

```
Code commit
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  CI — Continuous Integration                                 │
│  Build → Unit Test → Code Quality → Integration Test        │
│  → Artifact (Docker image / binary / package)               │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  CD — Continuous Delivery                                    │
│  Deploy to Staging → Acceptance Test → Manual approval gate │
│  → Ready to deploy to Production (human triggers)           │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  CD — Continuous Deployment                                  │
│  Auto-deploy to Production (no human gate)                  │
│  Monitor → Rollback if needed                               │
└─────────────────────────────────────────────────────────────┘
```

**Ba mức độ automation:**

| | Continuous Integration | Continuous Delivery | Continuous Deployment |
|---|---|---|---|
| Build & test | Auto | Auto | Auto |
| Deploy staging | Auto | Auto | Auto |
| Deploy production | N/A | **Manual trigger** | **Fully auto** |
| Risk | Low | Medium | Requires mature test coverage |

---

## Tại sao CI/CD?

**Vấn đề trước CI/CD:**
- "Integration hell" — merge code sau nhiều tuần → conflicts khổng lồ
- Release cadence: vài tháng/lần → risk cao, rollback khó
- "Works on my machine" — môi trường không nhất quán
- Feedback loop chậm — bug phát hiện sau nhiều ngày/tuần

**CI/CD giải quyết:**
- Integrate liên tục (mỗi commit) → conflicts nhỏ, dễ fix
- Release nhanh (multiple times/day) → risk nhỏ mỗi release
- Reproducible builds → container images là artifact chuẩn
- Fast feedback (phút, không phải ngày)

---

## Pipeline Anatomy

```
Trigger (push/PR/schedule/manual)
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Stage 1: Build                                             │
│  ├─ Checkout source                                          │
│  ├─ Restore cache (deps)                                    │
│  ├─ Compile / build                                         │
│  └─ Save artifact                                           │
├─────────────────────────────────────────────────────────────┤
│  Stage 2: Test                                              │
│  ├─ Unit tests                                              │
│  ├─ Code coverage check                                     │
│  └─ Lint / static analysis                                  │
├─────────────────────────────────────────────────────────────┤
│  Stage 3: Security Scan                                     │
│  ├─ SAST (source code)                                      │
│  ├─ SCA (dependencies)                                      │
│  └─ Secret scanning                                         │
├─────────────────────────────────────────────────────────────┤
│  Stage 4: Package                                           │
│  ├─ Build Docker image                                      │
│  ├─ Scan image (Trivy)                                      │
│  └─ Push to registry                                        │
├─────────────────────────────────────────────────────────────┤
│  Stage 5: Deploy Staging                                    │
│  ├─ Update manifest repo (image tag)                        │
│  └─ ArgoCD sync staging                                     │
├─────────────────────────────────────────────────────────────┤
│  Stage 6: Integration / E2E Test                            │
│  ├─ Smoke test                                              │
│  └─ E2E test suite                                          │
├─────────────────────────────────────────────────────────────┤
│  Stage 7: Deploy Production                                 │
│  ├─ Manual approval (Delivery) / Auto (Deployment)         │
│  ├─ Update manifest repo (prod)                             │
│  └─ ArgoCD sync production                                  │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
Monitor → Alert → Rollback if needed
```

---

## Key Concepts

### Artifact

Output của CI pipeline — immutable, versioned, deployable unit.

| App type | Artifact |
|---|---|
| Containerized app | Docker image (tagged với git SHA) |
| JVM app | JAR/WAR file |
| Node.js | npm package hoặc container |
| Binary | Compiled binary (Go, Rust) |
| Infrastructure | Terraform plan, Ansible bundle |

**Immutable artifact principle**: artifact build một lần, deploy nhiều môi trường. Không rebuild cho từng environment.

```
git commit abc123
    → build once → myapp:sha-abc123
    → deploy staging  (test với image này)
    → deploy prod     (same image — không rebuild)
```

### Pipeline as Code

Pipeline configuration trong version control cùng với source code:
- GitHub Actions: `.github/workflows/*.yml`
- GitLab CI: `.gitlab-ci.yml`
- Jenkins: `Jenkinsfile`

Benefits: review pipeline changes, rollback, audit history.

### Branch Strategy & Pipeline Triggers

| Branch | Trigger | Pipeline |
|---|---|---|
| Feature branch | Push | Build + Unit Test + Lint |
| `main` / `master` | Push (merge) | Full pipeline → deploy staging |
| Release tag `v*.*.*` | Push tag | Full pipeline → deploy production |
| Pull Request | Open/Update | Build + Test + Security scan |

### Environment Promotion

```
dev → staging → production
       │ auto       │ manual approval
       │            │ hoặc GitOps promote
```

Mỗi environment có:
- Separate namespace (K8s) hoặc cluster
- Own config (database URL, API keys)
- Own image tag hoặc same image tag (promoted)

---

## DORA Metrics

**4 Key Metrics đo lường DevOps performance** (Google DORA research):

| Metric | Elite | High | Medium | Low |
|---|---|---|---|---|
| **Deployment Frequency** | On-demand (multiple/day) | Weekly-monthly | Monthly-6mo | < 6mo |
| **Lead Time for Changes** | < 1 hour | 1 day – 1 week | 1 week – 1 month | > 6 months |
| **Change Failure Rate** | 0–5% | 5–10% | 10–15% | > 15% |
| **MTTR** (Mean Time to Restore) | < 1 hour | < 1 day | 1 day – 1 week | > 6 months |

**Cách đo:**
```
Deployment Frequency: số deploy/ngày tới production
Lead Time: thời gian từ commit → production
Change Failure Rate: % deploy cần hotfix/rollback
MTTR: thời gian từ incident → resolved
```

---

## Pipeline Best Practices

### Fast feedback principle
```
Fail fast: unit tests → linting → security scan → integration tests
(nhanh trước → chậm sau)

Unit test: < 5 phút
Full pipeline (build + test + scan): < 15 phút
Deploy staging: < 5 phút sau pipeline xong
```

### Trunk-based development
```
main branch luôn deployable
Feature branches tồn tại ngắn (< 1-2 ngày)
Small commits, frequent merges
Feature flags cho incomplete features
```

### Pipeline security
```
Secrets trong CI/CD secrets vault (không trong code)
Least privilege: pipeline service account chỉ có quyền cần thiết
Pin action versions: uses: actions/checkout@v4 (không @latest)
Audit pipeline runs
```

### Cache strategy
```
Cache dependencies (node_modules, pip, maven)
Cache Docker layers (BuildKit)
Cache test results (unchanged code → skip tests)
```

### Fail loudly
```
Exit code != 0 → pipeline fail → notification
Không silent failures
Test coverage threshold: pipeline fail nếu coverage drop
```

---

## CI vs CD — Separate Concerns

**Tại sao tách CI repo và CD repo (manifest repo)?**

```
CI Repo (app code):
  - Source code
  - Dockerfile
  - Unit/integration tests
  - .github/workflows/ci.yml
  → Output: Docker image pushed to registry

CD Repo (manifests):
  - Kubernetes YAML / Helm values
  - Per-environment configs
  - ArgoCD Application definitions
  → ArgoCD watches repo → sync to cluster
```

**Benefits của separation:**
- CI/CD concerns tách biệt — developer không cần biết về infra
- Audit trail: ai deploy gì, khi nào, lên môi trường nào
- Rollback: `git revert` trên manifest repo → ArgoCD tự rollback
- Access control: hạn chế ai được push vào production manifests

Chi tiết xem [[Pipeline Patterns]].

---

## Tool Landscape

| Category | Tools |
|---|---|
| **CI Platform** | GitHub Actions, GitLab CI, Jenkins, CircleCI, Tekton |
| **CD / GitOps** | ArgoCD, Flux, Spinnaker |
| **Image Registry** | Harbor, Docker Hub, ECR, GCR, GHCR |
| **Artifact Store** | Nexus, Artifactory, Harbor |
| **Secret Management** | Vault, AWS Secrets Manager, SOPS, Sealed Secrets |
| **SAST** | SonarQube, Semgrep, CodeQL |
| **SCA** | Dependabot, Snyk, OWASP Dependency-Check |
| **Container Scan** | Trivy, Grype, Clair |
| **Secret Scan** | Gitleaks, TruffleHog, detect-secrets |
| **E2E Test** | Playwright, Cypress, Selenium |
| **Promotion** | Kargo, ArgoCD Image Updater |
| **Notification** | Slack webhook, PagerDuty, OpsGenie |

---

## Gotchas

- **Flaky tests**: test fail intermittently → mất trust vào pipeline → người dùng ignore failures. Fix flaky tests trước khi thêm features.
- **Build không reproducible**: "works in CI, fails locally" hoặc ngược lại. Dùng container-based build environment (same image cho cả CI và local).
- **Secrets rotation**: hardcode secret version trong pipeline → khi rotate secret, phải update tất cả pipelines. Dùng secret stores với dynamic references.
- **Long-running pipelines**: pipeline 45 phút → developer không wait → context switch → slower delivery. Target < 15 phút cho full pipeline.
- **Mono-repo pitfall**: 1 commit → trigger build ALL services → slow và wasted resources. Implement path-based filtering (chỉ build services thay đổi).
- **Approval gates và toil**: quá nhiều manual approvals → bottleneck, người approve không đọc kỹ → rubber stamp. Tự động hóa verification, manual approval chỉ cho production.
