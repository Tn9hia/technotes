---
tags: [database, postgresql, postgres, mvcc, patroni, pgbouncer, pgvector, sql, devops, index]
---

# 🐘 PostgreSQL — Deep Dive

> PostgreSQL là "the most advanced open source relational database". Xem [[01-Terminology#MVCC]] và [[02-Architecture]] cho nền tảng chung.

## Contents

| File | Nội dung |
|---|---|
| [[DevOps/Database/05-PostgreSQL/Overview\|Overview]] | What/Why/When/Where/How, key config, security hardening, ops runbook, gotchas |
| [[Architecture & Internals]] | Process model (1 process/connection), MVCC + transaction isolation, WAL durability, VACUUM & XID wraparound, shared_buffers, TOAST |
| [[DevOps/Database/05-PostgreSQL/High Availability\|High Availability]] | Streaming replication setup & monitoring, replication slots, Patroni auto-failover + HAProxy, PgBouncer connection pooling, logical replication & zero-downtime upgrade |
| [[Performance Tuning]] | EXPLAIN ANALYZE (reading plans, red flags), index types (B-tree, GIN, GiST, BRIN, partial, covering), table partitioning (range/hash/list), configuration tuning, anti-patterns |
| [[Operations & Security]] | pg_dump / pg_basebackup / PITR với pgBackRest, pg_stat_* monitoring, Prometheus exporter, pgBadger log analysis, pg_hba.conf, SSL, RBAC least privilege, Row-Level Security, pgAudit |
| [[PostgreSQL on Kubernetes]] | CloudNativePG operator, ScheduledBackup to S3, PITR restore, CNPG Pooler (PgBouncer), storage class tuning, migration patterns (init container vs Job) |
| [[Extensions & Comparison]] | pg_stat_statements, PostGIS, TimescaleDB, pgvector, so sánh ưu/nhược với MySQL |

---

## Quick Reference

### Connection String

```bash
# psql
psql "postgresql://user:password@host:5432/dbname?sslmode=require"

# With pgBouncer (transaction mode)
psql "postgresql://user:password@pgbouncer-host:6432/dbname"

# CNPG primary
psql "postgresql://myapp_user:pass@myapp-db-rw.production:5432/myapp"
```

### Replication Health

```sql
-- Primary: xem replication lag
SELECT application_name, replay_lag FROM pg_stat_replication;

-- Standby: xem lag từ primary
SELECT now() - pg_last_xact_replay_timestamp() AS lag;
```

### Kill Blocking Queries

```sql
SELECT pg_cancel_backend(pid)    -- graceful (SIGINT)
FROM pg_stat_activity
WHERE (now() - query_start) > interval '30 seconds'
  AND state != 'idle';
```

### Vacuum Health

```sql
SELECT relname, n_dead_tup,
       round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```

### Slow Queries

```sql
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

---

## Extensions

PostgreSQL's killer feature: extensibility — pg_stat_statements, PostGIS, TimescaleDB, pgvector. Chi tiết đầy đủ (kèm ví dụ SQL) tại [[Extensions & Comparison]].

---

## Autovacuum

Reclaim dead tuples (MVCC bloat), update statistics, và ngăn transaction ID wraparound. Chi tiết cơ chế tại [[Architecture & Internals]], tuning parameters tại [[Performance Tuning]].

```sql
-- Check table bloat
SELECT relname, n_live_tup, n_dead_tup,
       n_dead_tup::float / NULLIF(n_live_tup + n_dead_tup, 0) * 100 AS dead_pct,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Check XID age (> 2 billion = freeze!)
SELECT datname, age(datfrozenxid)
FROM pg_database
ORDER BY age DESC;
-- If age > 1.5 billion → run VACUUM FREEZE manually!
```

---

## Related Notes

- [[00-Database-MOC]] — Hub trung tâm
- [[01-Terminology#MVCC]] — MVCC concepts
- [[01-Terminology#WAL]] — WAL concepts
- [[02-Architecture#WAL Architecture]] — WAL deep dive
- [[04-MySQL-MariaDB]] — MySQL comparison
- [[09-VectorDB]] — pgvector cho AI use cases
- [[10-Operations#Backup]] — pg_dump, pg_basebackup, PITR
- [[10-Operations#Connection Pooling]] — PgBouncer
