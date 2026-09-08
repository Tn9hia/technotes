---
title: Pipeline Patterns
tags:
  - cicd
  - gitops
  - pipeline
  - deep-dive
date: 2026-04-26
---

# Pipeline Patterns

## App Repo vs Manifest Repo — Separation of Concerns

**Tại sao tách?**

```
❌ Monorepo (code + manifests cùng nhau):
   - Mỗi k8s change trigger CI build lại (không cần thiết)
   - Khó audit: ai deploy gì lên prod?
   - Developer thay đổi infra config vô tình
   - RBAC khó: dev cần push code, ops cần push manifests

✓ Tách 2 repos:
   App repo  → CI: build, test, scan, push image
   Manifest repo → CD: ArgoCD watches, sync to cluster
```

**Two-repo pattern:**

```
┌──────────────────────┐     push image      ┌─────────────────┐
│   app-repo           │  ──────────────────► │    Registry     │
│   (source code)      │                      │ registry.int/   │
│                      │  update image tag    │ myapp:sha-abc   │
│   .github/workflows/ │  ──────────────────► └─────────────────┘
│   Dockerfile         │         │
│   src/               │         ▼
└──────────────────────┘  ┌─────────────────────────────┐
                           │   manifest-repo             │
                           │   (k8s-manifests)           │
                           │                             │
                           │   apps/myapp/               │
                           │   ├── staging/              │
                           │   │   └── values.yaml       │
                           │   │       image.tag: sha-abc│
                           │   └── production/           │
                           │       └── values.yaml       │
                           └──────────┬──────────────────┘
                                      │ watches
                                      ▼
                           ┌─────────────────────────────┐
                           │   ArgoCD                    │
                           │   Reconcile: desired ↔ actual│
                           └──────────┬──────────────────┘
                                      │ sync
                                      ▼
                           ┌─────────────────────────────┐
                           │   Kubernetes Cluster        │
                           │   staging / production ns   │
                           └─────────────────────────────┘
```

---

## Image Tag Update Flow

**Bước CI cập nhật image tag vào manifest repo:**

### Pattern 1: Direct git push từ CI

```bash
# Trong CI pipeline (sau docker push)
MANIFEST_REPO="https://x-access-token:${MANIFEST_TOKEN}@github.com/company/k8s-manifests.git"
IMAGE_TAG="sha-${CI_COMMIT_SHORT_SHA}"

git clone $MANIFEST_REPO /tmp/manifests
cd /tmp/manifests

# Update staging values
sed -i "s|tag: .*|tag: ${IMAGE_TAG}|g" apps/myapp/staging/values.yaml

git config user.email "ci-bot@company.com"
git config user.name "CI Bot"
git add apps/myapp/staging/values.yaml
git commit -m "chore(myapp): update staging image to ${IMAGE_TAG}

Source: ${CI_PROJECT_URL}/-/commit/${CI_COMMIT_SHA}
Pipeline: ${CI_PIPELINE_URL}"
git push origin main
```

### Pattern 2: GitHub Actions — github-script

```yaml
- name: Update manifest repo
  uses: actions/github-script@v7
  with:
    github-token: ${{ secrets.MANIFEST_REPO_TOKEN }}
    script: |
      const fs = require('fs')
      const imageTag = `sha-${context.sha.slice(0, 7)}`

      // Get current file content + SHA (needed for update)
      const { data: file } = await github.rest.repos.getContent({
        owner: 'company',
        repo: 'k8s-manifests',
        path: 'apps/myapp/staging/values.yaml',
        ref: 'main'
      })

      const content = Buffer.from(file.content, 'base64').toString()
      const updated = content.replace(/tag: .+/, `tag: ${imageTag}`)

      // Update file
      await github.rest.repos.createOrUpdateFileContents({
        owner: 'company',
        repo: 'k8s-manifests',
        path: 'apps/myapp/staging/values.yaml',
        message: `chore(myapp): update to ${imageTag}`,
        content: Buffer.from(updated).toString('base64'),
        sha: file.sha,
        branch: 'main'
      })
```

### Pattern 3: yq (YAML processor — chính xác hơn sed)

```bash
# yq thay vì sed (không bị lỗi với YAML syntax)
IMAGE_TAG="sha-${CI_COMMIT_SHORT_SHA}"

yq e ".image.tag = \"${IMAGE_TAG}\"" -i apps/myapp/staging/values.yaml
yq e ".image.repository = \"${REGISTRY}/${IMAGE_NAME}\"" -i apps/myapp/staging/values.yaml

git diff                  # verify changes
git add -A
git commit -m "chore(myapp): bump to ${IMAGE_TAG}"
git push
```

---

## ArgoCD Image Updater

Tự động detect image tag mới trong registry → cập nhật manifest repo (không cần CI push).

```
Registry
  │ new tag pushed
  ▼
ArgoCD Image Updater
  │ polls registry (mỗi 2 phút)
  │ detects new tag matching semver/sha pattern
  │ updates manifest repo (git write-back)
  ▼
ArgoCD
  │ detects change in manifest repo
  ▼
Cluster sync
```

**Cài đặt:**
```bash
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

**Annotate ArgoCD Application:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  namespace: argocd
  annotations:
    # Khai báo images cần update
    argocd-image-updater.argoproj.io/image-list: |
      myapp=registry.internal/platform/myapp

    # Update strategy: newest-build (by push date), semver, digest, name (alphabetical)
    argocd-image-updater.argoproj.io/myapp.update-strategy: newest-build

    # Chỉ match tag format sha-xxxxxxx
    argocd-image-updater.argoproj.io/myapp.allow-tags: regexp:^sha-[0-9a-f]{7}$

    # Write back: helm values file
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
    argocd-image-updater.argoproj.io/myapp.helm.image-name: image.repository
    argocd-image-updater.argoproj.io/myapp.helm.image-tag: image.tag
spec:
  source:
    repoURL: https://github.com/company/k8s-manifests
    targetRevision: main
    path: apps/myapp/staging
    helm:
      valueFiles:
        - values.yaml
```

---

## Kargo — Multi-Stage Promotion

Kargo là promotion controller — quản lý việc promote artifact (image tag) qua các environments (staging → production) theo policy.

```
app-repo (CI) → Registry
                    │ new image sha-abc
                    ▼
              Kargo Warehouse
              (poll registry)
                    │
                    ▼
              Kargo Stage: staging ──── auto promote
                    │
                    │ (manual approval / automated checks)
                    ▼
              Kargo Stage: production ─── promote after tests pass
```

**Kargo CRDs:**

```yaml
# Warehouse — nguồn artifact
apiVersion: kargo.akuity.io/v1alpha1
kind: Warehouse
metadata:
  name: myapp
  namespace: kargo-demo
spec:
  subscriptions:
    - image:
        repoURL: registry.internal/platform/myapp
        tagSelectionStrategy: NewestBuild   # hoặc SemVer, Digest
        semverConstraint: "^1.0.0"         # chỉ dùng với SemVer strategy
        discoveryLimit: 5
```

```yaml
# Stage — environment với promotion policy
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: staging
  namespace: kargo-demo
spec:
  subscriptions:
    freight:
      - warehouse: myapp    # subscribe tới warehouse

  promotionTemplate:
    spec:
      steps:
        # Update Helm values trong git
        - uses: git-clone
          config:
            repoURL: https://github.com/company/k8s-manifests.git
            branch: main

        - uses: helm-update-image
          config:
            path: apps/myapp/staging
            images:
              - image: registry.internal/platform/myapp
                key: image.tag
                value: Tag   # chỉ dùng tag (không phải digest)

        - uses: git-commit
          config:
            message: "chore(myapp): promote to staging"

        - uses: git-push

        - uses: argocd-update
          config:
            apps:
              - name: myapp-staging
                sources:
                  - repoURL: https://github.com/company/k8s-manifests.git
                    desiredRevision: HEAD

        - uses: argocd-update
          config:
            apps:
              - name: myapp-staging
                sources:
                  - repoURL: https://github.com/company/k8s-manifests.git

---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: production
  namespace: kargo-demo
spec:
  subscriptions:
    stages:
      - staging           # promote từ staging (không trực tiếp từ warehouse)

  # Verification sau khi deploy staging (trước khi cho promote production)
  verification:
    analysisTemplates:
      - name: smoke-test

  promotionTemplate:
    spec:
      steps:
        - uses: git-clone
        - uses: helm-update-image
          config:
            path: apps/myapp/production
            # ...
        - uses: git-commit
        - uses: git-push
        - uses: argocd-update
```

```yaml
# AnalysisTemplate — automated verification
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: smoke-test
  namespace: kargo-demo
spec:
  metrics:
    - name: smoke-test
      provider:
        job:
          spec:
            template:
              spec:
                containers:
                  - name: smoke-test
                    image: curlimages/curl:8.6.0
                    command:
                      - sh
                      - -c
                      - |
                        curl -f https://staging.myapp.internal/health
                        curl -f https://staging.myapp.internal/api/v1/ping
                restartPolicy: Never
```

```bash
# Kargo CLI
kargo get stages --project kargo-demo
kargo promote --project kargo-demo --stage production --freight abc123
kargo get freightcollections --project kargo-demo
```

---

## Mono-repo Strategy

Khi tất cả services trong 1 repo:

```yaml
# .github/workflows/ci.yml — path-based trigger
on:
  push:
    paths:
      - 'services/myapp/**'
      - 'shared/**'

jobs:
  detect-changes:
    outputs:
      myapp: ${{ steps.filter.outputs.myapp }}
      auth-service: ${{ steps.filter.outputs.auth }}
    steps:
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            myapp:
              - 'services/myapp/**'
            auth:
              - 'services/auth/**'
            shared:
              - 'shared/**'

  build-myapp:
    needs: detect-changes
    if: needs.detect-changes.outputs.myapp == 'true'
    # build only myapp

  build-auth:
    needs: detect-changes
    if: needs.detect-changes.outputs.auth == 'true'
    # build only auth-service
```

---

## Manifest Repo Structure

```
k8s-manifests/
├── apps/
│   ├── myapp/
│   │   ├── base/                      # Kustomize base hoặc Helm chart
│   │   │   ├── Chart.yaml
│   │   │   └── templates/
│   │   ├── staging/
│   │   │   ├── values.yaml            # image.tag: sha-abc
│   │   │   └── kustomization.yaml
│   │   └── production/
│   │       ├── values.yaml            # image.tag: sha-abc (promoted)
│   │       └── kustomization.yaml
│   └── auth-service/
│       ├── staging/
│       └── production/
│
├── platform/                          # shared infra (cert-manager, ingress, ...)
│   ├── cert-manager/
│   └── ingress-nginx/
│
└── argocd/                            # ArgoCD Application definitions
    ├── myapp-staging.yaml
    ├── myapp-production.yaml
    └── app-of-apps.yaml
```

---

## PR-based Promotion (với approval)

```
main branch (auto) → staging
                          │
                    CI creates PR:
                    "promote myapp v1.2.3 to production"
                          │
                    Reviewer approves PR
                          │
                    Merge → production manifest updated
                          │
                    ArgoCD auto-sync production
```

```yaml
# CI job tạo PR sau khi staging deploy thành công
create-production-pr:
  needs: [verify-staging]
  steps:
    - name: Create promotion PR
      uses: peter-evans/create-pull-request@v6
      with:
        token: ${{ secrets.MANIFEST_REPO_TOKEN }}
        commit-message: "chore(myapp): promote sha-${{ github.sha }} to production"
        branch: promote/myapp-${{ github.sha }}
        title: "🚀 Promote myapp sha-${{ github.sha }} to production"
        body: |
          ## Promotion Request

          **Service**: myapp
          **Version**: sha-${{ github.sha }}
          **From**: staging → production

          ### Changes
          - [View diff](...)
          - [Staging deployment](https://staging.myapp.internal)

          ### Checklist
          - [ ] Staging smoke test passed
          - [ ] Performance metrics normal
          - [ ] No alerts firing on staging
        reviewers: ops-team
        labels: promotion, production
```

---

## Gotchas

- **Git conflicts trong manifest repo**: khi nhiều services đồng thời CI → đồng thời push vào manifest repo → git conflict. Giải pháp: retry với pull+rebase, hoặc dùng Kargo/Image Updater thay vì direct git push.
- **ArgoCD sync timing**: sau khi CI push manifest, ArgoCD polling interval mặc định là 3 phút. Để nhanh hơn: `argocd app sync myapp` từ CI hoặc dùng ArgoCD webhook.
- **Image tag `latest` trong manifest**: tracking `latest` → không biết đang chạy version nào, rollback không rõ. Luôn dùng immutable tag (sha hoặc semver).
- **Kargo và ArgoCD version compatibility**: Kargo tích hợp chặt với ArgoCD. Check compatibility matrix trước khi upgrade một trong hai.
- **Manifest repo access từ CI**: dùng deploy key (read-write) cho manifest repo, separate với app repo. Rotate key định kỳ.
- **Drift detection**: ArgoCD có thể sync về manifest ngay cả khi ops team manual patch. `syncPolicy.automated.selfHeal: true` → tốt cho production consistency nhưng phải đảm bảo manifest là source of truth.
