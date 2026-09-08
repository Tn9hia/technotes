---
tags: [database, postgresql, extensions, pgvector, postgis, timescaledb, mysql]
---

# PostgreSQL — Extensions & So sánh MySQL

> PostgreSQL's killer feature: extensibility. Xem [[05-PostgreSQL#Extensions]] cho tổng quan, và [[09-VectorDB#pgvector]] cho vector search chi tiết.

## Extensions

### pg_stat_statements

```sql
-- Enable
CREATE EXTENSION pg_stat_statements;

-- Find top slow queries
SELECT query, calls, mean_exec_time, total_exec_time, rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Reset stats
SELECT pg_stat_statements_reset();
```

### PostGIS (Geospatial)

```sql
CREATE EXTENSION postgis;

-- Store geometry
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name TEXT,
    coords GEOGRAPHY(POINT, 4326)
);

-- Insert point (longitude, latitude)
INSERT INTO locations (name, coords)
VALUES ('Saigon', ST_MakePoint(106.6602, 10.7626));

-- Find locations within 5km radius
SELECT name, ST_Distance(coords, ST_MakePoint(106.6602, 10.7626)::geography) AS dist
FROM locations
WHERE ST_DWithin(coords, ST_MakePoint(106.6602, 10.7626)::geography, 5000)
ORDER BY dist;
```

### TimescaleDB (Time-series)

```sql
CREATE EXTENSION timescaledb;

-- Convert regular table to hypertable (auto-partitioned by time)
SELECT create_hypertable('metrics', 'time');

-- Automatic chunk management, compression, retention
SELECT add_compression_policy('metrics', INTERVAL '7 days');
SELECT add_retention_policy('metrics', INTERVAL '90 days');

-- Continuous aggregates (materialized views refreshed automatically)
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS hour, avg(value)
FROM metrics
GROUP BY hour;
```

### pgvector (Vector Search for AI)

```sql
CREATE EXTENSION vector;

-- Store embeddings
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(1536)  -- OpenAI ada-002 dimension
);

-- Create HNSW index for fast ANN search
CREATE INDEX ON documents USING hnsw(embedding vector_cosine_ops);

-- Semantic search
SELECT id, content, 1 - (embedding <=> '[0.1, 0.2, ...]') AS similarity
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'  -- <=> = cosine distance
LIMIT 5;
```

Xem chi tiết về Vector DB tại [[09-VectorDB]].

---

## Ưu/Nhược điểm vs MySQL

### PostgreSQL Ưu điểm
```
✅ MVCC implementation tốt hơn (không cần undo log)
✅ Extensions ecosystem (pgvector, PostGIS, TimescaleDB)
✅ JSONB native với GIN index
✅ CTE, window functions, lateral joins, range types
✅ Full-text search built-in (tsvector/tsquery)
✅ Table inheritance, partitioning native
✅ Better write scaling với logical replication
✅ Logical replication cho cross-version migration
✅ CHECK constraints, exclusion constraints
✅ Community-driven, no Oracle drama
```

### PostgreSQL Nhược điểm
```
❌ Slower than MySQL cho simple OLTP reads (due to MVCC overhead)
❌ autovacuum cần tune, có thể gây performance spikes
❌ Connection model: fork per connection → cần PgBouncer
❌ No thread pool (mỗi connection là 1 OS process → memory overhead)
❌ Replication more complex (cần Patroni/repmgr cho HA)
❌ Ít managed cloud options hơn MySQL (nhưng đang tăng)
❌ Upgrade major version cần pg_upgrade hoặc logical replication approach
```

Xem chi tiết MySQL tại [[04-MySQL-MariaDB]].

---

## Use Cases

| Use Case | PostgreSQL Feature |
|----------|-------------------|
| General OLTP | Core PostgreSQL + PgBouncer + Patroni |
| Geo-spatial | PostGIS extension |
| Time-series | TimescaleDB extension |
| Full-text search | tsvector + GIN index |
| JSON document store | JSONB + GIN index |
| AI/Semantic search | pgvector extension |
| Analytics | Window functions, CTEs, parallel query |
| Multi-tenant SaaS | Row-level security (RLS) |
| Event sourcing | Logical replication + LISTEN/NOTIFY |
