---
title: GitOps Patterns & Best Practices
tags:
  - gitops
  - patterns
  - best-practices
  - deep-dive
date: 2026-04-26
---

# GitOps Patterns & Best Practices

## Repo Structure Patterns

### Pattern 1: Monorepo (App code + Config tách thư mục)

```
myapp-repo/
├── src/                    ← app code
├── Dockerfile
├── .github/workflows/
│   └── ci.yaml             ← CI: build, test, push image
└── deploy/                 ← config (ArgoCD watches đây)
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── overlays/
        ├── dev/
        ├── staging/
        └── prod/
```

**Ưu:** Đơn giản, developer thấy code và config cùng chỗ.
**Nhược:** Developer có thể sửa prod config. CI trigger không cần thiết khi chỉ sửa docs. Khó phân quyền.

---

### Pattern 2: Separate Config Repo (Recommended)

```
app-repo/            ← developer owned
├── src/
├── Dockerfile
└── .github/workflows/ci.yaml
    # CI: build image → push → update image tag trong config-repo

config-repo/         ← ops/platform owned, ArgoCD watches
├── apps/
│   ├── myapp/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       │   ├── kustomization.yaml
│   │       │   └── values.yaml    ← image: myapp:v1.2.3
│   │       ├── staging/
│   │       └── prod/
│   └── otherapp/
└── infra/
    ├── cert-manager/
    ├── ingress-nginx/
    └── monitoring/
```

**Ưu:** Tách biệt rõ ràng. Developer không có write access vào prod manifest. Config repo có thể có stricter review process.
**Nhược:** 2 repo → phức tạp hơn, cần automation update image tag cross-repo.

---

### Pattern 3: Environment Branches

```
config-repo/
├── main branch    ← prod environment
├── staging branch ← staging environment
└── dev branch     ← dev environment
```

Promotion = merge PR từ `dev` → `staging` → `main`.

**Ưu:** Đơn giản, branch = environment.
**Nhược:** Merge conflict. Khó track "version X đang ở env nào". Anti-pattern theo GitOps best practice (environment nên là thư mục, không phải branch).

---

### Pattern 4: Environment Directories (Recommended)

```
config-repo/
├── environments/
│   ├── dev/
│   │   └── myapp/
│   │       └── values.yaml   ← image: myapp:v1.2.4-dev (latest)
│   ├── staging/
│   │   └── myapp/
│   │       └── values.yaml   ← image: myapp:v1.2.3
│   └── prod/
│       └── myapp/
│           └── values.yaml   ← image: myapp:v1.2.2 (stable)
└── base/
    └── myapp/
        └── deployment.yaml   ← template chung
```

**Promotion:** Sửa image tag trong `staging/values.yaml` → PR → merge → ArgoCD deploy.

---

## Image Tag Strategy

### Anti-pattern: dùng `latest`

```yaml
# ĐỪNG LÀM
image: myapp:latest
# latest = không immutable, không biết version nào đang chạy
# ArgoCD không detect change nếu tag không thay đổi
```

### Immutable tags

```yaml
# Git SHA (recommended)
image: registry.internal/myapp:a3f8b2c

# Semantic version
image: registry.internal/myapp:v1.2.3

# Date + SHA (dễ đọc hơn)
image: registry.internal/myapp:20260426-a3f8b2c
```

### Automated image update

**CI pipeline update image tag sau khi push:**

```yaml
# .github/workflows/ci.yaml
- name: Update image tag in config repo
  run: |
    git clone https://x-access-token:${{ secrets.CONFIG_REPO_TOKEN }}@github.com/myorg/config-repo
    cd config-repo
    
    # Update tag trong dev environment
    sed -i "s|image: registry.internal/myapp:.*|image: registry.internal/myapp:${{ github.sha }}|" \
      environments/dev/myapp/values.yaml
    
    git config user.email "ci@myorg.com"
    git config user.name "CI Bot"
    git add .
    git commit -m "chore: update myapp to ${{ github.sha }}"
    git push
```

**ArgoCD Image Updater** — tự động poll registry và update tag:

```yaml
# Annotation trên ArgoCD Application
metadata:
  annotations:
    argocd-image-updater.argoproj.io/image-list: myapp=registry.internal/myapp
    argocd-image-updater.argoproj.io/myapp.update-strategy: semver
    argocd-image-updater.argoproj.io/myapp.tag-match: "^v[0-9]+\.[0-9]+\.[0-9]+$"
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
```

---

## Environment Promotion Strategy

### Manual promotion (PR-based)

```
Dev (auto-deploy từ CI)
    │
    │ Developer/QA verify
    │ Open PR: "promote myapp v1.2.3 to staging"
    ▼
Staging (manual PR merge)
    │
    │ Full QA, performance test
    │ Open PR: "promote myapp v1.2.3 to prod"
    │ Require ≥ 2 approvals
    ▼
Prod (manual PR merge, change window)
```

**Điều kiện PR merge:**
- CI tests pass trên branch mới nhất
- Manual approval
- Change window (không merge vào prod lúc 5pm Friday)

### Kargo — Pipeline-based promotion

Kargo là tool chuyên biệt cho promotion flow trong GitOps:

```yaml
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: staging
spec:
  subscriptions:
    upstreamStages:
      - name: dev
        requestedFreight:
          - origin:
              kind: Warehouse
              name: myapp-warehouse
            desired: Latest
  promotionMechanisms:
    gitRepoUpdates:
      - repoURL: https://github.com/myorg/config-repo
        branch: main
        kustomize:
          images:
            - image: registry.internal/myapp
              path: environments/staging/myapp
```

---

## Rollback Strategy

### GitOps rollback = `git revert`

```bash
# Xem history
git log --oneline environments/prod/myapp/values.yaml

# Revert commit xấu
git revert <bad-commit-sha>
git push

# ArgoCD tự detect diff và sync về version trước
```

### ArgoCD rollback (không qua Git)

```bash
# Rollback nhanh không cần sửa Git
argocd app rollback myapp-prod <revision-id>

# Xem revision history
argocd app history myapp-prod

# LƯU Ý: rollback kiểu này tạo "out of sync" với Git
# → ArgoCD sẽ sync lại nếu có selfHeal
# → Phải sửa Git đồng thời để consistent
```

**Production rollback procedure:**

```
1. argocd app rollback myapp-prod <prev-revision>  ← fast, restore service
2. Đồng thời: git revert <bad-commit> + push       ← fix source of truth
3. ArgoCD re-sync với Git version mới              ← consistent state
4. Post-mortem
```

---

## Multi-environment Config Management

### Helm + multiple values files

```
config-repo/
└── myapp/
    ├── Chart.yaml
    ├── templates/
    │   └── deployment.yaml
    ├── values.yaml              ← defaults chung
    ├── values-dev.yaml          ← dev overrides
    ├── values-staging.yaml      ← staging overrides
    └── values-prod.yaml         ← prod overrides
```

```yaml
# ArgoCD Application cho prod
spec:
  source:
    helm:
      valueFiles:
        - values.yaml
        - values-prod.yaml    # override sau cùng
```

### Kustomize base + overlays

```
myapp/
├── base/
│   ├── deployment.yaml       ← template: replicas=1, resource limits thấp
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    │       # patches: replicas=1, image=:latest-dev
    ├── staging/
    │   └── kustomization.yaml
    │       # patches: replicas=2, resource limits medium
    └── prod/
        ├── kustomization.yaml
        │   # patches: replicas=5, resource limits high, PDB
        └── hpa.yaml          ← prod-only resource
```

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
  - hpa.yaml
  - pdb.yaml

images:
  - name: registry.internal/myapp
    newTag: v1.2.3

patches:
  - target:
      kind: Deployment
      name: myapp
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 5
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/cpu
        value: "500m"

commonLabels:
  environment: production
```

---

## Production Mindset

### 1. Git là source of truth — cluster là implementation detail

```
Đúng:  "Desired state là X (theo Git)"
Sai:   "Cluster đang chạy X"  ← cluster có thể drift

Consequence: mọi thay đổi phải đi qua Git, không kubectl apply thủ công
```

### 2. Immutability over mutability

```
Đúng:  deploy image mới với tag mới
Sai:   update image `latest` tại chỗ

Consequence: rollback = deploy version cũ, không phải "undo"
```

### 3. Everything as code — không có implicit knowledge

```
Tất cả config phải trong Git:
✓ Kubernetes manifests
✓ Helm values
✓ ArgoCD Applications
✓ RBAC policies
✓ Network policies
✗ "Tôi nhớ là đã chạy kubectl patch lần trước"
```

### 4. Fail fast, fail loudly

```yaml
# ArgoCD nên alert ngay khi OutOfSync
# Không để drift âm thầm trong nhiều ngày

# Notification khi OutOfSync > 5 phút
trigger.on-out-of-sync-too-long: |
  - when: app.status.sync.status == 'OutOfSync' && time.Now().Sub(app.status.operationState.startedAt) > 5*time.Minute
    send: [app-out-of-sync-alert]
```

### 5. Principle of least privilege

```
CI pipeline:
  - Write access vào config repo (chỉ image tag update)
  - KHÔNG có access vào K8s cluster

ArgoCD:
  - Read access vào Git
  - Write access vào K8s — nhưng chỉ namespace được phân công

Developer:
  - Write access vào app repo
  - Read-only ArgoCD access cho dev/staging
  - KHÔNG có access trực tiếp vào prod cluster
```

### 6. Change management cho production

```
Quy trình merge vào prod:
1. PR phải có description: what changed, why, risk level
2. ≥ 2 reviewer approve (kể cả infra/ops reviewer)
3. CI checks pass
4. Deploy trong change window (business hours)
5. Notify stakeholders (Slack/email)
6. Monitor 30 phút sau deploy
7. Rollback plan sẵn sàng
```

### 7. Test manifest trước khi merge

```yaml
# CI pipeline validate manifest trước khi merge vào config repo
- name: Lint Helm chart
  run: helm lint ./myapp --values values-prod.yaml

- name: Kustomize build
  run: kustomize build overlays/prod | kubectl apply --dry-run=client -f -

- name: Validate with kubeval/kubeconform
  run: kustomize build overlays/prod | kubeconform -strict -kubernetes-version 1.28.0

- name: Policy check with Conftest/OPA
  run: kustomize build overlays/prod | conftest test -
```

---

## Observability cho GitOps

```
Metrics cần monitor:
- ArgoCD app sync duration
- ArgoCD app health status (số app Degraded)
- Số lần sync fail / ngày
- OutOfSync duration (bao lâu trước khi được fix)
- Deployment frequency (số deploy / ngày)  ← DORA metric
- Change failure rate                       ← DORA metric
- Mean time to restore (MTTR)               ← DORA metric
```

```yaml
# Prometheus: ArgoCD tự expose metrics tại :8082/metrics
# Grafana dashboard: https://grafana.com/grafana/dashboards/14584
```

---

## Bootstrapping ArgoCD — Chicken and egg problem

```
Vấn đề: ArgoCD cần được install trước khi nó có thể manage chính nó
```

**Giải pháp: 2 bước bootstrap:**

```bash
# Bước 1: Install ArgoCD thủ công (1 lần duy nhất)
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Bước 2: Apply root Application (App of Apps)
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/myorg/gitops-repo
    path: bootstrap/     # chứa ArgoCD config + tất cả app-of-apps
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

# Từ đây, ArgoCD tự quản lý chính nó và toàn bộ cluster
```

**`bootstrap/` directory trong Git:**

```
bootstrap/
├── argocd/
│   ├── argocd-install.yaml      ← ArgoCD itself (Helm chart)
│   ├── argocd-cm.yaml           ← config
│   └── argocd-rbac-cm.yaml      ← RBAC
└── app-of-apps.yaml             ← root Application pointing to /apps
```
