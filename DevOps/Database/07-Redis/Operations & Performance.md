---
title: Redis Operations & Performance
tags:
  - redis
  - operations
  - performance
  - monitoring
  - memory
date: 2026-04-30
---

# Redis Operations & Performance

## Memory Management

### maxmemory và Eviction Policies

```bash
# redis.conf
maxmemory 4gb                    # hard limit (0 = no limit — dangerous in production)
maxmemory-policy allkeys-lru     # eviction policy khi đạt maxmemory

# Eviction policies:
# noeviction       — return error khi full (default, không auto-evict)
# allkeys-lru      — evict LRU key bất kỳ
# volatile-lru     — evict LRU key có TTL set
# allkeys-lfu      — evict LFU (Least Frequently Used) key bất kỳ  [Redis 4.0+]
# volatile-lfu     — evict LFU key có TTL set
# allkeys-random   — evict random key
# volatile-random  — evict random key có TTL set
# volatile-ttl     — evict key với TTL nhỏ nhất (sắp expire nhất)
```

**Chọn policy theo use case:**

| Use case | Policy |
|----------|--------|
| Pure cache (tất cả keys có thể evict) | `allkeys-lru` hoặc `allkeys-lfu` |
| Cache + persistent data (chỉ evict cache) | `volatile-lru` — cache keys có TTL, persistent keys không có TTL |
| Session store (evict sắp hết hạn nhất) | `volatile-ttl` |
| Không muốn mất data | `noeviction` — app phải xử lý OOM error |

### Memory Analysis

```bash
# Tổng quan memory
redis-cli INFO memory
# used_memory: 1073741824           ← bytes Redis đang dùng
# used_memory_rss: 1258291200       ← RSS từ OS (luôn >= used_memory)
# mem_fragmentation_ratio: 1.17     ← RSS/used_memory. > 1.5 = high frag
# maxmemory: 4294967296
# maxmemory_human: 4.00G
# mem_allocator: jemalloc-5.3.0

# Memory per key
redis-cli MEMORY USAGE user:1           # bytes (bao gồm key + value + overhead)
redis-cli MEMORY USAGE user:1 SAMPLES 5  # average samples cho nested structures

# Object encoding (internal representation)
redis-cli OBJECT ENCODING user:1
# string: int | embstr | raw
# list:   listpack (small) | quicklist (large)
# hash:   listpack (small) | hashtable (large)
# set:    listpack | intset | hashtable
# zset:   listpack | skiplist

# Find big keys (blocks — chỉ dùng maintenance window)
redis-cli --bigkeys
redis-cli --bigkeys --sleep 0.01        # add delay giữa scans

# Memory doctor
redis-cli MEMORY DOCTOR
# → gợi ý về fragmentation, maxmemory config, etc.

# Active defrag (Redis 4.0+ — jemalloc only)
# redis.conf
activedefrag yes
active-defrag-ignore-bytes 100mb      # bắt đầu defrag khi frag waste > 100MB
active-defrag-enabled yes
```

### Memory Encoding Thresholds

```bash
# redis.conf — tune encoding thresholds
hash-max-listpack-entries 128     # hash dùng listpack nếu <= 128 fields
hash-max-listpack-value 64        # và mỗi value <= 64 bytes

list-max-listpack-size -2         # quicklist node max: -2 = 8kb per node
list-compress-depth 0             # compress middle nodes (0=off, 1=compress all except 2 ends)

set-max-intset-entries 512        # set dùng intset nếu all integers và <= 512 entries
set-max-listpack-entries 128
set-max-listpack-value 64

zset-max-listpack-entries 128
zset-max-listpack-value 64
```

---

## Slow Log & Profiling

```bash
# redis.conf
slowlog-log-slower-than 10000    # microseconds (10ms)
slowlog-max-len 128              # giữ 128 entries

# View slow log
redis-cli SLOWLOG GET 10         # 10 slowest commands
redis-cli SLOWLOG LEN            # total slow entries
redis-cli SLOWLOG RESET

# Output format:
# 1) 1) (integer) 14           ← unique ID
#    2) (integer) 1705316400   ← Unix timestamp
#    3) (integer) 15234        ← execution time (microseconds)
#    4) 1) "LRANGE"            ← command + args
#       2) "events:log"
#       3) "0"
#       4) "-1"
#    5) "127.0.0.1:52341"      ← client addr
#    6) ""                      ← client name

# MONITOR — capture all commands (performance impact!)
redis-cli MONITOR               # print all commands in real-time
# ONLY dùng short-term debugging, không bật trên prod sustained
```

---

## Key Inspection & Management

```bash
# Safe key scan (cursor-based, non-blocking)
redis-cli SCAN 0 MATCH "user:*" COUNT 100
# Returns: cursor + list of keys
# Repeat with returned cursor until cursor = 0

# Delete keys by pattern (careful!)
redis-cli SCAN 0 MATCH "session:*" COUNT 100 | xargs redis-cli DEL

# Lazy delete (non-blocking)
redis-cli UNLINK key1 key2       # async delete (không block event loop)

# TTL management
redis-cli TTL key                # seconds remaining (-1=no TTL, -2=not exists)
redis-cli PTTL key               # milliseconds
redis-cli PERSIST key            # remove TTL (make permanent)

# Key count
redis-cli DBSIZE                 # total keys in current DB
redis-cli INFO keyspace          # breakdown by DB

# Dump & restore key (move between instances)
redis-cli DUMP key               # serialized value
redis-cli RESTORE new:key 0 <serialized>   # 0 = no TTL

# Inspect key
redis-cli TYPE key               # string/list/hash/set/zset/stream
redis-cli OBJECT FREQ key        # LFU frequency counter
redis-cli OBJECT IDLETIME key    # seconds since last access
redis-cli DEBUG OBJECT key       # internal details (encoding, serialized length)
```

---

## Configuration Management

```bash
# View config
redis-cli CONFIG GET maxmemory
redis-cli CONFIG GET "*max*"     # wildcard

# Live update (không cần restart)
redis-cli CONFIG SET maxmemory 8gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru
redis-cli CONFIG SET slowlog-log-slower-than 5000

# Persist live changes to redis.conf
redis-cli CONFIG REWRITE         # yêu cầu redis.conf có sẵn và writable

# Reset stats
redis-cli CONFIG RESETSTAT       # reset INFO statistics counters

# Flush
redis-cli FLUSHDB                # flush current DB (blocking)
redis-cli FLUSHDB ASYNC          # async (non-blocking)
redis-cli FLUSHALL ASYNC         # flush all DBs
```

---

## Monitoring với Prometheus

### redis_exporter

```yaml
# docker-compose.yml
redis-exporter:
  image: oliver006/redis_exporter:latest
  ports:
    - "9121:9121"
  environment:
    - REDIS_ADDR=redis://redis:6379
    - REDIS_PASSWORD=${REDIS_PASSWORD}
  command:
    - --web.listen-address=:9121
    - --redis.addr=redis://redis:6379
```

```yaml
# Kubernetes sidecar (trong Redis Pod)
- name: redis-exporter
  image: oliver006/redis_exporter:v1.62.0
  ports:
    - name: metrics
      containerPort: 9121
  env:
    - name: REDIS_ADDR
      value: "redis://localhost:6379"
    - name: REDIS_PASSWORD
      valueFrom:
        secretKeyRef:
          name: redis-secret
          key: password
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 100m
      memory: 128Mi
```

```yaml
# ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: redis
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: redis
  endpoints:
    - port: metrics
      interval: 30s
```

### Key Metrics & Alert Rules

```yaml
# PrometheusRule
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: redis-alerts
  namespace: monitoring
spec:
  groups:
    - name: redis
      rules:
        # Memory pressure
        - alert: RedisMemoryHigh
          expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Redis memory usage > 85%"

        # Evictions happening (data loss for noeviction sets, perf impact for LRU)
        - alert: RedisEvictionsHigh
          expr: rate(redis_evicted_keys_total[5m]) > 10
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "Redis evicting >10 keys/sec"

        # Connection exhaustion
        - alert: RedisTooManyConnections
          expr: redis_connected_clients / redis_config_maxclients > 0.9
          for: 5m
          labels:
            severity: critical

        # Replication lag
        - alert: RedisReplicationLag
          expr: redis_connected_slaves < 1
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Redis master has no connected replicas"

        - alert: RedisReplicationOffset
          expr: redis_master_repl_offset - redis_slave_repl_offset > 1000000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Replication lag > 1MB"

        # Keyspace miss rate (cache inefficiency)
        - alert: RedisCacheMissRateHigh
          expr: |
            rate(redis_keyspace_misses_total[5m]) /
            (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])) > 0.5
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Redis cache miss rate > 50%"

        # Instance down
        - alert: RedisDown
          expr: redis_up == 0
          for: 1m
          labels:
            severity: critical
```

### Key INFO sections

```bash
redis-cli INFO server        # version, uptime, config file
redis-cli INFO clients       # connected_clients, blocked_clients
redis-cli INFO memory        # memory usage, fragmentation
redis-cli INFO stats         # ops/sec, hits/misses, evictions
redis-cli INFO replication   # role, replicas, offset
redis-cli INFO cpu           # CPU usage
redis-cli INFO keyspace      # keys per DB, expiring keys, avg TTL
redis-cli INFO all           # everything
redis-cli INFO everything    # Redis 7+ (includes modules)

# Quick health check
redis-cli PING               # PONG
redis-cli LATENCY LATEST     # event latency
redis-cli LATENCY HISTORY command    # latency over time
redis-cli LATENCY RESET
```

---

## Performance Tuning

### OS-level (Linux)

```bash
# Disable Transparent Huge Pages (causes latency spikes with fork)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# vm.overcommit_memory — cho phép fork khi memory tight
sysctl vm.overcommit_memory=1
# Thêm vào /etc/sysctl.conf:
# vm.overcommit_memory = 1

# TCP backlog
sysctl net.core.somaxconn=65535
sysctl net.ipv4.tcp_max_syn_backlog=65535
# redis.conf: tcp-backlog 65535

# File descriptors
ulimit -n 65536
# /etc/security/limits.conf:
# redis soft nofile 65536
# redis hard nofile 65536
```

### Redis Config Tuning

```bash
# redis.conf — production settings
tcp-backlog 65535
timeout 300              # disconnect idle clients after 300s (0=disabled)
tcp-keepalive 60         # keepalive interval

databases 1              # chỉ dùng DB 0 trong production (cluster không hỗ trợ multi-DB)

# Connection limit
maxclients 10000         # default 10000

# Lazy free — async deletion
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
lazyfree-lazy-user-del yes       # UNLINK default behavior

# Threaded I/O (Redis 6+)
io-threads 4             # số CPU cores - 1 (không enable nếu CPU < 4)
io-threads-do-reads yes  # enable threaded reads
```

---

## Backup & Recovery

```bash
# Manual RDB snapshot
redis-cli BGSAVE
redis-cli LASTSAVE    # verify timestamp

# Copy RDB file
cp /var/lib/redis/dump.rdb /backup/dump-$(date +%Y%m%d).rdb

# Restore: stop Redis, replace dump.rdb, start Redis
# Redis sẽ load dump.rdb on startup

# AOF repair (nếu AOF bị corrupt)
redis-check-aof --fix appendonly.aof

# RDB check
redis-check-rdb dump.rdb

# Migrate data between instances
redis-cli --pipe-mode             # bulk import
redis-cli MIGRATE host port key 0 1000    # move key to another instance
# hoặc dùng redis-dump / redis-copy tools
```

---

## Gotchas

- **`CONFIG REWRITE` và comments**: Rewrite ghi lại redis.conf theo format Redis. Comments trong file gốc có thể bị xóa. Backup redis.conf trước khi rewrite lần đầu.
- **Fragmentation ratio > 1.5**: jemalloc giữ lại free pages từ deleted keys. `activedefrag yes` giải quyết dần, nhưng tốn CPU. Restart Redis (sau khi migrate traffic) là cách nhanh nhất.
- **MONITOR và throughput**: `MONITOR` gửi toàn bộ command stream tới client. Trên instance 100k ops/sec, MONITOR có thể double network traffic. Dùng `DEBUG SLEEP 0` để estimate current load trước.
- **maxmemory 0**: Mặc định không có limit — Redis dùng hết RAM của server → OS OOM killer kill Redis hoặc process khác. Luôn set maxmemory trong production.
- **Multiple databases (SELECT)**: Redis hỗ trợ 16 DBs (0-15) nhưng đây là anti-pattern trong microservices. DB separation không cung cấp isolation thực sự (FLUSHALL xóa tất cả, single-threaded cho tất cả DBs). Dùng separate Redis instances hoặc key namespace thay vì multiple DBs.
- **Blocking commands trong production**: `BLPOP/BRPOP` với timeout=0 block connection vô thời hạn. Connection pool cạn nếu có nhiều idle blocking connections. Luôn set reasonable timeout.
