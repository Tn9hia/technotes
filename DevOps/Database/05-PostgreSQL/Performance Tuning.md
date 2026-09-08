---
title: PostgreSQL Performance Tuning
tags:
  - postgresql
  - performance
  - indexing
  - query-optimization
date: 2026-04-27
---

# PostgreSQL Performance Tuning

## EXPLAIN ANALYZE — Reading Query Plans

```sql
-- Basic: chỉ plan, không execute
EXPLAIN SELECT * FROM orders WHERE user_id = 123;

-- Execute và measure actual time
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, u.email
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending'
  AND o.created_at > now() - interval '7 days';
```

### Đọc Query Plan

```
Hash Join  (cost=45.23..892.31 rows=150 width=120)
           (actual time=1.234..25.678 rows=142 loops=1)
  Buffers: shared hit=234 read=45
  ->  Seq Scan on orders  (cost=0.00..750.00 rows=1500 width=100)
                          (actual time=0.021..15.234 rows=1498 loops=1)
        Filter: ((status = 'pending') AND (created_at > ...))
        Rows Removed by Filter: 8502
  ->  Hash  (cost=20.00..20.00 rows=2000 width=20)
            (actual time=0.892..0.892 rows=2000 loops=1)
      ->  Seq Scan on users  (cost=0.00..20.00 rows=2000 width=20)
```

| Metric | Nghĩa |
|---|---|
| `cost=start..total` | Planner estimate (relative units) |
| `rows=N` | Planner estimated row count |
| `actual time=start..end` | Real milliseconds |
| `actual rows=N` | Actual rows returned |
| `loops=N` | Executed N times (×N for nested loops) |
| `shared hit=N` | Pages from buffer cache |
| `shared read=N` | Pages read from disk |
| `Rows Removed by Filter` | Wasted work — cần index |

**Red flags:**
- `Seq Scan` trên large table với Filter → cần index
- `actual rows` >> `rows` estimate → statistics stale → `ANALYZE`
- `loops=N` cao trong Nested Loop → N×M row problem
- `Rows Removed by Filter` >> `actual rows` → inefficient index hoặc không có index

### Visualize Plans

```sql
-- JSON format cho pgAdmin / explain.dalibo.com
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT ...;

-- Auto-explain: log slow queries với plans
-- postgresql.conf
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 1000    -- log queries > 1s
auto_explain.log_analyze = true
auto_explain.log_buffers = true
```

---

## Index Types

### B-tree (default)

```sql
-- Dùng cho: equality, range, ORDER BY, LIKE với prefix
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created ON orders(created_at DESC);

-- Composite: thứ tự columns quan trọng
-- Dùng được khi query filter trên (col1) hoặc (col1, col2) — không phải chỉ col2
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Include columns (covering index): avoid table fetch
CREATE INDEX idx_orders_status_include ON orders(status)
INCLUDE (created_at, total_amount);
-- Query: SELECT created_at, total_amount FROM orders WHERE status='pending'
-- → Index only scan, không cần fetch heap
```

### Partial Index

```sql
-- Index chỉ một subset rows — nhỏ hơn và nhanh hơn full index
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';          -- chỉ pending orders

-- Index chỉ non-null values
CREATE INDEX idx_users_email ON users(email)
WHERE email IS NOT NULL;

-- Query phải include điều kiện WHERE của partial index để dùng được
SELECT * FROM orders WHERE status = 'pending' AND created_at > '2026-01-01';
-- ✓ sẽ dùng idx_orders_pending
```

### GIN Index — Fulltext Search & Arrays

```sql
-- Fulltext search
CREATE INDEX idx_articles_search ON articles
USING gin(to_tsvector('english', title || ' ' || body));

SELECT * FROM articles
WHERE to_tsvector('english', title || ' ' || body) @@ to_tsquery('postgresql & performance');

-- Array operations
CREATE INDEX idx_tags ON posts USING gin(tags);

SELECT * FROM posts WHERE tags @> ARRAY['postgresql'];     -- contains
SELECT * FROM posts WHERE tags && ARRAY['postgresql', 'performance'];  -- overlap

-- JSONB
CREATE INDEX idx_metadata ON events USING gin(metadata);

SELECT * FROM events WHERE metadata @> '{"type": "login"}';
SELECT * FROM events WHERE metadata ? 'user_id';
```

### GiST Index — Geometric & Range Types

```sql
-- Range types
CREATE TABLE reservations (
    room_id int,
    period tstzrange        -- timestamp range
);
CREATE INDEX idx_reservations_period ON reservations USING gist(period);

-- Check overlap
SELECT * FROM reservations
WHERE period && '[2026-01-01, 2026-01-07)'::tstzrange;

-- PostGIS geographic queries
CREATE INDEX idx_locations_geom ON locations USING gist(geom);
SELECT * FROM locations WHERE ST_DWithin(geom, ST_Point(106.8, 10.8), 1000);
```

### BRIN Index — Time-Series / Sequential Data

```sql
-- BRIN (Block Range Index): rất nhỏ, chỉ useful cho naturally ordered data
-- Mỗi entry = min/max cho một range of disk blocks
CREATE INDEX idx_logs_created ON logs USING brin(created_at) WITH (pages_per_range = 128);

-- Good for: append-only tables (logs, metrics, events) với sequential timestamp
-- Not good for: randomly ordered data
```

### Index Maintenance

```sql
-- Check unused indexes (tốn write overhead, xem xét drop)
SELECT schemaname, tablename, indexname,
       idx_scan, idx_tup_read, idx_tup_fetch,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname NOT IN ('pg_catalog', 'pg_toast')
ORDER BY pg_relation_size(indexrelid) DESC;

-- Rebuild bloated indexes concurrently (không lock table)
REINDEX INDEX CONCURRENTLY idx_orders_user_id;

-- Check index bloat
SELECT indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## Table Partitioning

Partitioning chia large table thành smaller physical tables. Queries chỉ scan relevant partitions (partition pruning).

### Range Partitioning (phổ biến nhất — time-series)

```sql
-- Tạo partitioned table
CREATE TABLE logs (
    id bigint GENERATED ALWAYS AS IDENTITY,
    created_at timestamptz NOT NULL,
    level text,
    message text
) PARTITION BY RANGE (created_at);

-- Tạo partitions
CREATE TABLE logs_2026_01 PARTITION OF logs
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE logs_2026_02 PARTITION OF logs
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Default partition (rows không match bất kỳ partition nào)
CREATE TABLE logs_default PARTITION OF logs DEFAULT;

-- Index tự áp dụng cho tất cả partitions
CREATE INDEX ON logs(created_at);
CREATE INDEX ON logs(level, created_at);

-- Insert tự route đến đúng partition
INSERT INTO logs (created_at, level, message)
VALUES (now(), 'error', 'Something failed');

-- Detach old partitions (instant, không lock)
ALTER TABLE logs DETACH PARTITION logs_2026_01 CONCURRENTLY;
-- Partition tồn tại như standalone table → có thể archive hoặc drop
DROP TABLE logs_2026_01;
```

```sql
-- Auto-create monthly partitions với pg_partman
-- CREATE EXTENSION pg_partman;
SELECT partman.create_parent(
    p_parent_table := 'public.logs',
    p_control := 'created_at',
    p_interval := '1 month',
    p_premake := 3          -- tạo trước 3 partitions
);

-- pg_partman maintenance job (chạy qua pg_cron hoặc external scheduler)
SELECT partman.run_maintenance();
```

### Hash Partitioning (even distribution)

```sql
-- Chia đều theo hash value — good for lookup tables
CREATE TABLE users (
    id bigint,
    email text
) PARTITION BY HASH (id);

CREATE TABLE users_0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE users_1 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE users_2 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE users_3 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### List Partitioning

```sql
CREATE TABLE orders (
    id bigint,
    region text,
    amount numeric
) PARTITION BY LIST (region);

CREATE TABLE orders_vn PARTITION OF orders FOR VALUES IN ('VN');
CREATE TABLE orders_sg PARTITION OF orders FOR VALUES IN ('SG');
CREATE TABLE orders_th PARTITION OF orders FOR VALUES IN ('TH');
CREATE TABLE orders_other PARTITION OF orders DEFAULT;
```

---

## Configuration Tuning

```ini
# postgresql.conf — cho server 32GB RAM, 8 CPU, SSD

# ─── Memory ───
shared_buffers = 8GB                 # 25% RAM
effective_cache_size = 24GB          # estimate total (RAM + OS cache available for PG)
work_mem = 64MB                      # per-sort/hash op; adjust per workload
maintenance_work_mem = 2GB           # VACUUM, CREATE INDEX
huge_pages = try                     # use huge pages nếu OS configured

# ─── I/O ───
effective_io_concurrency = 200       # SSD: 200; HDD: 2; NVMe: 500-1000
random_page_cost = 1.1               # SSD: 1.1-2; HDD: 4 (default)
seq_page_cost = 1.0

# ─── WAL / Checkpoints ───
wal_buffers = 64MB                   # default -1 = 1/32 shared_buffers
checkpoint_completion_target = 0.9   # spread checkpoint I/O over 90% of interval
checkpoint_timeout = 15min           # tăng để reduce checkpoint frequency
max_wal_size = 4GB                   # tăng để reduce checkpoint frequency
min_wal_size = 1GB

# ─── Query Planner ───
default_statistics_target = 100      # tăng cho better estimates (cost: analyze time)
# Per-column statistics:
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;

# ─── Parallelism ───
max_worker_processes = 8
max_parallel_workers = 8
max_parallel_workers_per_gather = 4  # parallel seq scans / joins
max_parallel_maintenance_workers = 4 # parallel index builds

# ─── Logging ───
log_min_duration_statement = 1000    # log queries > 1s
log_checkpoints = on
log_connections = off                # noisy với PgBouncer
log_lock_waits = on
log_temp_files = 0                   # log all temp file usage (spills)
```

---

## Connection & Lock Monitoring

```sql
-- Active queries
SELECT pid, now() - pg_stat_activity.query_start AS duration,
       query, state, wait_event_type, wait_event
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 seconds'
  AND state != 'idle'
ORDER BY duration DESC;

-- Blocking queries
SELECT
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    now() - blocked.query_start AS blocked_duration
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE NOT blocked.granted;

-- Kill a query (graceful)
SELECT pg_cancel_backend(pid);

-- Kill a connection (force)
SELECT pg_terminate_backend(pid);

-- Lock waits
SELECT relation::regclass, mode, granted, pid, query
FROM pg_locks l
JOIN pg_stat_activity a USING (pid)
WHERE NOT granted;
```

---

## Statistics & Query Optimization

```sql
-- Refresh statistics (sau bulk load hoặc major changes)
ANALYZE orders;
ANALYZE;           -- tất cả tables

-- Extended statistics: multi-column correlations
CREATE STATISTICS orders_stats ON user_id, status FROM orders;
ANALYZE orders;
-- Planner giờ biết correlation giữa user_id và status

-- pg_stats: xem column statistics
SELECT attname, n_distinct, correlation,
       most_common_vals, most_common_freqs
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';

-- n_distinct < 0: fractional estimate (e.g., -0.1 = 10% distinct values)
-- correlation: 1.0 = physical order matches value order (perfect for range scans)
```

### Common Anti-Patterns

```sql
-- BAD: Function on indexed column → index unusable
SELECT * FROM orders WHERE date_trunc('day', created_at) = '2026-04-27';
-- GOOD: Rewrite as range
SELECT * FROM orders WHERE created_at >= '2026-04-27' AND created_at < '2026-04-28';

-- BAD: Leading wildcard → can't use B-tree
SELECT * FROM users WHERE email LIKE '%@gmail.com';
-- GOOD: pg_trgm index for arbitrary LIKE
CREATE INDEX idx_users_email_trgm ON users USING gin(email gin_trgm_ops);

-- BAD: NOT IN với NULL (returns empty!)
SELECT * FROM orders WHERE user_id NOT IN (SELECT id FROM banned_users);
-- Nếu có NULL trong banned_users → NOT IN returns nothing!
-- GOOD:
SELECT * FROM orders WHERE NOT EXISTS (
    SELECT 1 FROM banned_users WHERE banned_users.id = orders.user_id
);

-- BAD: SELECT * với JOIN (fetch unnecessary columns)
SELECT * FROM orders o JOIN users u ON o.user_id = u.id;
-- GOOD: explicit columns

-- BAD: OFFSET large values (scan all rows)
SELECT * FROM orders ORDER BY created_at DESC OFFSET 10000 LIMIT 20;
-- GOOD: keyset pagination
SELECT * FROM orders
WHERE created_at < :last_seen_created_at
ORDER BY created_at DESC
LIMIT 20;
```

---

## Gotchas

- **work_mem × max_connections**: work_mem không phải per-connection — per operation. Complex query có thể dùng work_mem nhiều lần (multiple sort/hash nodes). Nếu set 256MB × 100 connections × 3 ops/query → OOM. Monitor với `log_temp_files = 0` để biết spills, tăng dần cẩn thận.
- **Correlation và sequential scan**: Planner dùng `correlation` column để chọn index vs seq scan. Nếu correlation thấp (random physical order) → index scan đắt hơn seq scan → planner chọn seq scan → đúng! Đừng force index nếu planner không chọn — investigate nguyên nhân.
- **Parallel query và work_mem**: Parallel query với N workers × work_mem per sort = N×work_mem total. Với `max_parallel_workers_per_gather = 4` và `work_mem = 256MB` → 1GB per query cho sorting.
- **Partial index và implicit cast**: `WHERE status = 'pending'` có thể không dùng partial index `WHERE status = 'pending'::order_status` nếu types không match. Check cast types.
- **Autovacuum vs manual VACUUM**: Manual `VACUUM ANALYZE` sau bulk operations (DELETE/UPDATE nhiều rows) thay vì đợi autovacuum. Autovacuum có cost throttling → có thể lâu hơn manual.
