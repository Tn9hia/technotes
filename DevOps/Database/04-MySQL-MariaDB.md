---
tags: [database, mysql, mariadb, innodb, replication, galera, proxysql, sql, devops]
---

# 🐬 MySQL & MariaDB — Deep Dive

> MySQL vẫn là workhorse của phần lớn web applications. Nắm vững từ architecture đến HA setup thực tế. Xem [[01-Terminology#ACID]] và [[02-Architecture]] cho nền tảng.

---

## Architecture — InnoDB Deep Dive

```
┌──────────────────────────────────────────────────────────┐
│                     MySQL Server                          │
│                                                           │
│  ┌────────────────────────────────────────────────────┐  │
│  │              Connection Layer                       │  │
│  │  max_connections = 151 (default, often too low)    │  │
│  └────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────┐  │
│  │              SQL Layer                              │  │
│  │  Parser → Optimizer → Executor                     │  │
│  └────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────┐  │
│  │              InnoDB Storage Engine                  │  │
│  │                                                     │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │         Buffer Pool (70-80% RAM)             │  │  │
│  │  │  - Data pages (16KB default)                 │  │  │
│  │  │  - Index pages                               │  │  │
│  │  │  - Undo log pages                            │  │  │
│  │  │  - Insert buffer                             │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────┐  ┌───────────────────────┐  │  │
│  │  │   Redo Log      │  │    Undo Log            │  │  │
│  │  │ (ib_logfile0,1) │  │ (in system tablespace) │  │  │
│  │  │ WAL for crash   │  │ MVCC + Rollback        │  │  │
│  │  │ recovery        │  │                        │  │  │
│  │  └─────────────────┘  └───────────────────────┘  │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │           Data Files                         │  │  │
│  │  │  ibdata1 (system), *.ibd (per-table)         │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### InnoDB Clustered Index
- **Mọi table InnoDB đều có clustered index (thường là PRIMARY KEY)**
- Data rows được stored theo PRIMARY KEY order trong B-Tree
- **Secondary indexes:** Lưu PK value → 2 lookups (secondary B-Tree → PK → data row)

```sql
-- Clustered index structure
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,  -- Clustered on this
    user_id INT,
    total DECIMAL(10,2),
    INDEX idx_user (user_id)  -- Secondary: stores (user_id, id) pairs
);

-- Secondary index lookup:
-- 1. Search idx_user B-Tree for user_id = 100 → get id values
-- 2. Search PK B-Tree for each id → get full row
-- (called "double lookup" or "bookmark lookup")
```

### Redo Log (WAL)
```
innodb_log_files_in_group = 2       # ib_logfile0, ib_logfile1 (circular)
innodb_log_file_size = 1GB          # Larger = more recovery time, less I/O spikes
innodb_log_buffer_size = 256MB      # Buffer before write to redo log
innodb_flush_log_at_trx_commit = 1  # 0=no flush, 1=sync(safest), 2=OS cache
```

**`innodb_flush_log_at_trx_commit` là critical:**
- `1` = fsync sau mỗi COMMIT → full ACID, chậm nhất, SAFEST
- `2` = write to OS cache (flush mỗi giây) → mất tối đa 1 giây data nếu crash
- `0` = flush mỗi giây → có thể mất nhiều data, FASTEST

---

## Replication

### Async Binlog Replication (Traditional)
```
Primary                              Replica
┌─────────────────┐                 ┌──────────────────┐
│  Write → Binlog │ ──── network ──►│  IO Thread       │
│  (Statement/    │                 │  (copy binlog)   │
│   Row/Mixed)    │                 │  Relay Log       │
│                 │                 │  SQL Thread      │
│  Ack client ✓  │                 │  (apply events)  │
└─────────────────┘                 └──────────────────┘
                                     (may lag behind)
```

**Binlog formats:**
- `STATEMENT`: Log SQL statements — nhỏ nhưng non-deterministic queries có thể diverge
- `ROW`: Log actual row changes — lớn hơn nhưng accurate (production recommendation)
- `MIXED`: Dùng statement khi safe, row khi không

### GTID Replication (Global Transaction ID)
```
GTID = server_uuid:transaction_id
Example: 3E11FA47-71CA-11E1-9E33-C80AA9429562:1-100

Benefits:
- Easier failover (replica tự biết cần replicate từ đâu)
- Auto-skip duplicates
- Easy to verify replication state
```

```ini
# my.cnf
gtid_mode = ON
enforce_gtid_consistency = ON
```

### Semi-synchronous Replication
```
Primary: Write → Binlog → WAIT for ≥1 replica ACK → COMMIT → Ack client
                           (rEPL_SEMI_SYNC_MASTER_TIMEOUT = 10000ms)
                           (nếu timeout → fallback to async)
```

Bảo vệ chống mất data tốt hơn async, latency thấp hơn synchronous.

---

## High Availability

### MySQL Group Replication (MGR)
```
┌───────────────────────────────────────────────┐
│          MySQL Group Replication               │
│                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Primary  │  │ Secondary│  │ Secondary│   │
│  │(R/W)     │  │ (R only) │  │ (R only) │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │             │             │          │
│       └─────────────┴─────────────┘          │
│              Group Communication             │
│              (Paxos-based, virtual sync)     │
│                                               │
│  Single-Primary mode (default):               │
│  - 1 primary, N secondaries                  │
│  - Auto-failover                             │
│  Multi-Primary mode:                          │
│  - All nodes can write (conflict detection)  │
└───────────────────────────────────────────────┘
```

### Galera Cluster (MariaDB + Percona XtraDB)
```
┌──────────────────────────────────────────────────┐
│                Galera Cluster                     │
│                                                   │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐ │
│  │  Node 1  │     │  Node 2  │     │  Node 3  │ │
│  │ (R/W)    │◄───►│ (R/W)    │◄───►│ (R/W)    │ │
│  └──────────┘     └──────────┘     └──────────┘ │
│                                                   │
│  Synchronous certification-based replication      │
│  Every node is a master (active-active)           │
│  Write committed when certified by quorum         │
│                                                   │
│  SST (State Snapshot Transfer): Full state sync   │
│  IST (Incremental State Transfer): Delta sync     │
└──────────────────────────────────────────────────┘
```

**Galera vs MGR:**
- Galera: Mature, MariaDB native, Galera arbitrator
- MGR: MySQL native, better conflict handling
- Cả hai: Multi-primary với synchronous replication

### ProxySQL
```
Application → ProxySQL → Primary (writes)
                       → Replica 1 (reads)
                       → Replica 2 (reads)

ProxySQL features:
- Query routing (regex-based: SELECT → replicas, INSERT/UPDATE → primary)
- Connection pooling (multiplexing)
- Query caching
- Query rewriting/blocking
- Health monitoring + auto-failover
- Sharding support
```

```sql
-- ProxySQL routing rules
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup)
VALUES (1, 1, '^SELECT', 2),    -- reads → hostgroup 2 (replicas)
       (2, 1, '.*', 1);         -- everything else → hostgroup 1 (primary)
```

---

## Deployment Models

### Standalone
```
Single MySQL server → đơn giản, phù hợp dev/small apps
⚠️ SPOF (Single Point of Failure), limited scale
```

### Primary-Replica
```
Primary (R/W) ──async──► Replica 1 (R)
                    └───► Replica 2 (R)

+ Read scaling, backup from replica
- Manual failover, possible data loss
```

### MGR (MySQL Group Replication)
```
Primary (R/W) ◄──Paxos──► Secondary 1 (R)
                      ◄──► Secondary 2 (R)
+ Auto-failover, single primary or multi-primary
+ Built-in conflict detection
```

### Galera Cluster
```
Node 1 (R/W) ◄──sync──► Node 2 (R/W) ◄──sync──► Node 3 (R/W)
+ All nodes writable, zero data loss
+ Synchronous replication
- Write latency higher (sync overhead)
- Cluster-wide lock for DDL
```

---

## Performance Tuning

### Critical Parameters
```ini
[mysqld]
# Memory
innodb_buffer_pool_size = 12G         # 70-80% RAM (server 16GB → 12G)
innodb_buffer_pool_instances = 8      # 1 per GB, max 64
innodb_log_file_size = 2G             # Larger = fewer I/O spikes, longer recovery

# I/O
innodb_flush_log_at_trx_commit = 1    # 1=safest, 2=faster but risky
innodb_flush_method = O_DIRECT        # Bypass OS cache, avoid double-buffering
innodb_io_capacity = 2000             # IOPs available (SSD: 2000-10000)
innodb_io_capacity_max = 4000         # Max IOPs for background tasks

# Connections
max_connections = 500                 # Keep below 1000 (use ProxySQL pooling)
thread_cache_size = 100               # Reuse threads
back_log = 1000                       # Connection queue

# Slow Query
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1                   # Log queries > 1 second
log_queries_not_using_indexes = ON    # Log index-less queries

# Binary Log
binlog_format = ROW
expire_logs_days = 7
binlog_row_image = MINIMAL            # Reduce binlog size
```

### Indexing Best Practices
```sql
-- ✅ Composite index: most selective first, then range column last
CREATE INDEX idx_user_status_created ON orders(user_id, status, created_at);
-- Query: WHERE user_id = 1 AND status = 'active' ORDER BY created_at → uses full index

-- ❌ Leading wildcard → no index
SELECT * FROM products WHERE name LIKE '%phone%';

-- ✅ Full-text search instead
ALTER TABLE products ADD FULLTEXT INDEX ft_name(name);
SELECT * FROM products WHERE MATCH(name) AGAINST('phone');

-- ✅ Covering index (include all columns in query)
CREATE INDEX idx_covering ON orders(user_id, status, total);
SELECT total FROM orders WHERE user_id = 1 AND status = 'active';
-- → Index Only Scan, không cần read data rows!

-- ❌ Index trên low-cardinality column (boolean, enum với ít values)
-- MySQL không dùng index nếu selectivity thấp
CREATE INDEX idx_gender ON users(gender); -- Bad (only 2 values)

-- ✅ Partial index workaround với composite
CREATE INDEX idx_active ON orders(status, user_id) WHERE status = 'active';
-- Chỉ index active orders → nhỏ hơn, nhanh hơn
```

---

## EXPLAIN Output — Cách đọc

```sql
EXPLAIN SELECT u.name, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id;
```

```
+----+-------------+-------+------------+------+-------------------+----------+---------+------------------+------+----------+----------------------------------------------+
| id | select_type | table | partitions | type | possible_keys     | key      | key_len | ref              | rows | filtered | Extra                                        |
+----+-------------+-------+------------+------+-------------------+----------+---------+------------------+------+----------+----------------------------------------------+
|  1 | SIMPLE      | u     | NULL       | range| idx_created       | idx_created| 5     | NULL             | 5000 |   100.00 | Using index condition; Using temporary; Using filesort |
|  1 | SIMPLE      | o     | NULL       | ref  | idx_user_id       | idx_user_id| 4     | mydb.u.id        |    5 |   100.00 | NULL                                         |
+----+-------------+-------+------------+------+-------------------+----------+---------+------------------+------+----------+----------------------------------------------+
```

**Cột `type` (từ tệ đến tốt):**
```
ALL          → Full table scan ❌ (fix ngay!)
index        → Full index scan (khá là bad)
range        → Range scan using index ✅
ref          → Non-unique index lookup ✅
eq_ref       → Unique index lookup (JOIN) ✅✅
const/system → Single row (PK/UNIQUE = constant) ✅✅✅
```

**`Extra` cần chú ý:**
- `Using filesort` → Sort không dùng index → cần thêm ORDER BY index
- `Using temporary` → Tạo temp table → cần index cho GROUP BY/DISTINCT
- `Using index` → Covering index, không đọc data row → rất tốt
- `Using index condition` → Index Condition Pushdown → tốt
- `Using where` → Filter sau khi fetch rows

---

## MariaDB vs MySQL — Điểm khác biệt

| Feature | MySQL 8.0 | MariaDB 10.x |
|---------|-----------|--------------|
| Engine | InnoDB | InnoDB + Aria + ColumnStore |
| Galera | Plugin (Percona) | Native built-in |
| JSON | JSON datatype | Dynamic columns |
| Window Functions | ✅ MySQL 8.0+ | ✅ MariaDB 10.2+ |
| Invisible Indexes | ✅ | ✅ |
| Sequence | ❌ | ✅ Native |
| Spider Engine | ❌ | ✅ (sharding) |
| Temporal Tables | Limited | ✅ System-versioned tables |
| Thread Pool | Commercial only | ✅ Free |
| Oracle compatibility | Limited | ✅ Better (PL/SQL compat) |
| Licensing | GPL + commercial | GPL (community-friendly) |
| Performance | Generally similar | Often slightly faster for writes |
| MySQL 8.0 compat | N/A | ❌ Not fully compatible (diverging) |

**Khi nào chọn MariaDB:**
- Self-hosted, prefer GPL
- Cần Galera Cluster native
- European/privacy-conscious deployments (MariaDB Foundation là EU-based)
- Need ColumnStore for analytics

**Khi nào chọn MySQL:**
- AWS RDS/Aurora ecosystem (Aurora MySQL-compatible)
- Oracle support required
- Group Replication native

---

## Ưu/Nhược điểm

### Ưu điểm
```
✅ Mature, proven at scale (Facebook, Twitter, Google all use MySQL)
✅ Huge ecosystem (tools, ORMs, hosting)
✅ Read scaling tốt với replicas
✅ Strong InnoDB engine
✅ Great managed options (AWS RDS, Aurora, PlanetScale)
✅ Simple to get started
```

### Nhược điểm
```
❌ Write scaling limited (single primary)
❌ DDL operations lock tables (pt-online-schema-change needed)
❌ Weaker JSON support vs PostgreSQL JSONB
❌ No built-in full-text search (need ES integration)
❌ Limited advanced features (no RETURNING, limited window functions in older versions)
❌ Replication lag với heavy write workloads
❌ innodb_buffer_pool_size cần tune thủ công
```

---

## Quick Operations Reference

```bash
# Check replication status
SHOW SLAVE STATUS\G
SHOW MASTER STATUS;

# Check buffer pool hit rate
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';

# Find slow queries
mysqldumpslow -s t -n 10 /var/log/mysql/slow.log

# Check open connections
SHOW PROCESSLIST;
SHOW STATUS LIKE 'Threads_connected';

# Kill long-running query
KILL QUERY <process_id>;

# Analyze table (update statistics)
ANALYZE TABLE users;

# Check InnoDB status (deadlocks, transactions)
SHOW ENGINE INNODB STATUS\G
```

---

## Related Notes

- [[00-Database-MOC]] — Hub trung tâm
- [[01-Terminology#ACID]] — ACID properties
- [[01-Terminology#WAL]] — WAL concepts
- [[02-Architecture#Replication]] — Replication architecture
- [[05-PostgreSQL]] — PostgreSQL comparison
- [[10-Operations#Backup]] — mysqldump, xtrabackup
- [[10-Operations#Connection Pooling]] — ProxySQL
