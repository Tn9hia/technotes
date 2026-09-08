---
title: PostgreSQL High Availability
tags:
  - postgresql
  - high-availability
  - patroni
  - replication
date: 2026-04-27
---

# PostgreSQL High Availability

## Replication Overview

```
┌─────────────────────────────────────────────────────────┐
│ Replication types:                                       │
│                                                         │
│  Physical (streaming):                                  │
│    Primary → WAL stream → Standby                       │
│    Standby = byte-for-byte copy of primary              │
│    Use: HA failover, read replicas                      │
│                                                         │
│  Logical:                                               │
│    Primary → decoded WAL (SQL-level changes) → Replica  │
│    Can replicate subset (tables, schemas)               │
│    Replica can have different schema/indexes            │
│    Use: selective replication, zero-downtime upgrades   │
└─────────────────────────────────────────────────────────┘
```

---

## Streaming Replication

### Setup Primary

```ini
# postgresql.conf (primary)
wal_level = replica
max_wal_senders = 10          # max concurrent standby connections
wal_keep_size = 1GB           # keep 1GB of WAL for lagging standbys
# hoặc dùng replication slots (không auto-cleanup — có thể fill disk!)

synchronous_standby_names = '' # async replication (default)
# synchronous_standby_names = 'standby1'  # sync: chờ standby confirm WAL
```

```sql
-- Tạo replication user
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'strongpass';
```

```ini
# pg_hba.conf (primary) — allow replication connections
host replication replicator standby-ip/32 scram-sha-256
```

### Setup Standby

```bash
# Initial sync với pg_basebackup
pg_basebackup \
  -h primary-host \
  -U replicator \
  -D /var/lib/postgresql/data \
  -P \
  -Xs \                    # stream WAL during backup
  -R                       # tự tạo standby.signal và connection config

# pg_basebackup tạo tự động:
# - standby.signal (đánh dấu đây là standby)
# - postgresql.auto.conf với primary_conninfo
```

```ini
# postgresql.conf (standby)
hot_standby = on           # allow read queries on standby
hot_standby_feedback = on  # standby báo cho primary về active queries
                           # → primary không vacuum rows standby đang dùng
                           # → trade-off: primary accumulates more dead tuples

# primary_conninfo (trong postgresql.auto.conf hoặc recovery.conf)
primary_conninfo = 'host=primary-host port=5432 user=replicator password=strongpass application_name=standby1'
```

### Monitor Replication Lag

```sql
-- Trên primary: xem replication status
SELECT
    application_name,
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag,
    sync_state
FROM pg_stat_replication;

-- Lag in bytes
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- Trên standby: xem apply status
SELECT
    now() - pg_last_xact_replay_timestamp() AS replication_lag,
    pg_is_in_recovery() AS is_standby,
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn() AS replayed_lsn;
```

### Replication Slots

```sql
-- Replication slot: primary giữ WAL cho đến khi slot confirm đã nhận
-- Dùng khi standby có thể lag (tránh WAL bị rotate trước standby nhận được)
-- NGUY HIỂM: nếu standby offline lâu → WAL tích lũy → disk full → primary crash!

-- Tạo replication slot
SELECT pg_create_physical_replication_slot('standby1_slot');

-- Monitor slot lag (phải alert nếu lớn)
SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;

-- Drop slot khi standby bị remove
SELECT pg_drop_replication_slot('standby1_slot');
```

---

## Patroni — Automatic Failover

Patroni = HA template sử dụng etcd/Consul/ZooKeeper để leader election và automatic failover.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   ┌─────────┐     ┌─────────┐     ┌─────────┐          │
│   │ Patroni │     │ Patroni │     │ Patroni │          │
│   │ (primary)│    │ (standby)│    │ (standby)│         │
│   │  pg     │────►│  pg     │◄───►│  pg     │          │
│   └────┬────┘     └────┬────┘     └────┬────┘          │
│        │               │               │               │
│        └───────────────┴───────────────┘               │
│                         │                              │
│                   ┌─────▼─────┐                        │
│                   │   DCS     │ ← etcd / Consul / ZK   │
│                   │  (leader  │   (stores cluster       │
│                   │  election)│    state + config)      │
│                   └───────────┘                        │
│                                                         │
│   HAProxy / pgBouncer → route to primary only           │
└─────────────────────────────────────────────────────────┘
```

### Install & Configure

```yaml
# /etc/patroni/patroni.yml
scope: postgres-cluster          # cluster name
namespace: /service/             # DCS key prefix
name: pg-node-1                  # this node's name

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.0.1:8008

etcd3:
  hosts: etcd-1:2379,etcd-2:2379,etcd-3:2379

bootstrap:
  dcs:
    ttl: 30                      # leader lease TTL (seconds)
    loop_wait: 10                # how often Patroni runs main loop
    retry_timeout: 10
    maximum_lag_on_failover: 1048576   # max lag (1MB) để eligible for promotion
    postgresql:
      use_pg_rewind: true        # dùng pg_rewind để sync old primary
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: on
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: on        # cần cho pg_rewind

  initdb:
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums         # detect data corruption

  pg_hba:
    - local all all trust
    - host replication replicator 10.0.0.0/24 scram-sha-256
    - host all all 0.0.0.0/0 scram-sha-256

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.0.1:5432
  data_dir: /var/lib/postgresql/data
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: strongpass
    superuser:
      username: postgres
      password: adminpass
  parameters:
    shared_buffers: 256MB
    logging_collector: on
    log_destination: csvlog
    log_directory: log
    log_filename: postgresql-%Y-%m-%d.log

tags:
  nofailover: false    # set true để prevent node từ becoming primary
  noloadbalance: false # set true để exclude từ load balancer (hot standby reads)
```

### Patroni CLI

```bash
# Status
patronictl -c /etc/patroni/patroni.yml list
# + Cluster: postgres-cluster --------+----+-----------+
# | Member    | Host       | Role    | State   | TL | Lag in MB |
# +-----------+------------+---------+---------+----+-----------+
# | pg-node-1 | 10.0.0.1   | Leader  | running |  1 |           |
# | pg-node-2 | 10.0.0.2   | Replica | running |  1 |         0 |
# | pg-node-3 | 10.0.0.3   | Replica | running |  1 |         0 |

# Planned switchover (graceful, không mất data)
patronictl -c /etc/patroni/patroni.yml switchover postgres-cluster \
  --master pg-node-1 \
  --candidate pg-node-2

# Failover (khi primary down)
patronictl -c /etc/patroni/patroni.yml failover postgres-cluster \
  --master pg-node-1 \
  --candidate pg-node-2 \
  --force

# Pause auto-failover (khi làm maintenance)
patronictl -c /etc/patroni/patroni.yml pause postgres-cluster

# Edit DCS config (live, không cần restart)
patronictl -c /etc/patroni/patroni.yml edit-config postgres-cluster

# Reload config
patronictl -c /etc/patroni/patroni.yml reload postgres-cluster pg-node-1

# Reinitialize node (sau khi node diverge)
patronictl -c /etc/patroni/patroni.yml reinit postgres-cluster pg-node-2
```

### HAProxy cho Patroni

```ini
# haproxy.cfg
global
    maxconn 100

defaults
    mode tcp
    timeout connect 4s
    timeout client 30s
    timeout server 30s

# Primary — read/write
frontend pg-primary
    bind *:5000
    default_backend pg-primary-backend

backend pg-primary-backend
    option httpchk GET /master
    http-check expect status 200
    server pg-node-1 10.0.0.1:5432 maxconn 100 check port 8008
    server pg-node-2 10.0.0.2:5432 maxconn 100 check port 8008
    server pg-node-3 10.0.0.3:5432 maxconn 100 check port 8008

# Replica — read-only
frontend pg-replica
    bind *:5001
    default_backend pg-replica-backend

backend pg-replica-backend
    option httpchk GET /replica
    http-check expect status 200
    balance roundrobin
    server pg-node-1 10.0.0.1:5432 maxconn 100 check port 8008
    server pg-node-2 10.0.0.2:5432 maxconn 100 check port 8008
    server pg-node-3 10.0.0.3:5432 maxconn 100 check port 8008
```

Patroni expose REST endpoints:
- `GET /master` → 200 nếu primary, 503 nếu không
- `GET /replica` → 200 nếu healthy standby
- `GET /health` → 200 nếu running (bất kể role)

---

## PgBouncer — Connection Pooling

### Tại sao cần?

```
PostgreSQL: 1 connection = 1 process = ~5-10MB RAM + CPU
100 connections = 500MB RAM overhead
Nếu app có nhiều threads/instances: connection count bùng nổ

PgBouncer: lightweight proxy, pool connections
App → PgBouncer (nhiều connections) → PostgreSQL (ít connections)
```

### Pooling Modes

| Mode | How | Use case |
|---|---|---|
| **Session** | 1 PG conn per client session (hold đến disconnect) | Legacy apps với session state |
| **Transaction** | PG conn trả về pool sau mỗi transaction | Hầu hết apps — recommended |
| **Statement** | PG conn trả về sau mỗi statement | Apps không dùng transactions (rare) |

```ini
# pgbouncer.ini
[databases]
myapp = host=localhost port=5432 dbname=myapp

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

pool_mode = transaction           # recommended
max_client_conn = 1000            # max connections từ apps
default_pool_size = 20            # max PG connections per database/user pair
min_pool_size = 5                 # keep warm
reserve_pool_size = 5             # emergency pool
reserve_pool_timeout = 5s

# Timeout
server_idle_timeout = 600s        # close idle PG connections
client_idle_timeout = 0           # don't close idle client connections
query_timeout = 0                 # no global query timeout

# Stats
stats_period = 60
admin_users = pgbouncer_admin
stats_users = monitoring

log_connections = 1
log_disconnections = 1
```

```bash
# userlist.txt
"myapp_user" "password_hash"
"pgbouncer_admin" "admin_hash"

# Generate hash
echo -n "password" | md5sum | awk '{print "md5"$1}'
```

```bash
# PgBouncer admin console
psql -h localhost -p 6432 -U pgbouncer_admin pgbouncer

# Trong console
SHOW POOLS;         -- pool utilization
SHOW CLIENTS;       -- current client connections
SHOW SERVERS;       -- backend PostgreSQL connections
SHOW STATS;         -- throughput metrics
RELOAD;             -- reload config
PAUSE myapp;        -- pause connections (cho maintenance)
RESUME myapp;       -- resume
```

---

## Logical Replication

Logical replication cho phép replicate subset của data và có thể dùng cho zero-downtime major version upgrades.

```sql
-- PUBLISHER (source)
ALTER SYSTEM SET wal_level = logical;
-- restart PostgreSQL

-- Tạo publication
CREATE PUBLICATION myapp_pub FOR TABLE orders, products, customers;
-- hoặc tất cả tables
CREATE PUBLICATION myapp_pub FOR ALL TABLES;

-- SUBSCRIBER (destination)
CREATE SUBSCRIPTION myapp_sub
  CONNECTION 'host=primary-host port=5432 dbname=myapp user=replicator password=strongpass'
  PUBLICATION myapp_pub;

-- Monitor
SELECT * FROM pg_stat_subscription;
SELECT * FROM pg_publication;
SELECT * FROM pg_replication_slots WHERE slot_type = 'logical';
```

### Zero-Downtime Major Version Upgrade

```
Strategy:
  Old (PG 15) → Logical Replication → New (PG 16)
  1. Setup logical replication từ PG 15 → PG 16
  2. Wait for replication to catch up (lag ~ 0)
  3. Quick application switch: update connection string
  4. Verify, then decommission PG 15

Lưu ý:
  - Cần schemas và sequences migrate trước (pg_dump --schema-only)
  - Sequences không được replicate → update manually sau switch
  - DDL changes không được replicate
```

---

## Gotchas

- **Replication slots và disk**: Replication slot giữ WAL cho đến khi standby consume. Nếu standby offline lâu → disk fill → primary crash. Luôn monitor `pg_replication_slots.pg_wal_lsn_diff`. Set `max_slot_wal_keep_size` để limit WAL kept per slot.
- **synchronous replication và availability**: Với `synchronous_standby_names = 'standby1'`, nếu standby1 down → primary **block** trên mọi COMMIT. Cần `ANY 1 (standby1, standby2)` syntax cho quorum-based sync.
- **pg_rewind và timeline**: Sau failover, old primary có diverged WAL. `pg_rewind` rewind old primary về điểm phân kỳ, sau đó streaming replication tiếp tục. Không có `pg_rewind` → phải pg_basebackup lại (lâu hơn).
- **PgBouncer và prepared statements**: Transaction mode không tương thích với named prepared statements (PREPARE/EXECUTE). Apps dùng ORMs với prepared statements cần Session mode hoặc disable prepared statements.
- **hot_standby_feedback và bloat**: `hot_standby_feedback = on` tốt cho preventing query cancellation trên standby nhưng prevent vacuum trên primary → bloat. Monitor primary bloat khi có long-running queries trên standbys.
