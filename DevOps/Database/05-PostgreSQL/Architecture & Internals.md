---
title: PostgreSQL Architecture & Internals
tags:
  - postgresql
  - database
  - internals
date: 2026-04-27
---

# PostgreSQL Architecture & Internals

Hiểu internals giúp debug performance issues và đưa ra quyết định đúng về indexing, vacuuming, và replication.

## Process Architecture

```
┌──────────────────────────────────────────────────────┐
│  PostgreSQL Server                                    │
│                                                      │
│  postmaster (master process)                         │
│    ├─ postgres (backend) ← 1 per client connection   │
│    ├─ postgres (backend)                             │
│    ├─ autovacuum launcher                            │
│    ├─ autovacuum worker (0..N)                       │
│    ├─ WAL writer                                     │
│    ├─ background writer                              │
│    ├─ checkpointer                                   │
│    ├─ stats collector                                │
│    └─ logical replication launcher                   │
│                                                      │
│  Shared Memory (shared_buffers)                      │
│    ├─ Buffer pool (pages cached from disk)           │
│    ├─ WAL buffers                                    │
│    └─ Lock tables, catalog cache                     │
└──────────────────────────────────────────────────────┘
```

**Quan trọng:** Mỗi client connection = 1 OS process (không phải thread). Connection overhead = ~5MB/connection. Dùng PgBouncer để pool connections.

---

## MVCC — Multi-Version Concurrency Control

MVCC là cơ chế cho phép reads và writes không block lẫn nhau.

### Cách hoạt động

```sql
-- Mỗi row có hidden system columns:
-- xmin: transaction ID đã INSERT/UPDATE row này
-- xmax: transaction ID đã DELETE/UPDATE row này (0 = still live)
-- ctid: physical location (page, row) trong heap

SELECT xmin, xmax, ctid, * FROM orders LIMIT 5;
```

```
UPDATE orders SET status='shipped' WHERE id=1:

Row cũ (xmin=100, xmax=200, status='pending')   ← vẫn visible với txn < 200
Row mới (xmin=200, xmax=0,   status='shipped')  ← visible với txn >= 200

DELETE chỉ set xmax, không xóa row khỏi disk ngay.
→ cần VACUUM để reclaim space.
```

### Transaction Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| Read Uncommitted | ✗ (PG treats as RC) | ✓ | ✓ |
| Read Committed (default) | ✗ | ✓ | ✓ |
| Repeatable Read | ✗ | ✗ | ✗ (PG prevents) |
| Serializable | ✗ | ✗ | ✗ |

```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- snapshot tạo lúc BEGIN, không thấy changes từ các txn khác
COMMIT;

BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- SSI (Serializable Snapshot Isolation) — detect và rollback conflicts
-- Ứng dụng phải retry khi nhận serialization failure
```

---

## WAL — Write-Ahead Log

WAL đảm bảo durability và là nền tảng của replication.

### Cách hoạt động

```
COMMIT transaction
  │
  ├─ 1. Write WAL record to WAL buffer
  ├─ 2. Flush WAL buffer to WAL files (pg_wal/)
  │      → fsync() hoặc O_DSYNC
  │      → CHƯA write dirty pages to data files
  └─ 3. Return success to client

Background:
  WAL writer     → flush WAL buffer định kỳ
  Checkpointer   → write dirty pages to data files + create checkpoint record
  Background writer → pre-write dirty pages để reduce checkpoint I/O spike
```

```
Crash recovery:
  PostgreSQL start
    → Read latest checkpoint record (trong pg_control)
    → Replay WAL records từ checkpoint forward
    → Database consistent lại
```

### WAL Configuration

```ini
# postgresql.conf

# fsync: đảm bảo WAL flush thực sự (KHÔNG tắt trong production)
fsync = on

# synchronous_commit: trade durability cho latency
synchronous_commit = on          # safe: flush WAL trước khi return COMMIT
# synchronous_commit = off       # faster: COMMIT ngay, WAL flush async (có thể mất 1-2 txn trên crash)
# synchronous_commit = remote_apply  # replication: chờ standby apply WAL

# wal_level: lượng thông tin ghi vào WAL
wal_level = replica          # cần cho streaming replication
# wal_level = logical        # cần cho logical replication

# checkpoint_timeout: max interval giữa checkpoints
checkpoint_timeout = 5min    # default 5min; tăng để reduce I/O nhưng tăng recovery time

# max_wal_size: WAL accumulation trước khi force checkpoint
max_wal_size = 1GB           # tăng để giảm tần suất checkpoint

# archive_mode + archive_command: copy WAL files cho PITR
archive_mode = on
archive_command = 'aws s3 cp %p s3://my-pg-wal/%f'
```

---

## VACUUM & Autovacuum

### Tại sao cần VACUUM?

```
INSERT/UPDATE/DELETE → dead tuples tích lũy trong heap
Dead tuples:
  1. Chiếm disk space (bloat)
  2. Slows sequential scans (phải skip qua chúng)
  3. Wrap-around: Transaction ID (XID) là 32-bit = 4 billion max
     → Sau 2 billion transactions: "transaction ID wraparound"
     → PostgreSQL shutdown để tránh data loss nếu không vacuum đúng lúc!

VACUUM: mark dead tuples as reusable (KHÔNG shrink disk ngay)
VACUUM FULL: rewrite table, shrink disk (cần exclusive lock — disruptive)
ANALYZE: update statistics cho query planner
```

### Autovacuum

```sql
-- Xem autovacuum activity
SELECT schemaname, relname,
       n_live_tup, n_dead_tup,
       last_autovacuum, last_autoanalyze,
       autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Check table bloat
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total_size,
    pg_size_pretty(pg_relation_size(tablename::regclass)) AS table_size,
    n_dead_tup,
    n_live_tup,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY dead_pct DESC;
```

```ini
# Autovacuum tuning (postgresql.conf)

autovacuum = on                           # đừng tắt
autovacuum_max_workers = 3               # tăng cho busy clusters
autovacuum_naptime = 1min                # check interval

# Trigger threshold: vacuum khi dead tuples > base + scale * reltuples
autovacuum_vacuum_threshold = 50         # minimum dead tuples trước khi vacuum
autovacuum_vacuum_scale_factor = 0.2     # 20% of table size (giảm cho large tables)
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.1

# Cost-based throttling (prevent vacuum từ eating I/O)
autovacuum_vacuum_cost_delay = 2ms       # default 2ms; 0 = no throttle
autovacuum_vacuum_cost_limit = 200       # increase cho faster vacuum

# Per-table override (cho high-write tables)
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,   -- vacuum khi 1% dead (thay vì 20%)
    autovacuum_vacuum_cost_delay = 0          -- không throttle
);
```

```sql
-- XID wraparound: check tables gần ngưỡng nguy hiểm
SELECT relname, age(relfrozenxid) as xid_age,
       2000000000 - age(relfrozenxid) as xids_remaining
FROM pg_class
WHERE relkind = 'r'
ORDER BY xid_age DESC
LIMIT 20;
-- age > 1.5 billion: cần attention
-- age > 1.9 billion: URGENT — vacuum immediately

-- Force vacuum cho table cụ thể
VACUUM (VERBOSE, ANALYZE) orders;
```

---

## Shared Buffers & Memory

```
Query execution memory path:
  Disk → OS page cache → shared_buffers → work_mem (per operation)

shared_buffers: PostgreSQL buffer pool
  → Cache hot pages
  → Recommendation: 25% RAM (max ~8GB sau đó diminishing returns)
  → PostgreSQL cũng dùng OS page cache → "double buffering" ở trên 8GB

work_mem: memory per sort/hash operation
  → Mỗi node trong query plan có thể dùng work_mem
  → Complex query với nhiều joins → nhiều work_mem instances
  → Quá thấp → spill to disk (temp files) → slow
  → Quá cao × max_connections → OOM
  → Formula: (RAM - shared_buffers) / (max_connections * avg_parallel_ops)
  → Thường: 4-64MB

maintenance_work_mem: VACUUM, CREATE INDEX, ALTER TABLE
  → Có thể set cao hơn work_mem
  → Thường: 256MB - 1GB
```

```ini
# postgresql.conf — memory tuning cho 32GB RAM server
shared_buffers = 8GB
effective_cache_size = 24GB    # hint cho planner (OS cache + shared_buffers)
work_mem = 64MB
maintenance_work_mem = 1GB
```

---

## Table Storage & Heap

```
PostgreSQL heap file = sequence of 8KB pages

Page layout:
┌─────────────────────────────────┐
│ Page header (24 bytes)          │
├─────────────────────────────────┤
│ Item pointers (4 bytes each)    │
│ → point to tuples               │
├────────────── ↓ ────────────────┤
│ Free space                      │
├────────────── ↑ ────────────────┤
│ Tuples (rows) stored from end   │
│ Each tuple: header + data       │
└─────────────────────────────────┘

TOAST (The Oversized-Attribute Storage Technique):
  Column values > ~2KB → stored in separate TOAST table
  → Compressed and/or out-of-line
  → Transparent to queries
  → Affects: jsonb columns, text blobs, bytea
```

```sql
-- Check TOAST table size
SELECT
    relname AS table,
    pg_size_pretty(pg_total_relation_size(oid)) AS total,
    pg_size_pretty(pg_relation_size(oid)) AS heap,
    pg_size_pretty(pg_total_relation_size(oid) - pg_relation_size(oid)) AS toast_and_indexes
FROM pg_class
WHERE relkind = 'r'
ORDER BY pg_total_relation_size(oid) DESC
LIMIT 10;
```
