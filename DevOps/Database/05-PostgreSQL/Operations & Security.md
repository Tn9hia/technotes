---
title: PostgreSQL Operations & Security
tags:
  - postgresql
  - operations
  - backup
  - monitoring
  - security
date: 2026-04-27
---

# PostgreSQL Operations & Security

## Backup Strategies

### pg_dump — Logical Backup

```bash
# Dump single database (custom format — recommended)
pg_dump \
  -h localhost -U postgres \
  -Fc \                       # custom format: compressed, parallel restore
  -f /backup/myapp_$(date +%Y%m%d).dump \
  myapp

# Schema only
pg_dump -Fc --schema-only -f schema.dump myapp

# Data only (không có CREATE TABLE)
pg_dump -Fc --data-only -f data.dump myapp

# Specific tables
pg_dump -Fc -t orders -t products -f partial.dump myapp

# Dump all databases (global objects + all databases)
pg_dumpall -h localhost -U postgres -f /backup/all_$(date +%Y%m%d).sql
# → Bao gồm roles, tablespaces

# Restore
pg_restore \
  -h localhost -U postgres \
  -d myapp \
  -j 4 \              # parallel restore jobs
  -Fc \
  /backup/myapp_20260427.dump
```

### pg_basebackup — Physical Backup

```bash
# Full cluster backup (bao gồm WAL)
pg_basebackup \
  -h localhost -U replicator \
  -D /backup/base \
  -Ft \               # tar format
  -z \                # gzip
  -P \                # progress
  -Xs \               # stream WAL
  --checkpoint=fast

# Output: base.tar.gz (data) + pg_wal.tar.gz (WAL)
```

### PITR — Point-In-Time Recovery

PITR cho phép restore database đến bất kỳ thời điểm nào, không chỉ tại thời điểm backup.

```
Timeline:
  Base backup (Jan 1) ─── WAL archives ──► PITR target (Jan 15, 14:30)

Setup:
  1. Base backup (pg_basebackup)
  2. Archive WAL continuously (archive_command)
  3. Restore: base backup + replay WAL đến target time
```

```ini
# postgresql.conf — enable WAL archiving
archive_mode = on
archive_command = 'aws s3 cp %p s3://my-pg-wal-archive/%f'
archive_cleanup_command = 'pg_archivecleanup /var/lib/postgresql/wal_archive %r'

# Verify archive works
archive_status = on   # check pg_stat_archiver view
```

```sql
-- Monitor archiving
SELECT archived_count, last_archived_wal, last_archived_time,
       failed_count, last_failed_wal
FROM pg_stat_archiver;
```

```bash
# PITR Restore procedure

# 1. Stop PostgreSQL
systemctl stop postgresql

# 2. Restore base backup
rm -rf /var/lib/postgresql/data
tar -xzf /backup/base.tar.gz -C /var/lib/postgresql/data

# 3. Create recovery.conf (PG 11 và trước) hoặc recovery signal files (PG 12+)
# PG 12+:
touch /var/lib/postgresql/data/recovery.signal

cat >> /var/lib/postgresql/data/postgresql.conf <<EOF
restore_command = 'aws s3 cp s3://my-pg-wal-archive/%f %p'
recovery_target_time = '2026-04-15 14:30:00+07'
recovery_target_action = 'promote'  # promote to primary after recovery
EOF

# 4. Start PostgreSQL — tự replay WAL đến target time
systemctl start postgresql

# 5. Verify
psql -c "SELECT now();"  # xem current state
psql -c "SELECT pg_is_in_recovery();"  # should return false (promoted)
```

### pgBackRest (Production-grade)

pgBackRest là tool backup chuyên nghiệp: incremental, parallel, compression, S3 support.

```ini
# /etc/pgbackrest/pgbackrest.conf
[global]
repo1-type=s3
repo1-s3-bucket=my-pg-backup
repo1-s3-region=ap-southeast-1
repo1-s3-endpoint=s3.amazonaws.com
repo1-path=/pgbackrest
repo1-retention-full=4           # giữ 4 full backups
repo1-retention-diff=14          # giữ 14 differential backups

process-max=4                    # parallel processes
compress-type=lz4
compress-level=3
log-level-console=info
log-level-file=detail

[myapp-cluster]
pg1-path=/var/lib/postgresql/data
pg1-port=5432
```

```bash
# Tạo stanza (cluster definition)
pgbackrest --stanza=myapp-cluster stanza-create

# Full backup
pgbackrest --stanza=myapp-cluster --type=full backup

# Incremental (chỉ backup thay đổi từ lần full backup cuối)
pgbackrest --stanza=myapp-cluster --type=incr backup

# List backups
pgbackrest --stanza=myapp-cluster info

# Restore
pgbackrest --stanza=myapp-cluster restore
# PITR
pgbackrest --stanza=myapp-cluster --type=time \
  "--target=2026-04-15 14:30:00+07" restore
```

---

## Monitoring

### pg_stat_* Views

```sql
-- Database-level stats
SELECT datname,
       numbackends AS connections,
       xact_commit, xact_rollback,
       blks_hit, blks_read,
       round(blks_hit * 100.0 / NULLIF(blks_hit + blks_read, 0), 2) AS cache_hit_ratio,
       tup_returned, tup_fetched, tup_inserted, tup_updated, tup_deleted,
       conflicts, deadlocks
FROM pg_stat_database
WHERE datname NOT IN ('postgres', 'template0', 'template1');

-- Table-level stats
SELECT relname,
       seq_scan, seq_tup_read,
       idx_scan, idx_tup_fetch,
       n_tup_ins, n_tup_upd, n_tup_del,
       n_live_tup, n_dead_tup,
       last_vacuum, last_autovacuum,
       last_analyze, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;

-- Index usage
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Slow queries (pg_stat_statements extension)
CREATE EXTENSION pg_stat_statements;

SELECT query, calls, total_exec_time, mean_exec_time,
       rows, shared_blks_hit, shared_blks_read,
       round(shared_blks_hit * 100.0 / NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS hit_ratio
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Reset stats
SELECT pg_stat_statements_reset();
SELECT pg_stat_reset();
```

### Prometheus Exporter

```bash
# postgres_exporter — expose pg metrics cho Prometheus
docker run -d \
  -e DATA_SOURCE_NAME="postgresql://monitoring:pass@localhost:5432/postgres?sslmode=disable" \
  -p 9187:9187 \
  prometheuscommunity/postgres-exporter

# Hoặc helm
helm install prometheus-postgres-exporter \
  prometheus-community/prometheus-postgres-exporter \
  -n monitoring \
  --set config.datasource.host=postgres-primary \
  --set config.datasource.user=monitoring \
  --set config.datasource.passwordSecret.name=pg-monitoring-secret
```

```yaml
# ServiceMonitor cho kube-prometheus-stack
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: postgres-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: prometheus-postgres-exporter
  endpoints:
    - port: http
      interval: 30s
```

### Key Metrics để Alert

```yaml
# PrometheusRule
groups:
  - name: postgresql.alerts
    rules:
      # Connection saturation
      - alert: PostgreSQLConnectionSaturation
        expr: |
          pg_stat_database_numbackends / pg_settings_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "PostgreSQL connections at {{ $value | humanizePercentage }}"

      # Low cache hit ratio
      - alert: PostgreSQLLowCacheHit
        expr: |
          pg_stat_database_blks_hit /
          NULLIF(pg_stat_database_blks_hit + pg_stat_database_blks_read, 0) < 0.99
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Cache hit ratio {{ $value | humanizePercentage }} (target: 99%)"

      # Replication lag
      - alert: PostgreSQLReplicationLag
        expr: pg_replication_lag > 300    # > 5 minutes
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Replication lag {{ $value }}s on {{ $labels.instance }}"

      # Dead tuples bloat
      - alert: PostgreSQLHighDeadTuples
        expr: |
          pg_stat_user_tables_n_dead_tup /
          NULLIF(pg_stat_user_tables_n_live_tup, 0) > 0.1
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Table {{ $labels.relname }} has >10% dead tuples"

      # XID wraparound approaching
      - alert: PostgreSQLXIDWraparoundApproaching
        expr: pg_database_datfrozenxid > 1500000000
        labels:
          severity: critical
        annotations:
          summary: "XID wraparound approaching for {{ $labels.datname }}"
```

### pgBadger — Log Analysis

```bash
# Generate report từ PostgreSQL logs
pgbadger \
  /var/log/postgresql/postgresql-*.log \
  -o /var/www/html/pgbadger-report.html \
  --format csv \
  --sample 10

# Nên enable trước:
# log_min_duration_statement = 100  (log queries > 100ms)
# log_checkpoints = on
# log_lock_waits = on
# log_temp_files = 0
```

---

## Security

### pg_hba.conf — Authentication

```ini
# pg_hba.conf — format: TYPE DATABASE USER ADDRESS METHOD

# Local connections: trust cho postgres superuser (chỉ local socket)
local   all             postgres                                peer
local   all             all                                     scram-sha-256

# IPv4 local connections
host    all             all             127.0.0.1/32            scram-sha-256

# Replication connections
host    replication     replicator      10.0.0.0/24             scram-sha-256

# App connections từ K8s pods
host    myapp           myapp_user      10.0.0.0/8              scram-sha-256

# SSL required cho external connections
hostssl all             all             0.0.0.0/0               scram-sha-256

# KHÔNG dùng trust cho network connections
# KHÔNG dùng md5 (broken) → dùng scram-sha-256
```

### SSL/TLS

```ini
# postgresql.conf
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'ca.crt'          # verify client certs
ssl_min_protocol_version = 'TLSv1.2'
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'

# Require SSL cho specific users
# pg_hba.conf:
# hostssl all all 0.0.0.0/0 scram-sha-256
```

```bash
# Verify SSL
psql "host=db.internal dbname=myapp user=myapp_user sslmode=require"
# sslmode options: disable, allow, prefer, require, verify-ca, verify-full
```

### RBAC — Roles & Privileges

```sql
-- Principle of least privilege

-- Tạo roles (không phải users trực tiếp)
CREATE ROLE readonly_role NOLOGIN;
GRANT CONNECT ON DATABASE myapp TO readonly_role;
GRANT USAGE ON SCHEMA public TO readonly_role;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly_role;

CREATE ROLE readwrite_role NOLOGIN;
GRANT CONNECT ON DATABASE myapp TO readwrite_role;
GRANT USAGE, CREATE ON SCHEMA public TO readwrite_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite_role;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO readwrite_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO readwrite_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO readwrite_role;

-- Tạo users và assign roles
CREATE USER app_user WITH PASSWORD 'strongpass' CONNECTION LIMIT 50;
GRANT readwrite_role TO app_user;

CREATE USER reporting_user WITH PASSWORD 'strongpass';
GRANT readonly_role TO reporting_user;

-- Revoke default public schema privileges
REVOKE ALL ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE myapp FROM PUBLIC;
```

### Row-Level Security (RLS)

```sql
-- RLS: users chỉ thấy rows của họ
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy: mỗi user chỉ đọc orders của mình
CREATE POLICY orders_isolation ON orders
    USING (user_id = current_setting('app.current_user_id')::int);

-- Set current user context từ application
SET LOCAL app.current_user_id = '123';
SELECT * FROM orders;  -- chỉ thấy orders của user 123

-- Superuser bypass RLS (cần thiết cho admins)
CREATE POLICY orders_admin ON orders
    USING (current_user = 'admin_user');

-- Force policy ngay cả cho table owner
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
```

### Audit Logging với pgAudit

```bash
# Install pgaudit extension
apt-get install postgresql-16-pgaudit
```

```ini
# postgresql.conf
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl, role'   # log: read, write, function, role, ddl, misc, all
pgaudit.log_catalog = off          # không log catalog queries (noise)
pgaudit.log_parameter = on         # log query parameters
pgaudit.log_statement_once = off
```

```sql
-- Per-role audit
CREATE ROLE audited_user LOGIN PASSWORD 'pass';
ALTER ROLE audited_user SET pgaudit.log = 'all';  -- log everything này user làm

-- Per-object audit
SELECT pgaudit.set_object_audit('read,write', 'table', 'public', 'orders');
```

```
# Audit log format:
AUDIT: SESSION,1,1,DDL,CREATE TABLE,TABLE,public.orders,
"CREATE TABLE orders (id serial, user_id int, amount numeric)",<none>

AUDIT: OBJECT,2,1,READ,SELECT,TABLE,public.orders,
"SELECT * FROM orders WHERE user_id = 123",<none>
```

---

## Gotchas

- **pg_dump không consistent với pg_stat_statements**: pg_dump dùng serializable snapshot → consistent. Nhưng pg_dump không bao gồm WAL → không thể dùng cho PITR. Cần kết hợp pg_basebackup + WAL archiving cho PITR.
- **PITR và timezone**: `recovery_target_time` phải specify timezone rõ ràng. `'2026-04-15 14:30:00'` interpreted theo server timezone, không UTC. Luôn dùng `'2026-04-15 07:30:00+00'` để tránh nhầm lẫn.
- **scram-sha-256 và libpq compatibility**: Clients cũ (libpq < 10) không support scram-sha-256. Nếu có legacy apps → dùng `md5` cho user đó nhưng note là weaker.
- **RLS và prepared statements**: RLS evaluate lúc plan (security barrier). Nếu dùng `SECURITY DEFINER` function → RLS bypass. Luôn test RLS với actual user context, không chỉ superuser.
- **Backup và UNLOGGED tables**: `UNLOGGED TABLE` không được write đến WAL → nhanh hơn nhưng **bị truncate khi crash recovery** và **không replicated** và **không included trong pg_basebackup properly**. Không dùng cho persistent data.
