---
tags: [database, terminology, acid, cap, mvcc, index, transaction, devops]
---

# 📖 Database Terminology — Thuật ngữ cốt lõi

> Tất cả thuật ngữ quan trọng cần nắm vững trước khi đi sâu vào từng loại DB. Đây là nền tảng.

---

## ACID

ACID là tập hợp 4 thuộc tính đảm bảo tính toàn vẹn của database transaction. Xem kiến trúc thực tế tại [[02-Architecture#WAL Architecture]].

### A — Atomicity (Tính nguyên tử)

**Định nghĩa:** Một transaction hoặc thực hiện TOÀN BỘ hoặc không thực hiện GÌ cả. Không có trạng thái trung gian.

**Ví dụ thực tế:** Chuyển tiền ngân hàng
```sql
BEGIN;
  UPDATE accounts SET balance = balance - 1000000 WHERE id = 'A';  -- Trừ tiền A
  UPDATE accounts SET balance = balance + 1000000 WHERE id = 'B';  -- Cộng tiền B
COMMIT;
```
Nếu câu lệnh thứ 2 fail (ví dụ B không tồn tại) → toàn bộ transaction rollback, tiền A không bị trừ.

**Cơ chế:** Dựa vào Undo Log (InnoDB) hoặc WAL (PostgreSQL) để rollback khi cần.

### C — Consistency (Tính nhất quán)

**Định nghĩa:** Database luôn chuyển từ trạng thái hợp lệ này sang trạng thái hợp lệ khác. Mọi constraint, rule đều được đảm bảo.

**Ví dụ:** Constraint `balance >= 0` phải được giữ. Nếu transaction vi phạm → bị reject.
```sql
ALTER TABLE accounts ADD CONSTRAINT chk_balance CHECK (balance >= 0);
-- Bây giờ nếu trừ tiền xuống âm → transaction fail, rollback
```

**Lưu ý:** Consistency là trách nhiệm của cả ứng dụng lẫn database (constraints, triggers, etc.)

### I — Isolation (Tính độc lập)

**Định nghĩa:** Các transaction đồng thời không ảnh hưởng lẫn nhau. Mỗi transaction thấy một "snapshot" nhất quán của dữ liệu.

**4 Isolation Levels (từ thấp đến cao):**
```
READ UNCOMMITTED → có thể đọc dirty data (uncommitted)
READ COMMITTED   → chỉ đọc committed data (PostgreSQL default)
REPEATABLE READ  → same query trả về same result trong tx (MySQL default)
SERIALIZABLE     → hoàn toàn tuần tự, như chạy lần lượt
```

**Các vấn đề isolation:**
- **Dirty Read:** Đọc data chưa commit của tx khác
- **Non-repeatable Read:** Same query trong cùng tx trả về khác nhau
- **Phantom Read:** Lần đọc sau thấy thêm row mới (do tx khác INSERT)
- **Lost Update:** Hai tx cùng update, một cái bị ghi đè

**Giải pháp:** MVCC (xem phần dưới) giải quyết phần lớn vấn đề isolation mà không cần lock.

### D — Durability (Tính bền vững)

**Định nghĩa:** Sau khi COMMIT, dữ liệu được bảo đảm tồn tại dù server crash, mất điện.

**Cơ chế:** WAL (Write-Ahead Log) — ghi log trước khi ghi vào disk thực sự. Xem [[02-Architecture#WAL Architecture]].

```
Transaction COMMIT
     ↓
Ghi vào WAL (fsync)
     ↓
Ack về client "SUCCESS"
     ↓
(Sau đó) Ghi vào data files (background)

Nếu crash sau COMMIT → WAL replay khi restart → data vẫn còn
```

---

## BASE

BASE là triết lý ngược với ACID, phù hợp với hệ thống phân tán scale lớn. Xem thêm [[03-Types#Wide-Column]].

- **BA — Basically Available:** Hệ thống luôn available, nhưng có thể trả về stale data
- **S — Soft state:** Trạng thái có thể thay đổi theo thời gian, dù không có input mới
- **E — Eventually consistent:** Dữ liệu sẽ nhất quán "cuối cùng" — không phải ngay lập tức

**Ví dụ thực tế:** Cassandra write
```
Client ghi vào node A
→ Node A ack ngay (Basically Available)
→ Background: A replicates sang B, C (Soft state — đang propagate)
→ Sau vài ms: B, C có data → Consistent (Eventually consistent)
```

**Khi nào chấp nhận BASE?**
- Social media feeds (stale vài giây OK)
- Shopping cart (temporary inconsistency OK)
- Analytics counters (approximate OK)
- **KHÔNG OK:** Banking, healthcare, inventory critical

---

## CAP Theorem

**Định lý:** Một hệ thống phân tán chỉ có thể đảm bảo TỐI ĐA 2 trong 3 thuộc tính sau khi xảy ra network partition.

```
              Consistency (C)
                   /\
                  /  \
                 /    \
                /  ???  \
               /  pick   \
              /    two    \
             ──────────────
    Availability (A)    Partition Tolerance (P)
```

- **C (Consistency):** Mọi node đều trả về data mới nhất
- **A (Availability):** Mọi request đều nhận được response (dù có thể stale)
- **P (Partition Tolerance):** Hệ thống vẫn hoạt động khi network giữa các node bị đứt

**Thực tế:** P là bắt buộc trong distributed system (network LUÔN có thể fail). Do đó, chỉ có 2 lựa chọn thực tế: **CP** hoặc **AP**.

| DB | Loại | Giải thích |
|----|------|------------|
| MySQL/PostgreSQL (single node) | CA | Không phân tán, không cần P |
| HBase | CP | Consistency > Availability khi partition |
| MongoDB (default) | CP | Primary election khi partition |
| Cassandra | AP | Vẫn write/read khi partition, chấp nhận stale |
| DynamoDB | AP | Eventual consistency mặc định |
| CockroachDB | CP | Strong consistency distributed |
| Redis Cluster | AP | Có thể read stale từ replica |

### PACELC Extension

CAP chỉ nói về khi có partition. PACELC bổ sung: ngay cả khi KHÔNG có partition, vẫn có trade-off giữa Latency và Consistency.

```
PAC: If Partition → choose A or C
ELC: Else → choose L (Latency) or C (Consistency)

MySQL: PA/EL → During partition: Availability; Normal: Low Latency
DynamoDB: PA/EL → Available + Low Latency, eventual consistency
CockroachDB: PC/EC → Consistent always, higher latency
```

---

## Transaction, Commit, Rollback, Savepoint

```sql
-- Begin transaction
BEGIN; -- hoặc START TRANSACTION;

-- Savepoint để partial rollback
SAVEPOINT sp1;
INSERT INTO orders VALUES (1, 'item_a', 100);

SAVEPOINT sp2;
INSERT INTO orders VALUES (2, 'item_b', 200);

-- Rollback chỉ đến sp2, giữ lại sp1
ROLLBACK TO SAVEPOINT sp2;

-- Commit toàn bộ (chỉ còn order 1)
COMMIT;
```

**Distributed Transaction (2PC — Two-Phase Commit):**
```
Phase 1 — Prepare:
  Coordinator → "Can you commit?" → tất cả participants
  Participants → "Yes" / "No"

Phase 2 — Commit:
  Nếu tất cả Yes → Coordinator → "COMMIT"
  Nếu có bất kỳ No → Coordinator → "ROLLBACK"
```

---

## Index Types

Xem thêm chi tiết cho từng DB tại [[04-MySQL-MariaDB#Indexing]], [[05-PostgreSQL#Extensions]].

### B-Tree Index (mặc định)
- **Cấu trúc:** Balanced tree, O(log n) search
- **Phù hợp:** Equality (`=`), Range (`>`, `<`, `BETWEEN`), ORDER BY, LIKE 'abc%'
- **Không phù hợp:** LIKE '%abc', full-text search
- **Tất cả RDBMS đều có:** MySQL, PostgreSQL, SQLite

```
         [50]
        /    \
    [25]      [75]
   /    \    /    \
 [10] [30] [60] [90]

Query: WHERE id = 60
→ Root [50] → Right → [75] → Left → [60] ✓
```

### Hash Index
- **Cấu trúc:** Hash table, O(1) lookup
- **Phù hợp:** Chỉ equality (`=`), không hỗ trợ range
- **Dùng khi:** Lookup by exact key, very high cardinality
- **MySQL MEMORY engine, PostgreSQL**

### GIN (Generalized Inverted Index)
- **Phù hợp:** Full-text search, JSONB, arrays, tsvector
- **PostgreSQL đặc biệt mạnh**
```sql
-- Full-text search với GIN
CREATE INDEX idx_fts ON articles USING GIN(to_tsvector('english', content));
SELECT * FROM articles WHERE to_tsvector('english', content) @@ to_tsquery('database');

-- JSONB với GIN
CREATE INDEX idx_json ON data USING GIN(payload);
SELECT * FROM data WHERE payload @> '{"type": "error"}';
```

### GiST (Generalized Search Tree)
- **Phù hợp:** Geometric data, full-text search, range types
- **PostgreSQL + PostGIS**
```sql
-- Geo query với GiST
CREATE INDEX idx_geo ON locations USING GIST(coordinates);
SELECT * FROM locations WHERE ST_DWithin(coordinates, ST_Point(106.7, 10.8), 1000);
```

### BRIN (Block Range Index)
- **Cấu trúc:** Lưu min/max của từng block range, rất nhỏ
- **Phù hợp:** Data có correlation với physical storage order (timestamp, sequential ID)
- **Dùng cho:** Time-series, append-only data lớn
```sql
-- BRIN cho time-series
CREATE INDEX idx_ts ON events USING BRIN(created_at);
-- Index size: 1/1000 so với B-Tree nhưng vẫn hiệu quả với sequential scan
```

### Bitmap Index
- **Oracle, PostgreSQL (internally)**
- **Phù hợp:** Low cardinality columns (gender, status, boolean)
- **Kết hợp:** Bitmap AND/OR để combine multiple conditions hiệu quả

---

## Query Plan & EXPLAIN

```sql
-- PostgreSQL
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- Output example:
Index Scan using idx_email on users  (cost=0.43..8.45 rows=1 width=100)
                                      (actual time=0.05..0.06 rows=1 loops=1)
  Index Cond: (email = 'test@example.com')
Planning Time: 0.1 ms
Execution Time: 0.1 ms
```

**Các node types quan trọng:**
- `Seq Scan` → full table scan, BAD nếu bảng lớn
- `Index Scan` → dùng index, tốt
- `Index Only Scan` → chỉ đọc từ index, không cần heap, RẤT tốt
- `Bitmap Index Scan` → scan nhiều rows qua index
- `Nested Loop` → join bằng loop, tốt khi one side nhỏ
- `Hash Join` → hash one side, probe the other, tốt khi cả hai lớn
- `Merge Join` → merge sorted inputs, tốt khi cả hai đã sorted

**Query Optimizer:** Cost-based optimizer (CBO) — ước tính cost dựa trên statistics (pg_statistics). Phải `ANALYZE` table định kỳ để statistics accurate.

---

## Lock

### Shared Lock (S Lock / Read Lock)
- Multiple transactions có thể cùng hold shared lock
- Dùng cho READ operations
- Compatible với Shared Lock khác, KHÔNG compatible với Exclusive Lock

### Exclusive Lock (X Lock / Write Lock)
- Chỉ một transaction hold tại một thời điểm
- Dùng cho WRITE (INSERT/UPDATE/DELETE)
- KHÔNG compatible với bất kỳ lock nào khác

### Row-level vs Table-level Lock
```
Table Lock: LOCK TABLE users IN ACCESS EXCLUSIVE MODE;
→ Toàn bộ table bị lock, không ai khác có thể đọc/ghi
→ High contention, avoid in production

Row Lock: SELECT * FROM users WHERE id = 1 FOR UPDATE;
→ Chỉ lock row id=1
→ Other rows vẫn accessible
→ MySQL InnoDB, PostgreSQL default
```

### Deadlock
```
Transaction A:                 Transaction B:
LOCK row 1                     LOCK row 2
  (waiting for row 2) ←→        (waiting for row 1)
→ DEADLOCK! Database sẽ kill transaction có ít work nhất
```

**Phòng tránh deadlock:**
- Luôn lock resources theo thứ tự nhất quán
- Giữ transaction ngắn nhất có thể
- Sử dụng `SELECT ... FOR UPDATE SKIP LOCKED` cho queue patterns

---

## MVCC (Multi-Version Concurrency Control)

**Vấn đề:** Lock-based concurrency → readers block writers, writers block readers → performance thấp.

**Giải pháp MVCC:** Mỗi row có nhiều versions. Readers đọc version phù hợp với thời điểm transaction bắt đầu. Writers tạo version mới.

```
Time:  T1        T2        T3
       BEGIN     BEGIN
       READ id=1 (thấy v1)
                 UPDATE id=1 (tạo v2)
                 COMMIT
       READ id=1 (vẫn thấy v1 — snapshot tại T1)
       COMMIT
       
→ T1 không bị block bởi T2's write!
→ T2 không bị block bởi T1's read!
```

**PostgreSQL MVCC:** Mỗi row có `xmin` (tx tạo ra) và `xmax` (tx xóa/update).
**MySQL InnoDB:** Dùng Undo Log để reconstruct old versions.

**Dead tuple / Bloat:** Old versions cần cleanup. PostgreSQL dùng **autovacuum**. MySQL tự manage undo log.

Xem chi tiết tại [[05-PostgreSQL#Autovacuum]] và [[04-MySQL-MariaDB#Architecture]].

---

## Replication: Sync vs Async

### Asynchronous Replication
```
Client → PRIMARY → COMMIT (ack ngay) → Replica (lag có thể xảy ra)
```
- Latency thấp cho write
- Có thể mất data nếu primary crash trước khi replica nhận
- MySQL binlog replication default

### Synchronous Replication
```
Client → PRIMARY → ghi WAL → CHỜ Replica confirm → COMMIT → ack về client
```
- Durability cao hơn, zero data loss
- Write latency cao hơn (phụ thuộc vào network RTT đến replica)
- PostgreSQL synchronous_commit = on

### Semi-synchronous (MySQL)
- Primary chờ ÍT NHẤT 1 replica nhận data trước khi commit
- Balance giữa performance và durability

---

## Sharding vs Partitioning

### Partitioning (chia trong 1 node)
```
users table
├── users_2022 (partition by year)
├── users_2023
└── users_2024

→ Cùng DB server, khác physical files
→ Query routing tự động bởi DB engine
→ Giảm index size, cải thiện scan performance
```

**Horizontal Partitioning:** Chia rows (theo range, hash, list)
**Vertical Partitioning:** Chia columns (tách columns ít dùng ra bảng riêng)

### Sharding (chia nhiều node)
```
users table
├── Shard 1: user_id 1-1M     → DB Server 1
├── Shard 2: user_id 1M-2M    → DB Server 2
└── Shard 3: user_id 2M-3M    → DB Server 3

→ Khác DB server hoàn toàn
→ Cần application-level routing hoặc middleware (ProxySQL, Vitess)
→ Scale out storage + compute
```

**Sharding strategies:** Range, Hash, Directory, Consistent Hashing. Xem chi tiết [[02-Architecture#Sharding Strategies]].

---

## Connection Pool

**Vấn đề:** Tạo DB connection tốn kém (TCP handshake + auth + session setup). Nếu mỗi request tạo connection mới → performance thảm.

**Giải pháp:** Connection Pool — duy trì một pool connections sẵn sàng.

```
Application Servers          Connection Pool        Database
┌──────────┐                ┌─────────────┐        ┌───────┐
│ Request 1│──→ Borrow  →───│ Conn 1 (idle)│──────→│       │
│ Request 2│──→ Borrow  →───│ Conn 2 (busy)│──────→│  DB   │
│ Request 3│──→ Wait    ←───│ Conn 3 (busy)│──────→│       │
└──────────┘    (queue)     │ Conn 4 (idle)│        └───────┘
                            └─────────────┘
                             Max: 10 conns
```

**Tools:**
- PostgreSQL: **PgBouncer** (transaction mode rất hiệu quả)
- MySQL: **ProxySQL**, HikariCP (Java)
- Generic: HikariCP, c3p0, DBCP

**Các params quan trọng:**
- `min_pool_size`: Số conn tối thiểu maintain
- `max_pool_size`: Số conn tối đa
- `connection_timeout`: Wait bao lâu trước khi fail
- `idle_timeout`: Đóng conn idle bao lâu

Xem thêm [[10-Operations#Connection Pooling]].

---

## WAL (Write-Ahead Log)

**Nguyên lý:** Trước khi ghi vào data files, LUÔN ghi vào log (sequential write) trước. Log là source of truth.

```
Write Request
     ↓
[1] Ghi vào WAL (append-only, sequential → FAST)
     ↓
[2] Ack về client "COMMITTED"
     ↓
[3] (Background) Apply changes vào data pages (random write)

Recovery sau crash:
→ Replay WAL từ last checkpoint
→ Re-apply uncommitted writes
→ Rollback incomplete transactions
```

**Tại sao sequential write nhanh hơn random write?**
- HDD: Seek time + rotational latency expensive
- SSD: Sequential còn nhanh hơn random một chút, và tránh write amplification

**WAL trong các DB:**
- PostgreSQL: `pg_wal/` directory, WAL segments 16MB
- MySQL InnoDB: Redo Log (`ib_logfile0`, `ib_logfile1`)
- SQLite: WAL mode file

---

## LSM Tree vs B-Tree

### B-Tree
- **Read:** O(log n), rất tốt
- **Write:** Random I/O, update in-place
- **Use case:** OLTP, balanced read/write
- **DB dùng:** MySQL InnoDB, PostgreSQL

### LSM Tree (Log-Structured Merge Tree)
```
Write → MemTable (RAM) → flush → SSTable (disk, immutable)
                                  ↓
                         Background Compaction
                         (merge SSTables, remove tombstones)
```
- **Write:** Sequential I/O, RẤT nhanh (append only)
- **Read:** Cần check nhiều SSTables (Bloom filter giúp skip)
- **Space:** Cần compaction để reclaim space
- **Use case:** Write-heavy workloads
- **DB dùng:** RocksDB, Cassandra, HBase, LevelDB

---

## N+1 Problem

**Vấn đề kinh điển trong ORM:**
```python
# N+1 Problem
posts = Post.query.all()          # 1 query
for post in posts:                 # N queries (1 per post)
    print(post.author.name)       # SELECT * FROM users WHERE id = ?

# Nếu có 100 posts → 101 queries!
```

**Fix: Eager Loading / JOIN**
```python
# SQLAlchemy
posts = Post.query.options(joinedload(Post.author)).all()
# → 1 query với JOIN

# SQL thuần
SELECT p.*, u.name
FROM posts p
JOIN users u ON p.author_id = u.id;
# → 1 query, done
```

**Fix: Batch Loading**
```python
# Load all authors in one query
author_ids = [post.author_id for post in posts]
authors = {u.id: u for u in User.query.filter(User.id.in_(author_ids)).all()}
# → 2 queries total
```

---

## Related Notes

- [[00-Database-MOC]] — Hub trung tâm
- [[02-Architecture]] — Kiến trúc chi tiết các khái niệm trên
- [[03-Types]] — Áp dụng các concepts vào từng loại DB
- [[04-MySQL-MariaDB]] — MySQL implementation
- [[05-PostgreSQL]] — PostgreSQL implementation
- [[10-Operations]] — Áp dụng trong vận hành thực tế
