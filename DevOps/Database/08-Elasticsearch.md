---
tags: [database, elasticsearch, search, nosql, devops, elk, observability]
---

# Elasticsearch

> Elasticsearch — Distributed search and analytics engine built on Apache Lucene. Dùng cho full-text search, log analytics, và observability.

## Liên kết nhanh
- [[00-Database-MOC]] — Tổng quan
- [[01-Terminology]] — Thuật ngữ
- [[03-Types]] — Các loại Database
- [[10-Operations]] — Vận hành

---

## Architecture

### Cluster → Node → Index → Shard → Document

```
┌─────────────────────────────────────────────────────────┐
│                    ES Cluster                            │
│                                                          │
│  ┌────────────────┐  ┌────────────────┐  ┌───────────┐  │
│  │  Master Node   │  │  Data Node 1   │  │Data Node 2│  │
│  │ (cluster mgmt) │  │ ┌────────────┐ │  │┌─────────┐│  │
│  │                │  │ │  Index A   │ │  ││ Index A ││  │
│  │                │  │ │ Shard 0 (P)│ │  ││Shard 1(P│  │
│  │                │  │ │ Shard 1 (R)│ │  ││Shard 0(R││  │
│  └────────────────┘  │ └────────────┘ │  │└─────────┘│  │
│                      └────────────────┘  └───────────┘  │
│  ┌────────────────┐                                      │
│  │ Coordinating   │ ← Client requests vào đây            │
│  │    Node        │ → route đến đúng shard                │
│  └────────────────┘                                      │
└─────────────────────────────────────────────────────────┘
```

### Node Roles

| Role | Mô tả | Khi nào tách riêng |
|------|-------|-------------------|
| **master** | Quản lý cluster state, index creation/deletion | Cluster lớn (>10 nodes) |
| **data** | Lưu data, xử lý CRUD và search | Luôn cần |
| **data_hot** | Data nodes cho hot data (SSD) | ILM hot-warm-cold architecture |
| **data_warm** | Data nodes cho warm data (HDD) | |
| **data_cold** | Data nodes cho cold data (ít query) | |
| **ingest** | Pre-process documents trước khi index | Pipeline phức tạp |
| **coordinating** | Route requests, merge results | High-traffic clusters |
| **ml** | Machine learning nodes | X-Pack ML features |

```yaml
# elasticsearch.yml
node.roles: [master, data]          # single-node dev
node.roles: [master]                # dedicated master
node.roles: [data_hot]              # dedicated hot data node
node.roles: []                      # coordinating only
```

---

## Inverted Index — Cơ chế hoạt động

### Tại sao Inverted Index nhanh hơn LIKE query

**RDBMS LIKE:** `SELECT * FROM posts WHERE body LIKE '%redis%'`
→ Phải scan toàn bộ rows → O(n) → chậm với data lớn

**Inverted Index:**
```
Text: "Redis is fast, Redis supports clustering"

Inverted Index:
┌──────────────┬───────────────────────────────────────┐
│     Term     │  Posting List (doc_id : positions)    │
├──────────────┼───────────────────────────────────────┤
│ redis        │  doc1:[0,3], doc2:[0], doc3:[2,8]     │
│ fast         │  doc1:[2], doc5:[1]                   │
│ support      │  doc1:[4], doc2:[3]                   │  ← analyzed: "supports" → "support"
│ cluster      │  doc1:[5], doc4:[1]                   │  ← "clustering" → "cluster"
└──────────────┴───────────────────────────────────────┘

Query "redis clustering":
→ Lookup "redis" → {doc1, doc2, doc3}
→ Lookup "cluster" → {doc1, doc4}
→ Intersection → {doc1}  ← O(log n), cực nhanh
```

### Document Lifecycle

```
Document → Tokenizer → Token Filter → Inverted Index
                              ↓
POST /my-index/_doc/1
{
  "title": "Redis is AWESOME!"
}
                              ↓
Standard Analyzer:
  1. Character filter: "Redis is AWESOME!" (unchanged)
  2. Tokenizer: ["Redis", "is", "AWESOME"]
  3. Token filter:
     - lowercase: ["redis", "is", "awesome"]
     - stop words: ["redis", "awesome"]  ← "is" bị bỏ
     - stemming (nếu có): ["redis", "awesom"]
                              ↓
Stored in Lucene Segment (immutable)
                              ↓
Segment Merge (background) → Larger segments
```

### Segments

- Mỗi shard là một Lucene index gồm nhiều **segments**
- Segments là **immutable** — write tạo segment mới, delete đánh dấu tombstone
- **Segment merge** (background): merge nhỏ → lớn, xóa tombstones
- `refresh_interval: 1s` → tạo in-memory segment → data searchable
- `flush` → segment được write xuống disk

---

## Mapping

### Dynamic vs Explicit Mapping

```json
// Dynamic mapping (default) — ES tự đoán kiểu
// Vấn đề: "price": "99.99" → text (sai!), không thể sort/aggregate

// Explicit mapping — định nghĩa rõ ràng
PUT /products
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text",                 // full-text search
        "fields": {
          "keyword": {                  // multi-field: dùng cho exact match + aggregation
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "price":       { "type": "float" },
      "created_at":  { "type": "date", "format": "strict_date_optional_time" },
      "in_stock":    { "type": "boolean" },
      "tags":        { "type": "keyword" },     // array của keywords
      "description": { "type": "text", "analyzer": "english" },
      "location":    { "type": "geo_point" },
      "metadata":    { "type": "object" },
      "embedding":   { "type": "dense_vector", "dims": 768 }  // vector search!
    }
  }
}
```

### Field Types quan trọng

| Type | Dùng khi | Notes |
|------|----------|-------|
| `text` | Full-text search | Analyzed, không sort được |
| `keyword` | Exact match, aggregation, sort | Not analyzed |
| `integer/long/float` | Số | Range queries, aggregation |
| `date` | Timestamps | Nhiều format support |
| `boolean` | True/false | |
| `geo_point` | Lat/lon | Proximity search |
| `nested` | Array of objects với query độc lập | Tốn memory hơn |
| `dense_vector` | Vector embeddings | ANN search (8.0+) |

---

## Query DSL

### Basic Queries

```json
// Match query — full-text search (analyzed)
GET /products/_search
{
  "query": {
    "match": {
      "description": "redis cache performance"
    }
  }
}

// Term query — exact match (not analyzed)
GET /products/_search
{
  "query": {
    "term": { "tags": "database" }
  }
}

// Range query
{
  "query": {
    "range": {
      "price": { "gte": 10, "lte": 100 }
    }
  }
}

// Bool query — kết hợp nhiều conditions
{
  "query": {
    "bool": {
      "must": [
        { "match": { "description": "redis" } }
      ],
      "filter": [
        { "term": { "in_stock": true } },
        { "range": { "price": { "lte": 50 } } }
      ],
      "should": [
        { "term": { "tags": "featured" } }   // boost score nếu có
      ],
      "must_not": [
        { "term": { "tags": "deprecated" } }
      ]
    }
  }
}
```

**must vs filter:**
- `must`: tính vào relevance score
- `filter`: không tính score, được cache → nhanh hơn

### Aggregations

```json
GET /logs/_search
{
  "size": 0,
  "aggs": {
    "errors_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "1h"
      },
      "aggs": {
        "error_count": {
          "filter": { "term": { "level": "ERROR" } }
        },
        "avg_response_time": {
          "avg": { "field": "response_time_ms" }
        }
      }
    },
    "top_endpoints": {
      "terms": {
        "field": "endpoint.keyword",
        "size": 10
      }
    }
  }
}
```

---

## Cluster Health & Operations

### Cluster Status

```
GREEN  → Tất cả primary + replica shards assigned ✅
YELLOW → Tất cả primary shards OK, MỘT SỐ replica chưa assigned ⚠️
         (1-node cluster mặc định là YELLOW vì không thể tự replicate)
RED    → Một số PRIMARY shards chưa assigned ❌ (data loss possible!)
```

```bash
# Health check
GET /_cluster/health?pretty

# Shard allocation issues
GET /_cluster/allocation/explain

# Unassigned shards
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason

# Node stats
GET /_cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m,node.role
```

### Common Issues

```bash
# Disk watermark (mặc định)
# low: 85% → không allocate thêm shard
# high: 90% → di chuyển shards đi
# flood_stage: 95% → đặt index thành read-only!

# Fix flood stage
PUT /my-index/_settings
{
  "index.blocks.read_only_allow_delete": null
}

# Tăng watermark (không khuyến nghị lâu dài)
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.flood_stage": "98%"
  }
}
```

---

## Index Lifecycle Management (ILM)

Tự động quản lý vòng đời của index theo thời gian:

```
Hot Phase → Warm Phase → Cold Phase → Delete Phase
(active write/read) (read-only) (archived) (deleted)
```

```json
PUT /_ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "7d",        // rollover sau 7 ngày
            "max_size": "50gb"      // hoặc sau 50GB
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",            // 7 ngày sau rollover
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {},
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

### Index Template + ILM

```json
// Index template: áp dụng settings/mapping cho index mới tự động
PUT /_index_template/logs-template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy",
      "index.lifecycle.rollover_alias": "logs"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "level":      { "type": "keyword" },
        "message":    { "type": "text" },
        "service":    { "type": "keyword" }
      }
    }
  }
}

// Tạo initial index với alias
PUT /logs-000001
{
  "aliases": {
    "logs": { "is_write_index": true }
  }
}
```

---

## Tuning

### JVM Heap

```bash
# /etc/elasticsearch/jvm.options
# Rule: 50% RAM, max 32GB (vì > 32GB compressed OOPs tắt)
-Xms16g
-Xmx16g      # Xms = Xmx để tránh GC pauses khi resize

# Kiểm tra heap pressure
GET /_nodes/stats/jvm?pretty | grep heap_used_percent
```

### Performance Settings

```json
// Bulk indexing - tắt refresh tạm thời
PUT /my-index/_settings
{
  "refresh_interval": "-1",        // tắt auto-refresh
  "number_of_replicas": 0          // tắt replica trong lúc bulk
}

// Sau khi bulk xong
PUT /my-index/_settings
{
  "refresh_interval": "1s",
  "number_of_replicas": 1
}
POST /my-index/_forcemerge?max_num_segments=1

// Search performance
PUT /my-index/_settings
{
  "index.refresh_interval": "5s",          // tăng lên cho write-heavy
  "index.translog.durability": "async",    // async translog (mất < 5s data nếu crash)
  "index.translog.sync_interval": "5s"
}
```

### Bulk API (indexing nhiều docs)

```bash
# Khuyến nghị: 5-15MB per bulk request
curl -X POST "localhost:9200/_bulk" -H 'Content-Type: application/json' -d'
{"index": {"_index": "logs", "_id": "1"}}
{"@timestamp": "2026-03-15T00:00:00Z", "level": "INFO", "message": "Server started"}
{"index": {"_index": "logs", "_id": "2"}}
{"@timestamp": "2026-03-15T00:01:00Z", "level": "ERROR", "message": "Connection refused"}
'
```

---

## ELK / EFK Stack

```
                     ┌─────────────┐
Logs/Metrics ──────► │  Logstash   │ ─────► Elasticsearch ──► Kibana
                     │  (parse,    │              ▲
                     │  transform) │              │
                     └─────────────┘              │
                                                  │
Logs ──────────────► Filebeat/Fluentd ────────────┘
                     (lightweight shipper)
```

### Filebeat → Elasticsearch (đơn giản nhất)

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    paths:
      - /var/log/nginx/*.log
    fields:
      service: nginx

output.elasticsearch:
  hosts: ["https://elasticsearch:9200"]
  username: "elastic"
  password: "${ELASTIC_PASSWORD}"

setup.ilm.enabled: true
setup.template.enabled: true
```

### Fluentd → Elasticsearch (EFK stack - Kubernetes)

```yaml
# fluentd configmap
<match kubernetes.**>
  @type elasticsearch
  host elasticsearch-service
  port 9200
  logstash_format true
  logstash_prefix k8s
  include_tag_key true
  <buffer>
    @type file
    chunk_limit_size 10MB
    flush_interval 5s
    retry_max_interval 30
    retry_forever true
  </buffer>
</match>
```

---

## Ưu Nhược Điểm

### Ưu điểm
- **Full-text search** không có đối thủ trong world of databases
- **Horizontal scaling** tự nhiên qua sharding
- **Near real-time** search (1s default refresh)
- **Rich aggregations**: histogram, percentiles, cardinality
- **Schema-flexible**: dynamic mapping
- **Ecosystem**: Kibana, APM, Fleet, Elastic Agent

### Nhược điểm
- **Phức tạp** để vận hành đúng cách (shard sizing, ILM, JVM tuning)
- **Memory intensive**: JVM heap + OS cache
- **Không phải primary store**: mất data nếu không có backup
- **Eventual consistency**: search có thể delay 1s
- **Cost**: Elasticsearch cần nhiều tài nguyên hơn alternatives
- **GDPR challenges**: delete/update không thật sự xóa ngay (segments)

---

## Use Cases

| Use Case | Pattern | Notes |
|----------|---------|-------|
| **Full-text search** | Fuzzy, phrase, multi-field | E-commerce, documentation |
| **Log analytics** | ELK/EFK stack | Centralized logging |
| **APM / Observability** | Elastic APM, Metricbeat | Traces, metrics, logs |
| **Security analytics (SIEM)** | Elastic Security | UEBA, threat detection |
| **Geospatial search** | geo_point + geo_distance | Store finder, delivery |
| **Time-series** | date_histogram aggregation | Metrics, IoT (dù InfluxDB tốt hơn) |
| **Product catalog** | Multi-field, faceted search | Ecommerce filter/search |

---

## Related Notes

- [[00-Database-MOC]] — Tổng quan toàn bộ
- [[01-Terminology]] — Thuật ngữ: Index, Sharding, Replication
- [[03-Types]] — So sánh với các loại DB khác
- [[07-Redis]] — Thường dùng kết hợp Redis cache + Elasticsearch search
- [[09-VectorDB]] — ES 8.0+ có dense_vector field cho vector search
- [[10-Operations]] — ILM, backup, monitoring
