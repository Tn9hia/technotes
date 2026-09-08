---
title: Redis on Kubernetes
tags:
  - redis
  - kubernetes
  - helm
  - bitnami
  - sentinel
date: 2026-04-30
---

# Redis on Kubernetes

## Deployment Options

| Option | Use case | Complexity |
|--------|----------|-----------|
| Bitnami Redis Helm (standalone) | Dev, small prod | Low |
| Bitnami Redis Helm (sentinel) | Production HA | Medium |
| Bitnami Redis Cluster Helm | Large scale / sharding | High |
| Redis Enterprise Operator | Enterprise features | High |

---

## Bitnami Redis — Sentinel Mode (Production HA)

```yaml
# values.yaml
architecture: replication     # standalone | replication (với Sentinel)

auth:
  enabled: true
  existingSecret: redis-secret
  existingSecretPasswordKey: password

master:
  count: 1
  persistence:
    enabled: true
    storageClass: "gp3"
    size: 20Gi
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi

replica:
  replicaCount: 2
  persistence:
    enabled: true
    storageClass: "gp3"
    size: 20Gi
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi

sentinel:
  enabled: true
  quorum: 2
  masterSet: mymaster
  downAfterMilliseconds: 5000
  failoverTimeout: 60000

# Metrics sidecar
metrics:
  enabled: true
  image:
    repository: oliver006/redis_exporter
    tag: v1.62.0
  serviceMonitor:
    enabled: true
    namespace: monitoring
    interval: 30s

# redis.conf overrides
commonConfiguration: |-
  maxmemory 900mb
  maxmemory-policy allkeys-lru
  lazyfree-lazy-eviction yes
  lazyfree-lazy-expire yes
  lazyfree-lazy-server-del yes
  activedefrag yes
  tcp-keepalive 60
```

```bash
# Install
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Create secret
kubectl create secret generic redis-secret \
  --from-literal=password=$(openssl rand -base64 32) \
  -n redis

helm install redis bitnami/redis \
  -f values.yaml \
  -n redis \
  --create-namespace \
  --version 19.6.0

# Verify
kubectl get pods -n redis
kubectl exec -n redis redis-master-0 -- redis-cli -a $PASS INFO replication
```

### Service Endpoints

```bash
# Sentinel endpoint (clients dùng cái này)
redis-sentinel.redis.svc.cluster.local:26379

# Direct master (không dùng cho app — failover sẽ thay đổi IP)
redis-master.redis.svc.cluster.local:6379

# Replica (read-only)
redis-replicas.redis.svc.cluster.local:6379
```

### Application Connection (via Sentinel)

```yaml
# Kubernetes Deployment environment variables
env:
  - name: REDIS_SENTINEL_HOSTS
    value: "redis-sentinel.redis.svc.cluster.local:26379"
  - name: REDIS_MASTER_SET
    value: "mymaster"
  - name: REDIS_PASSWORD
    valueFrom:
      secretKeyRef:
        name: redis-secret
        key: password
```

```python
# Python redis-py với Sentinel
from redis.sentinel import Sentinel

sentinel = Sentinel([
    ('redis-sentinel.redis.svc.cluster.local', 26379),
], password=os.getenv('REDIS_PASSWORD'))

master = sentinel.master_for('mymaster')
replica = sentinel.slave_for('mymaster')
```

---

## StorageClass cho Redis

```yaml
# storageclass-gp3.yaml — AWS gp3 với tuned IOPS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-redis
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"         # 3000 IOPS baseline (increase cho write-heavy Redis)
  throughput: "125"    # MB/s
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain  # IMPORTANT: Retain data khi PVC bị xóa
allowVolumeExpansion: true
```

---

## NetworkPolicy cho Redis

```yaml
# Chỉ cho phép pods với label app=myapp kết nối tới Redis
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-allow-app
  namespace: redis
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: redis
  policyTypes:
    - Ingress
  ingress:
    # Application access
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: production
          podSelector:
            matchLabels:
              redis-client: "true"
      ports:
        - port: 6379
        - port: 26379

    # Prometheus scrape
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - port: 9121    # metrics exporter

    # Inter-pod replication (Redis cluster + sentinel)
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: redis
```

---

## Redis Cluster với Helm

```yaml
# values-cluster.yaml (bitnami/redis-cluster)
cluster:
  nodes: 6             # 3 masters + 3 replicas
  replicas: 1          # replicas per master

password: ""           # set via existingSecret
existingSecret: redis-cluster-secret
existingSecretPasswordKey: password

persistence:
  enabled: true
  storageClass: "gp3"
  size: 20Gi

redis:
  resources:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 2Gi
  configmap: |-
    maxmemory 1800mb
    maxmemory-policy allkeys-lru

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring
```

```bash
helm install redis-cluster bitnami/redis-cluster \
  -f values-cluster.yaml \
  -n redis \
  --version 11.0.0
```

---

## ConfigMap Override Pattern

```yaml
# redis-config.yaml — custom redis.conf snippet
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-extra-config
  namespace: redis
data:
  redis-extra.conf: |
    # Keyspace notifications (requires replication safe config)
    notify-keyspace-events "Ex"    # Expired events

    # Slow log
    slowlog-log-slower-than 5000
    slowlog-max-len 256

    # Latency monitoring
    latency-monitor-threshold 100
    latency-tracking yes

    # Disable dangerous commands
    rename-command FLUSHDB ""
    rename-command FLUSHALL ""
    rename-command DEBUG ""
    rename-command CONFIG ""        # hoặc đổi tên thành secret value
```

---

## Backup & Restore trên Kubernetes

```yaml
# CronJob backup RDB snapshot
apiVersion: batch/v1
kind: CronJob
metadata:
  name: redis-backup
  namespace: redis
spec:
  schedule: "0 2 * * *"    # 2AM daily
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: redis:7-alpine
              command:
                - /bin/sh
                - -c
                - |
                  redis-cli -h redis-master -a $REDIS_PASSWORD BGSAVE
                  sleep 5
                  redis-cli -h redis-master -a $REDIS_PASSWORD LASTSAVE
                  # Copy RDB via kubectl cp or S3 upload
                  kubectl cp redis/redis-master-0:/data/dump.rdb \
                    /backup/dump-$(date +%Y%m%d-%H%M%S).rdb
              env:
                - name: REDIS_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: redis-secret
                      key: password
```

---

## Keyspace Notifications trong K8s

Hữu ích để invalidate application cache khi Redis key expire.

```bash
# Enable trong redis.conf / ConfigMap
notify-keyspace-events "Ex"    # E = keyevent events, x = expired

# Subscribe tới expired events
redis-cli PSUBSCRIBE "__keyevent@0__:expired"
# → nhận notification mỗi khi key expire trong DB 0

# Use case: application cache invalidation
# Application subscribe và clear local cache khi nhận expired event
```

---

## Sidecar Pattern — Cache Warming

```yaml
# Init container warm cache trước khi app container start
initContainers:
  - name: cache-warmer
    image: myapp:latest
    command:
      - /bin/sh
      - -c
      - python warm_cache.py --redis-host redis-master.redis.svc.cluster.local
    env:
      - name: REDIS_PASSWORD
        valueFrom:
          secretKeyRef:
            name: redis-secret
            key: password
```

---

## Gotchas

- **StatefulSet và PVC lifecycle**: Helm uninstall không xóa PVCs (vì `reclaimPolicy: Retain`). Đây là intentional behavior — prevents data loss. Xóa PVCs manually sau khi confirm data không cần.
- **Sentinel và Kubernetes Service**: Sentinel trả về IP:port của master hiện tại. Trong K8s, IP của Pod thay đổi sau restart. Cần client hỗ trợ Sentinel protocol để resolve tên, không hardcode IP. Một số ORMs/frameworks không hỗ trợ Sentinel — check compatibility trước.
- **`rename-command CONFIG ""`**: Disable `CONFIG` command ngăn `CONFIG REWRITE` và `CONFIG SET` từ bên ngoài. Nếu dùng redis_exporter — exporter dùng `CONFIG GET maxmemory` để export metrics. Set `--redis-only-metrics` hoặc rename CONFIG thành secret string thay vì disable hoàn toàn.
- **Anti-affinity và 3 replicas/nodes**: Pod anti-affinity `requiredDuringSchedulingIgnoredDuringExecution` với topologyKey=kubernetes.io/hostname require >= 3 nodes cho 3 Redis pods. Với cluster nhỏ → pods Pending. Dùng `preferredDuringSchedulingIgnoredDuringExecution` hoặc tăng node count.
- **Bitnami Sentinel và failover**: Sau Sentinel failover, cũ master trở thành replica của master mới. Bitnami Helm set lại Service selector để trỏ tới master mới. Nhưng có window (~5-10s) mà master Service trỏ sai. Applications phải handle connection errors với retry logic.
- **Memory limit vs maxmemory**: Set `limits.memory` trong K8s > `maxmemory` trong redis.conf. Nếu `limits.memory <= maxmemory`, Redis có thể bị OOMKilled bởi kernel trước khi trigger eviction. Rule: `limits.memory = maxmemory × 1.2` (cho overhead: memory fragmentation, AOF buffer, connection buffers).
