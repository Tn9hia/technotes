---
tags: [database, architecture, storage-engine, replication, sharding, devops]
---

# 🏗️ Database Architecture — Kiến trúc nội tại

> Hiểu bên trong DB hoạt động thế nào để tune performance và debug sự cố. Đọc [[01-Terminology]] trước nếu chưa nắm terminology.

---

## Storage Engine Layer

Storage engine là trái tim của DB — nơi thực sự đọc/ghi dữ liệu. Các RDBMS thường có thể swap storage engine.

### InnoDB (MySQL default)
```
┌──────────────────────────────────────┐
│              InnoDB                   │
│                                       │
│  ┌─────────────┐  ┌───────────────┐  │
│  │ Buffer Pool │  │  Redo Log     │  │
│  │  (RAM cache)│  │  (WAL)        │  │
│  └─────────────┘  └───────────────┘  │
│  ┌─────────────┐  ┌───────────────┐  │
│  │  Undo Log   │  │  Data Files   │  │
│  │  (MVCC)     │  │  (.ibd)       │  │
│  └─────────────┘  └───────────────┘  │
│  ┌─────────────────────────────────┐  │
│  │     Change Buffer               │  │
│  │  (buffer secondary index writes)│  │
│  └─────────────────────────────────┘  │
└──────────────────────────────────────┘
```

**Key characteristics:**
- Row-level locking (B-Tree clustered index)
- ACID compliant với MVCC
- Foreign key support
- Crash recovery via Redo Log
- Buffer Pool là nơi cache data pages (đặt = 70-80% RAM)

Xem tuning chi tiết tại [[04-MySQL-MariaDB#Tuning]].

### RocksDB
```
Write → MemTable (RAM, sorted) 
      → WAL (crash safety)
      → Level 0 SSTable (disk, sorted)
      → Level 1..N (compaction, larger + sorted)
```

**Key characteristics:**
- LSM Tree based → write-optimized
- Dùng trong MyRocks (MySQL), TiKV (TiDB), MongoRocks, Cassandra (option)
- Excellent cho write-heavy workloads
- Compaction có thể gây I/O spikes

### WiredTiger (MongoDB default)
```
┌──────────────────────────────────────┐
│           WiredTiger                  │
│                                       │
│  ┌─────────────────────────────────┐  │
│  │       Cache (50% RAM default)   │  │
│  │  Pages: dirty / clean / evicted │  │
│  └─────────────────────────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  │
│  │  Journal     │  │  Data Files  │  │
│  │  (WAL/oplog) │  │  (.wt)       │  │
│  └──────────────┘  └──────────────┘  │
└──────────────────────────────────────┘
```

**Key characteristics:**
- Document-level concurrency control (MVCC)
- Snappy/zstd compression
- B-Tree và LSM mode available
- Journal = WAL cho crash safety

Xem [[06-MongoDB#Architecture]] cho chi tiết.

---

## Buffer Pool / Memory Architecture

Buffer Pool là "L3 cache" của database — giữ hot data trong RAM để tránh disk I/O.

```
┌─────────────────────────────────────────────────────────┐
│                    Memory Architecture                    │
│                                                           │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Buffer Pool (70-80% RAM)             │   │
│  │                                                    │   │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │   │
│  │  │ Hot Pages│  │Warm Pages│  │   Cold Pages   │  │   │
│  │  │(LRU head)│  │          │  │  (LRU tail)    │  │   │
│  │  │          │  │          │  │  → evict first │  │   │
│  │  └──────────┘  └──────────┘  └────────────────┘  │   │
│  │                                                    │   │
│  │  Free list → New pages from disk go here first    │   │
│  └──────────────────────────────────────────────────┘   │
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Sort Buffer│  │  Join Buffer │  │  Log Buffer  │  │
│  │  (per query) │  │  (per query) │  │  (WAL buf)   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**InnoDB Buffer Pool cụ thể:**
- Mặc định: 128MB (quá nhỏ cho production)
- Production: 70-80% RAM
- Multiple buffer pool instances (innodb_buffer_pool_instances = 8 cho server lớn)
- LRU algorithm với "midpoint insertion" để tránh large scans evict hot data

**Cache Hit Rate — metric quan trọng:**
```sql
-- MySQL
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';
-- Innodb_buffer_pool_reads = physical reads
-- Innodb_buffer_pool_read_requests = total reads
-- Hit rate = 1 - (reads/read_requests) → cần > 99%

-- PostgreSQL
SELECT
  sum(heap_blks_hit)::float / (sum(heap_blks_hit) + sum(heap_blks_read)) as cache_hit_ratio
FROM pg_statio_user_tables;
-- Cần > 0.99
```

---

## Write Path vs Read Path

### Write Path (Simplified)
```
Client WRITE Request
        │
        ▼
┌──────────────────┐
│   SQL Parser     │  Parse SQL → AST
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Query Optimizer │  Generate execution plan
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Lock Manager    │  Acquire row/page locks
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   Buffer Pool    │  Modify page in memory (dirty page)
│   (dirty page)   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   WAL/Redo Log   │  Append log record (SYNC to disk)
│   (fsync)        │
└────────┬─────────┘
         │
         ▼
   COMMIT → Ack to Client ✓
         │
         ▼ (background)
┌──────────────────┐
│  Checkpoint      │  Flush dirty pages to data files
│  (async)         │
└──────────────────┘
```

### Read Path
```
Client READ Request
        │
        ▼
┌──────────────────┐
│   SQL Parser     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Query Optimizer │  Choose best execution plan (index scan vs seq scan)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   Buffer Pool    │  Check cache
│                  │
│   Cache HIT? ────┼──→ Return data (fast path, no disk I/O)
│   Cache MISS?    │
└────────┬─────────┘
         │ (cache miss)
         ▼
┌──────────────────┐
│   Storage Engine │  Read from disk (slow)
│   (disk I/O)     │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   Buffer Pool    │  Cache the page
└────────┬─────────┘
         │
         ▼
   Return data to Client
```

---

## WAL Architecture

WAL (Write-Ahead Log) = sequential append log → crash recovery + replication.

```
WAL Architecture (PostgreSQL pg_wal)

┌─────────────────────────────────────────────────────────┐
│                      WAL Stream                          │
│                                                           │
│  LSN: 0/01000000           LSN: 0/02000000               │
│  ├──────────────────────────┤                            │
│  │  WAL Segment (16MB)      │  WAL Segment (16MB)  ...  │
│  │  ┌──────┬──────┬──────┐  │                            │
│  │  │Record│Record│Record│  │                            │
│  │  │ T1   │ T2   │ T1   │  │                            │
│  │  └──────┴──────┴──────┘  │                            │
│  └──────────────────────────┘                            │
│                                                           │
│  Checkpoint LSN ───────────────────► Current LSN         │
│  (last synced to data files)         (latest write)      │
│                                                           │
│  WAL between checkpoint and current = recovery window    │
└─────────────────────────────────────────────────────────┘

Crash Recovery:
1. Find last checkpoint from pg_control
2. Replay WAL from that checkpoint
3. Redo all committed transactions
4. Rollback uncommitted transactions
5. Database consistent and ready
```

**WAL settings (PostgreSQL):**
```
wal_level = replica  # hoặc logical cho logical replication
max_wal_size = 1GB   # max WAL accumulation trước khi force checkpoint
min_wal_size = 80MB
wal_keep_size = 0    # keep WAL cho replicas (hoặc dùng slots)
```

---

## Replication Architecture

### Primary-Replica (Master-Slave)
```
                    ┌─────────────────────────┐
                    │         PRIMARY          │
                    │  ┌────────┐  ┌────────┐ │
Write ─────────────►│  │ WAL/  │  │ Data   │ │
                    │  │ Binlog │  │ Files  │ │
Read (primary) ─────│  └───┬────┘  └────────┘ │
                    └──────┼──────────────────┘
                           │ WAL stream / binlog
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Replica 1 │ │ Replica 2 │ │ Replica 3 │
        │ (async)  │ │ (sync)   │ │ (async)  │
        └──────────┘ └──────────┘ └──────────┘
             │
        Read scaling (route reads to replicas)
```

**Failover:** Khi primary fail, replica được promote lên primary. Tools:
- MySQL: Orchestrator, MHA
- PostgreSQL: Patroni + etcd/consul
- MongoDB: Automatic via replica set election

### Multi-Primary (Active-Active)
```
Application
     │
     ├──────────────────────┐
     ▼                      ▼
┌──────────┐          ┌──────────┐
│ Primary 1│◄────────►│ Primary 2│
│ (write)  │  sync    │ (write)  │
└──────────┘  repl    └──────────┘
```
- MySQL: Galera Cluster, MySQL Group Replication
- Conflict resolution cần thiết (last-write-wins hoặc application-level)
- Higher write latency do cross-node coordination

### Quorum-based (Raft/Paxos)
```
         ┌────────┐
         │ Leader │ ← writes go here
         └────┬───┘
              │ replicate
         ┌────┴────┐
    ┌────┴─┐    ┌──┴───┐
    │Follow│    │Follow│
    │  er 1│    │  er 2│
    └──────┘    └──────┘

Quorum = majority (2/3 nodes must ack write)
Leader election via Raft if leader fails
```
- **etcd, CockroachDB, TiDB, MongoDB replica sets** (với Raft variant)
- Strong consistency guarantee
- Survives minority node failures

---

## Sharding Strategies

Xem thêm [[01-Terminology#Sharding vs Partitioning]].

### Range Sharding
```
user_id 1-1M      → Shard 1
user_id 1M-2M     → Shard 2
user_id 2M+       → Shard 3
```
- **Pro:** Range queries efficient, easy hotspot detection
- **Con:** Hotspot nếu new data concentrated (e.g., sequential IDs → Shard N luôn nhận write mới)

### Hash Sharding
```
shard = hash(user_id) % num_shards

hash(1) % 3 = 1 → Shard 1
hash(2) % 3 = 2 → Shard 2
hash(3) % 3 = 0 → Shard 3
```
- **Pro:** Even distribution, no hotspots
- **Con:** Range queries cần scatter-gather (query tất cả shards)
- **Con:** Thêm shard → rehash toàn bộ data (giải quyết bằng Consistent Hashing)

### Directory/Lookup Sharding
```
ShardMap table:
user_id_range → shard
1-1000        → shard_1
1001-2000     → shard_2
...

Application → query ShardMap → route to correct shard
```
- **Pro:** Flexible, có thể rebalance dễ dàng
- **Con:** ShardMap là single point of failure, cần cache

### Consistent Hashing
```
Hash ring: 0 ─────────────────────────────── MAX
                │         │         │
             Node A     Node B     Node C

Key hashes to a point on ring → next clockwise node handles it

Add Node D between A and B:
→ Only keys between A and D need remapping (not all keys!)
```
- **Pro:** Rebalancing minimal (chỉ ~1/N keys cần di chuyển)
- **Con:** Phức tạp hơn implement
- **Dùng trong:** Cassandra, DynamoDB, Redis Cluster (dùng hash slots variant)

---

## Query Lifecycle

```
SQL Query: SELECT u.name, COUNT(o.id)
           FROM users u
           LEFT JOIN orders o ON u.id = o.user_id
           WHERE u.created_at > '2024-01-01'
           GROUP BY u.id
           ORDER BY COUNT(o.id) DESC
           LIMIT 10;

┌─────────────────────────────────────────────────────────┐
│ STEP 1: PARSE                                            │
│  Lexer: tokenize SQL string                              │
│  Parser: build Abstract Syntax Tree (AST)                │
│  Validate: syntax check, table/column existence          │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│ STEP 2: PLAN (Query Optimizer)                          │
│  - Get table statistics (row count, column distribution) │
│  - Generate multiple execution plans                     │
│  - Estimate cost for each plan                           │
│  - Choose lowest cost plan                               │
│                                                           │
│  Options considered:                                      │
│  - Index scan on created_at vs seq scan                  │
│  - Hash join vs nested loop join                         │
│  - Sort vs index-based ORDER BY                          │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│ STEP 3: EXECUTE                                          │
│  - Acquire necessary locks (S lock for reads)            │
│  - Retrieve pages from Buffer Pool (or disk)             │
│  - Apply filters, joins, aggregations                    │
│  - Sort/limit results                                    │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│ STEP 4: RETURN                                          │
│  - Format result set                                     │
│  - Release locks (for non-SERIALIZABLE)                  │
│  - Send to client via wire protocol                      │
└─────────────────────────────────────────────────────────┘
```

---

## Connection Handling

### Thread-per-Connection (MySQL, PostgreSQL traditional)
```
Client 1 ─→ Thread 1 (dedicated, ~1MB stack)
Client 2 ─→ Thread 2
Client 3 ─→ Thread 3
...
Client N ─→ Thread N

Problem: 10,000 clients = 10,000 threads = 10GB+ RAM + context switching hell
```

**PostgreSQL giải pháp:** PgBouncer in transaction mode — pool 100 server connections cho 10,000 clients.
**MySQL giải pháp:** ProxySQL, thread pool plugin.

### Async/Event-loop (Redis, Nginx approach)
```
Single Thread Event Loop

Event queue: [req1, req2, req3, ...]
     │
     ▼
┌─────────────────┐
│  Event Loop     │
│  while(true):   │
│    poll events  │
│    dispatch I/O │
│    run callbacks│
└─────────────────┘

Pro: No thread overhead, handles millions of connections
Con: Blocking operations stall the loop (must use async I/O)
```

Redis là ví dụ điển hình — single-threaded nhưng handle hàng trăm nghìn ops/sec vì không có I/O wait (in-memory).

Xem [[07-Redis#Architecture]] cho chi tiết.

---

## Storage: Row-oriented vs Column-oriented

```
Row Store (OLTP):
┌────┬──────────┬──────────┬──────────┐
│ id │   name   │  salary  │   dept   │
├────┼──────────┼──────────┼──────────┤
│  1 │  Alice   │  50000   │   Eng    │  ← stored together
│  2 │  Bob     │  60000   │   Sales  │  ← stored together
│  3 │  Carol   │  55000   │   Eng    │  ← stored together
└────┴──────────┴──────────┴──────────┘
→ Fast INSERT/UPDATE/DELETE (single row I/O)
→ Slow analytics (must read all columns even if you need 1)

Column Store (OLAP):
id:     [1, 2, 3, ...]
name:   [Alice, Bob, Carol, ...]
salary: [50000, 60000, 55000, ...]  ← stored together
dept:   [Eng, Sales, Eng, ...]

→ Fast analytics (SELECT AVG(salary) reads only salary column)
→ Excellent compression (same-type data compresses well)
→ Slow single-row lookups
→ Examples: ClickHouse, Redshift, Parquet files
```

---

## Related Notes

- [[00-Database-MOC]] — Hub trung tâm
- [[01-Terminology]] — Các thuật ngữ nền tảng
- [[03-Types]] — Áp dụng architecture vào từng loại
- [[04-MySQL-MariaDB]] — InnoDB chi tiết
- [[05-PostgreSQL]] — PostgreSQL WAL, Patroni
- [[06-MongoDB]] — WiredTiger, Replica Set
- [[07-Redis]] — Event loop, in-memory architecture
- [[10-Operations]] — Vận hành thực tế dựa trên kiến trúc này
