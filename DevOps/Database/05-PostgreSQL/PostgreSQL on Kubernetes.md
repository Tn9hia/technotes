---
title: PostgreSQL on Kubernetes
tags:
  - postgresql
  - kubernetes
  - cloudnativepg
  - operators
date: 2026-04-27
---

# PostgreSQL on Kubernetes

## Tại sao chạy Postgres trên K8s?

```
Pros:
  ✓ Unified infrastructure (không cần separate DB servers)
  ✓ Operator tự động HA, failover, backup
  ✓ GitOps-friendly (cluster config as YAML)
  ✓ Resource isolation với namespaces/quotas
  ✓ Tốt cho dev/staging (spin up fast)

Cons:
  ✗ Latency: extra network hop (container networking)
  ✗ Storage: cần hiểu K8s storage classes, performance varies
  ✗ Complexity: operator upgrade, PVC management
  ✗ Production: managed DB (RDS, Cloud SQL) thường ít toil hơn

Production decision:
  Dev/Staging: K8s operators ✓
  Production non-critical: K8s operators ✓ (với proper operator + backup)
  Production critical: Managed DB (RDS/Cloud SQL) hoặc bare-metal + Patroni
```

---

## CloudNativePG (CNPG) — Recommended Operator

CloudNativePG là K8s operator cho PostgreSQL, donated to CNCF. Cách tiếp cận immutable: mỗi change = new pod (không edit in-place).

### Install

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.23/releases/cnpg-1.23.0.yaml

# Verify
kubectl get pods -n cnpg-system
```

### Basic Cluster

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: myapp-db
  namespace: production
spec:
  instances: 3              # 1 primary + 2 standbys
  imageName: ghcr.io/cloudnative-pg/postgresql:16.2

  # PostgreSQL configuration
  postgresql:
    parameters:
      shared_buffers: "256MB"
      effective_cache_size: "1GB"
      work_mem: "32MB"
      maintenance_work_mem: "256MB"
      max_connections: "100"
      log_min_duration_statement: "1000"
      pg_stat_statements.max: "10000"
      pg_stat_statements.track: all
    shared_preload_libraries:
      - pg_stat_statements

  # pg_hba.conf
  pg_hba:
    - host all all 10.0.0.0/8 scram-sha-256

  # Bootstrap: tạo database và user
  bootstrap:
    initdb:
      database: myapp
      owner: myapp_user
      secret:
        name: myapp-db-credentials    # K8s Secret với username/password
      encoding: UTF8
      localeCType: en_US.UTF-8
      localeCollate: en_US.UTF-8
      postInitSQL:
        - CREATE EXTENSION IF NOT EXISTS pg_stat_statements
        - GRANT ALL PRIVILEGES ON DATABASE myapp TO myapp_user

  # Storage
  storage:
    size: 50Gi
    storageClass: fast-ssd      # SSD storage class

  walStorage:
    size: 10Gi
    storageClass: fast-ssd      # separate PVC cho WAL (important for performance)

  # Resources
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "2Gi"
      cpu: "2000m"

  # Anti-affinity: spread pods across nodes
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: cnpg.io/cluster
                operator: In
                values: [myapp-db]
          topologyKey: kubernetes.io/hostname

  # Monitoring
  monitoring:
    enablePodMonitor: true      # tự tạo PodMonitor cho Prometheus

  # Superuser secret
  superuserSecret:
    name: myapp-db-superuser
```

```yaml
# Secret cho application user
apiVersion: v1
kind: Secret
metadata:
  name: myapp-db-credentials
  namespace: production
type: kubernetes.io/basic-auth
data:
  username: bXlhcHBfdXNlcg==    # base64 "myapp_user"
  password: <base64-password>
```

### Backup với CNPG

```yaml
# Scheduled backup to S3
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: myapp-db-backup
  namespace: production
spec:
  schedule: "0 2 * * *"         # 2am daily
  backupOwnerReference: self
  cluster:
    name: myapp-db

---
# Backup destination (trong Cluster spec)
spec:
  backup:
    barmanObjectStore:
      destinationPath: s3://my-pg-backup/production/myapp-db
      s3Credentials:
        accessKeyId:
          name: aws-s3-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: aws-s3-credentials
          key: ACCESS_SECRET_KEY
      wal:
        compression: gzip
        encryption: AES256
      data:
        compression: gzip
        encryption: AES256
        immediateCheckpoint: false
        jobs: 4
    retentionPolicy: "30d"       # giữ backups 30 ngày
```

```bash
# Manual backup
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: myapp-manual-backup-$(date +%Y%m%d)
  namespace: production
spec:
  cluster:
    name: myapp-db
EOF

# Restore cluster từ backup (Point-in-Time)
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: myapp-db-restored
  namespace: production
spec:
  instances: 1
  bootstrap:
    recovery:
      source: myapp-db
      recoveryTarget:
        targetTime: "2026-04-15T14:30:00+07:00"
  externalClusters:
    - name: myapp-db
      barmanObjectStore:
        destinationPath: s3://my-pg-backup/production/myapp-db
        s3Credentials:
          accessKeyId:
            name: aws-s3-credentials
            key: ACCESS_KEY_ID
          secretAccessKey:
            name: aws-s3-credentials
            key: ACCESS_SECRET_KEY
EOF
```

### CNPG Plugin

```bash
# Install kubectl cnpg plugin
kubectl krew install cnpg

# Status
kubectl cnpg status myapp-db -n production
# → Cluster summary: 3 instances, 1 primary, 2 standbys
# → Current LSN, lag, last backup

# Promote standby (planned switchover)
kubectl cnpg promote myapp-db myapp-db-2 -n production

# Psql into primary
kubectl cnpg psql myapp-db -n production

# Check logs
kubectl cnpg logs cluster myapp-db -n production

# Hibernate cluster (scale to 0 — cho dev/test cost saving)
kubectl cnpg hibernate on myapp-db -n production
kubectl cnpg hibernate off myapp-db -n production
```

---

## Connecting Applications

### Service Endpoints

CNPG tạo Services tự động:

```bash
# Services tạo bởi CNPG
kubectl get svc -n production | grep myapp-db

# myapp-db-rw     → primary (read/write)
# myapp-db-ro     → standbys (read-only, round-robin)
# myapp-db-r      → all instances (random)
# myapp-db-any    → any live instance
```

```yaml
# Application dùng primary endpoint
env:
  - name: DB_HOST
    value: myapp-db-rw.production.svc.cluster.local
  - name: DB_PORT
    value: "5432"
  - name: DB_NAME
    value: myapp
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: myapp-db-credentials
        key: username
  - name: DB_PASS
    valueFrom:
      secretKeyRef:
        name: myapp-db-credentials
        key: password

# Read replicas cho analytics/reporting
env:
  - name: DB_READONLY_HOST
    value: myapp-db-ro.production.svc.cluster.local
```

### PgBouncer với CNPG

```yaml
# Pooler CRD — CNPG managed PgBouncer
apiVersion: postgresql.cnpg.io/v1
kind: Pooler
metadata:
  name: myapp-db-pooler-rw
  namespace: production
spec:
  cluster:
    name: myapp-db
  instances: 2                  # 2 PgBouncer pods cho HA
  type: rw                      # rw = route to primary; ro = route to standbys

  pgbouncer:
    poolMode: transaction
    parameters:
      max_client_conn: "1000"
      default_pool_size: "25"
      reserve_pool_size: "5"

  template:
    spec:
      resources:
        requests:
          memory: "128Mi"
          cpu: "100m"
        limits:
          memory: "256Mi"
```

---

## Storage Considerations

```bash
# Storage class performance matters:
# Standard HDD → random IOPS ~100 → poor for Postgres
# SSD (gp3 EKS) → 3000 IOPS baseline → acceptable
# io1/io2 → up to 64000 IOPS → good for high-traffic

# Verify storage class
kubectl get storageclass

# EKS: annotate default SC hoặc specify trong CNPG
# gp3 recommended: better IOPS/cost ratio than gp2
```

```yaml
# StorageClass cho EKS với gp3
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer   # schedule pod trước, provision PVC sau
reclaimPolicy: Retain                     # KHÔNG tự xóa PVC khi cluster deleted
allowVolumeExpansion: true
```

---

## Database Migrations trong K8s

### Pattern: Init Container

```yaml
# Deployment với init container chạy migrations
spec:
  initContainers:
    - name: db-migrate
      image: myapp:v2.0.0            # same image as app
      command: ["./migrate", "up"]   # hoặc: flask db upgrade, alembic upgrade head
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-connection
              key: url
  containers:
    - name: app
      image: myapp:v2.0.0
```

### Pattern: Kubernetes Job

```yaml
# Tách migration thành Job riêng (chạy trước deployment)
apiVersion: batch/v1
kind: Job
metadata:
  name: myapp-migration-v2-0-0
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: myapp:v2.0.0
          command: ["./migrate", "up"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-connection
                  key: url
```

### Migration Safety Checklist

```
1. Backward-compatible migrations:
   - ADD COLUMN (nullable hoặc có DEFAULT) → safe
   - ADD INDEX CONCURRENTLY → safe (non-blocking)
   - DROP COLUMN/TABLE → phải đảm bảo app không dùng nữa (deploy app trước, migrate sau)

2. Dangerous operations:
   - ADD NOT NULL column không có DEFAULT → lock table
   - RENAME column/table → breaks queries
   - Change column type → may lock, may fail

3. Large table migrations:
   - ADD COLUMN với DEFAULT (PostgreSQL 11+): instant (stored as catalog default)
   - CREATE INDEX CONCURRENTLY: non-blocking, nhưng cần maintenance_work_mem
   - Large UPDATE: batch updates, không 1 UPDATE cho toàn bộ table

4. Zero-downtime pattern (expand-contract):
   Step 1: ADD new_column
   Step 2: Write to BOTH old_column and new_column (dual-write period)
   Step 3: Backfill new_column từ old_column
   Step 4: Switch reads to new_column
   Step 5: Remove old_column (sau khi stable)
```

---

## Gotchas

- **PVC và failover**: CNPG sử dụng PVC per pod. Khi failover, standby với sẵn data được promote (không provision PVC mới). Cần `reclaimPolicy: Retain` để không mất data nếu Cluster resource bị xóa.
- **WAL separate PVC**: CNPG khuyến khích `walStorage` riêng biệt với `storage`. WAL write pattern (sequential append) khác với heap (random). Separate PVC cho phép tune I/O riêng biệt và tránh WAL fill data disk.
- **PgBouncer và CNPG secret rotation**: CNPG hỗ trợ rotate credentials — sau khi rotate, PgBouncer cần reload. CNPG Pooler tự handle nếu secret rotation qua CNPG API.
- **Node maintenance và PDB**: CNPG tự tạo PodDisruptionBudget (minAvailable: 1). Node drain sẽ respect PDB — sẽ không drain nếu chỉ còn 1 postgres pod available.
- **Migration jobs và connection count**: Migration tools như Flyway/Liquibase mở nhiều connections. Với PgBouncer, verify migration tool tương thích với transaction pool mode.
- **Backup và encryption**: Backup files trên S3 nên được encrypt (AES256 trong CNPG barmanObjectStore config). S3 bucket cũng nên enable default encryption và block public access.
