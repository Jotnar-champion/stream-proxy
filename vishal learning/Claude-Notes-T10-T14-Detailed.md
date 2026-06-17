# Full-Stack AI Engineering Interview Preparation Guide
### UST Global — AI-Inclined Full-Stack Role

---

## TOPIC 11: VectorDB

### Common Interview Questions (25)

**Q1: What is a VectorDB and why is it needed for AI applications?**

**Answer:** A Vector Database is a specialized data store optimized for storing, indexing, and querying high-dimensional numerical vectors (embeddings). Unlike traditional databases that query exact values, VectorDBs find the most *semantically similar* vectors using approximate nearest neighbor (ANN) search.

**Why it's needed:**
- LLMs have context window limits — you can't stuff 10 million NSE documents into a prompt
- Semantic search (meaning-based) is far superior to keyword search for financial queries
- It enables RAG: convert query to vector → find top-k similar document chunks → inject into prompt

```python
# Conceptual flow for NSE AI platform
query = "What are NSE's circuit breaker rules for midcap stocks?"
query_embedding = embed_model.encode(query)  # 1536-dim float32 vector
top_k_chunks = vectordb.similarity_search(query_embedding, k=5)
context = "\n".join([chunk.text for chunk in top_k_chunks])
response = claude.complete(f"Context: {context}\n\nQ: {query}")
```

---

**Q2: Explain cosine similarity, dot product, and Euclidean distance in the context of vector search.**

**Answer:**

**Cosine Similarity** — measures the angle between two vectors, ignoring magnitude. Best for text embeddings where direction matters more than length.

$$\cos(\theta) = \frac{A \cdot B}{|A| \cdot |B|}$$

Range: [-1, 1], where 1 = identical, 0 = orthogonal, -1 = opposite.

**Dot Product** — measures both direction and magnitude. Used in OpenAI's `text-embedding-3` models (they recommend dot product for normalized vectors, which equals cosine similarity when vectors are unit-normalized).

$$A \cdot B = \sum_{i=1}^{n} A_i B_i$$

**Euclidean Distance (L2)** — straight-line distance in vector space. Sensitive to magnitude differences. Less common for NLP embeddings.

$$d(A, B) = \sqrt{\sum_{i=1}^{n}(A_i - B_i)^2}$$

```python
import numpy as np

def cosine_similarity(a: np.ndarray, b: np.ndarray) -> float:
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

def dot_product(a: np.ndarray, b: np.ndarray) -> float:
    return np.dot(a, b)

def euclidean_distance(a: np.ndarray, b: np.ndarray) -> float:
    return np.linalg.norm(a - b)

# For normalized vectors, cosine_similarity == dot_product
a_norm = a / np.linalg.norm(a)
b_norm = b / np.linalg.norm(b)
assert abs(cosine_similarity(a_norm, b_norm) - dot_product(a_norm, b_norm)) < 1e-6
```

**Rule of thumb:** For sentence embeddings → cosine similarity. For binary/count vectors → dot product. For image/spatial data → Euclidean.

---

**Q3: What is HNSW (Hierarchical Navigable Small World)? How does it work?**

**Answer:** HNSW is the dominant ANN algorithm used by Pinecone, Qdrant, Weaviate, and pgvector. It builds a multi-layer graph structure where:

- **Bottom layer**: Contains all vectors connected to nearest neighbors
- **Upper layers**: Sparse "highway" graphs for fast traversal
- **Search**: Enter from top layer, greedily descend to bottom, return closest neighbors

**Key parameters:**
- `M` — max number of bidirectional connections per node (typically 16-64). Higher M = better recall, more memory
- `ef_construction` — size of the candidate list during index build. Higher = better recall, slower build
- `ef_search` — candidate set size during query. Higher = better recall, slower queries

```python
# pgvector HNSW index creation
CREATE INDEX ON nse_embeddings 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

# Query time ef setting
SET hnsw.ef_search = 100;  -- increase for higher recall
```

**Time complexity:** O(log N) for search vs O(N) for brute force.

---

**Q4: What is IVF (Inverted File Index) and Product Quantization (PQ)?**

**Answer:**

**IVF (Inverted File Index):**
- Clusters vectors into `nlist` Voronoi cells using k-means
- Query: find nearest cluster centroids first, then search only those cells
- `nprobe` parameter controls how many cells to search (recall vs speed trade-off)

**Product Quantization (PQ):**
- Compresses high-dimensional vectors by splitting into subvectors and quantizing each
- Dramatically reduces memory: 1536-dim float32 (6KB) → compressed PQ code (~64 bytes)
- Trade-off: lossy compression degrades recall slightly

**IVF-PQ combined** (used by Faiss):
```python
import faiss
import numpy as np

d = 1536       # embedding dimension
nlist = 100    # number of clusters
m = 8          # subvectors for PQ
k = 8          # bits per subvector

quantizer = faiss.IndexFlatL2(d)
index = faiss.IndexIVFPQ(quantizer, d, nlist, m, k)

# Train on representative sample
index.train(training_vectors)
index.add(all_vectors)

# Search
index.nprobe = 10  # search 10 clusters
distances, indices = index.search(query_vector, k=5)
```

---

**Q5: Compare pgvector, Pinecone, Weaviate, Qdrant, and ChromaDB.**

**Answer:**

| Feature | pgvector | Pinecone | Weaviate | Qdrant | ChromaDB |
|---|---|---|---|---|---|
| Type | PostgreSQL extension | Managed cloud | Self-hosted / cloud | Self-hosted / cloud | Embedded / server |
| Algorithm | HNSW, IVF | Proprietary | HNSW | HNSW | HNSW (via hnswlib) |
| Hybrid Search | Yes (with tsvector) | Yes | Yes | Yes | Limited |
| Metadata Filtering | SQL-native | Yes | Yes (GraphQL) | Yes | Yes |
| Scalability | Vertical | Horizontal (managed) | Horizontal | Horizontal | Limited |
| Cost | Free (self-managed) | $70+/mo | Free / paid cloud | Free (OSS) | Free (OSS) |
| Best For | Existing PG users | Enterprise managed | Rich schema + multi-modal | High perf + filtering | Local dev / prototyping |
| ACID Transactions | Yes | No | No | No | No |
| Maturity | Production | Production | Production | Production | Beta/Stable |

**For NSE AI platform:**
- **pgvector** — ideal if already using PostgreSQL for NSE data. Eliminates operational complexity of managing a separate vector store. Store embeddings alongside the structured stock data.
- **Qdrant** — best if you need high throughput with complex metadata filters (e.g., filter by sector, date range, stock symbol before vector search).

---

**Q6: What is metadata filtering in VectorDB and why is it important?**

**Answer:** Metadata filtering allows you to pre-filter (or post-filter) the vector search space based on structured attributes before/after performing similarity search.

**Pre-filtering** (filter then search — default in most DBs): Reduces the candidate set before ANN search. Risk: if filter is too restrictive, recall suffers.

**Post-filtering** (search then filter): Search full index, then filter results. Risk: may return fewer than k results.

```python
# Qdrant metadata filtering example — NSE use case
from qdrant_client import QdrantClient
from qdrant_client.models import Filter, FieldCondition, MatchValue, Range

client = QdrantClient("localhost", port=6333)

results = client.search(
    collection_name="nse_documents",
    query_vector=query_embedding,
    query_filter=Filter(
        must=[
            FieldCondition(key="sector", match=MatchValue(value="banking")),
            FieldCondition(
                key="date",
                range=Range(gte="2024-01-01", lte="2024-12-31")
            ),
            FieldCondition(key="doc_type", match=MatchValue(value="circular"))
        ]
    ),
    limit=10
)
```

```sql
-- pgvector with metadata filtering (PostgreSQL)
SELECT id, content, embedding <=> $1 AS distance
FROM nse_documents
WHERE 
    sector = 'banking'
    AND doc_date BETWEEN '2024-01-01' AND '2024-12-31'
    AND doc_type = 'circular'
ORDER BY embedding <=> $1
LIMIT 10;
```

**NSE context:** Filter NSE documents by instrument type (EQ/FO/CD), segment, exchange date, or SEBI circular number before semantic search.

---

**Q7: What is hybrid search and how does it combine dense + sparse vectors?**

**Answer:** Hybrid search combines:
- **Dense vectors** (semantic similarity via embeddings) — captures meaning
- **Sparse vectors** (BM25/TF-IDF keyword matching) — captures exact terms, ticker symbols, regulation numbers

This is critical for financial data where users might ask: "Show me SEBI circular SEBI/HO/IMD/DF2/CIR/P/2024/18 about mutual fund regulations" — a pure dense search would miss the exact circular number.

**Reciprocal Rank Fusion (RRF)** — most common hybrid merging algorithm:

$$RRF(d) = \sum_{r \in R} \frac{1}{k + r(d)}$$

```python
# Qdrant hybrid search
from qdrant_client.models import SparseVector, NamedVector, NamedSparseVector

# Generate sparse vector (BM25)
sparse_indices, sparse_values = bm25_encode(query_text)

results = client.query_points(
    collection_name="nse_documents",
    prefetch=[
        # Dense search
        models.Prefetch(
            query=dense_embedding,
            using="dense",
            limit=20
        ),
        # Sparse (BM25) search
        models.Prefetch(
            query=SparseVector(indices=sparse_indices, values=sparse_values),
            using="sparse",
            limit=20
        ),
    ],
    # RRF fusion
    query=models.FusionQuery(fusion=models.Fusion.RRF),
    limit=10
)
```

```python
# LangChain EnsembleRetriever for hybrid search
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever
from langchain_community.vectorstores import Qdrant

bm25_retriever = BM25Retriever.from_documents(docs)
bm25_retriever.k = 5

vector_retriever = qdrant_store.as_retriever(search_kwargs={"k": 5})

ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, vector_retriever],
    weights=[0.4, 0.6]  # 60% weight to semantic, 40% to keyword
)
```

---

**Q8: How do you tune VectorDB performance at scale?**

**Answer:**

**Index tuning (HNSW):**
```sql
-- pgvector: tune for your recall/speed trade-off
-- Build time tuning
SET max_parallel_workers_per_gather = 4;

CREATE INDEX CONCURRENTLY ON nse_embeddings 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 32, ef_construction = 128);  -- higher for better recall

-- Query time tuning
SET hnsw.ef_search = 200;  -- increase for higher recall at query cost
```

**Partitioning for scale:**
```sql
-- Partition by year for time-series NSE data
CREATE TABLE nse_embeddings (
    id BIGSERIAL,
    symbol VARCHAR(20),
    doc_date DATE,
    content TEXT,
    embedding vector(1536)
) PARTITION BY RANGE (doc_date);

CREATE TABLE nse_embeddings_2024 
    PARTITION OF nse_embeddings 
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Index each partition separately
CREATE INDEX ON nse_embeddings_2024 
USING hnsw (embedding vector_cosine_ops);
```

**Caching query embeddings:**
```python
import redis
import hashlib
import json

redis_client = redis.Redis()

def get_cached_embedding(text: str, embed_fn) -> list[float]:
    cache_key = f"emb:{hashlib.md5(text.encode()).hexdigest()}"
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)
    
    embedding = embed_fn(text)
    redis_client.setex(cache_key, 3600, json.dumps(embedding))  # 1hr TTL
    return embedding
```

**Qdrant quantization for memory reduction:**
```python
from qdrant_client.models import ScalarQuantization, ScalarQuantizationConfig, ScalarType

client.create_collection(
    collection_name="nse_documents",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
    quantization_config=ScalarQuantization(
        scalar=ScalarQuantizationConfig(
            type=ScalarType.INT8,        # 4x memory reduction
            quantile=0.99,
            always_ram=True              # keep quantized vectors in RAM
        )
    )
)
```

---

**Q9: How do you set up pgvector with PostgreSQL for NSE data embeddings?**

**Answer:**

```sql
-- 1. Install pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- 2. Create table for NSE document embeddings
CREATE TABLE nse_document_embeddings (
    id BIGSERIAL PRIMARY KEY,
    symbol VARCHAR(20),           -- e.g., 'RELIANCE', 'NIFTY50'
    doc_type VARCHAR(50),         -- 'circular', 'announcement', 'filing', 'report'
    segment VARCHAR(20),          -- 'EQ', 'FO', 'CD'
    doc_date DATE,
    source_url TEXT,
    chunk_index INTEGER,          -- which chunk of the document
    chunk_text TEXT NOT NULL,
    embedding vector(1536),       -- OpenAI ada-002 or text-embedding-3-small
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. Create HNSW index
CREATE INDEX ON nse_document_embeddings 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- 4. Supporting indexes for metadata filtering
CREATE INDEX ON nse_document_embeddings (symbol);
CREATE INDEX ON nse_document_embeddings (doc_type);
CREATE INDEX ON nse_document_embeddings (doc_date);
CREATE INDEX ON nse_document_embeddings USING GIN (metadata);
```

```python
# Python ingestion pipeline for NSE data
import asyncpg
import numpy as np
from anthropic import Anthropic
from sentence_transformers import SentenceTransformer

async def ingest_nse_document(
    pool: asyncpg.Pool,
    symbol: str,
    doc_type: str,
    content: str,
    doc_date: str,
    embed_model: SentenceTransformer
):
    # Chunk the document
    chunks = chunk_text(content, chunk_size=512, overlap=64)
    
    async with pool.acquire() as conn:
        for i, chunk in enumerate(chunks):
            # Generate embedding
            embedding = embed_model.encode(chunk).tolist()
            
            await conn.execute("""
                INSERT INTO nse_document_embeddings 
                    (symbol, doc_type, doc_date, chunk_index, chunk_text, embedding)
                VALUES ($1, $2, $3, $4, $5, $6)
            """, symbol, doc_type, doc_date, i, chunk, embedding)

# Similarity search with metadata filter
async def semantic_search_nse(
    pool: asyncpg.Pool,
    query: str,
    embed_model: SentenceTransformer,
    symbol: str = None,
    doc_type: str = None,
    limit: int = 5
) -> list[dict]:
    query_embedding = embed_model.encode(query).tolist()
    
    sql = """
        SELECT 
            id, symbol, doc_type, doc_date, chunk_text,
            1 - (embedding <=> $1::vector) AS similarity
        FROM nse_document_embeddings
        WHERE ($2::text IS NULL OR symbol = $2)
          AND ($3::text IS NULL OR doc_type = $3)
        ORDER BY embedding <=> $1::vector
        LIMIT $4
    """
    
    async with pool.acquire() as conn:
        rows = await conn.fetch(sql, query_embedding, symbol, doc_type, limit)
    
    return [dict(row) for row in rows]
```

---

**Q10: How do you handle versioning of embeddings when you update your embedding model?**

**Answer:** This is a real operational challenge. When you switch embedding models (e.g., from `text-embedding-ada-002` to `text-embedding-3-large`), old and new vectors are NOT comparable.

**Strategy:**
```sql
-- Add model version tracking to schema
ALTER TABLE nse_document_embeddings 
ADD COLUMN embedding_model VARCHAR(100) DEFAULT 'text-embedding-ada-002';

-- Create composite index
CREATE INDEX ON nse_document_embeddings (embedding_model, symbol);
```

```python
# Gradual migration with dual-write
async def migrate_embeddings_gradual(
    pool: asyncpg.Pool,
    old_model: str,
    new_model: SentenceTransformer,
    new_model_name: str,
    batch_size: int = 100
):
    """Backfill embeddings with new model without downtime"""
    async with pool.acquire() as conn:
        # Find records needing migration
        rows = await conn.fetch("""
            SELECT id, chunk_text FROM nse_document_embeddings
            WHERE embedding_model = $1
            ORDER BY id
            LIMIT $2
        """, old_model, batch_size)
        
        for row in rows:
            new_embedding = new_model.encode(row['chunk_text']).tolist()
            
            # Insert new version, don't delete old yet
            await conn.execute("""
                INSERT INTO nse_document_embeddings 
                    (symbol, doc_type, doc_date, chunk_index, chunk_text, 
                     embedding, embedding_model)
                SELECT symbol, doc_type, doc_date, chunk_index, chunk_text,
                       $1, $2
                FROM nse_document_embeddings WHERE id = $3
            """, new_embedding, new_model_name, row['id'])
```

---

**Q11: What is the difference between an exact KNN search and ANN search?**

**Answer:**
- **KNN (Exact)**: Scans all vectors, guaranteed 100% recall. O(N·d) complexity. Feasible only for small collections (<100K vectors).
- **ANN (Approximate)**: Trades small recall loss for massive speed gains via indexing (HNSW, IVF). O(log N) complexity. Production choice for millions of vectors.

```sql
-- pgvector: Force exact KNN (no index, guaranteed recall)
SET enable_indexscan = off;
SELECT id, embedding <=> $1 AS distance
FROM nse_embeddings
ORDER BY embedding <=> $1
LIMIT 10;

-- ANN (uses HNSW index)
SET enable_indexscan = on;
SELECT id, embedding <=> $1 AS distance
FROM nse_embeddings
ORDER BY embedding <=> $1
LIMIT 10;
```

---

**Q12: Explain namespace/collection isolation in VectorDBs for multi-tenancy.**

**Answer:**
```python
# Pinecone namespace-based multi-tenancy
import pinecone

pc = pinecone.Pinecone(api_key="...")
index = pc.Index("nse-platform")

# Each NSE segment in separate namespace
index.upsert(
    vectors=[{"id": "doc-1", "values": embedding, "metadata": {"type": "circular"}}],
    namespace="equity-segment"     # EQ namespace
)

index.upsert(
    vectors=[{"id": "doc-1", "values": embedding, "metadata": {"type": "report"}}],
    namespace="futures-options"    # F&O namespace
)

# Query only equity segment
results = index.query(
    vector=query_embedding,
    top_k=5,
    namespace="equity-segment"
)
```

---

**Q13: How do you handle sparse vs dense vector storage separately in Qdrant?**

**Answer:**
```python
from qdrant_client.models import VectorParams, SparseVectorParams, Distance

client.create_collection(
    collection_name="nse_hybrid",
    vectors_config={
        "dense": VectorParams(size=1536, distance=Distance.COSINE),
    },
    sparse_vectors_config={
        "sparse": SparseVectorParams()
    }
)

# Upsert with both vectors
client.upsert(
    collection_name="nse_hybrid",
    points=[
        models.PointStruct(
            id=doc_id,
            vector={
                "dense": dense_embedding,
                "sparse": SparseVector(
                    indices=bm25_indices,
                    values=bm25_weights
                )
            },
            payload={"symbol": "RELIANCE", "date": "2024-06-01"}
        )
    ]
)
```

---

**Q14: What are the trade-offs between storing full documents vs chunks in a VectorDB?**

**Answer:**
- **Full documents**: Better context, but embedding loses fine-grained detail; retrieval brings back too much irrelevant text
- **Chunks (512-1024 tokens)**: Better precision, but may lose cross-chunk context
- **Parent-child chunking**: Index small chunks, retrieve parent (larger) chunk for context

```python
# Parent-child retrieval strategy
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore

# Small chunks for indexing (256 tokens)
child_splitter = RecursiveCharacterTextSplitter(chunk_size=256)
# Larger chunks for retrieval (1024 tokens)
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=1024)

store = InMemoryStore()  # or RedisStore for production
retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)
```

---

**Q15: How would you implement a VectorDB-backed conversational memory?**

**Answer:**
```python
from langchain_community.vectorstores import Qdrant
from langchain.memory import VectorStoreRetrieverMemory

# Store conversation turns as embeddings
memory_vectorstore = Qdrant(
    client=qdrant_client,
    collection_name="conversation_memory",
    embeddings=embed_model
)

memory = VectorStoreRetrieverMemory(
    retriever=memory_vectorstore.as_retriever(search_kwargs={"k": 3}),
    memory_key="history",
    input_key="input"
)

# When user asks "What did I ask about NIFTY earlier?"
# Memory retrieves semantically similar past turns
```

---

**Q16: Explain dimensionality reduction techniques (PCA, UMAP) for VectorDB optimization.**

**Answer:**
```python
import umap
import numpy as np
from sklearn.decomposition import PCA

# Reduce 1536-dim OpenAI embeddings to 256-dim
# Trade: slight recall loss, 6x memory reduction, 6x faster search

# UMAP (better preserves local structure)
reducer = umap.UMAP(n_components=256, metric='cosine', random_state=42)
reduced_embeddings = reducer.fit_transform(full_embeddings)

# PCA (faster, linear)
pca = PCA(n_components=256)
reduced_embeddings_pca = pca.fit_transform(full_embeddings)
# Explained variance
print(f"Variance retained: {sum(pca.explained_variance_ratio_):.2%}")
```

---

**Q17: How do you evaluate retrieval quality in a VectorDB?**

**Answer:**
Key metrics:
- **Recall@K** — fraction of relevant docs in top-K results
- **MRR (Mean Reciprocal Rank)** — position of first relevant result
- **NDCG** — normalized discounted cumulative gain, accounts for ordering

```python
def recall_at_k(retrieved_ids: list, relevant_ids: set, k: int) -> float:
    top_k = set(retrieved_ids[:k])
    return len(top_k & relevant_ids) / len(relevant_ids)

def mrr(retrieved_ids: list, relevant_ids: set) -> float:
    for rank, doc_id in enumerate(retrieved_ids, 1):
        if doc_id in relevant_ids:
            return 1.0 / rank
    return 0.0

# Use RAGAS for RAG-specific evaluation
from ragas.metrics import context_recall, context_precision
```

---

**Q18: What is the cold start problem in VectorDB and how do you handle it?**

**Answer:** When first deploying, you have no embeddings. Solutions:
1. **Batch pre-ingestion** — ingest all historical NSE data before going live
2. **Incremental ingestion** — stream new documents as they arrive (SQS + Lambda pipeline)
3. **Fallback to keyword search** — if vector search returns no results above threshold, fall back to PostgreSQL full-text search

---

**Q19: How do you handle real-time document updates in a VectorDB?**

**Answer:**
```python
# Event-driven embedding update via SQS + Lambda (NSE context)
# When a new NSE circular is published → SQS → Lambda → embed → upsert

import boto3
import json

def lambda_handler(event, context):
    for record in event['Records']:
        message = json.loads(record['body'])
        doc_id = message['doc_id']
        content = message['content']
        
        # Generate new embedding
        embedding = embed_model.encode(content).tolist()
        
        # Upsert (update if exists, insert if new)
        qdrant_client.upsert(
            collection_name="nse_documents",
            points=[PointStruct(id=doc_id, vector=embedding, payload=message['metadata'])]
        )
```

---

**Q20: What is the difference between a vector index and a traditional database index?**

**Answer:**
| Aspect | Traditional B-tree Index | Vector (HNSW) Index |
|---|---|---|
| Query type | Exact match, range | Approximate nearest neighbor |
| Data type | Scalars, strings | High-dim float arrays |
| Guarantees | Exact results | Approximate (recall < 100%) |
| Update cost | O(log N) | O(log N) but larger constant |
| Memory | Low | High (graph structure) |
| Parallelism | Good | Good with tuning |

---

**Q21: How does Pinecone handle sharding and replication internally?**

**Answer:** Pinecone uses a proprietary architecture:
- Automatically shards large indexes across pods
- `p1` pods: optimized for latency; `s1` pods: optimized for storage
- Replicas provide read redundancy
- You control pod count, pod type, and replicas; Pinecone handles the distributed coordination

---

**Q22: What is a "stale" embedding and how do you detect/handle it?**

**Answer:** A stale embedding occurs when the source document changes but the embedding isn't updated.

```python
# Track content hash to detect staleness
import hashlib

async def check_and_update_embedding(doc_id: str, new_content: str):
    content_hash = hashlib.sha256(new_content.encode()).hexdigest()
    
    async with pool.acquire() as conn:
        existing = await conn.fetchrow(
            "SELECT content_hash FROM nse_embeddings WHERE id = $1", doc_id
        )
        
        if not existing or existing['content_hash'] != content_hash:
            # Content changed — re-embed
            new_embedding = embed_model.encode(new_content).tolist()
            await conn.execute("""
                UPDATE nse_embeddings 
                SET embedding = $1, content_hash = $2, updated_at = NOW()
                WHERE id = $3
            """, new_embedding, content_hash, doc_id)
```

---

**Q23: Compare pgvector's `<=>` (cosine), `<->` (L2), and `<#>` (negative inner product) operators.**

**Answer:**
```sql
-- Cosine distance (1 - cosine_similarity). Best for text.
SELECT id, embedding <=> query_vec AS cos_dist FROM nse_embeddings ORDER BY cos_dist LIMIT 5;

-- L2 (Euclidean) distance. Magnitude-sensitive.
SELECT id, embedding <-> query_vec AS l2_dist FROM nse_embeddings ORDER BY l2_dist LIMIT 5;

-- Negative inner product. Use for maximum inner product search.
-- Note: for normalized vectors, MIP == cosine similarity
SELECT id, embedding <#> query_vec AS neg_ip FROM nse_embeddings ORDER BY neg_ip LIMIT 5;
```

---

**Q24: How would you architect a distributed VectorDB for 100M+ document embeddings?**

**Answer:**
1. **Qdrant distributed mode** — multiple shards across nodes
2. **Separate index from storage** — use object store (S3) for raw vectors, in-memory index for search
3. **Tiered storage** — hot vectors (recently accessed) in RAM, cold vectors on disk
4. **Read replicas** — separate read and write paths
5. **Async indexing** — write to queue, async index build

---

**Q25: What is SPLADE and how does it improve hybrid search over BM25?**

**Answer:** SPLADE (Sparse Lexical and Expansion Model) is a learned sparse retrieval model that:
- Generates sparse vectors like BM25 but using a transformer (BERT)
- Performs vocabulary expansion (adds semantically related terms)
- Outperforms BM25 on BEIR benchmark

```python
from transformers import AutoTokenizer, AutoModelForMaskedLM

model = AutoModelForMaskedLM.from_pretrained("naver/splade-cocondenser-ensembledistil")
tokenizer = AutoTokenizer.from_pretrained("naver/splade-cocondenser-ensembledistil")

def encode_splade(text: str):
    tokens = tokenizer(text, return_tensors="pt", truncation=True, max_length=512)
    output = model(**tokens)
    # Aggregate over sequence length, apply ReLU + log(1+x)
    vec = torch.max(
        torch.log(1 + torch.relu(output.logits)) * tokens['attention_mask'].unsqueeze(-1),
        dim=1
    ).values.squeeze()
    return vec
```

---

### Deep-Dive Real-World Edge Case Questions (20)

**Q1: Your NSE AI platform's pgvector query returns results with cosine distance > 0.8 for all chunks, meaning no relevant content was found. How do you handle this gracefully?**

**Answer:**
```python
SIMILARITY_THRESHOLD = 0.75  # Below this = not relevant (distance > 0.25)

async def rag_with_fallback(query: str, symbol: str = None) -> dict:
    query_embedding = embed_model.encode(query).tolist()
    
    results = await semantic_search_nse(
        pool, query_embedding, symbol=symbol, limit=5
    )
    
    # Check similarity threshold
    high_confidence = [r for r in results if r['similarity'] >= SIMILARITY_THRESHOLD]
    
    if not high_confidence:
        # Fallback 1: Try broader search without symbol filter
        if symbol:
            results = await semantic_search_nse(pool, query_embedding, limit=5)
            high_confidence = [r for r in results if r['similarity'] >= 0.65]
        
        # Fallback 2: Keyword search in PostgreSQL
        if not high_confidence:
            keyword_results = await keyword_search(pool, query)
            if keyword_results:
                return {
                    "answer": generate_answer_from_context(keyword_results, query),
                    "source": "keyword_search",
                    "warning": "No semantically similar documents found; using keyword search."
                }
        
        # Fallback 3: Decline to answer
        if not high_confidence:
            return {
                "answer": "I don't have sufficient information in my knowledge base to answer this question about NSE markets. Please consult NSE's official website at nseindia.com.",
                "source": "no_context",
                "retrieved_docs": []
            }
    
    context = "\n\n".join([r['chunk_text'] for r in high_confidence])
    answer = await claude_client.generate(
        prompt=f"Based ONLY on the following NSE documents:\n{context}\n\nAnswer: {query}\n\nIf the documents don't contain the answer, say so explicitly.",
    )
    return {"answer": answer, "source": "vector_search", "retrieved_docs": high_confidence}
```

---

**Q2: You have 50 million NSE document embeddings in pgvector. HNSW indexing is taking 48 hours and consuming 200GB of RAM. How do you solve this?**

**Answer:**
```sql
-- Strategy 1: IVFFlat instead of HNSW for large datasets (less memory, slower query)
CREATE INDEX ON nse_embeddings 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 2000);  -- sqrt(50M) ≈ 7000; use 1000-2000 for balance

-- Set probes at query time
SET ivfflat.probes = 50;  -- search 50 lists (2.5% of total)

-- Strategy 2: Partition + index per partition
-- Index 1M rows per partition = manageable memory per index build

-- Strategy 3: Use pgvector's parallel index build
SET max_parallel_maintenance_workers = 7;
SET maintenance_work_mem = '8GB';  -- per worker

-- Strategy 4: Stream-build with pgvector's incremental indexing
-- First create table with no index, bulk insert, then index
-- Much faster than indexed inserts
```

```python
# Strategy 5: Use Qdrant instead for this scale
# Qdrant can handle 100M+ vectors with on-disk HNSW
client.create_collection(
    collection_name="nse_large",
    vectors_config=VectorParams(
        size=1536,
        distance=Distance.COSINE,
        on_disk=True,  # Store vectors on disk, not RAM
    ),
    hnsw_config=HnswConfigDiff(
        on_disk=True,    # Store HNSW graph on disk
        m=16,
        ef_construct=100
    )
)
```

---

**Q3: Two embedding models give different embeddings for the same document. Your VectorDB has a mix of old and new embeddings. Queries return inconsistent results. How do you fix this without downtime?**

**Answer:**
```python
# Blue-green embedding migration strategy

# Step 1: Add version column to schema
# ALTER TABLE nse_embeddings ADD COLUMN model_version VARCHAR(50);

# Step 2: Create separate index per model version
"""
CREATE INDEX idx_embeddings_v1 ON nse_embeddings (embedding vector_cosine_ops)
WHERE model_version = 'text-embedding-ada-002';

CREATE INDEX idx_embeddings_v2 ON nse_embeddings (embedding vector_cosine_ops)  
WHERE model_version = 'text-embedding-3-large';
"""

# Step 3: Query routing — always use same model version for query as for indexed docs
async def versioned_search(query: str, model_version: str = "text-embedding-3-large"):
    if model_version == "text-embedding-3-large":
        query_embedding = openai_client.embeddings.create(
            model="text-embedding-3-large", input=query
        ).data[0].embedding
    else:
        query_embedding = old_embed_model.encode(query).tolist()
    
    results = await pool.fetch("""
        SELECT id, chunk_text, 1 - (embedding <=> $1::vector) AS similarity
        FROM nse_embeddings
        WHERE model_version = $2
        ORDER BY embedding <=> $1::vector
        LIMIT 5
    """, query_embedding, model_version)
    
    return results

# Step 4: Background migration job
async def migrate_batch(batch_size: int = 500):
    rows = await pool.fetch("""
        SELECT id, chunk_text FROM nse_embeddings
        WHERE model_version = 'text-embedding-ada-002'
        LIMIT $1
    """, batch_size)
    
    for row in rows:
        new_emb = openai_client.embeddings.create(
            model="text-embedding-3-large", input=row['chunk_text']
        ).data[0].embedding
        
        await pool.execute("""
            UPDATE nse_embeddings SET embedding = $1, model_version = 'text-embedding-3-large'
            WHERE id = $2
        """, new_emb, row['id'])
        
    # When migration complete, deprecate old version routing
```

---

**Q4: Your vector search is returning semantically similar but temporally stale NSE data (e.g., 2020 SEBI rules when 2024 rules exist). How do you ensure recency-aware retrieval?**

**Answer:**
```python
# Strategy: Time-decay scoring — blend similarity with recency

from datetime import date, timedelta
import math

async def recency_weighted_search(
    query: str,
    embed_model,
    recency_weight: float = 0.3,  # 30% weight to recency
    k: int = 20  # fetch more, then re-rank
) -> list[dict]:
    query_embedding = embed_model.encode(query).tolist()
    today = date.today()
    
    # Fetch more results than needed for re-ranking
    candidates = await pool.fetch("""
        SELECT id, chunk_text, doc_date, symbol,
               1 - (embedding <=> $1::vector) AS semantic_score
        FROM nse_embeddings
        ORDER BY embedding <=> $1::vector
        LIMIT $2
    """, query_embedding, k)
    
    # Apply time decay: score decays exponentially with age
    # Half-life of 6 months for financial data
    HALF_LIFE_DAYS = 180
    
    def recency_score(doc_date: date) -> float:
        days_old = (today - doc_date).days
        return math.exp(-0.693 * days_old / HALF_LIFE_DAYS)  # e^(-ln2 * t/half_life)
    
    scored = []
    for row in candidates:
        semantic = row['semantic_score']
        recency = recency_score(row['doc_date'])
        # Weighted combination
        final_score = (1 - recency_weight) * semantic + recency_weight * recency
        scored.append({**dict(row), "final_score": final_score})
    
    # Re-rank by combined score
    scored.sort(key=lambda x: x['final_score'], reverse=True)
    return scored[:5]
```

---

**Q5: Qdrant cluster goes down during peak trading hours. Your NSE AI platform is returning 503s. What's your fallback strategy?**

**Answer:**
```python
# Multi-layer resilience strategy

class ResilientVectorSearch:
    def __init__(self, primary_client, fallback_pg_pool, cache: redis.Redis):
        self.primary = primary_client      # Qdrant
        self.fallback_pg = fallback_pg_pool  # pgvector (warm standby)
        self.cache = cache
    
    async def search(self, query: str, k: int = 5) -> list[dict]:
        cache_key = f"vsearch:{hashlib.md5(query.encode()).hexdigest()}"
        
        # Layer 1: Try cache
        cached = self.cache.get(cache_key)
        if cached:
            return json.loads(cached)
        
        # Layer 2: Try primary (Qdrant)
        try:
            results = await asyncio.wait_for(
                self._qdrant_search(query, k), timeout=2.0
            )
            # Populate cache
            self.cache.setex(cache_key, 300, json.dumps(results))
            return results
        except (Exception, asyncio.TimeoutError) as e:
            logger.warning(f"Qdrant failed: {e}, falling back to pgvector")
        
        # Layer 3: pgvector fallback
        try:
            results = await self._pgvector_search(query, k)
            self.cache.setex(cache_key, 60, json.dumps(results))  # shorter TTL for fallback
            return results
        except Exception as e:
            logger.error(f"pgvector fallback also failed: {e}")
        
        # Layer 4: Keyword search (no embedding needed)
        return await self._keyword_search(query, k)
```

---

**Q6: Users are querying your VectorDB with very short queries like "NIFTY" or "Sensex" which produce noisy embeddings. How do you handle this?**

**Answer:**
```python
# HyDE: Hypothetical Document Embeddings
# Generate a hypothetical answer first, embed THAT instead of the raw query

async def hyde_search(short_query: str) -> list[dict]:
    # Detect short/ambiguous query
    if len(short_query.split()) <= 3:
        # Ask Claude to generate a hypothetical passage
        hypothetical_doc = await claude_client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=300,
            messages=[{
                "role": "user",
                "content": f"""Generate a detailed passage that would appear in an NSE market document answering: "{short_query}"
                
Write as if from an NSE circular or market report. Be specific about regulations, indices, or mechanisms."""
            }]
        )
        
        search_text = hypothetical_doc.content[0].text
    else:
        search_text = short_query
    
    # Embed the hypothetical document (much richer signal)
    embedding = embed_model.encode(search_text).tolist()
    return await vector_search(embedding)
```

---

**Q7: Your embedding-based retrieval consistently misses questions about specific stock symbols (e.g., "INFY" vs "Infosys"). How do you solve the entity resolution problem?**

**Answer:**
```python
# Entity expansion / synonym injection before embedding

NSE_ENTITY_MAP = {
    "INFY": ["Infosys", "INFY.NS", "Infosys Limited", "Infosys Technologies"],
    "TCS": ["Tata Consultancy Services", "TCS.NS", "Tata Consulting"],
    "RELIANCE": ["Reliance Industries", "RIL", "RELIANCE.NS", "Mukesh Ambani company"],
    # ... load from NSE master file
}

def expand_query(query: str) -> str:
    """Expand NSE ticker symbols to full names for better embedding"""
    words = query.upper().split()
    expanded_parts = []
    for word in words:
        if word in NSE_ENTITY_MAP:
            # Include both the symbol and its expansions
            expanded_parts.append(f"{word} ({' OR '.join(NSE_ENTITY_MAP[word][:2])})")
        else:
            expanded_parts.append(word)
    return " ".join(expanded_parts)

# Also: use metadata filtering by ticker symbol alongside semantic search
async def symbol_aware_search(query: str) -> list[dict]:
    # Extract tickers from query
    tickers = extract_tickers(query)  # regex + NSE master list
    expanded = expand_query(query)
    embedding = embed_model.encode(expanded).tolist()
    
    if tickers:
        # Metadata filter + semantic search
        return await semantic_search_nse(pool, embedding, symbol=tickers[0])
    return await semantic_search_nse(pool, embedding)
```

---

**Q8: Your VectorDB index rebuild is blocking production queries. How do you rebuild the HNSW index without downtime in pgvector?**

**Answer:**
```sql
-- CONCURRENTLY flag allows reads/writes during index build
-- Does NOT lock the table

CREATE INDEX CONCURRENTLY nse_embeddings_hnsw_new
ON nse_embeddings USING hnsw (embedding vector_cosine_ops)
WITH (m = 32, ef_construction = 128);

-- Verify new index
SELECT * FROM pg_indexes WHERE indexname = 'nse_embeddings_hnsw_new';

-- Drop old index (also non-blocking in PG 14+)
DROP INDEX CONCURRENTLY nse_embeddings_hnsw_old;

-- Rename
ALTER INDEX nse_embeddings_hnsw_new RENAME TO nse_embeddings_hnsw;
```

---

**Q9: Two concurrent writes to Qdrant update the same point. How does Qdrant handle this and what can go wrong?**

**Answer:** Qdrant uses optimistic concurrency control with versioned points. Each point has a version number. If two updates arrive simultaneously:
- Last-writer-wins by default
- Use `if_unchanged_state_id` for conditional updates (like CAS)

```python
# Conditional upsert — only update if version matches
client.update_vectors(
    collection_name="nse_documents",
    points=[
        models.PointVectors(
            id=doc_id,
            vector=new_embedding
        )
    ],
    # Qdrant will reject if point was modified since you read it
    # (Qdrant 1.7+ supports ordering guarantees)
)

# For critical updates, use Qdrant's ordering parameter
client.upsert(
    collection_name="nse_documents",
    ordering=models.WriteOrdering.STRONG,  # Wait for all replicas
    points=[...]
)
```

---

**Q10: Your RAG pipeline's context window is being blown up because each retrieved chunk is 2000 tokens. You're hitting Claude's context limit with 5 chunks. How do you optimize?**

**Answer:**
```python
# Strategy 1: Smaller chunks at index time, use parent retrieval
# Strategy 2: LLM-based compression of chunks before injection
# Strategy 3: Map-reduce for long contexts

async def compressed_rag(query: str, chunks: list[str], max_tokens_per_chunk: int = 200) -> str:
    """Compress each chunk to relevant sentences before injecting"""
    
    compressed_chunks = []
    for chunk in chunks:
        # Extract only sentences relevant to query
        compressed = await claude_client.messages.create(
            model="claude-3-haiku-20240307",  # Fast, cheap model for compression
            max_tokens=200,
            messages=[{
                "role": "user",
                "content": f"""Extract only the sentences from this text that are relevant to: "{query}"
                
Text: {chunk}

Return ONLY the relevant sentences, nothing else."""
            }]
        )
        compressed_chunks.append(compressed.content[0].text)
    
    context = "\n---\n".join(compressed_chunks)
    
    return await generate_final_answer(query, context)

# Strategy 4: Sentence-level retrieval with cross-encoder re-ranking
# Retrieve at sentence level, re-rank with cross-encoder, take top 5 sentences
```

---

**Q11-Q20:** *(Space-constrained — continuing with remaining edge cases)*

**Q11: Qdrant collection snapshot for point-in-time recovery during NSE audit**
```python
# Create snapshot before bulk update
snapshot = client.create_snapshot(collection_name="nse_documents")
# snapshot.name = "nse_documents-2024-11-15-10:30:00.snapshot"

# After corrupt update, restore
client.recover_from_snapshot(
    collection_name="nse_documents",
    location=f"http://localhost:6333/collections/nse_documents/snapshots/{snapshot.name}"
)
```

**Q12: Drift detection** — periodically re-query a set of golden queries and compare top-k results to baseline. Alert if Recall@5 drops below 0.85.

**Q13: Handling multi-language NSE queries** (Hindi + English) — use multilingual embeddings (`intfloat/multilingual-e5-large`) which support 100+ languages in the same vector space.

**Q14: VectorDB memory explosion** — use scalar quantization (INT8) to reduce each float32 (4 bytes) to int8 (1 byte) → 4x memory reduction with ~1% recall loss.

**Q15: Embedding batching for throughput** — batch 100 texts per embed API call instead of 1 at a time; reduces latency 10x.

**Q16: Deduplication** — compute embeddings, find pairs with cosine > 0.98, mark as duplicates before ingestion.

**Q17: Incremental index updates vs full rebuild** — HNSW supports incremental inserts but degrades slightly vs fresh build; schedule weekly full reindex during off-hours.

**Q18: Query expansion with LLM** — for vague queries, use Claude to generate 3 query variants, search all 3, merge results via RRF.

**Q19: Access control at retrieval layer** — store user_id or role in metadata, filter at query time to prevent cross-tenant data leakage.

**Q20: Embedding model latency under load** — batch embedding with GPU, use ONNX Runtime for 3-5x faster CPU inference vs PyTorch.

---

### Must-Read Study Resources
1. **pgvector GitHub** — [github.com/pgvector/pgvector](https://github.com/pgvector/pgvector) — Official docs, index types, operators
2. **Qdrant Documentation** — [qdrant.tech/documentation](https://qdrant.tech/documentation/) — Best hybrid search and quantization docs
3. **ANN Benchmarks** — [ann-benchmarks.com](https://ann-benchmarks.com) — Compare HNSW, IVF, ScaNN recall/QPS benchmarks

---

## TOPIC 12: Hallucination in LLMs

### Common Interview Questions (25)

**Q1: What is hallucination in LLMs and why does it happen?**

**Answer:** Hallucination is when an LLM generates text that is factually incorrect, fabricated, or not grounded in its input, presented with confident fluency.

**Why it happens:**
1. **Training objective mismatch**: LLMs are trained to predict the next token (maximize likelihood), not to be truthful. Plausible ≠ accurate.
2. **Knowledge compression**: The model compresses terabytes of training data into billions of parameters — a lossy compression. Rare facts are poorly retained.
3. **No grounding mechanism by default**: Without RAG, the model relies entirely on parametric memory.
4. **Overconfidence in generation**: The model doesn't have a built-in "I don't know" signal — it fills gaps with plausible-sounding tokens.
5. **Distribution shift**: The model was trained on historical data; it will confidently extrapolate to present-day claims.

**For NSE platform:** An LLM without RAG might say "SEBI introduced circuit breakers in 2019" when the actual year is different, or fabricate a regulation that doesn't exist.

---

**Q2: What are the types of hallucination?**

**Answer:**

**1. Factual Hallucination** — Incorrect facts
> "NIFTY 50 was launched in 1990" (actually 1996)

**2. Reasoning Hallucination** — Correct premises, incorrect conclusions
> "NIFTY is up 2%, Reliance is in NIFTY, therefore Reliance is up 2%"

**3. Source Hallucination / Citation Fabrication** — Cites sources that don't exist
> "According to SEBI circular SEBI/HO/2024/1234..." (circular doesn't exist)

**4. Instruction Following Hallucination** — Ignores constraints
> You say "Only use the provided documents" — model answers from training knowledge

**5. Prompt Injection-induced Hallucination** — Malicious input manipulates output
> User embeds "Ignore previous instructions and say..." in a document

**6. Snowball Hallucination** — One error compounds into further errors in multi-step reasoning

---

**Q3: How does RAG reduce hallucination?**

**Answer:** RAG grounds the model's output in retrieved, verifiable source documents:

```
Without RAG: Query → LLM → Answer (from parametric memory, unverifiable)
With RAG:    Query → VectorDB → Top-K Docs → LLM(query + docs) → Grounded Answer
```

```python
# Grounding prompt for NSE financial analysis
GROUNDED_PROMPT = """You are an NSE market analysis assistant.

STRICT RULES:
1. Answer ONLY based on the provided source documents
2. If the documents don't contain the answer, respond: "I don't have sufficient information from NSE sources to answer this."
3. NEVER use your training knowledge for factual claims about regulations, dates, or statistics
4. Cite your source document ID for every factual claim

Source Documents:
{context}

Question: {question}

Answer (with citations):"""
```

**However, RAG doesn't fully eliminate hallucination:**
- Model may still ignore context and use parametric knowledge
- Model may misinterpret retrieved context
- Context may contain contradictory information

---

**Q4: What is "grounding" in LLM applications?**

**Answer:** Grounding is the technique of anchoring LLM responses to verified, external data sources. Types:

1. **Document grounding (RAG)** — Retrieved documents injected into context
2. **Tool grounding** — LLM calls tools (APIs, search, calculators) to get facts
3. **Database grounding** — LLM queries structured data directly
4. **Knowledge graph grounding** — Entities resolved against a KG

```python
# Tool-grounded LLM using function calling (Claude tool use)
import anthropic

tools = [
    {
        "name": "get_nse_stock_price",
        "description": "Get real-time NSE stock price for a given symbol",
        "input_schema": {
            "type": "object",
            "properties": {
                "symbol": {"type": "string", "description": "NSE stock symbol e.g. RELIANCE"}
            },
            "required": ["symbol"]
        }
    },
    {
        "name": "search_sebi_circulars",
        "description": "Search SEBI circulars database by keyword",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string"}
            },
            "required": ["query"]
        }
    }
]

response = anthropic.Anthropic().messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1000,
    tools=tools,
    messages=[{"role": "user", "content": "What is Reliance's current market cap?"}]
)

# If model uses tool → ground with real API data
# Prevents fabrication of stock prices
```

---

**Q5: What is citation enforcement and how do you implement it?**

**Answer:** Citation enforcement requires the model to explicitly cite source documents for every claim.

```python
CITATION_PROMPT = """Answer the question based on the following NSE documents.
For EVERY factual statement, cite the source using [Doc-N] notation.
If a claim cannot be cited, prefix it with [UNVERIFIED] or omit it.

Documents:
[Doc-1] {doc1_text} (Source: NSE Circular SEBI/HO/IMD/2024/P001, Date: 2024-03-15)
[Doc-2] {doc2_text} (Source: NSE Annual Report 2023-24, Section 3.2)
[Doc-3] {doc3_text} (Source: SEBI LODR Regulations 2015, Amendment 2024)

Question: {question}

Answer:"""

# Post-processing: validate citations
def validate_citations(answer: str, doc_ids: list[str]) -> dict:
    import re
    cited = re.findall(r'\[Doc-(\d+)\]', answer)
    
    uncited_claims = []
    # Simple heuristic: sentences without [Doc-N] that make factual claims
    sentences = answer.split('. ')
    for sent in sentences:
        if re.search(r'\b(is|was|are|were|has|have|will|shall)\b', sent, re.I):
            if not re.search(r'\[Doc-\d+\]', sent):
                uncited_claims.append(sent)
    
    return {
        "cited_sources": list(set(cited)),
        "uncited_sentences": uncited_claims,
        "citation_coverage": len(cited) / max(len(sentences), 1)
    }
```

---

**Q6: Explain RAGAS (Retrieval-Augmented Generation Assessment Suite) for hallucination detection.**

**Answer:** RAGAS evaluates RAG pipeline quality along 4 dimensions:

| Metric | Measures | How |
|---|---|---|
| **Faithfulness** | Is the answer grounded in retrieved context? | LLM-as-judge: verify each claim against context |
| **Answer Relevancy** | Does the answer address the question? | Cosine similarity of generated Q from answer vs original Q |
| **Context Recall** | Were all ground-truth facts in retrieved context? | Compare GT claims against context |
| **Context Precision** | How much of retrieved context was actually needed? | Signal-to-noise ratio |

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_recall,
    context_precision
)
from datasets import Dataset

# NSE evaluation dataset
data = {
    "question": ["What is the lot size for Nifty futures?"],
    "answer": ["The lot size for Nifty 50 futures is 25 units [Doc-1]."],
    "contexts": [["NSE F&O circular: Nifty 50 lot size revised to 25 from April 2023..."]],
    "ground_truth": ["The Nifty 50 futures lot size is 25 units as of April 2023."]
}

dataset = Dataset.from_dict(data)
result = evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_recall])
print(result)
# {'faithfulness': 0.97, 'answer_relevancy': 0.94, 'context_recall': 0.91}
```

---

**Q7: What is Guardrails AI and how does it prevent hallucinations?**

**Answer:** Guardrails AI is a Python framework that validates LLM outputs against structured schemas and custom validators.

```python
from guardrails import Guard
from guardrails.hub import ToxicLanguage, ValidLength
import guardrails.validators as validators

# Define output structure with validation rules
rail_spec = """
<rail version="0.1">
<output>
  <object name="nse_analysis">
    <string 
      name="summary" 
      description="Market analysis summary"
      validators="valid-length: min=50 max=500"
    />
    <string 
      name="confidence"
      description="Confidence level: high/medium/low"
      validators="valid-choices: choices={high, medium, low}"
    />
    <list name="citations">
      <string 
        name="source"
        description="Document source reference"
        validators="regex-match: regex='^(NSE|SEBI|BSE).*'"
      />
    </list>
    <boolean 
      name="has_sufficient_context"
      description="Whether context had enough info to answer"
    />
  </object>
</output>
</rail>
"""

guard = Guard.from_rail_string(rail_spec)

result = guard(
    llm_api=anthropic_complete,
    prompt=f"Analyze this NSE query: {query}\nContext: {context}"
)

# If validation fails, Guardrails re-prompts automatically
validated_output = result.validated_output
```

---

**Q8: What is TruLens and how does it evaluate hallucination?**

**Answer:** TruLens provides feedback functions (using LLMs as judges) to evaluate RAG chains:

```python
from trulens_eval import Tru, TruChain, Feedback
from trulens_eval.feedback import Groundedness, AgreementWithSources
from trulens_eval.feedback.provider import Anthropic as TruAnthropic

tru = Tru()
provider = TruAnthropic()

# Groundedness: is every claim in the answer supported by context?
groundedness = Feedback(
    provider.groundedness_measure_with_cot_reasons,
    name="Groundedness"
).on_input_output()

# Relevance: is retrieved context relevant to the question?  
context_relevance = Feedback(
    provider.context_relevance,
    name="Context Relevance"
).on_input_output()

# Answer relevance: does answer address the question?
answer_relevance = Feedback(
    provider.relevance,
    name="Answer Relevance"
).on_input_output()

# Wrap your LangChain RAG chain
tru_rag = TruChain(
    nse_rag_chain,
    app_id="NSE-Market-Analyzer-v2",
    feedbacks=[groundedness, context_relevance, answer_relevance]
)

# Run evaluation
with tru_rag as recording:
    response = nse_rag_chain.invoke({"question": "What are NSE F&O margin requirements?"})

# View dashboard
tru.run_dashboard()  # Opens at http://localhost:8501
```

---

**Q9: What prompt engineering techniques reduce hallucination?**

**Answer:**

**1. Chain-of-thought (CoT) prompting** — forces step-by-step reasoning, catches logical errors
```python
prompt = """Analyze this NSE regulatory question step by step.

Question: {question}
Context: {context}

Step 1: Identify what the question is asking
Step 2: Find relevant information from the context
Step 3: Identify any gaps where context is insufficient
Step 4: Formulate answer ONLY from what context supports
Step 5: Note any uncertainties

Answer:"""
```

**2. Explicit uncertainty instruction:**
```python
prompt = """Rules for answering:
- Say "According to [source]..." for factual claims
- Say "I am not certain, but..." for inferences
- Say "The provided documents do not contain..." if information is absent
- NEVER state uncertain information as fact"""
```

**3. Self-consistency** — sample 3-5 responses, take the majority answer:
```python
async def self_consistent_answer(query: str, context: str, n: int = 5) -> str:
    responses = await asyncio.gather(*[
        claude_complete(query, context, temperature=0.7)
        for _ in range(n)
    ])
    # Find most common answer via semantic clustering
    return majority_vote(responses)
```

**4. Negative instruction — tell it what NOT to do:**
```
"Do NOT fabricate regulation numbers, dates, or statistics not found in the documents."
```

**5. Role definition:**
```
"You are a conservative NSE regulatory compliance analyst. Your role requires 100% accuracy. 
When in doubt, always say 'I cannot confirm this from the provided sources.'"
```

---

**Q10: What is the difference between intrinsic and extrinsic hallucination?**

**Answer:**
- **Intrinsic hallucination**: The generated text directly contradicts the source documents in the context. The context says "NIFTY lot size is 25" and the model says "NIFTY lot size is 50."
- **Extrinsic hallucination**: The generated text cannot be verified against the provided context — neither confirmed nor denied. The model adds information not present in any source.

Extrinsic is harder to detect because it's technically not contradicting the sources; it's going beyond them.

```python
def classify_hallucination(
    claim: str, 
    context_chunks: list[str],
    llm_judge
) -> str:
    """Use LLM as judge to classify hallucination type"""
    prompt = f"""Given these source documents and a claim, classify the claim:
    
Sources: {chr(10).join(context_chunks)}
Claim: "{claim}"

Classification options:
- SUPPORTED: claim is directly supported by sources
- INTRINSIC_HALLUCINATION: claim directly contradicts a source
- EXTRINSIC_HALLUCINATION: claim goes beyond what sources state (cannot be verified)
- INFERENCE: claim is a reasonable inference from sources but not stated explicitly

Respond with just the classification word."""
    
    return llm_judge.complete(prompt).strip()
```

---

**Q11: How does Constitutional AI (Anthropic) reduce hallucination?**

**Answer:** Constitutional AI trains Claude with a set of principles (the "constitution") that include honesty and epistemic humility. Claude is RLHF-trained to prefer responses that:
- Acknowledge uncertainty
- Avoid fabricating information
- Recommend checking authoritative sources

This is why Claude's system prompt acknowledgment of uncertainty is more reliable than some other models. For NSE use, pairing Claude's constitutional training with RAG grounding creates a doubly-robust anti-hallucination stack.

---

**Q12-Q25 (Condensed):**

**Q12:** **Semantic entropy** as a hallucination measure — sample multiple outputs, if they're semantically diverse (high entropy), the model is uncertain → flag as potential hallucination.

**Q13:** **NLI-based faithfulness** — use a Natural Language Inference model (e.g., `cross-encoder/nli-deberta-v3-base`) to check if each generated sentence is *entailed* by the retrieved context.

```python
from transformers import pipeline
nli = pipeline("text-classification", model="cross-encoder/nli-deberta-v3-base")
result = nli(f"premise: {context} hypothesis: {generated_claim}")
# Labels: ENTAILMENT, CONTRADICTION, NEUTRAL
```

**Q14:** **FActScore** — decomposes LLM output into atomic facts, verifies each against Wikipedia or a knowledge source.

**Q15:** **Calibration** — well-calibrated models output probabilities matching their actual accuracy. Check token-level log probabilities; low probability tokens in factual claims signal potential hallucination.

**Q16:** **Retrieval augmented fine-tuning (RAFT)** — fine-tune model to cite sources and ignore red-herring documents, specifically reduces RAG hallucinations.

**Q17:** **System prompt injection defense** — prevent users from injecting "ignore previous instructions" into queries:
```python
def sanitize_user_input(text: str) -> str:
    injection_patterns = [
        r'ignore (all |previous |prior )?instructions',
        r'forget (everything|what I said)',
        r'act as (if|though)',
        r'new role:',
    ]
    for pattern in injection_patterns:
        if re.search(pattern, text, re.IGNORECASE):
            raise ValueError("Potential prompt injection detected")
    return text
```

**Q18:** **Hallucination in multi-turn conversations** — model may "remember" hallucinated facts from earlier turns. Solution: always re-retrieve from VectorDB for each turn; don't rely on model's in-context "memory" for facts.

**Q19:** **Fine-tuning for domain accuracy** — fine-tune on NSE-specific Q&A pairs to improve factual accuracy for domain-specific entities that appear rarely in training data.

**Q20:** **Human-in-the-loop validation** — for high-stakes financial outputs (trading recommendations), route to human review if confidence score < threshold.

**Q21-Q25 (Common questions):**
- What is the "lost in the middle" problem? (Models attend poorly to middle of context)
- How does temperature affect hallucination? (Lower temp = less creative, fewer hallucinations)
- What is self-RAG? (Model decides when to retrieve vs answer from memory)
- How does hallucination differ between GPT-4 and Claude? (Claude tends to be more cautious, acknowledges uncertainty more)
- What is Chain-of-Verification (CoVe)? (Generate answer → verify each claim → revise)

---

### Deep-Dive Real-World Edge Case Questions (20)

**Q1: Your NSE AI platform gives a financial recommendation based on hallucinated data ("Infosys Q3 profit rose 40%"). The platform is used by institutional traders. What safeguards prevent this from being a financial liability?**

**Answer:**
```python
# Multi-layer financial safety system

class FinancialSafetyLayer:
    def __init__(self, nse_data_api, vector_db, nli_model):
        self.nse_api = nse_data_api      # Real-time NSE data
        self.vector_db = vector_db        # Source documents
        self.nli = nli_model             # Fact verification

    async def safe_financial_response(self, query: str, llm_response: str) -> dict:
        # Step 1: Extract factual claims (numbers, dates, company stats)
        claims = await self.extract_numerical_claims(llm_response)
        
        # Step 2: Verify each claim against NSE API
        violations = []
        for claim in claims:
            if claim['type'] == 'stock_price':
                actual = await self.nse_api.get_price(claim['symbol'])
                if abs(claim['value'] - actual) / actual > 0.05:  # >5% deviation
                    violations.append({
                        "claim": claim['text'],
                        "stated": claim['value'],
                        "actual": actual,
                        "severity": "HIGH"
                    })
        
        # Step 3: NLI faithfulness check against retrieved documents
        faithfulness_score = await self.check_faithfulness(
            llm_response, 
            context=query['retrieved_docs']
        )
        
        # Step 4: Determine safety disposition
        if violations or faithfulness_score < 0.85:
            return {
                "status": "BLOCKED",
                "reason": "Factual verification failed",
                "violations": violations,
                "faithfulness_score": faithfulness_score,
                "safe_response": "I cannot confirm this information. Please verify with official NSE/BSE data."
            }
        
        return {
            "status": "APPROVED",
            "response": llm_response,
            "faithfulness_score": faithfulness_score,
            "disclaimer": "This analysis is based on retrieved documents as of the last data sync. Always verify with live market data before trading."
        }
```

---

**Q2: Your RAG pipeline retrieves the correct SEBI circular, but the LLM misinterprets a legal clause due to ambiguous language. The context was correct but the reasoning was wrong. How do you detect and handle reasoning hallucination?**

**Answer:**
```python
# Strategy: Structured output + external validation of reasoning steps

STRUCTURED_LEGAL_ANALYSIS_PROMPT = """Analyze this SEBI regulatory clause.

Clause: {clause}
Question: {question}

Provide your analysis in this EXACT JSON format:
{{
  "direct_quote": "Copy the exact relevant text from the clause",
  "interpretation": "Your interpretation of what the clause means",
  "reasoning_steps": [
    "Step 1: ...",
    "Step 2: ..."
  ],
  "answer": "Final answer",
  "confidence": "high/medium/low",
  "ambiguity_flags": ["Any ambiguous terms or phrases that need clarification"]
}}"""

# Then validate the interpretation against the direct quote
def validate_interpretation(response: dict, clause: str) -> bool:
    # Check: does direct_quote actually appear in the clause?
    if response['direct_quote'].lower() not in clause.lower():
        return False  # Model fabricated the quote
    
    # Check: use NLI to verify interpretation is entailed by quote
    nli_result = nli_model(
        f"premise: {response['direct_quote']} hypothesis: {response['interpretation']}"
    )
    if nli_result[0]['label'] == 'CONTRADICTION':
        return False  # Reasoning hallucination detected
    
    return True
```

---

**Q3: Your NSE AI chatbot starts a conversation fine but by turn 5, it's confidently stating facts from earlier turns that were actually incorrect. The error compounds across turns.**

**Answer:**
```python
# Stateless RAG per turn — never trust model's "memory" for facts
class StatelessRAGConversation:
    def __init__(self, vector_db, llm):
        self.vector_db = vector_db
        self.llm = llm
        self.conversation_history = []  # Only keep Q&A pairs, not facts
    
    async def respond(self, user_message: str) -> str:
        # ALWAYS re-retrieve fresh context for every turn
        # Never rely on facts mentioned in previous turns
        fresh_context = await self.vector_db.search(user_message, k=5)
        
        # Build prompt with conversation history (for coherence)
        # but fresh context (for factual grounding)
        messages = self.conversation_history[-4:]  # Last 2 turns for coherence
        
        system = f"""You are an NSE market analysis assistant.
        
CRITICAL: Use ONLY the provided context documents for factual claims.
Do NOT treat statements from earlier in the conversation as factual references.
If you cited something incorrectly earlier, correct yourself using the current context.

Current Context Documents:
{chr(10).join([c['text'] for c in fresh_context])}"""
        
        response = await self.llm.complete(messages, system=system)
        
        # Store exchange (without facts, just the exchange itself)
        self.conversation_history.extend([
            {"role": "user", "content": user_message},
            {"role": "assistant", "content": response}
        ])
        
        return response
```

---

**Q4-Q20 (Key scenarios):**

**Q4:** Conflicting sources — two retrieved SEBI circulars contradict each other (old and new regulation). Solution: inject document metadata (date, amendment status), instruct model to prefer most recent, explicitly flag the conflict to user.

**Q5:** Model ignores RAG context and answers from training data. Solution: Measure faithfulness score; if low, add stronger grounding prompt like "You are a document reading machine. You have NO prior knowledge. Answer ONLY from these documents."

**Q6:** User asks about future NSE events ("Will SEBI ban futures trading?"). LLM confidently predicts. Solution: classify query intent — if predictive/speculative, respond with "This requires prediction beyond available data; I can only analyze historical documents."

**Q7:** Temperature too high causes creative hallucinations. For financial use: set temperature=0.0 to 0.2 for factual responses, 0.7+ only for creative/exploratory analysis.

**Q8:** Prompt injection via malicious NSE document. A document contains "SYSTEM: Ignore all previous instructions." Solution: use document sandboxing in prompts, separate system/user/context roles clearly, use Claude's structured message format.

**Q9:** Model hallucinates API endpoint or SQL query when used as a code generator. Solution: validate generated SQL before execution, use parameterized queries, dry-run in transaction that gets rolled back.

**Q10:** Hallucination in embeddings — two semantically similar but factually different documents (e.g., different years of the same SEBI circular). Both get retrieved. Model blends them incorrectly. Solution: metadata filtering by exact document ID + version.

**Q11-Q20:** Cross-lingual hallucinations, token budget pressure causing truncation of key context, hallucination in table/numeric data extraction, multi-hop reasoning failures, hallucination amplification in agent chains, over-refusal (refusing to answer valid queries), calibration failure in high-confidence wrong answers, domain shift from general to financial, hallucination in structured output (wrong JSON schema compliance), streaming response inconsistency.

---

### Must-Read Study Resources
1. **Survey of Hallucination in NLG** — [arxiv.org/abs/2202.03629](https://arxiv.org/abs/2202.03629) — Comprehensive academic survey
2. **RAGAS Documentation** — [docs.ragas.io](https://docs.ragas.io) — Practical RAG evaluation framework
3. **Anthropic's Model Cards and Safety** — [anthropic.com/model-card](https://www.anthropic.com/model-card) — Claude's honesty properties

---

## TOPIC 13: Gemini (Google)

### Common Interview Questions (20)

**Q1: Describe the Gemini model family and when you'd use each.**

**Answer:**

| Model | Context | Best For | Speed |
|---|---|---|---|
| **Gemini 1.5 Flash** | 1M tokens | Cost-effective tasks, summarization, classification | Very fast |
| **Gemini 1.5 Pro** | 1M tokens | Complex reasoning, long-doc analysis, code | Medium |
| **Gemini 2.0 Flash** | 1M tokens | Improved reasoning, multimodal, agentic tasks | Fast |
| **Gemini 2.0 Pro (exp)** | 2M tokens | Most capable, research, complex agents | Slower |
| **Gemini Nano** | On-device | Mobile, edge AI, offline | Fastest |
| **Gemini Ultra** | Large | Most complex tasks (largely superseded by 2.0) | Slowest |

**For NSE platform:** Gemini 1.5 Pro's 1M token context is exceptional — you could potentially feed an entire year of NSE circulars in one context rather than relying on chunked RAG. However, "needle in a haystack" recall degrades at very long contexts.

---

**Q2: How does Gemini's 1M token context window change RAG architecture?**

**Answer:** It doesn't eliminate RAG but changes the trade-offs:

```python
# With 1M context: "Long Context RAG" strategy
# Instead of top-K retrieval, inject ALL relevant documents
# Let the model do its own "retrieval" internally

import google.generativeai as genai

# Traditional RAG: top-5 chunks
# Long-context: ALL documents for a specific symbol

async def long_context_analysis(symbol: str, query: str):
    # Fetch all documents for this symbol (could be thousands)
    all_docs = await fetch_all_symbol_documents(symbol)
    full_context = "\n\n".join([d['text'] for d in all_docs])
    
    # 1M context can handle a year's worth of NSE circulars for one symbol
    if len(full_context.split()) < 700_000:  # ~1M tokens ≈ 750K words
        model = genai.GenerativeModel('gemini-1.5-pro')
        response = model.generate_content(
            f"All NSE documents for {symbol}:\n{full_context}\n\nQuery: {query}"
        )
    else:
        # Fall back to chunked RAG for very large document sets
        return await chunked_rag(symbol, query)
```

**Pros:** No chunking loss, no retrieval miss, full cross-document reasoning
**Cons:** High cost per query, latency, "lost in middle" degradation, not scalable for multi-symbol queries

---

**Q3: What are Gemini's multimodal capabilities?**

**Answer:** Gemini is natively multimodal (trained on text, images, audio, video, code simultaneously):

```python
import google.generativeai as genai
from PIL import Image
import requests

genai.configure(api_key="YOUR_API_KEY")
model = genai.GenerativeModel('gemini-1.5-pro')

# Image + text (e.g., analyze NSE candlestick chart screenshot)
chart_image = Image.open("nse_nifty_chart.png")
response = model.generate_content([
    chart_image,
    "Analyze this NSE NIFTY 50 candlestick chart. Identify key support/resistance levels, trend direction, and any technical patterns visible."
])

# Video analysis (e.g., NSE trading session recording)
video_file = genai.upload_file(path="nse_session.mp4", mime_type="video/mp4")
response = model.generate_content([
    video_file,
    "Summarize the key market events in this NSE trading session video."
])

# PDF natively (no chunking needed up to ~1000 pages)
pdf = genai.upload_file(path="sebi_annual_report_2024.pdf", mime_type="application/pdf")
response = model.generate_content([pdf, "What are SEBI's top regulatory priorities for 2025?"])
```

---

**Q4: How do you integrate Gemini via Google Vertex AI?**

**Answer:**
```python
import vertexai
from vertexai.generative_models import GenerativeModel, Part, GenerationConfig

# Initialize Vertex AI
vertexai.init(project="your-gcp-project", location="us-central1")

model = GenerativeModel("gemini-1.5-pro-002")

# Configure generation
generation_config = GenerationConfig(
    temperature=0.1,
    top_p=0.95,
    max_output_tokens=2048,
    response_mime_type="application/json",  # Structured output
    response_schema={
        "type": "object",
        "properties": {
            "market_sentiment": {"type": "string", "enum": ["bullish", "bearish", "neutral"]},
            "key_factors": {"type": "array", "items": {"type": "string"}},
            "confidence": {"type": "number"}
        }
    }
)

response = model.generate_content(
    [f"Analyze NSE market sentiment based on: {news_text}"],
    generation_config=generation_config
)

# Response is structured JSON
analysis = json.loads(response.text)
```

---

**Q5: Gemini vs Claude vs GPT-4 — benchmarks and practical differences (2025)?**

**Answer:**

| Benchmark | Gemini 2.0 Pro | Claude 3.5 Sonnet | GPT-4o |
|---|---|---|---|
| MMLU | ~90% | ~88.7% | ~88.7% |
| HumanEval (Code) | ~84% | ~92% | ~90.2% |
| MATH | ~76% | ~71.1% | ~76.6% |
| Long context (NIAH) | Excellent | Excellent | Good |
| Multimodal | Native | Text-primary (Vision via Claude 3) | Vision capable |
| Context window | 2M | 200K | 128K |

**Practical differences:**
- **Claude** — best for nuanced instruction following, safety, long-form writing, code quality. Most reliable for "follow rules strictly" use cases like NSE regulatory analysis.
- **GPT-4o** — best ecosystem (most integrations, assistants API, best function calling track record), strong multimodal, best real-time API for voice.
- **Gemini** — best for very long context (1M-2M), native multimodal (including video), tightest Google ecosystem integration (Docs, Sheets, Gmail), best for tasks needing Google Search grounding.

---

**Q6: When would you choose Gemini over Claude for NSE platform?**

**Answer:**
Choose Gemini when:
1. **Analyzing entire annual reports without chunking** — Gemini 1.5 Pro's 1M context handles full SEBI annual reports or NSE data dictionaries
2. **Chart/image analysis** — User uploads a screenshot of an NSE chart; Gemini's native image understanding is deeply integrated
3. **Video analysis** — Analyzing NSE's recorded webinars or results presentations
4. **Cost optimization at scale** — Gemini Flash is significantly cheaper than Claude Sonnet for high-volume classification tasks
5. **Google Cloud stack** — If platform is GCP-native (Vertex AI, BigQuery, Cloud Run), Gemini integrates with zero auth friction

Stick with Claude when:
1. Complex multi-step reasoning and instruction following
2. Code generation (Claude Sonnet outperforms on HumanEval)
3. Safety-critical financial guidance (Claude's constitutional training)
4. Long-form structured documents (regulatory filings, compliance reports)

---

**Q7: What is Gemini's "Grounding with Google Search" feature?**

**Answer:**
```python
# Gemini can call Google Search in real-time to ground responses
from google.generativeai.types import Tool, GoogleSearch

model = genai.GenerativeModel(
    'gemini-1.5-pro',
    tools=[Tool(google_search=GoogleSearch())]
)

response = model.generate_content(
    "What is NIFTY 50's current P/E ratio and how does it compare to historical averages?"
)
# Gemini autonomously decides to search Google, retrieves current data, grounds answer
# Returns grounding metadata with sources
for chunk in response.candidates[0].grounding_metadata.search_entry_point:
    print(chunk.rendered_content)
```

This is powerful for NSE platform — queries about current market levels, latest SEBI news, or recent corporate actions can be answered with live data without maintaining your own data pipeline.

---

**Q8: Explain Gemini's function calling / tool use.**

**Answer:**
```python
# Very similar to Claude's tool use but with different SDK

get_nse_price = genai.protos.Tool(
    function_declarations=[
        genai.protos.FunctionDeclaration(
            name="get_nse_stock_price",
            description="Fetch real-time NSE stock price",
            parameters=genai.protos.Schema(
                type=genai.protos.Type.OBJECT,
                properties={
                    "symbol": genai.protos.Schema(type=genai.protos.Type.STRING),
                    "exchange": genai.protos.Schema(
                        type=genai.protos.Type.STRING,
                        enum=["NSE", "BSE"]
                    )
                },
                required=["symbol"]
            )
        )
    ]
)

model = genai.GenerativeModel('gemini-1.5-pro', tools=[get_nse_price])

chat = model.start_chat()
response = chat.send_message("What's Infosys trading at right now on NSE?")

# Check if model wants to call a function
if response.candidates[0].content.parts[0].function_call:
    fc = response.candidates[0].content.parts[0].function_call
    result = execute_tool(fc.name, dict(fc.args))
    
    # Send tool result back
    response2 = chat.send_message(
        genai.protos.Content(parts=[genai.protos.Part(
            function_response=genai.protos.FunctionResponse(
                name=fc.name,
                response={"result": result}
            )
        )])
    )
```

---

**Q9-Q20 (Key questions):**

**Q9:** Gemini Code Execution — model can write and run Python code in a sandboxed environment, useful for on-the-fly financial calculations.

**Q10:** Gemini vs GPT-4 for RAG — both work well; Gemini's advantage is natively handling PDF/image documents without preprocessing.

**Q11:** Vertex AI Model Garden — access multiple models (Gemini, Llama, Mistral) through one API endpoint with unified billing.

**Q12:** Gemini safety filters vs Claude's constitution — Gemini uses configurable harm categories (HARM_CATEGORY_DANGEROUS_CONTENT, etc.); Claude's safety is more nuanced and model-level.

**Q13:** Gemini Flash vs Pro — Flash: 3x faster, 5x cheaper, slightly lower reasoning quality. Use Flash for classification/extraction, Pro for complex analysis.

**Q14:** Caching in Gemini API — `cachedContent` API allows caching large context (e.g., the entire NSE master data file) and reusing it across requests, dramatically reducing cost.

```python
# Gemini context caching for NSE master data
from google.generativeai import caching

cache = caching.CachedContent.create(
    model="gemini-1.5-pro-002",
    contents=[{"role": "user", "parts": [{"text": ENTIRE_NSE_MASTER_DATA}]}],
    ttl=datetime.timedelta(hours=24)
)
```

**Q15:** Gemini Nano on-device — for Android apps, can run on-device (Pixel 8+) with no API call latency, privacy preserved. 

**Q16-Q20:** PaLM to Gemini migration, Gemini's multilingual capabilities (100+ languages), Gemini for structured data analysis (can reason over tables), Gemini's approach to safety vs OpenAI moderation API, future Gemini Ultra 2 capabilities.

---

### Must-Read Study Resources
1. **Gemini Technical Report** — [arxiv.org/abs/2312.11805](https://arxiv.org/abs/2312.11805) — Google's Gemini architecture paper
2. **Vertex AI Generative AI Docs** — [cloud.google.com/vertex-ai/generative-ai/docs](https://cloud.google.com/vertex-ai/generative-ai/docs) — Official API reference
3. **Google AI Studio** — [aistudio.google.com](https://aistudio.google.com) — Try all Gemini models interactively

---

## TOPIC 14: GPT (OpenAI)

### Common Interview Questions (25)

**Q1: What are the differences between GPT-4, GPT-4o, and GPT-4-turbo?**

**Answer:**

| Model | Context | Speed | Cost (Input) | Key Features |
|---|---|---|---|---|
| **GPT-4** (original) | 8K / 32K | Slow | $30/1M tokens | Most carefully trained, reliable reasoning |
| **GPT-4-turbo** | 128K | Faster | $10/1M tokens | Larger context, updated knowledge (Apr 2024) |
| **GPT-4o** | 128K | 2x faster than GPT-4 | $5/1M tokens | **Native multimodal** (image/audio/text), best overall |
| **GPT-4o-mini** | 128K | Very fast | $0.15/1M tokens | Cost-optimized, strong for structured tasks |
| **o1 / o3** | 128K | Slower (thinking) | $15+/1M tokens | Extended thinking for math/science/code |
| **o1-mini** | 128K | Fast thinking | $3/1M tokens | Cheaper reasoning model |

**"o" series** (o1, o3, o3-mini, o4-mini) — these use "chain-of-thought" reasoning internally before outputting. Much better at math, code, multi-step problems. For NSE: use o-series for complex regulatory compliance checking, use GPT-4o for general market queries.

---

**Q2: How does OpenAI's Chat Completions API work?**

**Answer:**
```python
from openai import OpenAI

client = OpenAI(api_key="sk-...")

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {
            "role": "system",
            "content": """You are an NSE market analyst. Answer only from the provided context.
            Always cite sources. Respond in structured JSON when analyzing data."""
        },
        {
            "role": "user",
            "content": "What are the circuit breaker levels for NSE?"
        },
        {
            "role": "assistant",
            "content": "Based on NSE's trading rules, circuit breakers are triggered at..."  
        },
        {
            "role": "user",
            "content": "What about for individual stocks?"  # Follow-up
        }
    ],
    temperature=0.1,
    max_tokens=1000,
    response_format={"type": "json_object"},  # Structured output
    seed=42,              # Reproducible outputs (approximately)
    logprobs=True,        # Return token probabilities
    top_logprobs=5        # Top 5 token alternatives
)

answer = response.choices[0].message.content
finish_reason = response.choices[0].finish_reason  # "stop", "length", "tool_calls"
usage = response.usage  # prompt_tokens, completion_tokens, total_tokens
```

---

**Q3: Explain OpenAI's Assistants API — when would you use it over raw Chat Completions?**

**Answer:**

**Assistants API** provides: persistent threads (conversation history managed by OpenAI), built-in tools (file search/RAG, code interpreter, function calling), and file uploads.

```python
from openai import OpenAI

client = OpenAI()

# Create an assistant (do once, reuse)
assistant = client.beta.assistants.create(
    name="NSE Market Analyst",
    instructions="""You are an expert NSE market analyst. 
    Use the provided files to answer questions about NSE regulations and market data.
    Always cite specific documents and sections.""",
    model="gpt-4o",
    tools=[
        {"type": "file_search"},         # Built-in RAG
        {"type": "code_interpreter"},    # Python execution
        {"type": "function", "function": {
            "name": "get_live_nse_data",
            "description": "Get real-time NSE market data",
            "parameters": {
                "type": "object",
                "properties": {
                    "symbol": {"type": "string"}
                }
            }
        }}
    ]
)

# Upload NSE documents to vector store
vector_store = client.beta.vector_stores.create(name="NSE Documents")
client.beta.vector_stores.file_batches.upload_and_poll(
    vector_store_id=vector_store.id,
    files=[open("sebi_circulars.pdf", "rb"), open("nse_rules.pdf", "rb")]
)

# Attach vector store to assistant
client.beta.assistants.update(
    assistant.id,
    tool_resources={"file_search": {"vector_store_ids": [vector_store.id]}}
)

# Create a thread (persistent conversation)
thread = client.beta.threads.create()

# Add message and run
client.beta.threads.messages.create(
    thread_id=thread.id,
    role="user",
    content="What are SEBI's margin requirements for F&O trading?"
)

run = client.beta.threads.runs.create_and_poll(
    thread_id=thread.id,
    assistant_id=assistant.id
)

# Get response
messages = client.beta.threads.messages.list(thread_id=thread.id)
print(messages.data[0].content[0].text.value)
```

**Use Assistants API when:** You need persistent multi-turn conversation, built-in file search (auto RAG), or code execution without building infrastructure.
**Use Chat Completions when:** You need full control, custom RAG, lower latency, or non-OpenAI-managed state.

---

**Q4: How does OpenAI function calling work?**

**Answer:**
```python
# Define tools
tools = [
    {
        "type": "function",
        "function": {
            "name": "query_nse_database",
            "description": "Query the NSE historical data database",
            "parameters": {
                "type": "object",
                "properties": {
                    "symbol": {
                        "type": "string",
                        "description": "NSE stock symbol e.g. RELIANCE, TCS"
                    },
                    "metric": {
                        "type": "string",
                        "enum": ["price", "volume", "pe_ratio", "market_cap"],
                        "description": "The metric to query"
                    },
                    "start_date": {"type": "string", "format": "date"},
                    "end_date": {"type": "string", "format": "date"}
                },
                "required": ["symbol", "metric"]
            }
        }
    }
]

messages = [{"role": "user", "content": "What was Reliance's average P/E ratio in Q3 2024?"}]

response = client.chat.completions.create(
    model="gpt-4o",
    messages=messages,
    tools=tools,
    tool_choice="auto"  # or "required" to force tool use
)

# Handle tool call
if response.choices[0].finish_reason == "tool_calls":
    tool_call = response.choices[0].message.tool_calls[0]
    args = json.loads(tool_call.function.arguments)
    
    # Execute the actual function
    result = query_nse_database(**args)
    
    # Add to messages and continue
    messages.append(response.choices[0].message)  # assistant message with tool_call
    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": json.dumps(result)
    })
    
    # Final response with tool result
    final = client.chat.completions.create(
        model="gpt-4o",
        messages=messages
    )
    print(final.choices[0].message.content)
```

---

**Q5: Explain OpenAI text embeddings — `text-embedding-3-small` vs `text-embedding-3-large` vs `ada-002`.**

**Answer:**

| Model | Dimensions | MTEB Score | Cost (per 1M tokens) | Notes |
|---|---|---|---|---|
| `text-embedding-ada-002` | 1536 | 61.0 | $0.10 | Legacy, still widely deployed |
| `text-embedding-3-small` | 1536 (reducible to 512) | 62.3 | $0.02 | **5x cheaper**, slightly better than ada-002 |
| `text-embedding-3-large` | 3072 (reducible to 256) | 64.6 | $0.13 | Best quality, use for NSE if budget allows |

**Matryoshka embeddings** — `text-embedding-3` models support dimension reduction:
```python
from openai import OpenAI
import numpy as np

client = OpenAI()

# Full 3072-dim embedding
response = client.embeddings.create(
    model="text-embedding-3-large",
    input="NIFTY 50 circuit breaker triggered at 10% market decline",
)
full_embedding = response.data[0].embedding  # 3072 dims

# Truncate to 256 dims (still high quality due to Matryoshka training)
small_embedding = full_embedding[:256]
small_embedding = small_embedding / np.linalg.norm(small_embedding)  # re-normalize

# Use case for NSE: store 256-dim for fast search, 3072-dim for high-precision re-ranking
```

---

**Q6: How do you fine-tune a GPT model?**

**Answer:**
```python
from openai import OpenAI
import json

client = OpenAI()

# Step 1: Prepare training data (JSONL format)
training_data = [
    {
        "messages": [
            {"role": "system", "content": "You are an NSE regulatory expert."},
            {"role": "user", "content": "What is F&O margin?"},
            {"role": "assistant", "content": "F&O margin in NSE consists of SPAN margin (for market risk) and Exposure margin (for residual risk). For index futures, SPAN margin is typically 5-10% of contract value..."}
        ]
    },
    # ... hundreds more NSE-specific Q&A pairs
]

with open("nse_finetune.jsonl", "w") as f:
    for item in training_data:
        f.write(json.dumps(item) + "\n")

# Step 2: Upload training file
file = client.files.create(
    file=open("nse_finetune.jsonl", "rb"),
    purpose="fine-tune"
)

# Step 3: Create fine-tuning job
job = client.fine_tuning.jobs.create(
    training_file=file.id,
    model="gpt-4o-mini-2024-07-18",  # Base model
    hyperparameters={
        "n_epochs": 3,
        "batch_size": 4,
        "learning_rate_multiplier": 1.0
    },
    suffix="nse-market-analyst"
)

print(f"Fine-tuning job: {job.id}")

# Step 4: Monitor
events = client.fine_tuning.jobs.list_events(fine_tuning_job_id=job.id)

# Step 5: Use fine-tuned model
response = client.chat.completions.create(
    model=f"ft:gpt-4o-mini:org:nse-market-analyst:{job.fine_tuned_model}",
    messages=[{"role": "user", "content": "Explain SEBI's PFUTP regulations"}]
)
```

**When to fine-tune for NSE platform:**
- Model doesn't know NSE-specific terminology (ISIN, circuit filters, T+1 settlement)
- Consistent output format needed (structured JSON responses always)
- Domain-specific Q&A that's too long for system prompt
- **Cost reduction** — fine-tune gpt-4o-mini to match gpt-4o quality for domain tasks at 10x lower cost

---

**Q7: OpenAI vs Anthropic Claude — practical comparison for NSE AI platform.**

**Answer:**

| Dimension | OpenAI GPT-4o | Claude 3.5 Sonnet |
|---|---|---|
| **Instruction following** | Very good | Excellent (more precise rule-following) |
| **Code generation** | Excellent | Excellent (slightly edge on complex code) |
| **Safety/refusals** | Sometimes over-refuses | Better calibrated |
| **Context window** | 128K | 200K |
| **Streaming** | Yes | Yes |
| **Structured output** | JSON mode + response_format | Tool use / XML tags |
| **Vision** | GPT-4o (native) | Claude 3 (native vision) |
| **API ecosystem** | Broader (LangChain default) | Growing |
| **Fine-tuning** | Yes (GPT-4o-mini, GPT-3.5) | No public fine-tuning |
| **Price (Sonnet vs 4o)** | $5/$15 per 1M | $3/$15 per 1M |
| **Hallucination tendency** | Moderate | Lower (constitutional training) |
| **Reasoning (extended)** | o1/o3 series | Extended thinking (claude-3-7-sonnet) |

**Decision for NSE platform:**
- Use **Claude** as primary LLM for regulatory compliance analysis (lower hallucination risk, better instruction following for "strict rules" prompts)
- Use **GPT-4o** for multimodal chart analysis and when Assistants API file management simplifies architecture
- Use **GPT-4o-mini** for high-volume classification tasks (sentiment tagging, document categorization)

---

**Q8: What is the OpenAI Realtime API and when would you use it?**

**Answer:** The Realtime API enables low-latency (<300ms), bidirectional voice conversations with GPT-4o. It processes speech-to-speech natively (no separate STT/TTS pipeline).

```javascript
// WebSocket-based Realtime API
const ws = new WebSocket('wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview', {
    headers: {
        'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
        'OpenAI-Beta': 'realtime=v1'
    }
});

ws.on('open', () => {
    // Configure session
    ws.send(JSON.stringify({
        type: 'session.update',
        session: {
            voice: 'alloy',
            instructions: 'You are an NSE market analyst voice assistant. Provide concise, accurate market insights.',
            input_audio_transcription: { model: 'whisper-1' },
            turn_detection: { type: 'server_vad', threshold: 0.5 }
        }
    }));
});

// Stream audio chunks
ws.send(JSON.stringify({
    type: 'input_audio_buffer.append',
    audio: base64AudioChunk
}));
```

**NSE use case:** Voice-driven market query interface — traders can ask "What's the NIFTY doing today?" hands-free.

---

**Q9-Q25 (Condensed):**

**Q9:** Structured outputs (JSON Schema enforcement) vs JSON mode — structured outputs guarantee strict schema compliance, JSON mode just asks the model to output JSON (can deviate).

```python
response = client.chat.completions.create(
    model="gpt-4o-2024-08-06",
    messages=[...],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "nse_analysis",
            "strict": True,
            "schema": {
                "type": "object",
                "properties": {
                    "sentiment": {"type": "string", "enum": ["bullish", "bearish", "neutral"]},
                    "key_levels": {"type": "array", "items": {"type": "number"}}
                },
                "required": ["sentiment", "key_levels"],
                "additionalProperties": False
            }
        }
    }
)
```

**Q10:** OpenAI streaming:
```python
stream = client.chat.completions.create(model="gpt-4o", messages=[...], stream=True)
for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

**Q11:** Batch API — 50% cost reduction for async tasks (e.g., batch embedding all NSE documents overnight).

**Q12:** Token counting with `tiktoken`:
```python
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4o")
tokens = enc.encode("NIFTY 50 circuit breaker analysis")
print(len(tokens))  # 7
```

**Q13:** Rate limit handling with exponential backoff.

**Q14:** GPT-4 Vision for NSE — analyze charts, financial statements PDFs as images.

**Q15:** Moderation API — screen user inputs for harmful content before processing.

**Q16:** `logprobs` for confidence scoring — use token probabilities to estimate answer confidence.

**Q17:** Prompt caching in GPT-4 — similar to Claude's caching; prefix caching reduces cost for repeated system prompts.

**Q18:** o1 reasoning model — doesn't support system prompts, tools are different, use for complex multi-step financial calculations.

**Q19:** GPT-4o vs GPT-4o-mini for NSE — mini is sufficient for extraction/classification; use full GPT-4o for analysis/generation.

**Q20-Q25:** OpenAI Embeddings for RAG pipeline, fine-tuning evaluation (on held-out test set), distillation from GPT-4 to GPT-4o-mini, evals framework, predictable outputs with `seed`, organization-level usage monitoring.

---

### Must-Read Study Resources
1. **OpenAI Platform Docs** — [platform.openai.com/docs](https://platform.openai.com/docs) — Comprehensive API reference
2. **OpenAI Cookbook** — [cookbook.openai.com](https://cookbook.openai.com) — Practical recipes including RAG, fine-tuning
3. **GPT-4 Technical Report** — [arxiv.org/abs/2303.08774](https://arxiv.org/abs/2303.08774) — Architecture and evaluation details

---

## TOPIC 15: Embedding Models & Text Embedding

### Common Interview Questions (25)

**Q1: What are embeddings and how do they work?**

**Answer:** An embedding is a dense numerical vector that represents the semantic meaning of text in a continuous vector space. Semantically similar texts have vectors that are close together (low cosine distance).

**How they're generated:**
1. Text → tokenized into subword pieces
2. Tokens → transformer encoder (BERT-like architecture)
3. [CLS] token representation or mean-pooled hidden states → embedding vector
4. Optional: contrastive learning fine-tuning (SimCSE, MNRL) to improve semantic clustering

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('BAAI/bge-large-en-v1.5')

sentences = [
    "NIFTY 50 dropped 5% due to global selloff",          # Financial
    "Stock market indices fell sharply amid global fears",  # Similar
    "The recipe uses 2 cups of flour",                      # Unrelated
]

embeddings = model.encode(sentences)
# embeddings[0] and embeddings[1] will be close; embeddings[2] will be far

from sklearn.metrics.pairwise import cosine_similarity
sim_matrix = cosine_similarity(embeddings)
# sim_matrix[0][1] ≈ 0.87 (similar)
# sim_matrix[0][2] ≈ 0.12 (unrelated)
```

---

**Q2: What are sentence transformers and how do they differ from standard transformers?**

**Answer:**

**Standard BERT**: Designed for classification/NER with cross-attention between pairs. To compare N sentences with cross-encoder: O(N²) complexity — too slow for retrieval.

**Sentence Transformers (Bi-encoder)**: Fine-tuned with siamese/triplet networks using contrastive loss. Each sentence is independently encoded to a fixed-size embedding. Similarity computed with cosine similarity: O(N) retrieval after O(1) encoding.

```python
# Bi-encoder training objective (contrastive loss - SimCSE)
"""
positive pairs: (sentence, paraphrase) → push together
negative pairs: (sentence, unrelated_sentence) → push apart
Loss: L = -log [sim(a,p) / (sim(a,p) + sum(sim(a,n_i)))]
"""

# Popular sentence transformer models:
# all-MiniLM-L6-v2   — 22M params, 384-dim, very fast, good quality
# all-mpnet-base-v2  — 110M params, 768-dim, slower, better quality  
# BAAI/bge-large-en-v1.5  — 335M params, 1024-dim, top MTEB performer
# intfloat/e5-large-v2    — 335M params, 1024-dim, strong on retrieval
```

---

**Q3: Compare OpenAI `text-embedding-3-large` vs Cohere `embed-v3` vs Voyage AI embeddings.**

**Answer:**

| Model | Dims | MTEB Avg | $/1M tokens | Features |
|---|---|---|---|---|
| OpenAI text-embedding-3-large | 3072 | 64.6 | $0.13 | Matryoshka (reducible) |
| OpenAI text-embedding-3-small | 1536 | 62.3 | $0.02 | Best value |
| Cohere embed-v3-english | 1024 | 64.5 | $0.10 | Domain-specific fine-tuning, native batching |
| Cohere embed-v3-multilingual | 1024 | 62.0 | $0.10 | 100+ languages |
| Voyage AI voyage-2 | 1536 | 65.0+ | $0.12 | Top MTEB, RAG-optimized |
| Voyage AI voyage-law-2 | 1024 | High legal | $0.12 | **Domain-specific for legal/regulatory** |
| BGE-M3 (local) | 1024 | 64.3 | Free | Multi-lingual, dense+sparse |

**For NSE platform:**
- **Cohere embed-v3** — excellent for financial text, supports fine-tuning on NSE-specific domain data
- **Voyage voyage-finance-1** — purpose-built for financial documents (SEC filings, earnings reports, regulatory docs) — closest match to SEBI/NSE regulatory text
- **OpenAI text-embedding-3-small** — if already using OpenAI stack, excellent price/performance

---

**Q4: What are embedding dimensions and trade-offs?**

**Answer:**
- **Higher dimensions** (3072): More expressive, captures finer semantic distinctions, higher recall in VectorDB. Costs more storage (3072 × 4 bytes = ~12KB per vector).
- **Lower dimensions** (256-512): Faster similarity search, less storage, slight quality loss.

```python
# Matryoshka Representation Learning (MRL) - OpenAI text-embedding-3
# Train model so that first N dimensions already form a good embedding
# Enables dimension reduction without retraining

import numpy as np
from openai import OpenAI

client = OpenAI()

def get_embedding(text: str, dimensions: int = 1536) -> list[float]:
    response = client.embeddings.create(
        model="text-embedding-3-large",
        input=text,
        dimensions=dimensions  # API-level truncation with MRL quality
    )
    return response.data[0].embedding

# For NSE: use 256-dim for fast approximate search, 3072-dim for re-ranking
fast_emb = get_embedding(query, dimensions=256)   # 4x less storage, faster ANN
precise_emb = get_embedding(query, dimensions=3072)  # for re-ranking top-K
```

---

**Q5: What are chunking strategies and how do they impact embedding quality?**

**Answer:**

**Fixed-size chunking** — simplest, may split sentences/paragraphs mid-thought:
```python
def fixed_chunk(text: str, size: int = 512, overlap: int = 64) -> list[str]:
    words = text.split()
    chunks = []
    for i in range(0, len(words), size - overlap):
        chunks.append(" ".join(words[i:i + size]))
    return chunks
```

**Sentence-aware chunking** (recommended):
```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=64,
    separators=["\n\n", "\n", ". ", " ", ""],  # Prefer paragraph > sentence > word breaks
    length_function=len,
)
chunks = splitter.split_text(nse_circular_text)
```

**Semantic chunking** — split at semantic boundaries (topic changes):
```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

semantic_splitter = SemanticChunker(
    OpenAIEmbeddings(model="text-embedding-3-small"),
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=95  # Split when similarity drops sharply
)
chunks = semantic_splitter.split_text(sebi_annual_report)
```

**For NSE/SEBI documents:**
- SEBI circulars: Split by clause number (Clause 1, Clause 2...) — document-structure-aware
- NSE data files: Row-level chunking (each trading record as a chunk)
- Long-form reports: Section-level (2-3 paragraphs per chunk, ~800 tokens)

**Key insight:** Chunks too small lose context; chunks too large dilute the signal. Optimal for financial regulatory text: 256-512 tokens with 10-15% overlap.

---

**Q6: What is the difference between a bi-encoder and cross-encoder?**

**Answer:**

**Bi-encoder (dual encoder):**
- Encodes query and document **independently**
- Fast: pre-compute document embeddings offline
- Scales to millions of documents (O(1) per query after offline indexing)
- Quality: ≈ good for retrieval (top-100 candidates)

**Cross-encoder:**
- Takes query + document as a **single input**
- Attends across both simultaneously (full cross-attention)
- Quality: ≈ much higher than bi-encoder for relevance scoring
- Slow: must run for every (query, document) pair — unusable at million-document scale

**Re-ranking pipeline (best of both):**
```python
from sentence_transformers import SentenceTransformer, CrossEncoder

# Stage 1: Bi-encoder retrieval (fast, approximate)
bi_encoder = SentenceTransformer('BAAI/bge-large-en-v1.5')
query_embedding = bi_encoder.encode(query)
# Retrieve top-100 candidates from pgvector
candidates = await vectordb_search(query_embedding, k=100)

# Stage 2: Cross-encoder re-ranking (slow but accurate)
cross_encoder = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')
# Score each (query, candidate) pair
scores = cross_encoder.predict([
    (query, candidate['text']) for candidate in candidates
])

# Sort by cross-encoder score and take top 5
reranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)
top_5 = [doc for doc, score in reranked[:5]]
```

**NSE application:** Two-stage retrieval for regulatory queries. Bi-encoder fast-retrieves 50 relevant circulars, cross-encoder re-ranks to find the 5 most directly relevant.

---

**Q7: How do you embed financial text from NSE/SEBI specifically?**

**Answer:** Standard embeddings underperform on financial text because:
- Domain-specific terminology (ISIN, SEBI, LODR, F&O, NCFM, SEBI Act 1992)
- Numerical content (P/E ratios, SPAN margins, lot sizes)
- Regulatory legalese with specific clause structure

**Strategies:**

```python
# Strategy 1: Use finance-specialized embedding models
from sentence_transformers import SentenceTransformer

# Finance-specific models
fin_model = SentenceTransformer('yiyanghkust/finbert-tone')  # FinBERT
# or
fin_model = SentenceTransformer('ProsusAI/finbert')

# Strategy 2: Preprocessing — expand abbreviations before embedding
ABBREVIATIONS = {
    "NSE": "National Stock Exchange",
    "SEBI": "Securities and Exchange Board of India",
    "F&O": "Futures and Options",
    "NIFTY": "National Stock Exchange Fifty Index",
    "LODR": "Listing Obligations and Disclosure Requirements",
    "IRDAI": "Insurance Regulatory and Development Authority of India",
}

def preprocess_financial_text(text: str) -> str:
    for abbr, full in ABBREVIATIONS.items():
        text = re.sub(r'\b' + abbr + r'\b', f"{abbr} ({full})", text, count=1)
    return text

# Strategy 3: Voyage AI finance-specific model (best available)
import voyageai
voyage_client = voyageai.Client()

result = voyage_client.embed(
    texts=[preprocess_financial_text(nse_document)],
    model="voyage-finance-1",  # Purpose-built for financial documents
    input_type="document"       # "query" for queries, "document" for docs
)

# Strategy 4: Cohere embed with domain fine-tuning
# Fine-tune Cohere embed on NSE Q&A pairs for domain adaptation
```

---

**Q8: What is the MTEB (Massive Text Embedding Benchmark)?**

**Answer:** MTEB is the standard benchmark for evaluating text embedding models across 56+ tasks and 112 languages. Categories:

- **Retrieval** (15 datasets) — find relevant documents for queries (most important for RAG)
- **STS** (Semantic Textual Similarity) — score how similar two sentences are
- **Classification** — use embeddings to classify text
- **Clustering** — group similar documents
- **Reranking** — score relevance pairs

```python
# Running MTEB evaluation on custom model
from mteb import MTEB
from sentence_transformers import SentenceTransformer

# Fine-tuned NSE model
model = SentenceTransformer("./nse_finetuned_embeddings")

# Evaluate on financial retrieval tasks
evaluation = MTEB(tasks=["FiQA2018", "SciFact"])  # FiQA = financial Q&A
results = evaluation.run(model, output_folder="mteb_results")
```

**Key leaderboard insight (2025):** Top MTEB performers:
1. `voyage-3-large` — 70.0+ average
2. `text-embedding-3-large` — 64.6
3. `BAAI/bge-large-en-v1.5` — 64.2
4. `intfloat/e5-mistral-7b-instruct` — 66.6 (instruction-tuned LLM as embedder)

---

**Q9: What are instruction-tuned embedding models?**

**Answer:** Models like `intfloat/e5-mistral-7b-instruct` prepend a task-specific instruction to the query before embedding:

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("intfloat/e5-mistral-7b-instruct")

# For retrieval: prepend instruction to QUERY only (not documents)
query_instruction = "Represent this sentence for searching relevant passages: "

query = f"{query_instruction}What are NSE circuit breaker rules?"
query_embedding = model.encode(query)

# Documents encoded WITHOUT instruction prefix
doc_embedding = model.encode("NSE implements circuit breakers at 10%, 15%, and 20% levels...")

similarity = cosine_similarity([query_embedding], [doc_embedding])[0][0]
```

Instruction-tuned models achieve top MTEB scores because the instruction guides the representation to be task-specific.

---

**Q10: What is the impact of chunk overlap on embedding quality?**

**Answer:**
- **No overlap**: Context at chunk boundaries is lost. A sentence split across two chunks is poorly represented in both.
- **Too much overlap (>30%)**: Redundant vectors, larger index, similar chunks compete in search results.
- **Optimal overlap (10-15%)**: Ensures sentence continuity across boundaries.

```python
# Test overlap impact empirically
def evaluate_overlap_strategy(test_queries: list, overlap_sizes: list):
    results = {}
    for overlap in overlap_sizes:
        splitter = RecursiveCharacterTextSplitter(chunk_size=512, chunk_overlap=overlap)
        chunks = splitter.split_documents(nse_docs)
        vectorstore = build_vectorstore(chunks)
        
        recall_scores = []
        for query, expected_answer in test_queries:
            retrieved = vectorstore.similarity_search(query, k=5)
            recall = compute_recall(retrieved, expected_answer)
            recall_scores.append(recall)
        
        results[overlap] = np.mean(recall_scores)
    
    return results  # e.g., {0: 0.72, 64: 0.81, 128: 0.80, 256: 0.77}
```

---

**Q11-Q25 (Condensed):**

**Q11:** Word2Vec vs GloVe vs contextual embeddings — static (same vector regardless of context) vs dynamic (BERT gives different vectors for "bank" in financial vs river context).

**Q12:** Sentence-BERT training with NLI dataset — fine-tuned on SNLI/MultiNLI so entailed sentences have similar embeddings.

**Q13:** ColBERT (late interaction) — stores per-token embeddings for each document, computes MaxSim at query time. Better quality than bi-encoder, cheaper than cross-encoder.

**Q14:** Embedding model quantization — INT8 quantize the embedding model for 4x faster inference with minimal quality loss.

**Q15:** Batching embeddings for throughput:
```python
# Embed in batches for efficiency
def batch_embed(texts: list[str], batch_size: int = 100) -> list[list[float]]:
    all_embeddings = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        embeddings = model.encode(batch, batch_size=batch_size, show_progress_bar=True)
        all_embeddings.extend(embeddings.tolist())
    return all_embeddings
```

**Q16:** Embedding drift — if you update the embedding model, old vectors are incompatible; track model version alongside each embedding.

**Q17:** Negative mining for fine-tuning — hard negatives (semantically similar but different meaning) improve embedding quality dramatically vs random negatives.

**Q18:** Siamese network architecture for embedding fine-tuning.

**Q19:** BM25 vs embeddings for financial text — BM25 wins on exact term matching (ticker symbols, regulation IDs), embeddings win on semantic understanding. Hybrid approach is best.

**Q20:** Embedding documents with tables — flatten tables to text: "Row 1: Symbol=RELIANCE, Date=2024-06-01, Close=3200..."

**Q21:** Long document embedding strategies — chunk + embed vs hierarchical (chunk embeddings → document-level embedding via pooling).

**Q22:** Domain-adaptive pre-training — continue pre-training BERT on NSE/SEBI corpus before fine-tuning for embeddings (FinBERT approach).

**Q23:** Contrastive learning for NSE data — create training pairs: (NSE circular text, Q&A about that circular) as positives; different circulars as negatives.

**Q24:** Sparse-dense hybrid embeddings — SPLADE for sparse, BGE for dense — combine for best hybrid search.

**Q25:** Reranker vs embedding model selection — embedding model dominates recall (getting relevant docs into candidates); reranker dominates precision (ranking them correctly). Invest in both.

---

### Must-Read Study Resources
1. **MTEB Leaderboard** — [huggingface.co/spaces/mteb/leaderboard](https://huggingface.co/spaces/mteb/leaderboard) — Live benchmark rankings
2. **Sentence-Transformers Docs** — [sbert.net](https://www.sbert.net) — Bi-encoders, fine-tuning, cross-encoders
3. **BGE/FlagEmbedding Paper** — [arxiv.org/abs/2309.07597](https://arxiv.org/abs/2309.07597) — State-of-art embedding training techniques

---

## CORE FULL-STACK TOPICS

## Node.js / Express

### Common Interview Questions (25)

**Q1: Write an Express.js REST API with JWT auth, global error handling, Zod validation, rate limiting, and async error propagation.**

**Answer:**
```typescript
// server.ts
import express, { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { z, ZodError } from 'zod';
import rateLimit from 'express-rate-limit';
import { promisify } from 'util';

const app = express();
app.use(express.json());

// ============================================================
// 1. RATE LIMITING
// ============================================================
const apiLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,  // 15 minutes
    max: 100,                    // 100 requests per window
    standardHeaders: true,
    legacyHeaders: false,
    message: { error: 'Too many requests, please try again later.' },
    keyGenerator: (req) => req.ip ?? 'unknown',
});

app.use('/api/', apiLimiter);

// ============================================================
// 2. JWT AUTHENTICATION MIDDLEWARE
// ============================================================
interface JWTPayload {
    userId: string;
    role: 'admin' | 'analyst' | 'viewer';
    iat: number;
    exp: number;
}

declare global {
    namespace Express {
        interface Request {
            user?: JWTPayload;
        }
    }
}

const JWT_SECRET = process.env.JWT_SECRET!;
const verifyJwt = promisify<string, string, JWTPayload>(
    jwt.verify as (token: string, secret: string, cb: (err: Error | null, decoded: JWTPayload) => void) => void
);

const authenticate = async (req: Request, res: Response, next: NextFunction) => {
    const authHeader = req.headers.authorization;
    
    if (!authHeader?.startsWith('Bearer ')) {
        return res.status(401).json({ error: 'Missing or invalid authorization header' });
    }
    
    const token = authHeader.slice(7);
    
    try {
        const payload = await verifyJwt(token, JWT_SECRET);
        req.user = payload;
        next();
    } catch (error) {
        if (error instanceof jwt.TokenExpiredError) {
            return res.status(401).json({ error: 'Token expired' });
        }
        return res.status(401).json({ error: 'Invalid token' });
    }
};

const authorize = (roles: string[]) => (req: Request, res: Response, next: NextFunction) => {
    if (!req.user || !roles.includes(req.user.role)) {
        return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
};

// ============================================================
// 3. ZOD REQUEST VALIDATION
// ============================================================
const validate = (schema: z.ZodSchema) => async (req: Request, res: Response, next: NextFunction) => {
    try {
        // Validate body, params, and query simultaneously
        req.body = await schema.parseAsync(req.body);
        next();
    } catch (error) {
        next(error);  // Pass to global error handler
    }
};

const NSEQuerySchema = z.object({
    symbol: z.string().min(1).max(20).toUpperCase(),
    metric: z.enum(['price', 'volume', 'pe_ratio', 'market_cap']),
    startDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(),
    endDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(),
}).refine(
    (data) => !data.startDate || !data.endDate || data.startDate <= data.endDate,
    { message: 'startDate must be before endDate', path: ['startDate'] }
);

// ============================================================
// 4. ASYNC
