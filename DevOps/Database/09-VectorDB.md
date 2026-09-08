---
tags: [database, vector-db, ai, llm, rag, embeddings, devops]
---

# Vector Database

> Vector Database — Cơ sở dữ liệu chuyên biệt để lưu trữ và tìm kiếm vector embeddings (high-dimensional float arrays). Nền tảng cho AI/ML applications hiện đại.

## Liên kết nhanh
- [[00-Database-MOC]] — Tổng quan
- [[05-PostgreSQL]] — pgvector extension
- [[08-Elasticsearch]] — dense_vector field
- [[03-Types]] — Các loại Database

---

## Vector Database là gì?

### Từ Text → Vector Embedding

```
"Redis là một in-memory database"
           │
    Embedding Model
    (text-embedding-3-small, BGE, E5, etc.)
           │
           ▼
[0.023, -0.156, 0.891, 0.034, -0.445, ..., 0.112]
           ← 1536 dimensions (OpenAI ada-002) →
```

**Embedding** là cách biểu diễn semantic meaning của dữ liệu dưới dạng vector số học:
- Hai câu có ý nghĩa tương tự → vectors gần nhau trong không gian
- Hai câu trái nghĩa → vectors xa nhau
- Khái niệm trừu tượng được mã hóa trong relationships giữa các dimensions

### Vì sao cần Vector DB?

| Nhu cầu | Traditional DB | Vector DB |
|---------|---------------|-----------|
| Tìm "redis cache tutorial" | LIKE '%redis%' | Tìm docs về caching dù không có từ "redis" |
| Tìm ảnh tương tự | Không làm được | So sánh embedding của ảnh |
| Semantic search | Không | Tìm theo ý nghĩa, không phải từ khóa |
| Scale đến hàng triệu vectors | B-Tree index không đủ | ANN index chuyên biệt |

---

## RAG — Retrieval Augmented Generation

Kiến trúc phổ biến nhất dùng Vector DB với LLMs:

```
                    ┌─────────────────────────────────────────┐
INDEXING PHASE      │                                         │
                    │  Documents/Text                         │
                    │       │                                 │
                    │       ▼                                 │
                    │  Embedding Model                        │
                    │  (e.g., text-embedding-3-small)         │
                    │       │                                 │
                    │       ▼                                 │
                    │  Vector DB (stored chunks + embeddings) │
                    └─────────────────────────────────────────┘

                    ┌─────────────────────────────────────────┐
QUERY PHASE         │                                         │
                    │  User: "Redis dùng khi nào?"            │
                    │       │                                 │
                    │       ▼                                 │
                    │  Embedding Model → Query Vector          │
                    │       │                                 │
                    │       ▼                                 │
                    │  Vector DB: ANN Search                   │
                    │  → Top K relevant chunks                 │
                    │       │                                 │
                    │       ▼                                 │
                    │  LLM (GPT-4, Claude, Gemini...)         │
                    │  Prompt = Query + Retrieved Context      │
                    │       │                                 │
                    │       ▼                                 │
                    │  Answer dựa trên actual documents ✅     │
                    └─────────────────────────────────────────┘
```

**RAG giải quyết:**
- **Hallucination**: LLM có context thật từ documents
- **Knowledge cutoff**: Không cần retrain khi có data mới
- **Private data**: LLM không cần thấy data trực tiếp trong training

---

## Similarity Metrics

### Cosine Similarity (phổ biến nhất)

```
                    A · B
cos(θ) =  ─────────────────────
           ||A|| × ||B||

Kết quả: [-1, 1]
  1.0  = identical direction (rất tương đồng)
  0.0  = orthogonal (không liên quan)
 -1.0  = opposite direction (hoàn toàn trái ngược)
```

- **Không quan tâm magnitude**, chỉ quan tâm direction
- Tốt nhất cho text embeddings
- Thường normalize vectors trước → Cosine = Dot Product

### Euclidean Distance (L2)

```
d = √(Σ(aᵢ - bᵢ)²)

Kết quả: [0, ∞)
  0 = identical
  nhỏ hơn = tương đồng hơn
```

- Quan tâm cả magnitude lẫn direction
- Dùng cho image embeddings, spatial data

### Dot Product (Inner Product)

```
A · B = Σ(aᵢ × bᵢ)

Kết quả: (-∞, ∞)
  lớn hơn = tương đồng hơn (với normalized vectors)
```

- Nhanh nhất tính toán
- Với normalized vectors = Cosine similarity
- OpenAI embeddings được normalize sẵn → dùng Dot Product

---

## Indexing Algorithms — ANN Search

Vector DB không dùng exact search (quá chậm với millions of vectors) mà dùng **Approximate Nearest Neighbor (ANN)** — đánh đổi 1-5% accuracy để tăng tốc độ 100-1000x.

### HNSW — Hierarchical Navigable Small World

**Thuật toán phổ biến nhất**, được dùng bởi Weaviate, Qdrant, pgvector, Milvus.

```
Layer 2 (ít nodes nhất):  1 ──────────────── 5
                           │                  │
Layer 1:              1 ──2──3──────────5──6──7
                      │        │         │
Layer 0 (tất cả):  1─2─3─4─5─6─7─8─9─10─11─12─...
```

**Cách hoạt động:**
1. Graph multi-layer — layer trên có ít nodes hơn (skip list concept)
2. Search bắt đầu từ layer trên (coarse navigation)
3. "Greedy" traverse xuống layer dưới dần
4. Layer 0 là tất cả vectors, tìm exact neighbors tại đây

**Parameters:**
- `M` (16-64): số connections mỗi node — tăng → accuracy tăng, memory tăng
- `ef_construction` (100-500): số candidates khi build — tăng → quality tăng, build time tăng
- `ef_search` (50-200): số candidates khi search — tăng → accuracy tăng, latency tăng

**Trade-offs:**
- ✅ Rất nhanh (sub-millisecond với millions of vectors)
- ✅ High recall (~99% accuracy với tuning đúng)
- ❌ Memory intensive (graph structure tốn RAM)
- ❌ Build time chậm hơn IVF

### IVF — Inverted File Index

```
Training Phase:
  1. K-Means clustering → tạo N centroids (Voronoi cells)

Search Phase:
  Query → tìm N_probe nearest centroids → search trong những cells đó
```

**Dùng kết hợp với PQ (IVF-PQ) trong FAISS:**
```
IVF: reduce search space (tìm đúng cluster)
PQ:  quantize vectors để tiết kiệm memory
```

**Khi nào dùng IVF:**
- Dataset rất lớn (>10M vectors) cần tiết kiệm memory
- Có thể trade recall lấy speed
- Batch processing, không cần real-time insertion

### FAISS — Facebook AI Similarity Search

- Library của Meta, không phải database — thường được embedded trong các Vector DB
- Hỗ trợ nhiều index types: Flat (exact), IVF, HNSW, PQ
- Tốt cho research, local experimentation, custom integrations

### Product Quantization (PQ)

```
1536-dim vector → chia thành 64 subvectors (24-dim mỗi cái)
→ mỗi subvector quantize thành 1 byte (256 centroids)
→ 1536 floats (6KB) → 64 bytes (1/96 size!)
```

- Giảm memory 50-96x với accuracy loss nhỏ
- Thường dùng kết hợp: HNSW + PQ hoặc IVF + PQ

---

## Các Vector Database Phổ Biến

### Pinecone — Fully Managed

```python
import pinecone

pc = pinecone.Pinecone(api_key="YOUR_API_KEY")
index = pc.Index("my-index")

# Upsert vectors
index.upsert(vectors=[
    {"id": "doc1", "values": [0.1, 0.2, ...], "metadata": {"text": "Redis tutorial"}},
    {"id": "doc2", "values": [0.3, 0.1, ...], "metadata": {"text": "PostgreSQL guide"}},
])

# Query
results = index.query(
    vector=[0.1, 0.2, ...],
    top_k=5,
    include_metadata=True,
    filter={"category": {"$eq": "database"}}
)
```

**Đặc điểm:**
- ✅ Fully managed — không cần ops
- ✅ Scale tự động, SLA 99.9%
- ✅ Serverless tier (pay per use)
- ❌ Vendor lock-in
- ❌ Data ra ngoài infrastructure của bạn
- ❌ Costly ở scale lớn
- **Phù hợp:** Startup, prototype nhanh, khi không có infra team

### Weaviate — Open Source, Hybrid Search

```python
import weaviate

client = weaviate.connect_to_local()  # hoặc Weaviate Cloud

# Schema
client.collections.create(
    "Document",
    vectorizer_config=weaviate.classes.config.Configure.Vectorizer.text2vec_openai(),
    properties=[
        weaviate.classes.config.Property(name="content", data_type=weaviate.classes.config.DataType.TEXT),
        weaviate.classes.config.Property(name="source", data_type=weaviate.classes.config.DataType.TEXT),
    ]
)

# Insert (vectorize tự động)
collection = client.collections.get("Document")
collection.data.insert({"content": "Redis is fast", "source": "blog"})

# Hybrid search (vector + BM25 keyword)
results = collection.query.hybrid(
    query="in-memory database",
    alpha=0.7,   # 0 = pure BM25, 1 = pure vector
    limit=5
)
```

**Đặc điểm:**
- ✅ Hybrid search (vector + BM25) — tốt hơn pure vector search
- ✅ Built-in vectorizer modules (OpenAI, Cohere, HuggingFace)
- ✅ GraphQL API + REST + gRPC
- ✅ Multi-tenancy cho SaaS apps
- ❌ Resource intensive hơn Qdrant
- **Phù hợp:** Production RAG, hybrid search, multi-tenant SaaS

### Milvus — Cloud-Native, Large Scale

```python
from pymilvus import connections, Collection, FieldSchema, CollectionSchema, DataType

connections.connect("default", host="localhost", port="19530")

# Schema
fields = [
    FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
    FieldSchema(name="content", dtype=DataType.VARCHAR, max_length=1000),
    FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=1536),
]
schema = CollectionSchema(fields)
collection = Collection("documents", schema)

# Insert
collection.insert([
    ["Redis tutorial", "PostgreSQL guide"],
    [[0.1, 0.2, ...], [0.3, 0.1, ...]]
])

# Create index
collection.create_index("embedding", {
    "index_type": "HNSW",
    "metric_type": "COSINE",
    "params": {"M": 16, "efConstruction": 200}
})

# Search
results = collection.search(
    data=[[0.1, 0.2, ...]],
    anns_field="embedding",
    param={"metric_type": "COSINE", "ef": 100},
    limit=10,
    output_fields=["content"]
)
```

**Đặc điểm:**
- ✅ Kubernetes-native, horizontal scaling
- ✅ Supports tỷ vectors
- ✅ Nhiều index types (HNSW, IVF-PQ, DiskANN)
- ✅ Streaming insert với Kafka/Pulsar integration
- ❌ Phức tạp hơn khi self-host (nhiều components: etcd, MinIO, Pulsar)
- **Phù hợp:** Large-scale production (>100M vectors), enterprise

### Chroma — Lightweight, Dev-Friendly

```python
import chromadb
from chromadb.utils import embedding_functions

client = chromadb.Client()  # in-memory
# hoặc
client = chromadb.PersistentClient(path="/data/chroma")  # persistent
# hoặc
client = chromadb.HttpClient(host="localhost", port=8000)  # server mode

openai_ef = embedding_functions.OpenAIEmbeddingFunction(
    api_key="YOUR_API_KEY",
    model_name="text-embedding-3-small"
)

collection = client.create_collection("docs", embedding_function=openai_ef)

# Add documents (auto-embed)
collection.add(
    documents=["Redis is fast", "PostgreSQL is reliable"],
    metadatas=[{"source": "blog"}, {"source": "docs"}],
    ids=["doc1", "doc2"]
)

# Query (auto-embed query)
results = collection.query(
    query_texts=["in-memory database"],
    n_results=2
)
```

**Đặc điểm:**
- ✅ Cực đơn giản, Python-first
- ✅ In-memory hoặc persistent
- ✅ Tích hợp tốt với LangChain, LlamaIndex
- ❌ Không scale tốt (không phải production-grade)
- ❌ Ít features hơn so với Weaviate, Milvus
- **Phù hợp:** Prototype, local dev, learning, hackathon

### Qdrant — Rust-based, Fast & Efficient

```python
from qdrant_client import QdrantClient
from qdrant_client.models import Distance, VectorParams, PointStruct

client = QdrantClient(host="localhost", port=6333)

# Create collection
client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
)

# Insert points
client.upsert(
    collection_name="documents",
    points=[
        PointStruct(id=1, vector=[0.1, 0.2, ...], payload={"text": "Redis tutorial"}),
        PointStruct(id=2, vector=[0.3, 0.1, ...], payload={"text": "PG guide"}),
    ]
)

# Search with filter
results = client.search(
    collection_name="documents",
    query_vector=[0.1, 0.2, ...],
    query_filter={"must": [{"key": "category", "match": {"value": "database"}}]},
    limit=5
)
```

**Đặc điểm:**
- ✅ Rất nhanh (Rust implementation)
- ✅ Memory efficient hơn nhiều alternatives
- ✅ Rich filtering + payload storage
- ✅ Quantization support (tiết kiệm RAM)
- ✅ Kubernetes-ready, gRPC + REST
- ❌ Nhỏ hơn về ecosystem so với Weaviate/Milvus
- **Phù hợp:** Production self-hosted, cần performance cao + resource efficient

### pgvector — PostgreSQL Extension

```sql
-- Install extension
CREATE EXTENSION vector;

-- Table với vector column
CREATE TABLE documents (
    id BIGSERIAL PRIMARY KEY,
    content TEXT,
    source VARCHAR(255),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    embedding vector(1536)
);

-- Insert
INSERT INTO documents (content, embedding)
VALUES ('Redis is fast', '[0.1, 0.2, ...]'::vector);

-- Create HNSW index
CREATE INDEX ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Semantic search
SELECT content, source,
       1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
FROM documents
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;

-- Hybrid: vector + keyword filter
SELECT content, source,
       1 - (embedding <=> '[0.1, 0.2, ...]'::vector) AS similarity
FROM documents
WHERE source = 'blog'
  AND created_at > NOW() - INTERVAL '30 days'
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 10;

-- Operators:
-- <=>  : Cosine distance
-- <->  : Euclidean (L2) distance
-- <#>  : Negative dot product (inner product)
```

**Đặc điểm:**
- ✅ Không cần infra mới nếu đã có PostgreSQL
- ✅ ACID transactions, JOINs với existing tables
- ✅ Familiar SQL interface
- ✅ IVFFlat và HNSW index support
- ❌ Không scale tốt như dedicated Vector DBs (>10M vectors)
- ❌ Memory-intensive (HNSW index phải fit trong RAM)
- **Phù hợp:** Khi đã có PostgreSQL, dataset nhỏ/vừa (<5M vectors), cần JOIN với relational data

---

## So Sánh

| | Pinecone | Weaviate | Milvus | Chroma | Qdrant | pgvector |
|---|---|---|---|---|---|---|
| **Type** | Managed | Open Source | Open Source | Open Source | Open Source | PG Extension |
| **Scale** | Tự động | Medium-Large | Very Large | Small | Medium-Large | Small-Medium |
| **Setup** | API key | Docker/K8s | K8s | pip install | Docker/K8s | CREATE EXTENSION |
| **Hybrid Search** | ✅ | ✅ (best) | ✅ | ❌ | ✅ | ❌ (manual) |
| **Memory** | Cloud | Medium | High | Low | Low-Medium | High (HNSW) |
| **Production** | ✅ | ✅ | ✅ | ❌ | ✅ | Depends |
| **Vendor Lock** | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **SQL/JOINs** | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## Khi nào dùng pgvector vs Dedicated Vector DB?

### Dùng pgvector khi:
- Đã có PostgreSQL infrastructure
- Dataset < 5 triệu vectors
- Cần JOIN vector search với relational data
- Team quen PostgreSQL, không muốn học tool mới
- Budget/resource hạn chế
- ACID transactions là requirement

### Dùng Dedicated Vector DB khi:
- Dataset > 5-10 triệu vectors
- Cần cực nhanh (sub-10ms với 100M+ vectors)
- Hybrid search quan trọng (Weaviate)
- Multi-tenant SaaS (Weaviate, Qdrant)
- Kubernetes-native scaling (Milvus)
- Không muốn ops overhead (Pinecone)

---

## Chunking Strategy cho RAG

Cách chia documents thành chunks ảnh hưởng trực tiếp đến RAG quality:

```python
# Fixed-size chunking (đơn giản nhất, thường không tốt nhất)
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,          # characters per chunk
    chunk_overlap=50,        # overlap để không mất context ở ranh giới
    separators=["\n\n", "\n", ". ", " ", ""]
)

# Semantic chunking (tốt hơn - split ở điểm semantic boundary)
# → dùng embedding similarity để detect topic shifts

# Hierarchical: parent + child chunks
# Child chunks (nhỏ) để search chính xác
# Parent chunks (lớn) để cung cấp context đầy đủ cho LLM
```

---

## Use Cases

| Use Case | Công nghệ | Mô tả |
|----------|-----------|-------|
| **Chatbot với knowledge base** | RAG + Vector DB | LLM trả lời dựa trên docs nội bộ |
| **Semantic search** | Vector DB | Tìm theo ý nghĩa, không phải keyword |
| **Code search** | Vector DB | Tìm code tương tự, detect duplication |
| **Image similarity** | Vector DB | Tìm ảnh tương tự (CLIP embeddings) |
| **Recommendation** | Vector DB | "Users like you also liked..." |
| **Anomaly detection** | Vector DB | Outlier points xa các clusters |
| **Document deduplication** | Vector DB | Tìm near-duplicate documents |
| **Customer support** | RAG | Tự động trả lời từ knowledge base |

---

## Related Notes

- [[00-Database-MOC]] — Tổng quan toàn bộ
- [[01-Terminology]] — Thuật ngữ: Index, Replication
- [[03-Types]] — Vector DB trong landscape database
- [[05-PostgreSQL]] — pgvector extension cho PostgreSQL
- [[08-Elasticsearch]] — dense_vector, KNN search trong ES 8.0+
- [[10-Operations]] — Vận hành, monitoring Vector DB
