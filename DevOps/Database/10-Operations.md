---
tags: [database, operations, devops, backup, monitoring, tuning, dr]
---

# Database Operations

> Vận hành database production: backup, monitoring, migration, security, disaster recovery.

## Liên kết nhanh
- [[00-Database-MOC]] — Tổng quan
- [[04-MySQL-MariaDB]] — MySQL/MariaDB Operations
- [[05-PostgreSQL]] — PostgreSQL Operations
- [[06-MongoDB]] — MongoDB Operations
- [[07-Redis]] — Redis Operations
- [[08-Elasticsearch]] — Elasticsearch Operations

---

## Backup Strategies

### Logical vs Physical Backup

```
Logical Backup:
  Data → SQL statements / JSON / BSON
  Pros: portable, can restore to different version/schema
  Cons: slow (có thể mất giờ với data lớn), high CPU

Physical Backup:
  Data files trực tiếp (binary copy)
  Pros: rất nhanh, consistent snapshot
  Cons: phải cùng major version, cùng OS/architecture
```

### MySQL / MariaDB

```bash
# Logical: mysqldump (đơn giản, không phù hợp DB lớn)
mysqldump -h localhost -u root -p \
  --single-transaction \          # consistent snapshot (InnoDB)
  --routines \                     # include stored procedures
  --triggers \                     # include triggers
  --databases mydb \
  | gzip > /backup/mydb-$(date +%Y%m%d).sql.gz

# Restore
gunzip < mydb-20260315.sql.gz | mysql -h localhost -u root -p mydb

# Physical: Percona XtraBackup (production recommended)
# Full backup
xtrabackup --backup \
  --user=root --password=secret \
  --target-dir=/backup/full/

# Prepare (apply redo logs → consistent state)
xtrabackup --prepare --target-dir=/backup/full/

# Incremental backup (sau full backup)
xtrabackup --backup \
  --target-dir=/backup/inc1/ \
  --incremental-basedir=/backup/full/

# Restore
systemctl stop mysql
xtrabackup --copy-back --target-dir=/backup/full/
chown -R mysql:mysql /var/lib/mysql/
systemctl start mysql
```

### PostgreSQL

```bash
# Logical: pg_dump
pg_dump -h localhost -U postgres \
  -F c \                           # custom format (compressed, allows parallel restore)
  -d mydb \
  -f /backup/mydb-$(date +%Y%m%d).dump

# pg_dumpall: dump tất cả databases + global objects (roles, tablespaces)
pg_dumpall -U postgres > /backup/all-$(date +%Y%m%d).sql

# Restore
pg_restore -h localhost -U postgres \
  -d mydb \
  -j 4 \                           # 4 parallel jobs
  /backup/mydb-20260315.dump

# Physical: pg_basebackup (streaming backup)
pg_basebackup -h localhost -U replicator \
  -D /backup/base/ \
  -Ft \                            # tar format
  -z \                             # gzip compress
  -P \                             # progress
  --wal-method=stream              # stream WAL during backup

# WAL archiving (cần cho PITR)
# postgresql.conf:
archive_mode = on
archive_command = 'cp %p /archive/%f'   # hoặc upload S3
wal_level = replica
```

### MongoDB

```bash
# mongodump (logical)
mongodump \
  --host localhost:27017 \
  --username admin \
  --password secret \
  --authenticationDatabase admin \
  --db mydb \
  --out /backup/mongo-$(date +%Y%m%d)/

# Restore
mongorestore \
  --host localhost:27017 \
  --username admin \
  --password secret \
  --drop \                          # drop existing collections trước
  --db mydb \
  /backup/mongo-20260315/mydb/

# mongodump toàn bộ
mongodump --uri "mongodb+srv://..." --out /backup/

# Cloud: MongoDB Atlas có built-in automated backup
```

### Redis

```bash
# RDB snapshot
redis-cli BGSAVE
redis-cli LASTSAVE    # timestamp

# Backup file
cp /var/lib/redis/dump.rdb /backup/redis-$(date +%Y%m%d).rdb

# AOF backup
cp /var/lib/redis/appendonly.aof /backup/aof-$(date +%Y%m%d).aof

# Restore
systemctl stop redis
cp /backup/redis-20260315.rdb /var/lib/redis/dump.rdb
systemctl start redis
```

---

## PITR — Point-in-Time Recovery

Khôi phục database về một thời điểm cụ thể trong quá khứ (cực quan trọng khi có accidental DELETE):

### PostgreSQL PITR

```bash
# Setup: pg_basebackup + WAL archiving (xem phần Backup)

# Kịch bản: 14:30 có ai đó DROP TABLE. Cần restore về 14:29.

# 1. Stop PostgreSQL
systemctl stop postgresql

# 2. Restore base backup
rm -rf /var/lib/postgresql/data/*
tar -xzf /backup/base.tar.gz -C /var/lib/postgresql/data/

# 3. Tạo recovery config
cat > /var/lib/postgresql/data/postgresql.conf << EOF
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2026-03-15 14:29:00+07'
recovery_target_action = 'promote'
EOF

touch /var/lib/postgresql/data/recovery.signal

# 4. Start PostgreSQL → tự apply WAL đến 14:29
systemctl start postgresql

# Monitor recovery progress
tail -f /var/log/postgresql/postgresql.log
```

### MySQL PITR với binlog

```bash
# Kịch bản: DROP TABLE lúc 14:30, cần restore về 14:29

# 1. Restore full backup (từ XtraBackup)
xtrabackup --copy-back --target-dir=/backup/full/

# 2. Apply binlog từ sau full backup đến trước DROP TABLE
mysqlbinlog \
  --start-datetime="2026-03-15 09:00:00" \
  --stop-datetime="2026-03-15 14:29:00" \
  /var/log/mysql/binlog.000001 \
  /var/log/mysql/binlog.000002 \
  | mysql -u root -p

# Tìm exact position của DROP TABLE
mysqlbinlog --base64-output=DECODE-ROWS -v /var/log/mysql/binlog.000003 | grep -B5 "DROP TABLE"
# → tìm position number

# Apply đến trước position đó
mysqlbinlog --stop-position=12345 /var/log/mysql/binlog.000003 | mysql -u root -p
```

---

## Monitoring — Metrics quan trọng

### Universal Metrics (mọi DB)

```
QPS (Queries Per Second)        — throughput
Query Latency (p50/p95/p99)    — hiệu suất
Error Rate                      — stability
Replication Lag                 — data freshness
Connection Count / Pool Usage   — resource saturation
Disk I/O (read/write IOPS)     — storage bottleneck
Disk Usage %                    — capacity
CPU Usage                       — compute bottleneck
Memory Usage                    — memory pressure
```

### MySQL / MariaDB Metrics

```sql
-- Slow queries
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';

-- Key metrics
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Questions';
SHOW GLOBAL STATUS LIKE 'Slow_queries';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests';
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads';
-- Buffer pool hit rate = (read_requests - reads) / read_requests * 100
-- Target: > 99%

SHOW GLOBAL STATUS LIKE 'Innodb_row_lock_waits';
SHOW GLOBAL STATUS LIKE 'Aborted_connects';

-- Replication lag
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 0 là ideal
```

### PostgreSQL Metrics

```sql
-- Active connections
SELECT count(*), state FROM pg_stat_activity GROUP BY state;

-- Longest running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';

-- Cache hit rate (target > 99%)
SELECT
  sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_rate
FROM pg_statio_user_tables;

-- Index usage
SELECT relname, idx_scan, seq_scan FROM pg_stat_user_tables
WHERE seq_scan > 0 ORDER BY seq_scan DESC;

-- Bloat (dead tuples cần vacuum)
SELECT relname, n_dead_tup, n_live_tup,
  round(n_dead_tup::numeric/nullif(n_live_tup,0)*100, 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000 ORDER BY n_dead_tup DESC;

-- Replication lag
SELECT client_addr,
  pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

### Redis Metrics

```bash
redis-cli INFO stats | grep -E "instantaneous_ops|rejected|evicted|keyspace"
redis-cli INFO memory | grep -E "used_memory_human|mem_fragmentation_ratio"
redis-cli INFO replication | grep -E "master_repl_offset|slave_repl_offset|lag"

# Cache hit rate
redis-cli INFO stats | grep -E "keyspace_hits|keyspace_misses"
# Hit rate = hits / (hits + misses) * 100  →  target > 90%

# Latency
redis-cli --latency -h localhost
redis-cli --latency-history -h localhost -i 1
```

### Elasticsearch Metrics

```bash
# Cluster health
curl -s "localhost:9200/_cluster/health?pretty"

# Node stats
curl -s "localhost:9200/_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m"

# JVM heap (critical: keep < 75%)
curl -s "localhost:9200/_nodes/stats/jvm" | jq '.nodes[].jvm.mem.heap_used_percent'

# Indexing + search rate
curl -s "localhost:9200/_cat/indices?v&h=index,docs.count,store.size,pri.search.query_total"

# Pending tasks (should be 0)
curl -s "localhost:9200/_cluster/pending_tasks"
```

### Tools

```
Percona Monitoring and Management (PMM):
  → MySQL, PostgreSQL, MongoDB monitoring
  → pmm-server (Docker) + pmm-client trên mỗi DB host
  → Built-in dashboards, query analytics, slow query analysis

pgBadger:
  → Analyze PostgreSQL log files → HTML report
  pgbadger /var/log/postgresql/postgresql.log -o report.html

Redis Insight:
  → GUI quản lý Redis, memory analysis, profiler

Grafana + Prometheus:
  → mysqld_exporter, postgres_exporter, redis_exporter
  → Dashboard từ grafana.com/dashboards

Datadog / New Relic / Dynatrace:
  → APM + DB monitoring tích hợp, correlate với application traces
```

---

## Connection Pooling

### PgBouncer (PostgreSQL)

```ini
# pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Pool mode
pool_mode = transaction      # RECOMMENDED: connection released sau mỗi transaction
                             # session: giữ connection suốt session (giống direct)
                             # statement: released sau mỗi statement

max_client_conn = 1000       # max connections từ app
default_pool_size = 25       # connections đến Postgres per database+user
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 3

# Admin
admin_users = postgres
stats_users = stats
```

```bash
# Xem stats
psql -h localhost -p 6432 pgbouncer -U postgres
SHOW POOLS;
SHOW CLIENTS;
SHOW SERVERS;
SHOW STATS;
```

**Transaction vs Session pooling:**
- `session`: 1 client = 1 Postgres connection (ít lợi)
- `transaction`: connection share khi transaction xong → 1000 app connections dùng 25 PG connections
- Lưu ý: transaction mode không support prepared statements, `SET` commands, advisory locks

### ProxySQL (MySQL)

```sql
-- Thêm backend MySQL servers
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES
(0, '10.0.0.1', 3306),  -- write hostgroup
(1, '10.0.0.2', 3306),  -- read hostgroup
(1, '10.0.0.3', 3306);

-- Query routing rules
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply) VALUES
(1, 1, '^SELECT .* FOR UPDATE', 0, 1),    -- SELECT FOR UPDATE → master
(2, 1, '^SELECT', 1, 1);                  -- SELECT → replica

-- Load balancing
LOAD MYSQL SERVERS TO RUNTIME;
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

---

## Schema Migration

### Nguyên tắc an toàn

```
1. Backward compatible migrations (không break code đang chạy)
2. Tách migration thành nhiều bước nhỏ
3. Test trên staging trước
4. Có rollback plan
5. Monitor sau khi deploy
```

### Quy trình an toàn cho ALTER TABLE

```sql
-- ❌ NGUY HIỂM trên table lớn (lock table trong giờ!)
ALTER TABLE orders ADD COLUMN metadata JSONB;

-- ✅ AN TOÀN: thêm column nullable trước
ALTER TABLE orders ADD COLUMN metadata JSONB;
-- → Với PostgreSQL 11+: instant cho nullable column (no table rewrite)

-- ❌ NGUY HIỂM: thêm NOT NULL ngay
ALTER TABLE orders ADD COLUMN status VARCHAR(50) NOT NULL DEFAULT 'pending';

-- ✅ AN TOÀN: multi-step
-- Step 1: Thêm nullable column
ALTER TABLE orders ADD COLUMN status VARCHAR(50);
-- Step 2: Backfill data (batched)
UPDATE orders SET status = 'pending' WHERE id BETWEEN 1 AND 10000;
-- ... repeat in batches
-- Step 3: Add NOT NULL constraint
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;
```

### pt-online-schema-change (MySQL)

```bash
# Alter table không lock:
# 1. Tạo shadow table với schema mới
# 2. Copy data sang shadow table
# 3. Sync incremental changes bằng triggers
# 4. Rename tables (atomic swap)

pt-online-schema-change \
  --host=localhost \
  --user=root \
  --password=secret \
  --alter="ADD INDEX idx_status (status)" \
  --execute \
  D=mydb,t=orders

# gh-ost (GitHub Online Schema Change) — alternative không dùng triggers
gh-ost \
  --host=master.mysql \
  --user=root \
  --password=secret \
  --database=mydb \
  --table=orders \
  --alter="ADD INDEX idx_status (status)" \
  --execute
```

### Migration Tools

```
Flyway (Java/SQL-based):
  Versioned migrations: V1__init.sql, V2__add_index.sql
  Flyway validate → migrate

Liquibase:
  XML/YAML/JSON changelogs
  Rollback support

golang-migrate (Go projects):
  migrate -path ./migrations -database postgres://... up
  migrate -path ./migrations -database postgres://... down 1

Alembic (Python/SQLAlchemy):
  alembic upgrade head
  alembic downgrade -1
```

---

## Disaster Recovery

### RTO vs RPO

```
                    Disaster
                       │
RPO ─────────────────►│◄─────────────── RTO
(data loss tolerance)  │   (recovery time tolerance)
                       │
  "Mất tối đa         │    "Service phải up
   bao nhiêu data?"   │     trong bao lâu?"

RPO = 0        → synchronous replication, zero data loss
RPO = 1 hour   → backup mỗi giờ
RPO = 24 hours → daily backup

RTO = 15 min   → automated failover
RTO = 4 hours  → manual failover
RTO = 24 hours → restore from backup
```

### DR Strategies theo mức độ

```
Tier 1 — Active-Active (RPO≈0, RTO≈0)
  → Multi-region active deployment
  → Data synchronously replicated
  → Cost: rất cao
  → Use: financial systems, payment

Tier 2 — Active-Passive Hot Standby (RPO<1min, RTO<15min)
  → Patroni/MySQL Group Replication
  → Replica warm, auto-failover
  → Cost: 2x infrastructure

Tier 3 — Warm Standby (RPO<1h, RTO<1h)
  → Replica ở region khác, sync async
  → Manual failover
  → Cost: 1.5x infrastructure

Tier 4 — Backup + Restore (RPO=hours, RTO=hours)
  → Daily backup lên S3/GCS
  → Restore when needed
  → Cost: thấp nhất
```

### Failover Automation

```bash
# Patroni (PostgreSQL HA)
# patronictl xem trạng thái cluster
patronictl -c /etc/patroni/patroni.yml list

# Manual failover
patronictl -c /etc/patroni/patroni.yml failover --master current-master --candidate new-master

# MySQL Router + Group Replication
# Automatic failover khi primary fail
mysqlsh -- cluster status  # xem cluster status
mysqlsh -- cluster setPrimaryInstance('mysql2:3306')  # manual switchover
```

---

## Security

### Encryption

```bash
# Encryption at rest
# MySQL: innodb_encrypt_tables, InnoDB tablespace encryption
# PostgreSQL: pg_tde (Transparent Data Encryption) extension
# MongoDB: WiredTiger encryption at rest
# → Hoặc disk-level: LUKS, AWS EBS encryption, GCP disk encryption

# Encryption in transit (TLS)
# MySQL:
[mysqld]
require_secure_transport = ON
ssl-ca   = /etc/mysql/certs/ca.pem
ssl-cert = /etc/mysql/certs/server-cert.pem
ssl-key  = /etc/mysql/certs/server-key.pem

# PostgreSQL:
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file  = '/etc/ssl/private/server.key'
# pg_hba.conf: hostssl all all 0.0.0.0/0 md5
```

### RBAC (Role-Based Access Control)

```sql
-- PostgreSQL: Principle of least privilege
CREATE ROLE readonly;
GRANT CONNECT ON DATABASE mydb TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly;

CREATE USER app_user WITH PASSWORD 'strong_password';
GRANT readonly TO app_user;

-- App chỉ cần INSERT/UPDATE/SELECT — không cần DROP, TRUNCATE
CREATE ROLE app_role;
GRANT SELECT, INSERT, UPDATE ON orders TO app_role;
GRANT USAGE ON SEQUENCE orders_id_seq TO app_role;
```

```sql
-- MySQL:
CREATE USER 'app'@'10.0.0.%' IDENTIFIED BY 'password';
GRANT SELECT, INSERT, UPDATE ON mydb.* TO 'app'@'10.0.0.%';

-- Không bao giờ:
GRANT ALL PRIVILEGES ON *.* TO 'app'@'%';  -- ❌
```

### Audit Logging

```bash
# MySQL: General Query Log (tốn I/O, chỉ bật khi cần debug)
SET GLOBAL general_log = ON;
SET GLOBAL general_log_file = '/var/log/mysql/general.log';

# PostgreSQL: pgaudit extension
# postgresql.conf:
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl'   # log INSERT/UPDATE/DELETE/DDL

# MongoDB: Audit log
# mongod.conf:
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: '{ atype: { $in: ["authenticate", "createCollection", "dropCollection"] } }'
```

### Network Security

```bash
# Không bao giờ expose database port ra internet trực tiếp!

# Firewall rules (ví dụ iptables):
iptables -A INPUT -p tcp --dport 5432 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 5432 -j DROP

# PostgreSQL: pg_hba.conf
# TYPE  DATABASE  USER   ADDRESS       METHOD
host    mydb      app    10.0.0.0/8    scram-sha-256
local   all       all                  peer
# Từ chối tất cả connections khác

# Dùng SSH tunnel cho admin access (không mở port DB ra ngoài):
ssh -L 5432:db-server:5432 jump-host
# → psql -h localhost -p 5432  (tunnel qua SSH)

# Secrets management:
# Không hard-code credentials!
# → Kubernetes Secrets + External Secrets Operator
# → HashiCorp Vault
# → AWS Secrets Manager / GCP Secret Manager
```

---

## Capacity Planning

```
Công thức cơ bản cho sizing:

Database Size:
  Current size × growth_rate × 1.3 (buffer)

IOPS:
  (read_qps × avg_read_io) + (write_qps × avg_write_io)
  MySQL InnoDB: 1 write ≈ 2-4 IOPS (data + WAL)

RAM:
  Buffer pool = 70-80% RAM cho dedicated DB server
  PostgreSQL shared_buffers = 25% RAM (OS cache làm phần còn lại)
  Redis = dataset size + 30% overhead

Connections:
  max_connections = (available_ram - overhead) / connection_ram
  PostgreSQL: mỗi connection ≈ 5-10MB
  MySQL: mỗi connection ≈ 1-4MB
  → Dùng connection pool để không cần max_connections cao

Storage:
  SSDs bắt buộc cho write-heavy databases
  NVMe cho high IOPS workloads
  RAID 10 nếu không dùng cloud storage với redundancy
```

---

## Related Notes

- [[00-Database-MOC]] — Tổng quan toàn bộ
- [[04-MySQL-MariaDB]] — MySQL/MariaDB specific operations
- [[05-PostgreSQL]] — PostgreSQL: Patroni, pgBouncer, PITR
- [[06-MongoDB]] — MongoDB backup, Replica Set failover
- [[07-Redis]] — Redis persistence, Sentinel, Cluster
- [[08-Elasticsearch]] — ILM, snapshot, cluster ops
- [[01-Terminology]] — RTO, RPO, ACID, Replication
