---
title: Redis Architecture & Data Structures
tags:
  - redis
  - architecture
  - data-structures
  - persistence
date: 2026-04-30
---

# Redis Architecture & Data Structures

## Architecture Overview

```
Client 1 ──┐
Client 2 ──┤──► Event Loop (single-threaded) ──► In-Memory Store
Client 3 ──┘         │
                      ├─ I/O Threads (6.0+ — network read/write, không process commands)
                      ├─ BIO Threads (background: fsync, lazy free, close fd)
                      └─ Background Workers: BGSAVE, BGREWRITEAOF
```

**Single-threaded command processing:**
- Tất cả commands được xử lý tuần tự → không cần lock, không có race condition
- Memory allocator: jemalloc (reduces fragmentation)
- Latency thường < 1ms với mạng nội bộ

---

## Data Types

### String

```bash
# Basic get/set với TTL
SET user:1:name "Nghia"
SET counter 100 EX 3600 NX          # set only if NOT EXISTS, expire 1h
GET user:1:name
GETEX user:1:name EX 7200           # get + reset TTL (Redis 6.2+)
GETDEL user:1:name                  # get + delete

# Atomic counter
INCR page:views                     # atomic increment
INCRBY page:views 10
INCRBYFLOAT price 1.5

# Bulk
MSET k1 v1 k2 v2 k3 v3
MGET k1 k2 k3

# Bit operations — compact boolean flags
SETBIT user:1:features 0 1          # feature flag 0 = enabled
GETBIT user:1:features 0
BITCOUNT user:active:2024-01-15     # count active users (1 bit per user ID)

# Pattern: rate limiting
SET ratelimit:user:1 0 EX 60
INCR ratelimit:user:1               # nếu > threshold → block
```

### List — Queue / Stack

```bash
# Queue (FIFO)
LPUSH jobs:email task1 task2        # push left
RPOP  jobs:email                    # pop right (FIFO)

# Stack (LIFO)
LPUSH stack item1
LPOP  stack

# Blocking pop — worker pattern
BLPOP jobs:email jobs:sms 30        # block tối đa 30s, check từng queue

# Inspect
LRANGE jobs:email 0 -1              # tất cả elements
LLEN   jobs:email
LINDEX jobs:email 0                 # element tại index

# Trim (giữ window cuối cùng)
LTRIM events:log 0 999              # giữ 1000 items mới nhất

# Atomic pop + push — reliable queue
RPOPLPUSH jobs:email jobs:processing
LMOVE jobs:email jobs:processing LEFT RIGHT   # Redis 6.2+
```

### Hash — Object / Session

```bash
# User session
HSET session:abc123 user_id 1 username nghia role admin
HGET session:abc123 username
HMGET session:abc123 user_id role
HGETALL session:abc123
HDEL session:abc123 role

# Bulk update
HSET user:1 name "Nghia" email "nghia@co" age 30
HINCRBY user:1 age 1

# Check existence
HEXISTS user:1 email
HKEYS user:1
HVALS user:1
HLEN user:1

# Pattern: shopping cart
HSET cart:user:1 product:42 3       # product_id → quantity
HINCRBY cart:user:1 product:42 1    # add 1 more
```

### Set — Unique Collection

```bash
# Tags
SADD article:1:tags redis database nosql
SMEMBERS article:1:tags
SISMEMBER article:1:tags redis      # 1 hoặc 0
SCARD article:1:tags                # count

# Set operations
SINTER tags:redis tags:database     # intersection
SUNION tags:redis tags:nosql        # union
SDIFF  tags:redis tags:database     # difference

# Store result
SINTERSTORE common:tags tags:redis tags:database

# Pattern: unique visitors per day
SADD visitors:2024-01-15 user:1 user:2 user:3
SCARD visitors:2024-01-15           # unique count

# Random sample (lottery, recommendation)
SRANDMEMBER tags:redis 3            # 3 random, no remove
SPOP tags:redis                     # 1 random + remove
```

### Sorted Set — Ranked / Time-ordered

```bash
# Leaderboard
ZADD leaderboard 1500 "player:alice"
ZADD leaderboard 2300 "player:bob"
ZADD leaderboard NX 1800 "player:charlie"   # add only if NOT EXISTS

ZRANK leaderboard "player:bob"              # rank (0-indexed, ascending)
ZREVRANK leaderboard "player:bob"           # rank (descending = highest score first)
ZSCORE leaderboard "player:bob"
ZINCRBY leaderboard 100 "player:alice"

# Range queries
ZRANGE leaderboard 0 -1 WITHSCORES          # all, ascending
ZREVRANGE leaderboard 0 9 WITHSCORES        # top 10

# By score (Redis 6.2+: ZRANGE with BYSCORE REV LIMIT)
ZRANGEBYSCORE leaderboard 1000 2000 WITHSCORES LIMIT 0 10
ZRANGEBYSCORE leaderboard -inf +inf         # all

# Remove
ZREM leaderboard "player:alice"
ZREMRANGEBYSCORE leaderboard -inf 1000      # remove low scorers

# Pattern: rate limiting với sliding window
ZADD ratelimit:user:1 1705316400 "req:uuid1"
ZREMRANGEBYSCORE ratelimit:user:1 -inf (now-60s)   # remove old
ZCARD ratelimit:user:1             # count in last 60s

# Pattern: delayed job queue (score = timestamp to execute)
ZADD delayed:jobs 1705317000 "job:send-email:42"
ZRANGEBYSCORE delayed:jobs -inf (current_timestamp)  # jobs ready to run
```

### Stream — Event Log / Message Queue

```bash
# Producer
XADD events:orders '*' order_id 42 user_id 1 amount 150.00
# '*' = auto-generate ID (timestamp-sequence)
# Returns: "1705316400000-0"

XADD events:orders MAXLEN 10000 '*' ...    # cap stream at 10000 entries

# Consumer (simple)
XREAD COUNT 10 STREAMS events:orders 0     # read từ beginning
XREAD COUNT 10 BLOCK 5000 STREAMS events:orders $  # block, chỉ new msgs

# Consumer Group — distributed processing
XGROUP CREATE events:orders order-service $ MKSTREAM

# Worker reads
XREADGROUP GROUP order-service worker-1 COUNT 10 BLOCK 5000 STREAMS events:orders >

# Acknowledge processed
XACK events:orders order-service 1705316400000-0

# Inspect
XLEN events:orders
XRANGE events:orders - +                   # all messages
XRANGE events:orders 1705316400000-0 +     # from specific ID
XINFO GROUPS events:orders
XPENDING events:orders order-service - + 10   # unacked messages
```

### Geospatial

```bash
GEOADD locations 106.6602 10.7769 "hcm"
GEOADD locations 105.8412 21.0245 "hanoi"

GEODIST locations hcm hanoi km          # distance in km
GEOPOS locations hcm                    # lat/lon

# Find locations within radius
GEOSEARCH locations FROMMEMBER hcm BYRADIUS 500 km ASC COUNT 10
```

### HyperLogLog — Cardinality Estimation

```bash
# Count unique items với ~0.81% error, cố định 12KB memory
PFADD visitors:2024-01-15 user:1 user:2 user:3
PFCOUNT visitors:2024-01-15             # ~unique count

# Merge multiple HLL
PFMERGE visitors:week visitors:2024-01-15 visitors:2024-01-16
```

---

## Persistence

### RDB (Snapshot)

```
Điểm mạnh: compact file, fast restart, low overhead
Điểm yếu: có thể mất data giữa 2 snapshots
```

```bash
# redis.conf
save 900 1          # save nếu >= 1 change trong 900s
save 300 10         # save nếu >= 10 changes trong 300s
save 60 10000       # save nếu >= 10000 changes trong 60s
# Tắt: save ""

dbfilename dump.rdb
dir /var/lib/redis

# Manual trigger
BGSAVE              # fork → child save, không block
LASTSAVE            # timestamp của snapshot cuối
```

**Fork behavior:** Redis fork process để save. Kernel sử dụng Copy-on-Write (CoW) — memory tăng tạm thời khi parent tiếp tục modify pages.

### AOF (Append-Only File)

```
Điểm mạnh: durability cao (mất tối đa 1s data)
Điểm yếu: file lớn hơn RDB, restart chậm hơn
```

```bash
# redis.conf
appendonly yes
appendfilename "appendonly.aof"

appendfsync everysec    # flush mỗi giây (recommended, balance durability/perf)
# appendfsync always   # flush mỗi write (durability cao nhất, chậm nhất)
# appendfsync no       # OS quyết định (fastest, least durable)

auto-aof-rewrite-percentage 100   # rewrite khi AOF tăng gấp đôi
auto-aof-rewrite-min-size 64mb    # minimum size để trigger rewrite

# Manual rewrite (compact AOF)
BGREWRITEAOF
```

### Hybrid Mode (Recommended cho production)

```bash
# redis.conf — Redis 4.0+
aof-use-rdb-preamble yes    # AOF bắt đầu bằng RDB snapshot, append-only phần còn lại
```

### Persistence Summary

| Mode | Data Loss | Restart Speed | File Size | Recommended |
|------|-----------|---------------|-----------|-------------|
| None | All | Fastest | 0 | Cache only |
| RDB only | Minutes | Fast | Small | Low-priority cache |
| AOF everysec | ~1s | Medium | Medium | Most workloads |
| AOF always | ~0 | Slow | Large | Financial/audit |
| RDB + AOF hybrid | ~1s | Fast | Small | **Production** |

---

## Pipelining & Transactions

### Pipelining

```bash
# Batch commands — 1 round-trip cho nhiều commands
redis-cli --pipe < commands.txt

# Python (redis-py)
# pipe = r.pipeline()
# pipe.set('k1', 'v1')
# pipe.incr('counter')
# pipe.execute()    # 1 RTT cho cả 3 commands
```

### Transactions (MULTI/EXEC)

```bash
MULTI               # start transaction
SET k1 v1
INCR counter
EXEC                # execute atomically

DISCARD             # cancel transaction

# Optimistic locking (watch + multi)
WATCH user:1:balance    # watch key
MULTI
DECRBY user:1:balance 100
INCRBY user:2:balance 100
EXEC                # EXEC returns nil nếu user:1:balance thay đổi → retry
```

### Lua Scripting (Atomic)

```bash
# Script chạy atomically, không bị interrupt
EVAL "
  local current = redis.call('GET', KEYS[1])
  if tonumber(current) >= tonumber(ARGV[1]) then
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return 1
  end
  return 0
" 1 stock:item:42 5    # KEYS[1]=stock:item:42, ARGV[1]=5

# Load script (avoid sending script every time)
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# → returns SHA1
EVALSHA abc123sha1 1 mykey
```

---

## Pub/Sub

```bash
# Subscriber
SUBSCRIBE channel:notifications
PSUBSCRIBE "events:*"          # pattern subscribe

# Publisher
PUBLISH channel:notifications "new message"

# Inspect
PUBSUB CHANNELS "events:*"     # active channels matching pattern
PUBSUB NUMSUB channel:notifications
```

> Pub/Sub là fire-and-forget — không có persistence, subscribers miss messages nếu offline. Dùng **Streams** cho reliable messaging.

---

## Key Design Patterns

```bash
# Naming convention: type:id:field
user:1:profile
user:1:sessions
product:42:inventory
order:99:items

# Avoid overly long keys (memory overhead)
# Avoid spaces/special chars trong key names

# TTL strategy
SET session:abc123 data EX 3600       # absolute expiry
EXPIRE key seconds                     # set/update TTL
PERSIST key                            # remove TTL
TTL key                                # remaining seconds (-1 = no expiry, -2 = not exists)
PTTL key                               # milliseconds
EXPIREAT key 1705317000                # Unix timestamp

# Scan keys safely (never KEYS * in production)
SCAN 0 MATCH "user:*" COUNT 100        # cursor-based, COUNT là hint
# Loop: next cursor = 0 means done
```

---

## Gotchas

- **`KEYS *` blocks event loop**: Trả về tất cả keys — O(N). Với 10M keys block ~100ms. Dùng `SCAN` với cursor trong production.
- **Fork + CoW memory spike**: `BGSAVE` fork parent process. Nếu Redis đang write heavy khi fork, CoW duplicate nhiều pages → memory spike gấp đôi. Monitor `used_memory` + `mem_allocator_frag_ratio`.
- **AOF rewrite timing**: `BGREWRITEAOF` tạo temp file. Nếu disk I/O cao → rewrite chậm → AOF file tiếp tục grow. Tune `auto-aof-rewrite-min-size` và disk throughput.
- **Stream consumer group và unacked messages**: Messages `XREADGROUP` nhận nhưng chưa `XACK` nằm trong PEL (Pending Entry List). PEL grow unbounded nếu workers crash. Implement re-delivery logic với `XAUTOCLAIM` (Redis 6.2+).
- **HyperLogLog và false merge**: `PFMERGE` cộng cardinalities. Nếu 2 HLL có overlap (same elements), merged count sẽ cao hơn thực tế vì HLL không track individual elements.
- **Sorted set với cùng score**: Elements có cùng score sẽ được sort theo lexicographic order. Design score carefully nếu order matters.
