# PostgreSQL — 16.x
Tags: #postgresql #database #relational #sql #acid
Last updated: 2026-04-30

---

## 1. What — Nó là cái gì?

PostgreSQL là relational database management system (RDBMS) mã nguồn mở — lưu data trên disk, đảm bảo ACID (Atomicity, Consistency, Isolation, Durability), và hỗ trợ full SQL với extensions mạnh (JSONB, PostGIS, full-text search, partitioning). Đây là source of truth cho data cần persist bền vững trong hầu hết microservices stack.

---

## 2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?

Nếu không có PostgreSQL (hay một RDBMS tương đương), không có nơi nào đảm bảo data không bị mất khi server crash, không có cơ chế transaction để đảm bảo chuyển tiền ngân hàng hoặc trừ stock không bị partial write.

Ba vấn đề PostgreSQL giải quyết mà các alternatives không làm tốt bằng:
- **ACID transactions**: `BEGIN; UPDATE accounts SET balance = balance - 100 WHERE id = 1; UPDATE accounts SET balance = balance + 100 WHERE id = 2; COMMIT;` — hai operations này hoặc đều thành công, hoặc đều không xảy ra. Redis, MongoDB không đảm bảo điều này ở mức độ tương đương.
- **Complex queries**: JOIN nhiều bảng, window functions, CTEs, aggregations — những thứ mà Redis và document stores không làm được, và application code không nên phải tự implement.
- **Data integrity**: Foreign keys, CHECK constraints, NOT NULL, UNIQUE — database tự enforce, application không cần validate thủ công.

---

## 3. When — Dùng khi nào / KHÔNG dùng khi nào?

**Dùng khi:**
- Data là source of truth và không thể mất (user accounts, orders, transactions, inventory)
- Cần JOIN nhiều entities (user + orders + products + payments)
- Cần complex queries: aggregation, window functions, full-text search
- Cần ACID transactions (chuyển tiền, giảm stock, booking)
- Audit trail, compliance — cần lịch sử thay đổi, row-level security
- Data có structure rõ ràng và schema tương đối stable

**KHÔNG dùng khi:**
- Data chủ yếu là read và có thể tolerate stale (→ Redis cache trước)
- Time-series data với write throughput cực cao (millions/s) — dùng TimescaleDB hoặc VictoriaMetrics
- Unstructured blob, file storage — dùng S3/MinIO
- Schema thay đổi rất nhanh và unpredictable — có thể cân nhắc MongoDB, nhưng JSONB của PostgreSQL thường đủ
- Cần horizontal write scaling (sharding) tự động — PostgreSQL Citus hoặc chuyển sang distributed DB

---

## 4. Where — Architecture — Nó nằm ở đâu trong hệ thống?

```
                    ┌─────────────────────────────────────────┐
                    │           Microservices Layer            │
                    │  Service A   Service B   Service C       │
                    └──────┬────────────┬───────────┬─────────┘
                           │            │           │
                    ┌──────▼────────────▼───────────▼─────────┐
                    │              PgBouncer (connection pool)  │
                    │         port 5432 → pooled connections    │
                    └──────────────────┬──────────────────────┘
                                       │
                    ┌──────────────────▼──────────────────────┐
                    │            PostgreSQL Cluster (Patroni)  │
                    │                                          │
                    │   Primary ──WAL stream──► Replica 1      │
                    │      │              ──► Replica 2        │
                    │      │                                   │
                    │   etcd (Patroni DCS — quorum + leader)  │
                    └──────────────────────────────────────────┘
                           │
                    ┌──────▼─────────────────────────────────┐
                    │   Disk (WAL + Data files + TOAST)       │
                    │   Backup: pgBackRest → S3/MinIO         │
                    └────────────────────────────────────────┘
```

**Dependency:**
- Services → PgBouncer (5432) → PostgreSQL Primary (5433)
- Read replicas → PgBouncer replica pool hoặc direct connection
- HA: Patroni dùng etcd (hoặc Consul/ZooKeeper) làm DCS để bầu leader
- Monitoring: postgres_exporter → Prometheus → Grafana
- Kubernetes: CloudNativePG (CNPG) Operator

---

## 5. How — Cơ chế hoạt động

**Process-per-connection:**
Mỗi client connection = 1 OS process (~5-10MB RAM). Với 500 connections trực tiếp → ~2.5GB RAM chỉ cho processes. Đây là lý do **luôn cần PgBouncer** ở trước — pool 500 app connections thành 20-50 actual DB connections.

**MVCC (Multi-Version Concurrency Control):**
Thay vì lock rows khi update, PostgreSQL tạo version mới của row (xmin/xmax). Readers không block writers, writers không block readers. Trade-off: dead tuples tích lũy sau updates/deletes → cần VACUUM để clean up và prevent XID wraparound.

**WAL (Write-Ahead Log):**
Mọi change được ghi vào WAL trước khi apply vào data files. Crash recovery = replay WAL từ last checkpoint. Streaming replication = replica nhận WAL stream liên tục từ primary. Đây cũng là basis cho logical replication và PITR.

**Query planner:**
`EXPLAIN ANALYZE` để xem planner chọn execution plan gì — seq scan vs index scan, nested loop vs hash join. Planner dùng statistics (pg_statistic) để estimate costs. Nếu statistics stale → bad plan. `ANALYZE` cập nhật statistics; autovacuum chạy ANALYZE tự động.

**Checkpoints:**
PostgreSQL flush dirty pages từ shared_buffers ra disk định kỳ (checkpoint). Sau crash, chỉ cần replay WAL từ last checkpoint. `checkpoint_timeout` (default 5 min) và `max_wal_size` kiểm soát tần suất. Checkpoint quá thường → I/O spike. Quá hiếm → recovery lâu sau crash.

---

## 6. Key Config — Cấu hình cần nhớ

```ini
# postgresql.conf — production settings cho 32GB RAM server

# Memory
shared_buffers = 8GB              # 25% RAM — page cache của PostgreSQL
effective_cache_size = 24GB       # 75% RAM — hint cho planner (không allocate thực)
work_mem = 64MB                   # per sort/hash operation. Cẩn thận: 50 conns × 64MB = 3.2GB
maintenance_work_mem = 2GB        # cho VACUUM, CREATE INDEX

# WAL & Replication
wal_level = replica               # cần cho streaming replication (default từ PG10+)
max_wal_senders = 10
wal_keep_size = 1GB               # giữ WAL cho replicas bị lag
synchronous_commit = on           # off = faster writes nhưng có thể mất ~wal_writer_delay data

# Checkpoint
checkpoint_timeout = 15min        # default 5min — tăng để giảm I/O spike
max_wal_size = 4GB                # trigger checkpoint khi WAL vượt qua

# Connection
max_connections = 200             # với PgBouncer ở trước, giá trị này không cần cao
```

**Default values nguy hiểm:**
- `shared_buffers = 128MB` — default quá thấp cho production. Set 25% RAM.
- `max_connections = 100` — với microservices không có pooler, 100 connections hết rất nhanh.
- `synchronous_commit = on` — không phải nguy hiểm, nhưng biết rằng `off` tăng write throughput ~30% với trade-off mất tối đa `wal_writer_delay` (200ms) data khi crash.
- `log_min_duration_statement = -1` — disabled by default. Set `1000` (log queries > 1s) trong production để debug slow queries.

---

## 7. Security Considerations

**Attack surface:**
- Port 5432 exposed ra internet: PostgreSQL không có rate limiting built-in. Brute force password, CVE exploits.
- Superuser `postgres` không có password: default installation trên một số distros.
- `pg_hba.conf trust` method: ai connect từ localhost đều không cần password — nguy hiểm trong shared environments.
- SQL injection qua application: parameterized queries là bắt buộc, không dùng string concatenation.

**Hardening checklist tối thiểu:**
```sql
-- 1. Đổi password postgres superuser
ALTER USER postgres PASSWORD 'strong-random-password';

-- 2. Tạo user riêng cho mỗi service (least privilege)
CREATE USER myapp_user WITH PASSWORD 'app-password';
GRANT CONNECT ON DATABASE myapp TO myapp_user;
GRANT USAGE ON SCHEMA public TO myapp_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO myapp_user;
-- KHÔNG GRANT CREATE, DROP, TRUNCATE trừ khi migration user

-- 3. Row Level Security cho multi-tenant
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.current_tenant')::int);
```

```ini
# pg_hba.conf — không dùng trust, chỉ scram-sha-256
# TYPE  DATABASE  USER      ADDRESS         METHOD
local   all       postgres                  scram-sha-256
hostssl all       all       10.0.0.0/8      scram-sha-256
# KHÔNG có dòng: host all all 0.0.0.0/0 trust
```

```ini
# postgresql.conf — audit logging
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl'    # log tất cả writes và DDL changes
log_connections = on
log_disconnections = on
```

**Kubernetes-specific:**
- CNPG tự động tạo TLS cho in-cluster connections
- Credentials qua Kubernetes Secret, không hardcode trong CNPG Cluster spec
- NetworkPolicy: chỉ cho phép pods trong namespace `production` connect port 5432

---

## 8. Ops Runbook — Production Notes

**Health check:**
```bash
# Replication health
psql -h primary -U postgres -c "
  SELECT client_addr, state, sent_lsn, write_lsn,
         pg_wal_lsn_diff(sent_lsn, write_lsn) AS lag_bytes
  FROM pg_stat_replication;"

# Connection usage
psql -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"

# Long-running queries
psql -c "
  SELECT pid, now() - query_start AS duration, query
  FROM pg_stat_activity
  WHERE state = 'active' AND query_start < now() - interval '30 seconds'
  ORDER BY duration DESC;"

# Bloat check
psql -c "SELECT schemaname, tablename, n_dead_tup, n_live_tup,
                round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
         FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;"
```

**Metrics cần alert:**
| Metric | Threshold | Ý nghĩa |
|--------|-----------|---------|
| `pg_up` | == 0 | Instance down |
| `pg_stat_replication lag` | > 100MB | Replica lag lớn |
| `pg_database_size_bytes` | > 80% disk | Disk sắp đầy |
| Cache hit ratio | < 95% | `shared_buffers` quá nhỏ hoặc working set lớn hơn RAM |
| `pg_stat_bgwriter_maxwritten_clean` | tăng liên tục | Checkpoint quá thường |
| XID age | > 1.5 billion | VACUUM wraparound warning |
| Active connections | > 80% max_connections | Cần tăng pool size |

**Log quan trọng:**
```
FATAL: remaining connection slots reserved for replication      ← max_connections đầy
ERROR: deadlock detected                                        ← application bug
WARNING: out of shared memory                                   ← shared_buffers quá nhỏ
LOG: checkpoint occurring too frequently                        ← tăng max_wal_size
PANIC: could not write to file "pg_wal/..."                    ← disk full
```

**Patroni failover procedure:**
```bash
# Xem trạng thái cluster
patronictl -c /etc/patroni/patroni.yml list

# Manual switchover (graceful — planned maintenance)
patronictl -c /etc/patroni/patroni.yml switchover --master primary-1 --candidate replica-1

# Force failover (emergency)
patronictl -c /etc/patroni/patroni.yml failover --master primary-1 --force

# Reinit replica sau khi fix
patronictl -c /etc/patroni/patroni.yml reinit <cluster-name> <replica-member>
```

---

## 9. Gotchas & Lessons Learned

- **`work_mem` × connections = OOM**: `work_mem = 256MB` với 100 connections, mỗi query có 3 sort operations → 100 × 3 × 256MB = 76GB. Set `work_mem` thấp (16-64MB), chỉ tăng cho specific sessions cần sort lớn với `SET LOCAL work_mem = '256MB'`.
- **Replication slot và disk full**: Replication slot giữ WAL cho replica chưa consumed. Nếu replica bị drop mà quên xóa slot → WAL tích lũy → disk full → primary crash. Monitor `pg_replication_slots` và set `max_slot_wal_keep_size`.
- **VACUUM không chạy kịp → XID wraparound**: PostgreSQL dừng tất cả writes khi XID age > 2 billion để force VACUUM. Monitor `age(datfrozenxid)`, alert > 1.5 billion, panic > 1.9 billion.
- **Index không được dùng vì function trên column**: `WHERE LOWER(email) = 'nghia@co'` không dùng index trên `email`. Phải tạo functional index: `CREATE INDEX ON users (LOWER(email))`.
- **PgBouncer transaction mode và prepared statements**: `pgbouncer pool_mode = transaction` không tương thích với server-side prepared statements. Nhiều ORMs (Django, Rails) dùng prepared statements by default → phải disable hoặc dùng session mode, hoặc PgBouncer 1.21+ với prepared statement tracking.
- **CNPG và `reclaimPolicy`**: Mặc định PVC của CNPG dùng `reclaimPolicy: Delete`. Helm uninstall hoặc xóa Cluster CRD → mất data. Dùng `reclaimPolicy: Retain` trong StorageClass và backup trước mọi thao tác xóa.

---

## 10. Resources

- [PostgreSQL Documentation](https://www.postgresql.org/docs/current/) — official, đặc biệt chương "Server Administration"
- [pgTune](https://pgtune.leopard.in.ua/) — generate postgresql.conf từ hardware specs
- [Use the Index, Luke](https://use-the-index-luke.com/) — indexing và query optimization, rất thực tế
- [Patroni Documentation](https://patroni.readthedocs.io/) — HA setup
- [CloudNativePG](https://cloudnative-pg.io/documentation/) — PostgreSQL trên Kubernetes
- [pgBackRest](https://pgbackrest.org/user-guide.html) — backup/PITR
