---
tags: [database, redis, cache, nosql, devops, in-memory, index]
---

# Redis

> Redis (Remote Dictionary Server) — In-memory data structure store, dùng làm cache, message broker, session store, leaderboard, và distributed lock. Single-threaded command processing với sub-millisecond latency.

## Contents

| File | Nội dung |
|------|----------|
| [[DevOps/Database/07-Redis/Overview\|Overview]] | What/Why/When/Where/How, key config, security hardening, ops runbook, gotchas |
| [[Architecture & Data Structures]] | Single-threaded model, data types (String/List/Hash/Set/ZSet/Stream/HLL/Geo), persistence (RDB/AOF/hybrid), pipelining, transactions, Lua scripting, pub/sub, key design patterns |
| [[DevOps/Database/07-Redis/High Availability\|High Availability]] | Replication, Redis Sentinel (auto-failover), Redis Cluster (hash slots, sharding), deployment mode comparison |
| [[Operations & Performance]] | Memory management, eviction policies, object encoding, SLOWLOG, SCAN, CONFIG, backup, Prometheus metrics, OS tuning |
| [[Redis on Kubernetes]] | Bitnami Helm (sentinel mode), StorageClass, NetworkPolicy, Redis Cluster Helm, backup CronJob, gotchas |

---

## Quick Reference

### Connection

```bash
export REDIS_HOST=localhost
export REDIS_PORT=6379
export REDIS_PASS=yourpassword

redis-cli -h $REDIS_HOST -p $REDIS_PORT -a $REDIS_PASS
redis-cli -u "redis://:$REDIS_PASS@$REDIS_HOST:$REDIS_PORT"

# Test
redis-cli PING          # PONG
redis-cli INFO server | grep redis_version
```

### Common Operations

```bash
# String
SET key value EX 3600 NX          # set if not exists, expire 1h
GET key
INCR counter

# Hash
HSET user:1 name nghia email a@b.com
HGETALL user:1

# List (queue)
LPUSH queue task1
RPOP queue
BLPOP queue 30          # blocking pop

# Set
SADD tags:post:1 redis database
SMEMBERS tags:post:1

# Sorted Set (leaderboard)
ZADD leaderboard 1500 "player:1"
ZREVRANGE leaderboard 0 9 WITHSCORES    # top 10

# Stream
XADD events '*' key value
XREADGROUP GROUP mygroup worker1 COUNT 10 BLOCK 5000 STREAMS events >
XACK events mygroup <id>

# TTL
TTL key            # seconds (-1=no TTL, -2=not exists)
PERSIST key        # remove TTL
EXPIRE key 3600    # set TTL

# Safe key scan
SCAN 0 MATCH "user:*" COUNT 100
```

### Health Check

```bash
# Instance info
redis-cli INFO replication    # role, replicas, offset
redis-cli INFO memory         # used_memory, fragmentation
redis-cli INFO stats          # ops/sec, hits, misses, evictions
redis-cli INFO keyspace       # keys per DB

# Sentinel status
redis-cli -p 26379 SENTINEL masters
redis-cli -p 26379 SENTINEL sentinels mymaster
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster

# Cluster status
redis-cli CLUSTER INFO
redis-cli CLUSTER NODES

# Latency
redis-cli LATENCY LATEST
redis-cli --latency -h $REDIS_HOST    # continuous latency test
```

### Slow Log

```bash
redis-cli SLOWLOG GET 10       # 10 slowest commands
redis-cli SLOWLOG LEN
redis-cli SLOWLOG RESET
```

### Memory Analysis

```bash
redis-cli MEMORY USAGE key               # bytes cho specific key
redis-cli --bigkeys                      # scan for big keys (blocking!)
redis-cli MEMORY DOCTOR                  # recommendations
redis-cli INFO memory | grep -E "used_memory_human|mem_fragmentation_ratio"
```

### Config

```bash
redis-cli CONFIG GET maxmemory
redis-cli CONFIG SET maxmemory-policy allkeys-lru
redis-cli CONFIG REWRITE    # persist to redis.conf
```

---

## Architecture

Redis xử lý command **single-threaded** qua event loop (I/O multiplexing epoll/kqueue) → không race condition, mọi command atomic by default. Từ Redis 6.0: I/O threads cho network read/write (command execution vẫn single-thread). Chi tiết đầy đủ (memory architecture, object encoding) tại [[Architecture & Data Structures]].

---

## Data Type Cheat Sheet

| Type | Use case | Key commands |
|------|----------|-------------|
| String | Counter, cache, rate limit, bitmap | SET/GET/INCR/SETEX/SETNX |
| List | Queue, stack, recent items | LPUSH/RPOP/BLPOP/LRANGE |
| Hash | Object, session, shopping cart | HSET/HGET/HGETALL/HINCRBY |
| Set | Tags, unique items, intersection | SADD/SMEMBERS/SINTER/SUNION |
| Sorted Set | Leaderboard, priority queue, rate limit | ZADD/ZREVRANGE/ZRANGEBYSCORE |
| Stream | Event log, message queue | XADD/XREADGROUP/XACK |
| HyperLogLog | Unique count (approx) | PFADD/PFCOUNT |
| Bitmap | Feature flags, analytics | SETBIT/BITCOUNT |
| Geo | Location search | GEOADD/GEOSEARCH |

---

## Common Patterns

### Cache-Aside (Lazy Loading)

```python
def get_user(user_id):
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)

    user = db.query(f"SELECT * FROM users WHERE id = {user_id}")
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user
```

### Distributed Lock

```bash
# Acquire lock (SET NX EX = atomic)
SET lock:resource:1 <unique-token> NX EX 30

# Release lock (Lua — check + delete atomically)
EVAL "
  if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
  end
  return 0
" 1 lock:resource:1 <unique-token>
```

### Rate Limiting (Sliding Window)

```bash
# Sorted set: score = timestamp, member = request UUID
ZADD ratelimit:user:1 <now> <uuid>
ZREMRANGEBYSCORE ratelimit:user:1 -inf <now-window>
ZCARD ratelimit:user:1     # count in window → compare with limit
EXPIRE ratelimit:user:1 <window>
```

### Session Store

```bash
HSET session:<token> user_id 1 username nghia created_at 1705316400
EXPIRE session:<token> 86400    # 24h
HGETALL session:<token>
DEL session:<token>    # logout
```

---

## Related Notes

- [[00-Database-MOC]] — Tổng quan toàn bộ
- [[01-Terminology]] — Thuật ngữ: CAP theorem, replication, sharding
- [[03-Types]] — So sánh các loại DB
- [[05-PostgreSQL]] — pgvector nếu cần vector search thay Redis
- [[09-VectorDB]] — Vector database cho AI workloads
- [[10-Operations]] — Monitoring, backup, deployment
- Redis Sentinel trong Kubernetes: [[Redis on Kubernetes]]
- Redis secrets via Vault/ESO: [[Secrets Management]]
- Redis ServiceMonitor cho Prometheus: [[Monitoring]]
