---
title: Deployment Strategies
tags:
  - cicd
  - deployment
  - kubernetes
  - deep-dive
date: 2026-04-26
---

# Deployment Strategies

## Tổng quan

| Strategy | Downtime | Risk | Rollback speed | Resource cost | Use case |
|---|---|---|---|---|---|
| **Recreate** | Yes | High | Fast (re-deploy old) | Low | Dev/staging, stateful apps |
| **Rolling** | No | Medium | Slow (wait rollout) | Low | Standard production |
| **Blue/Green** | No | Low | Instant | 2x | Critical services |
| **Canary** | No | Very Low | Instant | Slightly more | High-traffic, gradual rollout |
| **A/B Testing** | No | Very Low | Instant | More | Feature testing per segment |

---

## Recreate

Stop tất cả old pods → start new pods. Brief downtime.

```yaml
# Kubernetes Deployment
spec:
  strategy:
    type: Recreate
```

```
v1 v1 v1
   ↓ stop all
         ↓ start all
v2 v2 v2
```

**Khi nào dùng:**
- Dev/staging environment
- Database migrations không tương thích (không thể chạy song song v1+v2)
- Stateful apps với exclusive resource access

---

## Rolling Update (default K8s)

Replace pods từng phần, đảm bảo luôn có minimum capacity.

```yaml
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # tối đa thêm 1 pod (tổng = 5)
      maxUnavailable: 1    # tối đa 1 pod unavailable (tổng ≥ 3)
      # Cả 2 có thể là số hoặc % (25%)
```

```
Replicas=4, maxSurge=1, maxUnavailable=1:

Bước 1: v1 v1 v1 v1        (4 running)
Bước 2: v1 v1 v1 v2 v2     (5 running — surge)
Bước 3: v1 v1 v2 v2 v2     (v1 being terminated)
Bước 4: v1 v2 v2 v2        (1 old + 3 new)
Bước 5: v2 v2 v2 v2        (done)
```

**Zero-downtime rolling:**
```yaml
# maxUnavailable=0: không có pod nào bị terminate trước khi new pod healthy
rollingUpdate:
  maxSurge: 1
  maxUnavailable: 0

# Quan trọng: readinessProbe phải đúng
# New pod không nhận traffic cho đến khi readinessProbe pass
spec:
  containers:
    - readinessProbe:
        httpGet:
          path: /health/ready
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5
        failureThreshold: 3
```

**Rollback:**
```bash
kubectl rollout undo deployment/myapp            # rollback về revision trước
kubectl rollout undo deployment/myapp --to-revision=3  # rollback cụ thể
kubectl rollout history deployment/myapp         # xem history
kubectl rollout status deployment/myapp          # monitor progress
kubectl rollout pause deployment/myapp           # pause rolling update
kubectl rollout resume deployment/myapp          # resume
```

---

## Blue/Green Deployment

Duy trì 2 environments song song — chuyển traffic bằng 1 thao tác.

```
                    ┌──────────────────────┐
Users ─► Load Balancer / Ingress            │
                    └──┬───────────────────┘
                       │ 100% traffic
                       ▼
               ┌───────────────┐    ┌───────────────┐
               │  Blue (v1)    │    │  Green (v2)   │
               │  ACTIVE       │    │  STANDBY      │
               └───────────────┘    └───────────────┘
                    ^current                 ^new version deployed here
```

```
Switch:
               ┌───────────────┐    ┌───────────────┐
               │  Blue (v1)    │    │  Green (v2)   │
               │  STANDBY      │    │  ACTIVE ◄─────┼── 100% traffic
               └───────────────┘    └───────────────┘
                    ^keep for rollback
```

### Implementation với K8s Service selector

```yaml
# Blue deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
        - image: myapp:v1

---
# Green deployment (deploy và test trước khi switch)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
        - image: myapp:v2

---
# Service — switch bằng cách thay đổi selector
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue    # ← thay thành "green" để switch
  ports:
    - port: 80
      targetPort: 8080
```

```bash
# Deploy green và test
kubectl apply -f myapp-green.yaml

# Smoke test green trực tiếp (qua pod IP)
kubectl run test --rm -it --image=curlimages/curl -- \
  curl http://myapp-green-pod-ip:8080/health

# Switch traffic
kubectl patch service myapp -p '{"spec":{"selector":{"version":"green"}}}'

# Verify
kubectl get endpoints myapp

# Rollback (instant)
kubectl patch service myapp -p '{"spec":{"selector":{"version":"blue"}}}'

# Cleanup blue sau khi stable
kubectl delete deployment myapp-blue
```

### Với Ingress (Nginx/Traefik)

```yaml
# Production ingress → blue
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  rules:
    - host: myapp.internal
      http:
        paths:
          - backend:
              service:
                name: myapp-blue    # ← switch to myapp-green
                port:
                  number: 80

# Preview ingress → green (test trước khi switch)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-preview
spec:
  rules:
    - host: myapp-preview.internal
      http:
        paths:
          - backend:
              service:
                name: myapp-green
                port:
                  number: 80
```

---

## Canary Deployment

Route một tỷ lệ nhỏ traffic tới version mới, tăng dần sau khi verify.

```
                    ┌─────────────────────────────┐
Users ─► Ingress ──►│ 95% → stable (v1)           │
                    │  5% → canary (v2)            │
                    └─────────────────────────────┘
```

```
Stage 1: 5% canary   → monitor error rate, latency
Stage 2: 20% canary  → monitor
Stage 3: 50% canary  → monitor
Stage 4: 100% canary → full rollout (rename to stable)
```

### Nginx Ingress Canary

```yaml
# Stable deployment (v1)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9      # 90% của capacity
  template:
    spec:
      containers:
        - image: myapp:v1

---
# Canary deployment (v2)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1      # 10% của capacity
  template:
    spec:
      containers:
        - image: myapp:v2

---
# Stable service
apiVersion: v1
kind: Service
metadata:
  name: myapp-stable
spec:
  selector:
    app: myapp
    version: stable

---
# Canary service
apiVersion: v1
kind: Service
metadata:
  name: myapp-canary
spec:
  selector:
    app: myapp
    version: canary

---
# Main ingress (stable — 100%)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  rules:
    - host: myapp.internal
      http:
        paths:
          - backend:
              service:
                name: myapp-stable
                port:
                  number: 80

---
# Canary ingress (weight-based)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% traffic
    # Hoặc header-based:
    # nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    # nginx.ingress.kubernetes.io/canary-by-header-value: "true"
spec:
  rules:
    - host: myapp.internal
      http:
        paths:
          - backend:
              service:
                name: myapp-canary
                port:
                  number: 80
```

```bash
# Tăng canary weight dần
kubectl annotate ingress myapp-canary \
  nginx.ingress.kubernetes.io/canary-weight=25 --overwrite

kubectl annotate ingress myapp-canary \
  nginx.ingress.kubernetes.io/canary-weight=50 --overwrite

# Full rollout
kubectl annotate ingress myapp-canary \
  nginx.ingress.kubernetes.io/canary-weight=100 --overwrite

# Rollback (ngay lập tức)
kubectl annotate ingress myapp-canary \
  nginx.ingress.kubernetes.io/canary-weight=0 --overwrite
```

### Argo Rollouts (advanced canary)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 5           # 5% traffic
        - pause: {duration: 5m}  # đợi 5 phút
        - setWeight: 20
        - pause: {}              # manual approval
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
      canaryService: myapp-canary
      stableService: myapp-stable
      trafficRouting:
        nginx:
          stableIngress: myapp
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 2
        args:
          - name: service-name
            value: myapp-canary

  selector:
    matchLabels:
      app: myapp
  template:
    # ... pod spec

---
# AnalysisTemplate — auto rollback nếu error rate cao
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.95    # 95% success rate
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[1m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[1m]))
```

```bash
# Argo Rollouts CLI
kubectl argo rollouts get rollout myapp
kubectl argo rollouts promote myapp          # tiếp tục pause
kubectl argo rollouts abort myapp            # rollback về stable
kubectl argo rollouts set image myapp myapp=myapp:v2  # deploy new version
```

---

## A/B Testing

Route traffic theo user segment (header, cookie, user ID) — khác canary (không phải random %).

```yaml
# Nginx Ingress — header-based routing
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-beta
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Beta-User"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
    # Hoặc cookie:
    # nginx.ingress.kubernetes.io/canary-by-cookie: "beta_user"
spec:
  rules:
    - host: myapp.internal
      http:
        paths:
          - backend:
              service:
                name: myapp-v2
                port:
                  number: 80
```

```bash
# Beta users nhận v2
curl -H "X-Beta-User: true" https://myapp.internal/

# Regular users nhận v1 (không có header)
curl https://myapp.internal/
```

---

## Smoke Tests sau Deploy

```bash
#!/bin/bash
# smoke-test.sh — chạy sau mỗi deployment
set -e

BASE_URL=${1:-https://staging.myapp.internal}
MAX_RETRIES=10
RETRY_DELAY=10

echo "Running smoke tests against $BASE_URL"

# Wait for deployment to be ready
for i in $(seq 1 $MAX_RETRIES); do
  HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL/health")
  if [ "$HTTP_CODE" = "200" ]; then
    echo "Health check passed"
    break
  fi
  echo "Attempt $i/$MAX_RETRIES: got $HTTP_CODE, retrying in ${RETRY_DELAY}s..."
  sleep $RETRY_DELAY
done

# Functional tests
curl -sf "$BASE_URL/health" | jq -e '.status == "ok"'
curl -sf "$BASE_URL/api/v1/ping" | jq -e '.pong == true'
curl -sf -o /dev/null -w "%{http_code}" "$BASE_URL/api/v1/users" | grep -q "200\|401"

echo "All smoke tests passed!"
```

---

## Gotchas

- **Rolling update và database migrations**: nếu v2 cần DB schema mà v1 không hỗ trợ → rolling update có lúc cả v1 và v2 chạy song song → fail. Pattern: **expand-then-contract** (add column compatible với v1, deploy v2, remove column trong release sau).
- **Session stickiness trong blue/green**: nếu user sessions lưu in-memory (không distributed), switch từ blue sang green → user mất session. Dùng Redis/Memcached cho session store.
- **Canary và stateful services**: canary với databases hay message queues cần cẩn thận — v1 và v2 có thể write schema-incompatible data.
- **Readiness probe quan trọng hơn liveness**: liveness probe fail → container restart (disruptive). Readiness probe fail → remove từ service endpoints (graceful). Rolling update phụ thuộc readiness probe → nếu probe sai → update treo.
- **`maxUnavailable: 0` và slow rollout**: khi `maxUnavailable=0`, rollout cần thêm pod mới healthy trước khi terminate old → cần đủ cluster capacity. Nếu cluster tight → rollout treo vì không schedule được pod mới.
- **Blue/Green và PVC**: stateful apps với PVC không thể dùng Blue/Green dễ dàng — PVC chỉ có thể bound bởi 1 pod (RWO). Cần shared storage (RWX) hoặc data migration plan.
