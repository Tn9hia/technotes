---
title: ArgoCD
tags:
  - argocd
  - gitops
  - deep-dive
date: 2026-04-26
---

# ArgoCD — Deep Dive

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    ArgoCD Components                     │
│                                                          │
│  ┌─────────────┐   ┌──────────────┐   ┌──────────────┐  │
│  │ argocd-     │   │ repo-server  │   │ application- │  │
│  │ server      │   │              │   │ controller   │  │
│  │ (API + UI)  │   │ (git clone,  │   │ (reconcile   │  │
│  │             │   │  render      │   │  loop)       │  │
│  │ :8080 HTTP  │   │  manifests)  │   │              │  │
│  └──────┬──────┘   └──────┬───────┘   └──────┬───────┘  │
│         │                 │                  │           │
│         └─────────────────┴──────────────────┘           │
│                           │                              │
│                    ┌──────┴──────┐                       │
│                    │    Redis    │  ← cache, queue       │
│                    └─────────────┘                       │
│                                                          │
│  ┌─────────────┐   ┌──────────────┐                      │
│  │    Dex      │   │ argocd-      │                      │
│  │  (OIDC SSO) │   │ applicationset│                     │
│  │             │   │ -controller  │                      │
│  └─────────────┘   └──────────────┘                      │
└──────────────────────────────────────────────────────────┘
```

### Vai trò từng component

| Component                     | Vai trò                                                                               |
| ----------------------------- | ------------------------------------------------------------------------------------- |
| **argocd-server**             | API server + Web UI. Nhận request từ CLI/UI, expose REST/gRPC API                     |
| **repo-server**               | Clone Git repo, render manifests (Helm/Kustomize/plain YAML). Stateless, có thể scale |
| **application-controller**    | Reconciliation loop chính — compare desired vs actual, trigger sync                   |
| **dex**                       | OIDC identity provider — SSO với GitHub, GitLab, LDAP, Okta                           |
| **redis**                     | Cache repo state, application state. Nếu redis down → ArgoCD vẫn chạy nhưng chậm      |
| **applicationset-controller** | Xử lý ApplicationSet CRD — tạo Application theo template                              |

---

## Application CRD — Đơn vị cơ bản

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd          # luôn trong namespace argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # cascade delete
spec:
  project: default           # ArgoCD Project (RBAC boundary)

  source:
    repoURL: https://github.com/myorg/gitops-repo
    targetRevision: HEAD      # branch, tag, hoặc commit SHA
    path: apps/myapp          # đường dẫn trong repo

    # Nếu dùng Helm:
    helm:
      valueFiles:
        - values.yaml
        - values-prod.yaml
      parameters:
        - name: image.tag
          value: v1.2.3

    # Nếu dùng Kustomize:
    kustomize:
      version: v4.5.7
      images:
        - myapp=registry.internal/myapp:v1.2.3

  destination:
    server: https://kubernetes.default.svc  # in-cluster
    # server: https://prod-cluster.internal:6443  # remote cluster
    namespace: myapp-prod

  syncPolicy:
    automated:
      prune: true          # xoá resource không còn trong Git
      selfHeal: true       # tự fix nếu ai đó sửa thủ công
      allowEmpty: false    # không sync nếu render ra empty
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true   # chỉ apply resource thực sự changed
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  ignoreDifferences:          # bỏ qua field nhất định khi compare
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas      # bỏ qua replicas nếu HPA đang quản lý
```

### Application Status

```
Sync Status:
  Synced       = cluster khớp Git
  OutOfSync    = có diff
  Unknown      = chưa sync lần nào

Health Status:
  Healthy      = tất cả resources healthy
  Progressing  = đang deploy
  Degraded     = có resource unhealthy
  Suspended    = bị pause
  Missing      = resource không tồn tại trong cluster
  Unknown      = không xác định được
```

---

## Sync Phases & Hooks

ArgoCD sync chia làm 3 phase. Hook là resource chỉ chạy trong phase cụ thể.

```
PreSync Phase
    │  ← chạy Job/Hook trước khi apply
    │  e.g.: database migration, backup
    ▼
Sync Phase
    │  ← apply tất cả resources
    ▼
PostSync Phase
    │  ← chạy sau khi sync xong
    │  e.g.: smoke test, notification
    ▼
SyncFail Phase  ← chạy nếu sync fail (cleanup, alert)
```

### Định nghĩa Hook

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
    # HookSucceeded = xoá Job sau khi thành công
    # HookFailed    = xoá sau khi fail
    # BeforeHookCreation = xoá Job cũ trước khi tạo mới
spec:
  template:
    spec:
      containers:
        - name: migration
          image: myapp:v1.2.3
          command: ["./migrate.sh"]
      restartPolicy: Never
```

### Sync Waves — Thứ tự trong cùng phase

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # deploy trước (wave thấp hơn = trước)
---
# Wave mặc định là 0
# Negative wave (-1, -2) = deploy đầu tiên
# e.g.:
# wave -2: CRD definitions
# wave -1: namespaces, RBAC
# wave 0:  ConfigMaps, Secrets
# wave 1:  Deployments
# wave 2:  Ingress
```

**Ví dụ thực tế:**

```yaml
# 1. CRD trước
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
---
# 2. Database (cần chạy trước app)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
---
# 3. App deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

---

## App of Apps Pattern

Dùng một ArgoCD Application để quản lý nhiều Application khác. Bootstrap toàn bộ cluster từ 1 entry point.

```
Root App (app-of-apps)
    │
    ├── App: monitoring (prometheus-stack)
    ├── App: ingress-nginx
    ├── App: cert-manager
    ├── App: myapp-dev
    ├── App: myapp-staging
    └── App: myapp-prod
```

**Git structure:**

```
gitops-repo/
├── apps/                      ← root app trỏ vào đây
│   ├── monitoring.yaml        ← Application manifest
│   ├── ingress-nginx.yaml
│   ├── cert-manager.yaml
│   └── myapp.yaml
└── charts/                    ← actual app configs
    ├── monitoring/
    ├── ingress-nginx/
    └── myapp/
        ├── dev/
        ├── staging/
        └── prod/
```

**Root Application:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/myorg/gitops-repo
    targetRevision: HEAD
    path: apps              # thư mục chứa Application manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd       # Applications luôn trong namespace argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## ApplicationSet — Template cho nhiều Applications

ApplicationSet tự động tạo nhiều Application từ template, dựa trên generator.

### Git Generator — tạo App cho mỗi thư mục trong repo

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-environments
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/myorg/gitops-repo
        revision: HEAD
        directories:
          - path: "environments/*"    # match env/dev, env/staging, env/prod
  template:
    metadata:
      name: "myapp-{{path.basename}}"   # myapp-dev, myapp-staging, myapp-prod
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/gitops-repo
        targetRevision: HEAD
        path: "{{path}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "myapp-{{path.basename}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### Cluster Generator — deploy lên nhiều cluster

```yaml
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            env: production    # chỉ cluster có label env=production
  template:
    metadata:
      name: "myapp-{{name}}"   # name = cluster name
    spec:
      destination:
        server: "{{server}}"   # server = cluster API URL
        namespace: myapp
```

### Matrix Generator — kết hợp nhiều generator

```yaml
spec:
  generators:
    - matrix:
        generators:
          - git:
              # list of apps
          - clusters:
              # list of clusters
  # → tạo App cho mọi (app, cluster) combination
```

---

## Drift Detection & Auto-remediation

```
Actual State (cluster)  ≠  Desired State (Git)
              ↑
         DRIFT detected
              │
    ┌─────────┴─────────┐
    │  selfHeal: true?  │
    └─────────┬─────────┘
             yes → ArgoCD tự sync lại
             no  → Status: OutOfSync, alert
```

**Vì sao drift xảy ra:**
- Ai đó `kubectl apply` / `kubectl edit` thủ công
- HPA thay đổi replicas (nên ignore với `ignoreDifferences`)
- Operator tự thêm/sửa field
- Secret được rotate bởi external tool

**`ignoreDifferences`** — bỏ qua drift trên field cụ thể:

```yaml
spec:
  ignoreDifferences:
    # HPA quản lý replicas → bỏ qua
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas

    # Webhook tự inject caBundle → bỏ qua
    - group: admissionregistration.k8s.io
      kind: MutatingWebhookConfiguration
      jqPathExpressions:
        - .webhooks[].clientConfig.caBundle
```

---

## RBAC trong ArgoCD

ArgoCD RBAC chia 2 lớp:

### 1. ArgoCD Projects — Phân vùng resources

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production workloads

  # Cho phép sync từ repo nào
  sourceRepos:
    - "https://github.com/myorg/gitops-repo"
    - "https://charts.bitnami.com/bitnami"

  # Cho phép deploy lên cluster/namespace nào
  destinations:
    - namespace: "prod-*"
      server: https://kubernetes.default.svc

  # Giới hạn resource type được phép deploy
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota   # không cho phép tạo ResourceQuota

  # Yêu cầu có ít nhất N approver khi sync
  syncWindows:
    - kind: deny
      schedule: "* * * * *"   # deny all sync
      duration: 24h
      applications: ["*"]
      # Tạo "maintenance window" rõ ràng

  # Orphaned resources policy
  orphanedResources:
    warn: true    # alert khi có resource trong namespace không thuộc App nào
```

### 2. RBAC Policy

```yaml
# argocd-rbac-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly    # default: chỉ xem

  policy.csv: |
    # Format: p, <role>, <resource>, <action>, <object>
    # Format: g, <user/group>, <role>

    # Admin role
    p, role:admin, *, *, *

    # Developer role: chỉ deploy lên dev/staging
    p, role:developer, applications, get, */dev-*
    p, role:developer, applications, sync, */dev-*
    p, role:developer, applications, get, */staging-*
    p, role:developer, applications, sync, */staging-*

    # Ops role: deploy mọi nơi nhưng không xoá Application
    p, role:ops, applications, *, */*
    p, role:ops, clusters, get, *
    p, role:ops, repositories, *, *

    # Assign groups (từ SSO/OIDC)
    g, myorg:platform-team, role:admin
    g, myorg:dev-team, role:developer
    g, myorg:ops-team, role:ops
```

---

## Multi-cluster Deployment

```yaml
# Đăng ký external cluster vào ArgoCD
argocd cluster add prod-cluster --name prod
argocd cluster add staging-cluster --name staging

# Xem clusters
argocd cluster list

# Application deploy lên external cluster
spec:
  destination:
    server: https://prod-cluster.internal:6443
    namespace: myapp
```

**ArgoCD tạo ServiceAccount trong remote cluster** với quyền cần thiết, lưu kubeconfig trong Secret.

---

## Notifications

```yaml
# argocd-notifications-cm
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.slack: |
    token: $slack-token
  
  template.app-sync-succeeded: |
    slack:
      message: |
        Application {{.app.metadata.name}} synced successfully.
        Revision: {{.app.status.sync.revision}}
  
  trigger.on-sync-succeeded: |
    - when: app.status.sync.status == 'Synced'
      send: [app-sync-succeeded]
  
  trigger.on-sync-failed: |
    - when: app.status.sync.status == 'OutOfSync'
      send: [app-sync-failed]
```

---

## CLI cheatsheet

```bash
# Login
argocd login argocd.internal.com --username admin

# App management
argocd app list
argocd app get myapp
argocd app diff myapp               # xem diff Git vs cluster
argocd app sync myapp               # sync thủ công
argocd app sync myapp --dry-run     # dry run
argocd app sync myapp --force       # force replace (xoá + tạo lại)
argocd app sync myapp --resource apps:Deployment:myapp  # sync 1 resource

# Rollback
argocd app history myapp
argocd app rollback myapp <revision>

# Refresh (re-read Git)
argocd app get myapp --hard-refresh

# Logs
argocd app logs myapp --container main

# Cluster
argocd cluster list
argocd cluster add <context-name>

# Repo
argocd repo list
argocd repo add https://github.com/myorg/repo --username git --password <token>

# Project
argocd proj list
argocd proj get production
```

---

## Gotchas

- **`prune: true` nguy hiểm nếu render fail**: nếu Helm/Kustomize render ra empty manifest → ArgoCD xoá toàn bộ resource. Cần `allowEmpty: false` trong syncPolicy.
- **HPA conflict với replicas**: HPA thay đổi `spec.replicas` → ArgoCD thấy drift → revert về Git value → HPA scale lại → loop. Fix: `ignoreDifferences` trên `/spec/replicas`.
- **Sync wave và health check**: ArgoCD đợi resource ở wave trước **Healthy** mới chuyển sang wave sau. Nếu không set health check đúng → stuck.
- **ArgoCD không quản lý namespace mặc định**: phải có `CreateNamespace=true` trong syncOptions hoặc tạo namespace manifest riêng.
- **Webhook secret**: không set webhook secret → bất kỳ ai biết URL đều có thể trigger sync. Luôn set `secret` trong webhook config.
- **Dex và network policy**: Dex cần kết nối ra ngoài (OIDC provider). Nếu cluster có NetworkPolicy strict → Dex không authenticate được.
