---
tags: [database, types, nosql, sql, rdbms, document, key-value, graph, timeseries, vector, devops]
---

# 🗂️ Các loại Database — Database Types

> Mỗi loại DB được thiết kế cho một bài toán cụ thể. Dùng sai tool = production incident đang chờ. Xem [[01-Terminology]] cho thuật ngữ cơ bản.

---

## 1. Relational Database (RDBMS)

### Đặc điểm cốt lõi
- **Schema-on-write:** Phải định nghĩa schema trước, data phải conform
- **ACID transactions:** Đảm bảo data integrity tuyệt đối
- **SQL:** Ngôn ngữ query standard, powerful (joins, aggregations, subqueries)
- **Normalized data:** Tránh data duplication, maintain referential integrity
- **B-Tree indexes:** Default, hiệu quả cho phần lớn workloads

### Khi nào dùng RDBMS?
```
✅ Cần ACID (banking, e-commerce orders, inventory)
✅ Data có relationships rõ ràng (joins cần thiết)
✅ Schema stable, ít thay đổi
✅ Complex queries, reporting, analytics
✅ Compliance requirements (audit trail)

❌ Schema thay đổi liên tục (early-stage startup)
❌ Extreme write throughput (millions/sec)
❌ Unstructured/semi-structured data
❌ Global distribution với low-latency writes
```

### Players chính
| DB | Strengths | Best For |
|----|-----------|----------|
| **PostgreSQL** | Feature-rich, extensions (PostGIS, pgvector), JSONB | General purpose, AI, Geo |
| **MySQL/MariaDB** | Simple, fast, huge ecosystem | Web apps, CMS |
| **SQLite** | Embedded, zero-config | Mobile, local dev, testing |
| **Oracle** | Enterprise, RAC, partitioning | Large enterprise, legacy |
| **SQL Server** | .NET ecosystem, SSRS | Microsoft shops |

Xem chi tiết [[04-MySQL-MariaDB]] và [[05-PostgreSQL]].

---

## 2. Document Database

### Đặc điểm cốt lõi
- **Schema-less (schema-flexible):** Documents có thể có fields khác nhau
- **JSON/BSON format:** Natural fit cho application objects
- **Embedded documents:** Denormalized, one-document = one entity
- **No JOINs needed:** Related data embedded trong document
- **Index on any field:** Flexible indexing

### Data model
```json
// User document trong MongoDB
{
  "_id": "user123",
  "name": "Nguyễn Văn A",
  "email": "a@example.com",
  "address": {
    "city": "Ho Chi Minh",
    "district": "1"
  },
  "orders": [
    {"id": "ord1", "total": 150000, "status": "delivered"},
    {"id": "ord2", "total": 300000, "status": "pending"}
  ],
  "tags": ["premium", "b2b"]
}
// → Một document, không cần JOIN
```

### Khi nào dùng Document DB?
```
✅ Schema thay đổi liên tục (products với attributes khác nhau)
✅ Hierarchical/nested data (orders với line items)
✅ Developer productivity quan trọng
✅ Semi-structured content (blog posts, product catalog)
✅ Cần scale writes horizontally

❌ Cần complex multi-document transactions (MongoDB 4.0+ có nhưng chậm hơn RDBMS)
❌ Data highly relational (nhiều JOINs)
❌ Financial/audit data (ACID critical)
```

### Players chính
- **MongoDB** — market leader, mature, Replica Sets + Sharding → [[06-MongoDB]]
- **CouchDB** — multi-master sync, offline-first
- **Amazon DocumentDB** — MongoDB-compatible, managed
- **Firestore** — Google, real-time sync, mobile-friendly
- **DynamoDB** — AWS, key-value + document, serverless scale

---

## 3. Key-Value Database

### Đặc điểm cốt lõi
- **Simplest model:** key → value (value là opaque blob hoặc structured)
- **O(1) reads/writes:** Hash table based
- **No query language:** Get/Set/Delete only (simple DSL trong Redis)
- **Extreme performance:** 100K-1M+ ops/sec
- **In-memory hoặc on-disk**

### Khi nào dùng Key-Value?
```
✅ Caching (HTML fragments, API responses, DB query results)
✅ Session storage
✅ Rate limiting counters
✅ Feature flags
✅ Real-time leaderboards (Redis Sorted Sets)
✅ Pub/Sub messaging

❌ Complex queries (filtering, range queries)
❌ Relational data
❌ Primary data store cho business-critical data (thường dùng làm cache layer)
```

### Players chính
- **Redis** — In-memory, rich data structures, Cluster → [[07-Redis]]
- **Memcached** — Pure caching, simpler than Redis, multi-threaded
- **DynamoDB** — Serverless, unlimited scale, AWS native
- **etcd** — Distributed KV, consensus (Raft), dùng cho K8s, Patroni

---

## 4. Wide-Column Database (Column-Family)

### Đặc điểm cốt lõi
- **Row key → Column families → Columns:** Sparse, multi-dimensional map
- **Distributed:** Sharding tự động, scale to petabytes
- **AP (Cassandra) / CP (HBase):** Configurable consistency
- **Write-optimized:** LSM tree, append-only
- **Tunable consistency:** ONE, QUORUM, ALL

### Data model (Cassandra)
```
Table: user_events
Partition Key: user_id
Clustering Key: event_time

user_id    | event_time          | event_type | data
──────────────────────────────────────────────────────
user_001   | 2024-01-01 00:01:00 | login      | {...}
user_001   | 2024-01-01 00:02:00 | view       | {...}
user_001   | 2024-01-01 00:05:00 | purchase   | {...}
user_002   | 2024-01-01 00:01:30 | login      | {...}

→ Tất cả events của user_001 trong cùng partition → fast range scan
→ Distributed: user_001 → Shard A, user_002 → Shard B
```

### Khi nào dùng Wide-Column?
```
✅ Massive scale (petabytes, millions of writes/sec)
✅ Time-series data (IoT, events, logs) với known access patterns
✅ Multi-datacenter active-active replication
✅ Always-on availability (AP system)

❌ Complex queries, ad-hoc analytics
❌ Frequent schema changes
❌ Strong consistency required
❌ Small-medium scale (overhead không worth it)
```

### Players chính
- **Cassandra** — Apache, Netflix/Apple/Discord dùng, AP, CQL (Cassandra Query Language)
- **HBase** — Hadoop ecosystem, CP, strong consistency
- **Google Bigtable** — Managed, massive scale
- **ScyllaDB** — Cassandra-compatible nhưng C++ (lower latency)

---

## 5. Graph Database

### Đặc điểm cốt lõi
- **Nodes + Edges + Properties:** Native graph storage
- **Relationship-first:** Traversal O(1) per hop (vs JOIN O(n) scan trong SQL)
- **Graph query languages:** Cypher (Neo4j), Gremlin
- **Best for:** Highly connected data với complex traversal

### Khi nào SQL JOIN sucks, Graph shines:
```sql
-- SQL: "Find all 3-hop connections from user X"
-- Cần self-join 3 lần, slow với million-node graphs

-- Cypher (Neo4j):
MATCH (u:User {id: "X"})-[:FOLLOWS*1..3]->(friend)
RETURN friend.name
-- O(k) where k = số connections, không phải O(n) full table scan
```

### Khi nào dùng Graph DB?
```
✅ Social networks (followers, friends)
✅ Recommendation engines ("users like you also bought")
✅ Fraud detection (transaction networks)
✅ Knowledge graphs
✅ Network/topology management
✅ Access control (permission graphs)

❌ Simple CRUD apps
❌ Large volume analytical queries
❌ Not a primary data store cho simple data
```

### Players chính
- **Neo4j** — Market leader, Cypher, ACID, enterprise support
- **Amazon Neptune** — Managed, Gremlin + SPARQL
- **ArangoDB** — Multi-model (graph + document + KV)
- **TigerGraph** — Real-time analytics graph

---

## 6. Time-Series Database

### Đặc điểm cốt lõi
- **Optimized cho time-indexed data:** Metrics, events, sensor readings
- **High ingest rate:** Hàng triệu data points/sec
- **Auto-downsampling/retention:** Data cũ tự động aggregate/delete
- **Built-in time functions:** rate(), irate(), avg_over_time()
- **Compression:** Time-series data compresses extremely well

### Khi nào dùng Time-Series DB?
```
✅ Infrastructure metrics (CPU, memory, network)
✅ Application APM metrics
✅ IoT sensor data
✅ Financial tick data
✅ Log-based analytics với time dimension

❌ General-purpose data storage
❌ Relational data với complex queries
```

### Players chính
| DB | Notes |
|----|-------|
| **InfluxDB** | Popular, InfluxQL/Flux, retention policies |
| **VictoriaMetrics** | Prometheus-compatible, lower resource usage, open-source |
| **TimescaleDB** | PostgreSQL extension → SQL + time-series → [[05-PostgreSQL#Extensions]] |
| **Prometheus** | Pull-based, designed for monitoring, short retention |
| **OpenTSDB** | HBase-backed, massive scale |

---

## 7. Search Engine Database

### Đặc điểm cốt lõi
- **Inverted Index:** Word → [doc1, doc2, ...] mapping
- **Full-text search:** Relevance scoring, tokenization, stemming
- **Near real-time:** Documents searchable within seconds of indexing
- **Distributed:** Sharding + replication built-in
- **Aggregations:** Analytics on top of search

Xem chi tiết [[08-Elasticsearch]].

### Khi nào dùng Search Engine?
```
✅ Full-text search (product search, blog search)
✅ Log analytics (ELK stack)
✅ Observability data (metrics + logs + traces correlation)
✅ Faceted search (filter by category, price range, etc.)
✅ Autocomplete/suggestions

❌ Primary OLTP data store
❌ Complex transactions
❌ Write-heavy workloads (indexing overhead)
```

### Players chính
- **Elasticsearch** — Market leader, ELK stack → [[08-Elasticsearch]]
- **OpenSearch** — AWS fork of Elasticsearch
- **Typesense** — Simpler, typo-tolerant, developer-friendly
- **Meilisearch** — Lightweight, instant search, Rust-based
- **Solr** — Apache, older, still used in enterprise

---

## 8. NewSQL

### Vấn đề NewSQL giải quyết
```
RDBMS          NoSQL
Strong ACID ✅  Scale horizontally ✅
SQL ✅          High availability ✅
Complex queries ✅  Low latency distributed ✅
Scale vertically ❌  ACID ❌
Distributed ❌   SQL ❌

NewSQL = Best of both worlds (in theory)
```

### Đặc điểm
- **Distributed SQL:** Full SQL compliance + horizontal sharding
- **ACID trong distributed setting:** Dùng distributed transactions (2PC + Paxos/Raft)
- **Auto-sharding:** Transparent to application
- **Geo-distributed:** Multi-region với locality routing

### Players chính
| DB | Notes |
|----|-------|
| **CockroachDB** | Strong consistency, geo-partitioning, PostgreSQL wire-compatible |
| **TiDB** | MySQL-compatible, TiKV (RocksDB) + TiFlash (columnar) |
| **Spanner** | Google, global, TrueTime clock |
| **YugabyteDB** | PostgreSQL-compatible, distributed |
| **Vitess** | MySQL sharding middleware (không phải DB mới) |

### Khi nào dùng NewSQL?
```
✅ Need global distribution với ACID
✅ Scale beyond single-node PostgreSQL/MySQL
✅ Multi-region active-active writes
✅ Existing SQL apps cần scale

❌ Simple workloads (overhead không worth it)
❌ Ultra-low latency (distributed tx overhead ~50-200ms cross-region)
```

---

## 9. Vector Database

> Vector DB là category mới nhất, driven bởi AI/LLM revolution. Xem chi tiết đầy đủ tại [[09-VectorDB]].

### Đặc điểm
- **Store high-dimensional vectors (embeddings)**
- **Similarity search** thay vì exact match
- **ANN (Approximate Nearest Neighbor):** HNSW, IVF algorithms
- **Metadata filtering:** Kết hợp vector search + traditional filters

### Players: Pinecone, Weaviate, Milvus, Chroma, Qdrant, pgvector

---

## 10. In-Memory Database

### Đặc điểm
- **All data in RAM:** Microsecond latency
- **Persistence optional:** RDB snapshots, AOF
- **Limited by RAM:** Cannot scale beyond available memory (Redis Cluster giải quyết phần nào)

### Khi nào dùng?
```
✅ Extreme low latency required (< 1ms)
✅ Caching layer
✅ Session store
✅ Real-time leaderboards, counters
✅ Message queues

❌ Large datasets vượt quá RAM
❌ Primary durable data store (trừ khi cấu hình persistence đúng)
```

---

## 📊 Bảng so sánh tổng hợp

| Loại DB | ACID | Scale | Latency | Query Power | Cost Complexity |
|---------|------|-------|---------|-------------|-----------------|
| RDBMS (PG/MySQL) | ✅ Strong | Vertical + Read replica | Low-Medium | ✅✅✅ SQL | Medium |
| Document (MongoDB) | ✅ (multi-doc limited) | Horizontal | Low | ✅✅ + aggregation | Low-Medium |
| Key-Value (Redis) | ✅ (single key) | Cluster | ✅ Microsecond | ✅ Limited | Low |
| Wide-Column (Cassandra) | ❌ Eventual | ✅✅ Massive | Low | ✅ CQL (limited) | High |
| Graph (Neo4j) | ✅ Strong | Limited | Fast for traversal | ✅✅ Cypher | High |
| Time-Series | ✅ | Horizontal | Low | ✅ (time-aware) | Medium |
| Search (ES) | ❌ | ✅ Horizontal | Near real-time | ✅✅ DSL | High |
| NewSQL (CockroachDB) | ✅ Distributed | ✅ Horizontal | Medium | ✅✅✅ SQL | High |
| Vector | ❌ | Horizontal | Low-Medium | Similarity only | Medium |
| In-Memory (Redis) | ✅ Limited | Cluster | ✅✅ Microsecond | ✅ Limited | Low |

---

## 🎯 Decision Flowchart

```
Bạn cần lưu gì?
│
├── Structured records với relationships → RDBMS (PostgreSQL/MySQL)
│   └── Need distributed? → CockroachDB / TiDB
│
├── JSON documents, flexible schema → MongoDB
│
├── Simple key lookup, caching → Redis
│
├── IoT/metrics/telemetry time-series → VictoriaMetrics/InfluxDB/TimescaleDB
│
├── Full-text search, log analytics → Elasticsearch
│
├── Highly connected data (social/recommendations) → Neo4j
│
├── Massive write throughput (IoT scale) → Cassandra
│
└── AI embeddings, semantic search → pgvector / Milvus / Weaviate
    └── See: [[09-VectorDB]]
```

---

## Related Notes

- [[00-Database-MOC]] — Hub trung tâm
- [[01-Terminology]] — ACID, CAP, BASE concepts
- [[02-Architecture]] — Storage engines
- [[04-MySQL-MariaDB]] — RDBMS deep dive
- [[05-PostgreSQL]] — PostgreSQL features
- [[06-MongoDB]] — Document DB deep dive
- [[07-Redis]] — Key-Value/In-Memory deep dive
- [[08-Elasticsearch]] — Search Engine deep dive
- [[09-VectorDB]] — Vector DB deep dive
- [[10-Operations]] — Vận hành các loại DB
