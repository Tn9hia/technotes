---
tags: [database, moc, devops, sql, nosql]
---

# 🗄️ Database — Map of Content

> Hub trung tâm cho toàn bộ kiến thức về Database. Mọi con đường đều dẫn về đây.

## 📚 Hệ thống ghi chú này gồm gì?

Bộ notes này cover toàn bộ kiến thức Database từ fundamental đến production-grade operations — từ lý thuyết ACID/CAP cho đến cách vận hành, tune performance thực tế.

---

## 🗺️ Navigation

| File | Nội dung |
|------|----------|
| [[01-Terminology]] | Thuật ngữ cốt lõi: ACID, CAP, MVCC, WAL, Index types |
| [[02-Architecture]] | Kiến trúc nội tại: Storage Engine, Buffer Pool, Replication |
| [[03-Types]] | Các loại DB: RDBMS, Document, KV, Graph, TimeSeries, Vector |
| [[04-MySQL-MariaDB]] | MySQL & MariaDB: InnoDB, Replication, HA, Tuning |
| [[05-PostgreSQL]] | PostgreSQL: MVCC/WAL internals, Patroni HA, Performance Tuning, Operations & Security (backup/PITR/RLS/pgAudit), CloudNativePG on K8s, Extensions (pgvector/PostGIS/TimescaleDB) |
| [[06-MongoDB]] | MongoDB: WiredTiger, Sharding, Aggregation |
| [[07-Redis]] | Redis: Data structures & patterns, Sentinel/Cluster HA, Memory/Eviction/Ops, Redis on Kubernetes |
| [[08-Elasticsearch]] | Elasticsearch: Inverted Index, Query DSL, ILM |
| [[09-VectorDB]] | Vector DB: HNSW, RAG, Pinecone, Weaviate, Milvus |
| [[10-Operations]] | Vận hành: Backup, PITR, Monitoring, DR |

---

## 🏗️ Sơ đồ tổng quan — Khi nào dùng loại DB nào?

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATABASE DECISION TREE                        │
└─────────────────────────────────────────────────────────────────┘

Dữ liệu có cấu trúc rõ ràng?
├── YES → Cần ACID transaction?
│         ├── YES → RDBMS
│         │         ├── Cần scale + distributed → PostgreSQL / CockroachDB
│         │         ├── Web app truyền thống   → MySQL / MariaDB
│         │         └── Enterprise / legacy    → Oracle / SQL Server
│         └── NO (eventual consistency OK?)
│               ├── Cần throughput cao         → Cassandra (Wide-Column)
│               └── Cần geo-distributed        → CockroachDB / TiDB (NewSQL)
│
└── NO → Dữ liệu dạng gì?
          ├── JSON/Documents                   → MongoDB / CouchDB
          ├── Key-Value đơn giản / cache        → Redis / Memcached
          ├── Graph (relationships)             → Neo4j / Neptune
          ├── Time-series metrics               → InfluxDB / VictoriaMetrics / TimescaleDB
          ├── Full-text search / log analytics  → Elasticsearch / OpenSearch
          ├── AI/ML embeddings / semantic search→ pgvector / Pinecone / Weaviate / Milvus
          └── Huge analytical queries           → ClickHouse / BigQuery / Redshift

┌─────────────────────────────────────────────────────────────────┐
│                    SCALE DECISION                                │
│                                                                  │
│   Single Node  ──────────────────────────────► Distributed      │
│   SQLite → MySQL → PostgreSQL → TiDB/Cockroach → Cassandra      │
│   Redis  → Redis Cluster                                         │
│   Mongo  → MongoDB Sharded Cluster                              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    CAP THEOREM QUICK VIEW                        │
│                                                                  │
│              Consistency                                         │
│                   /\                                             │
│                  /  \                                            │
│         MySQL  /    \ HBase                                      │
│       Postgres/      \Cassandra                                  │
│              /   ??   \                                          │
│             /  (pick   \                                         │
│            /    two)    \                                        │
│           ──────────────                                         │
│      Availability    Partition                                   │
│                      Tolerance                                   │
│                                                                  │
│  CA: MySQL, PostgreSQL (no partition tolerance in single node)  │
│  CP: HBase, MongoDB (strong consistency over availability)      │
│  AP: Cassandra, DynamoDB (availability over consistency)        │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔑 Core Concepts Quick Reference

### ACID vs BASE
- **ACID** → Relational DBs (MySQL, PostgreSQL) — [[01-Terminology#ACID]]
- **BASE** → NoSQL distributed systems (Cassandra, DynamoDB) — [[01-Terminology#BASE]]

### Replication Patterns
- **Primary-Replica** → MySQL, PostgreSQL, MongoDB — [[02-Architecture#Replication]]
- **Multi-Primary** → Galera, MySQL Group Replication — [[04-MySQL-MariaDB]]
- **Quorum-based** → MongoDB, Cassandra — [[06-MongoDB]]

### HA Solutions
- MySQL/MariaDB → ProxySQL + Galera / MGR — [[04-MySQL-MariaDB]]
- PostgreSQL → Patroni + etcd + pgBouncer — [[05-PostgreSQL]]
- MongoDB → Replica Set tự động failover — [[06-MongoDB]]
- Redis → Sentinel hoặc Cluster — [[07-Redis]]

---

## 🚀 Production Cheatsheet

```
Workload Type          → Recommended DB
─────────────────────────────────────────
OLTP (transactions)    → PostgreSQL / MySQL
OLAP (analytics)       → ClickHouse / Redshift
Caching                → Redis
Session store          → Redis
Job queue              → Redis Streams / RabbitMQ
Full-text search       → Elasticsearch
Log storage            → Elasticsearch + ILM
Time-series metrics    → VictoriaMetrics / InfluxDB
Document store         → MongoDB
Graph traversal        → Neo4j
AI/Semantic search     → pgvector / Milvus / Weaviate
Geo-spatial            → PostgreSQL + PostGIS
Multi-model            → PostgreSQL (JSONB + pgvector + PostGIS)
```

---

## 🔗 Related Areas

- [[../Kubernetes/00-K8s-MOC|Kubernetes]] — StatefulSet cho DB deployment
- [[../CI-CD/00-CICD-MOC|CI/CD]] — Schema migration trong pipeline
- [[../Monitoring/00-Monitoring-MOC|Monitoring]] — Grafana + PMM cho DB metrics
- [[../Security/00-Security-MOC|Security]] — Encryption at rest/in transit

---

## 📖 Related Notes

- [[01-Terminology]]
- [[02-Architecture]]
- [[03-Types]]
- [[04-MySQL-MariaDB]]
- [[05-PostgreSQL]]
- [[06-MongoDB]]
- [[07-Redis]]
- [[08-Elasticsearch]]
- [[09-VectorDB]]
- [[10-Operations]]
