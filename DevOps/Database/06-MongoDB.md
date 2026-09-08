---
tags: [database, mongodb, nosql, document, wiredtiger, replication, sharding, devops]
---

# 🍃 MongoDB — Deep Dive

> MongoDB là Document DB market leader. Schema flexibility + horizontal scale + rich query. Xem [[03-Types#Document Database]] cho context.

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     mongod Process                        │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              Connection Pool                        │ │
│  └─────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              Query Engine                           │ │
│  │  Parser → Optimizer (cost-based) → Executor         │ │
│  └─────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────┐ │
│  │           WiredTiger Storage Engine                 │ │
│  │                                                     │ │
│  │  ┌────────────────────────────────────────────┐    │ │
│  │  │  WiredTiger Cache (50% RAM default)        │    │ │
│  │  │  - B-Tree pages (document data)            │    │ │
│  │  │  - Index pages                             │    │ │
│  │  └────────────────────────────────────────────┘    │ │
│  │  ┌────────────────┐  ┌────────────────────────┐   │ │
│  │  │   Journal      │  │   Data Files           │   │ │
│  │  │  (WAL, 100MB  │  │   (collection.wt)       │   │ │
│  │  │   segments)    │  │   (index.wt)            │   │ │
│  │  └────────────────┘  └────────────────────────┘   │ │
│  └─────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

### WiredTiger
- **Document-level concurrency control** (không phải collection-level lock)
- **MVCC** (snapshot isolation)
- **Compression:** Snappy (default), zlib, zstd
- **Cache:** 50% RAM mặc định → `wiredTigerCacheSizeGB` trong mongod.conf

---

## Document Model — BSON

MongoDB lưu BSON (Binary JSON) — superset của JSON với thêm types.

```javascript
// Document example (BSON internally)
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),  // 12-byte: timestamp+machineID+pid+counter
  "name": "Laptop Gaming",
  "price": 25000000,
  "specs": {                          // Embedded document
    "cpu": "Intel i9",
    "ram": 32,
    "storage": "1TB NVMe"
  },
  "tags": ["gaming", "portable"],    // Array
  "created_at": ISODate("2024-01-15"),
  "reviews": [                        // Array of embedded docs
    {
      "user_id": ObjectId("..."),
      "rating": 5,
      "comment": "Great laptop!"
    }
  ]
}
```

### Embedded Documents vs References

**Embedded (Denormalized):** Tốt khi data luôn được đọc cùng nhau
```javascript
// ✅ User với địa chỉ (luôn đọc cùng nhau)
{
  "_id": ObjectId("..."),
  "name": "Nghia",
  "address": {
    "street": "123 Nguyễn Huệ",
    "city": "Ho Chi Minh"
  }
}
// → 1 document = 1 read, no JOIN needed
```

**References (Normalized):** Tốt khi data được chia sẻ nhiều
```javascript
// ✅ Order references product (product dùng trong nhiều orders)
// orders collection
{
  "_id": ObjectId("ord1"),
  "user_id": ObjectId("usr1"),         // Reference to users collection
  "product_id": ObjectId("prod1"),     // Reference to products collection
  "quantity": 2
}
// → Cần $lookup (JOIN equivalent) để get product details
```

**Rule of thumb:**
- **1-to-1 hoặc 1-to-few:** Embed
- **1-to-many (unbounded):** References (embedded array có thể grow vô hạn → document size limit 16MB)
- **Many-to-many:** References + junction hoặc duplicate data

---

## Replication — Replica Set

```
┌─────────────────────────────────────────────────────────┐
│                    Replica Set (3 nodes)                 │
│                                                           │
│  ┌────────────────┐                                      │
│  │   PRIMARY      │ ← All writes go here                 │
│  │   (votes: 1)   │                                      │
│  └───────┬────────┘                                      │
│          │ oplog replication (async)                     │
│          ├──────────────────────────────┐                │
│          ▼                              ▼                │
│  ┌────────────────┐            ┌────────────────┐        │
│  │   SECONDARY 1  │            │   SECONDARY 2  │        │
│  │   (votes: 1)   │            │   (votes: 1)   │        │
│  │   can serve    │            │   can serve    │        │
│  │   reads        │            │   reads        │        │
│  └────────────────┘            └────────────────┘        │
│                                                           │
│  Election: majority votes needed (2/3 = 2 votes)        │
│  If primary fails → remaining 2 elect new primary        │
└─────────────────────────────────────────────────────────┘
```

### Oplog (Operations Log)
```javascript
// oplog is a capped collection in local.oplog.rs
{
  "ts": Timestamp(1705312800, 1),
  "op": "i",        // i=insert, u=update, d=delete, c=command
  "ns": "mydb.orders",
  "o": { "_id": ObjectId("..."), "total": 150000 }  // new document
}
// All secondaries replay oplog to stay in sync
```

### Read Preference
```javascript
// Primary (default) - always read from primary
MongoClient.connect(uri, { readPreference: 'primary' });

// primaryPreferred - primary if available, else secondary
// secondary - always read from secondary (may be stale)
// secondaryPreferred - secondary if available, else primary
// nearest - lowest network latency

// Use case: Read-heavy analytics → secondaryPreferred
db.collection('orders').find({}).read('secondaryPreferred');
```

---

## Sharding

```
┌──────────────────────────────────────────────────────────────┐
│                   MongoDB Sharded Cluster                     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                   mongos (query router)               │   │
│  │  Routes queries to correct shard based on shard key  │   │
│  └───────────────────────┬──────────────────────────────┘   │
│                          │                                   │
│         ┌────────────────┼────────────────┐                 │
│         ▼                ▼                ▼                 │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐          │
│  │  Shard 1   │   │  Shard 2   │   │  Shard 3   │          │
│  │ (Replica   │   │ (Replica   │   │ (Replica   │          │
│  │  Set)      │   │  Set)      │   │  Set)      │          │
│  └────────────┘   └────────────┘   └────────────┘          │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Config Servers (Replica Set)             │   │
│  │  Stores cluster metadata + chunk locations           │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### Shard Key Strategy
```javascript
// Shard key là critical — chọn sai = performance disaster

// ❌ Monotonically increasing key (ObjectId, timestamp)
// → All writes go to last shard (hotspot!)
sh.shardCollection("mydb.orders", { "_id": 1 });

// ✅ Hashed shard key (even distribution)
sh.shardCollection("mydb.orders", { "_id": "hashed" });
// Pros: Even distribution
// Cons: Range queries broadcast to all shards

// ✅ Compound shard key (zone sharding)
sh.shardCollection("mydb.users", { "region": 1, "user_id": 1 });
// Pros: Co-locate data by region, range queries within region efficient
// Cons: Imbalanced if regions have different cardinality

// ✅ Zone sharding (geo)
sh.addShardTag("shard1", "SEA");
sh.addTagRange("mydb.users",
  { region: "SEA", user_id: MinKey },
  { region: "SEA", user_id: MaxKey },
  "SEA"
);
// → All SEA users on shard1 (low latency for SEA reads)
```

---

## Aggregation Pipeline

MongoDB's powerful data transformation framework.

```javascript
db.orders.aggregate([
  // Stage 1: Filter
  { $match: { status: "completed", created_at: { $gte: new Date("2024-01-01") } } },

  // Stage 2: Join with users (like LEFT JOIN)
  { $lookup: {
      from: "users",
      localField: "user_id",
      foreignField: "_id",
      as: "user"
  }},

  // Stage 3: Unwind array
  { $unwind: "$user" },

  // Stage 4: Group and aggregate
  { $group: {
      _id: "$user.city",
      total_revenue: { $sum: "$total" },
      order_count: { $count: {} },
      avg_order: { $avg: "$total" }
  }},

  // Stage 5: Sort
  { $sort: { total_revenue: -1 } },

  // Stage 6: Limit
  { $limit: 10 },

  // Stage 7: Reshape output
  { $project: {
      city: "$_id",
      total_revenue: 1,
      order_count: 1,
      avg_order: { $round: ["$avg_order", 2] }
  }}
]);
```

**Aggregation best practices:**
- `$match` và `$limit` sớm nhất có thể → reduce data volume
- `$match` trước `$lookup` → filter trước khi join
- Dùng `explain()` để check aggregation plan
- `allowDiskUse: true` cho large aggregations (pipeline > 100MB)

---

## Indexing

```javascript
// Single field index
db.users.createIndex({ email: 1 });  // 1=ascending, -1=descending

// Compound index
db.orders.createIndex({ user_id: 1, status: 1, created_at: -1 });

// Text index (full-text search)
db.products.createIndex({ name: "text", description: "text" });
db.products.find({ $text: { $search: "gaming laptop" } });

// Geospatial index (2dsphere)
db.locations.createIndex({ coords: "2dsphere" });
db.locations.find({
  coords: {
    $near: {
      $geometry: { type: "Point", coordinates: [106.66, 10.76] },
      $maxDistance: 5000  // meters
    }
  }
});

// TTL index (auto-delete documents after expiry)
db.sessions.createIndex(
  { created_at: 1 },
  { expireAfterSeconds: 3600 }  // Delete after 1 hour
);

// Partial index (index only subset of documents)
db.orders.createIndex(
  { user_id: 1 },
  { partialFilterExpression: { status: "active" } }
);
// Only index active orders → smaller, faster

// Explain query
db.orders.find({ user_id: ObjectId("...") }).explain("executionStats");
```

---

## Tuning

### WiredTiger Cache
```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  # Default: (RAM - 1GB) / 2 → set to ~60% RAM
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy  # snappy (fast) or zstd (better compression)
    indexConfig:
      prefixCompression: true
```

### readConcern / writeConcern
```javascript
// writeConcern: How many nodes must ack write?
db.orders.insertOne(
  { total: 150000 },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
  // w: majority = majority of replica set must ack
  // j: true = must be written to journal before ack
  // wtimeout: 5000ms timeout
);

// readConcern: Which version of data to read?
db.orders.find().readConcern("majority");
// "local" (default): read from primary's local state (may not be majority committed)
// "majority": read data acknowledged by majority (stronger)
// "linearizable": strongest, slowest (serial read)
// "snapshot": read from consistent snapshot (multi-document transactions)
```

---

## Ưu/Nhược điểm

### Ưu điểm
```
✅ Flexible schema (great for evolving data models)
✅ Natural JSON mapping (developer-friendly, no ORM needed)
✅ Horizontal scale via sharding (transparent)
✅ Rich query language + aggregation pipeline
✅ Embedded documents = fewer JOINs = lower latency
✅ Atlas (managed) is excellent
✅ Change streams (real-time CDC)
✅ Multi-document ACID transactions (MongoDB 4.0+)
✅ Geospatial, text search built-in
```

### Nhược điểm
```
❌ Higher memory usage (BSON overhead, cache)
❌ Multi-document transactions chậm hơn RDBMS
❌ No built-in JOIN (phải dùng $lookup, kém hiệu quả)
❌ Schema-less = schema chaos nếu không discipline
❌ Shard key chọn sai = rất khó sửa sau này
❌ Atlas đắt, self-hosted complex
❌ 16MB document size limit
❌ No full SQL support (complex analytics khó)
❌ Index không hiệu quả nếu array field large
```

---

## Use Cases

| Use Case | Sao phù hợp |
|----------|-------------|
| Product catalog | Flexible attributes (every product có fields khác nhau) |
| User profiles | Nested data (preferences, addresses, history) |
| Content management | Blog posts, articles với varying structure |
| Real-time analytics | Change streams + aggregation |
| IoT data | Time-series events per device |
| Mobile backend | Atlas Device Sync, offline-first |
| E-commerce | Product variants, inventory |
| Gaming | Player state, leaderboards |

---

## Quick Reference Commands

```javascript
// Switch database
use mydb;

// CRUD
db.users.insertOne({ name: "Nghia", email: "test@example.com" });
db.users.insertMany([{...}, {...}]);

db.users.findOne({ email: "test@example.com" });
db.users.find({ age: { $gte: 25, $lte: 35 } }).sort({ name: 1 }).limit(10);

db.users.updateOne({ _id: id }, { $set: { name: "New Name" }, $push: { tags: "vip" } });
db.users.updateMany({ status: "inactive" }, { $set: { archived: true } });

db.users.deleteOne({ _id: id });
db.users.deleteMany({ created_at: { $lt: new Date("2020-01-01") } });

// Check replica set status
rs.status();
rs.isMaster();

// Check sharding
sh.status();

// Collection stats
db.orders.stats();
db.orders.totalSize();
```

---

## Related Notes

- [[00-Database-MOC]] — Hub trung tâm
- [[01-Terminology#Sharding vs Partitioning]] — Sharding concepts
- [[02-Architecture#Replication Architecture]] — Replication patterns
- [[03-Types#Document Database]] — Document DB overview
- [[10-Operations]] — Backup, monitoring MongoDB
