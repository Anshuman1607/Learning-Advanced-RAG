# PRP: Codebase Documentation RAG System
### Product Requirements Prompt — Senior Architect Edition

---

## TABLE OF CONTENTS

1. High-Level Architecture Explanation
2. End-to-End Workflow
3. Component Responsibilities
4. Folder Structure Explanation
5. API Design
6. Data Flow Diagrams (Text-Based)
7. Celery + Redis Workflow Explanation
8. Retrieval Pipeline Explanation
9. Re-Ranking Explanation
10. Error Handling Strategy
11. Scalability Considerations
12. Future Improvements
13. Security Considerations
14. Deployment Architecture
15. Step-by-Step Implementation Roadmap
16. Suggested Development Order
17. Resume-Worthy Project Highlights
18. Interview Questions from This Project
19. Metrics to Evaluate the RAG System
20. Common Failure Cases and Solutions

---

## 1. HIGH-LEVEL ARCHITECTURE EXPLANATION

### What Is This System?

This is a **production-grade Retrieval-Augmented Generation (RAG)** system that lets developers upload technical documentation (README files, markdown docs, API references) and ask natural language questions about their codebase. Instead of asking an LLM to hallucinate answers from memory, the system retrieves the most relevant text chunks from uploaded documents and passes them as grounded context to the LLM before generating a response.

### Why RAG?

LLMs have fixed knowledge cutoffs and cannot know about your private codebase. RAG solves this by:
- Storing your documentation in a vector database
- Retrieving relevant passages at query time
- Injecting them into the LLM prompt as context
- Generating answers grounded in your actual documentation

### Two Core Subsystems

**Ingestion Pipeline** — Takes uploaded files and processes them into a searchable index:
- Parses documents into text chunks
- Converts chunks to dense vector embeddings (OpenAI)
- Stores embeddings in Qdrant (vector DB)
- Builds a sparse BM25 index for keyword search
- Runs fully asynchronously via Celery + Redis

**Query Pipeline** — Answers user questions using hybrid retrieval:
- Runs dense vector search (Qdrant) and sparse BM25 search in parallel
- Fuses results using Reciprocal Rank Fusion (RRF)
- Re-ranks top candidates using a cross-encoder model
- Passes top 5 chunks to OpenAI for generation
- Streams the response token-by-token via Server-Sent Events (SSE)

### The "Hybrid" Advantage

Using only dense (vector) retrieval misses exact keyword matches. Using only sparse (BM25) retrieval misses semantic similarity. Combining both gives you the best of both worlds — this is the industry standard approach in production RAG systems.

---

## 2. END-TO-END WORKFLOW

### Ingestion Workflow

```
Developer uploads file (POST /ingest)
  │
  ├── FastAPI receives file, validates it
  ├── Saves file to temp storage
  ├── Creates a job record in Redis (status: "pending")
  ├── Dispatches Celery task → returns job_id immediately
  │
  └── Celery Worker picks up the task:
        │
        ├── LangChain Document Loader reads file
        ├── RecursiveCharacterTextSplitter chunks it
        │     └── chunk_size=500, overlap=50
        │     └── Metadata: filename, chunk_index, source
        ├── OpenAI Embeddings API generates dense vectors
        ├── Vectors stored in Qdrant collection
        ├── BM25 index built from chunk text
        ├── BM25 index serialized and saved (Redis or disk)
        └── Job status updated → "completed"
```

### Query Workflow

```
Developer sends question (POST /chat)
  │
  ├── FastAPI receives session_id + query
  ├── Retrieves conversation history from Redis (by session_id)
  ├── Runs Dense Retrieval:
  │     └── Embed query → search Qdrant → top 20 chunks
  ├── Runs Sparse Retrieval:
  │     └── Tokenize query → BM25 score all chunks → top 20 chunks
  ├── Reciprocal Rank Fusion (RRF):
  │     └── Merge 20+20 candidates → unified ranked list
  ├── Cross-Encoder Re-ranking:
  │     └── Score each (query, chunk) pair → select top 5
  ├── Context Construction:
  │     └── Concatenate top 5 chunks + conversation history
  ├── LLM Generation (OpenAI):
  │     └── Stream response token-by-token
  └── SSE stream returned to client
```

---

## 3. COMPONENT RESPONSIBILITIES

### FastAPI (API Layer)
- Handles HTTP requests and responses
- Validates input (file type, size, required fields)
- Delegates work to services — does NOT contain business logic
- Manages SSE streaming connections
- Returns immediate responses (job_id for ingestion)

### Celery Workers (Async Processing)
- Execute the long-running ingestion tasks
- Decouple file processing from HTTP request lifecycle
- Enable retry logic for failed tasks
- Allow horizontal scaling (run more workers when needed)

### Redis (Message Broker + State Store)
- Acts as Celery's task queue (broker)
- Stores task results (result backend)
- Stores job status (pending/processing/completed/failed)
- Stores conversation history per session_id
- Stores serialized BM25 index

### Qdrant (Vector Database)
- Stores dense vector embeddings (768 or 1536 dimensions)
- Enables approximate nearest neighbor (ANN) search
- Supports metadata filtering (filter by filename, date, etc.)
- Persists data across restarts
- Can scale to millions of vectors

### LangChain (Document Processing)
- Used narrowly: loading, chunking, embedding wrappers, LLM wrappers
- RecursiveCharacterTextSplitter handles markdown structure intelligently
- OpenAIEmbeddings wraps the embedding API call
- ChatOpenAI wraps the generation API call with streaming

### rank-bm25 (Sparse Retrieval)
- Implements BM25 (Best Match 25) keyword scoring algorithm
- Fast, interpretable, great for exact term matches
- Custom-implemented retrieval (not delegated to LangChain)
- Index is built from corpus of chunks at ingestion time

### sentence-transformers Cross-Encoder (Re-ranking)
- Takes (query, chunk) pairs and scores their relevance
- More accurate than bi-encoder embeddings (full attention)
- Slower — used only on the 20 fused candidates, not the full index
- model: cross-encoder/ms-marco-MiniLM-L-6-v2

### OpenAI API (LLM)
- Generates the final answer using provided context
- Streamed via the streaming=True parameter
- Receives: system prompt + conversation history + retrieved context + user query

---

## 4. FOLDER STRUCTURE EXPLANATION

```
app/
│
├── api/                        # HTTP Layer
│   ├── routes/
│   │   ├── ingest.py           # POST /ingest endpoint
│   │   ├── chat.py             # POST /chat endpoint
│   │   ├── status.py           # GET /status/{job_id}
│   │   └── health.py           # GET /health
│   └── dependencies.py         # Shared FastAPI dependencies (auth, rate limiting)
│
├── services/                   # Business Logic Layer
│   ├── ingestion_service.py    # Orchestrates ingestion: save file, dispatch task
│   ├── query_service.py        # Orchestrates query: retrieval → rerank → generate
│   ├── embedding_service.py    # Wraps OpenAI embedding calls
│   └── chat_history_service.py # Manages session history in Redis
│
├── pipelines/                  # Core RAG Logic
│   ├── ingestion_pipeline.py   # End-to-end ingestion: load → chunk → embed → store
│   └── query_pipeline.py       # End-to-end query: retrieve → fuse → rerank → generate
│
├── workers/                    # Celery Task Definitions
│   ├── celery_app.py           # Celery app initialization
│   └── tasks.py                # @celery.task definitions (wraps pipeline calls)
│
├── db/                         # Database Clients
│   ├── qdrant_client.py        # Qdrant connection and collection management
│   └── redis_client.py         # Redis connection and key management
│
├── utils/                      # Shared Utilities
│   ├── rrf.py                  # Reciprocal Rank Fusion implementation
│   ├── bm25_utils.py           # BM25 index build, serialization, retrieval
│   ├── reranker.py             # Cross-encoder re-ranking logic
│   └── chunker.py              # LangChain splitter wrapper
│
├── core/                       # Application Configuration
│   ├── config.py               # Settings via pydantic-settings (env vars)
│   └── logging.py              # Structured logging setup
│
└── models/                     # Pydantic Data Models
    ├── request_models.py       # ChatRequest, IngestResponse, etc.
    └── response_models.py      # StatusResponse, StreamEvent, etc.
```

### Why This Structure?

- **api/** is thin — it just routes requests to services. No business logic here.
- **services/** owns orchestration. Each service has one responsibility.
- **pipelines/** owns the RAG-specific logic. This is the intellectual core of the system.
- **workers/** is isolated so Celery can import tasks without importing the entire app.
- **db/** wraps all external DB clients so they can be swapped or mocked in tests.
- **utils/** contains pure, stateless functions (RRF, BM25, reranking) — fully unit-testable.
- **core/** keeps config in one place, using environment variables for everything secret.
- **models/** defines all data contracts. Pydantic catches malformed data at the boundary.

---

## 5. API DESIGN

### POST /ingest

**Purpose:** Upload a documentation file for asynchronous ingestion.

**Request:**
```
Content-Type: multipart/form-data
Body: file (binary), collection_name (optional string)
```

**Response (202 Accepted):**
```json
{
  "job_id": "a3f8c2d1-...",
  "status": "pending",
  "message": "File accepted. Processing started."
}
```

**Design Decisions:**
- Returns `202 Accepted` (not `200`) — the work is not done yet
- `job_id` is a UUID so clients can poll status
- File is saved to temp storage before dispatching the Celery task (prevents task from failing if the web process exits)

---

### GET /status/{job_id}

**Purpose:** Check the status of an ingestion job.

**Response (200 OK):**
```json
{
  "job_id": "a3f8c2d1-...",
  "status": "completed",    // pending | processing | completed | failed
  "num_chunks": 42,
  "error": null             // Populated on failure
}
```

---

### POST /chat

**Purpose:** Ask a question. Returns a streaming response.

**Request:**
```json
{
  "session_id": "user-session-abc",
  "query": "How does the authentication middleware work?"
}
```

**Response (200 OK, text/event-stream):**
```
data: {"token": "The "}
data: {"token": "authentication "}
data: {"token": "middleware "}
...
data: {"token": "[DONE]"}
```

**Design Decisions:**
- `session_id` enables conversational memory across turns
- SSE is used instead of WebSockets — simpler, stateless, HTTP-compatible
- `[DONE]` sentinel token signals stream completion to the client

---

### GET /health

**Purpose:** Liveness/readiness probe for load balancers and orchestrators.

**Response (200 OK):**
```json
{
  "status": "ok",
  "qdrant": "connected",
  "redis": "connected",
  "celery": "connected"
}
```

---

## 6. DATA FLOW DIAGRAMS (TEXT-BASED)

### Ingestion Data Flow

```
┌─────────────┐     HTTP POST      ┌─────────────┐
│   Client    │ ─────────────────► │   FastAPI   │
└─────────────┘                    └──────┬──────┘
                                          │ save file
                                          ▼
                                   ┌─────────────┐
                                   │ Temp Storage│
                                   └──────┬──────┘
                                          │ dispatch task
                                          ▼
                                   ┌─────────────┐
       ┌──── job_id ◄──────────── │    Redis    │ ◄─── Celery broker
       │                           └──────┬──────┘
       ▼                                  │
┌─────────────┐                    ┌──────▼──────┐
│   Client    │                    │   Celery    │
└─────────────┘                    │   Worker    │
                                   └──────┬──────┘
                                          │
                             ┌────────────┼────────────┐
                             ▼            ▼             ▼
                      ┌──────────┐ ┌──────────┐ ┌──────────┐
                      │LangChain │ │  OpenAI  │ │  Qdrant  │
                      │ Loader + │ │Embeddings│ │ (Dense)  │
                      │ Chunker  │ │   API    │ │ Storage  │
                      └──────────┘ └──────────┘ └──────────┘
                                                      │
                                               ┌──────▼──────┐
                                               │  BM25 Index │
                                               │  (Redis)    │
                                               └─────────────┘
```

### Query Data Flow

```
┌─────────────┐    POST /chat     ┌─────────────┐
│   Client    │ ────────────────► │   FastAPI   │
└─────────────┘                   └──────┬──────┘
                                         │
                            ┌────────────┤
                            ▼            ▼
                     ┌──────────┐ ┌──────────┐
                     │  Qdrant  │ │  BM25    │
                     │  Dense   │ │  Sparse  │
                     │ Retrieval│ │ Retrieval│
                     └────┬─────┘ └────┬─────┘
                          │  top 20    │  top 20
                          └─────┬──────┘
                                ▼
                         ┌─────────────┐
                         │     RRF     │
                         │   Fusion    │
                         └──────┬──────┘
                                │ fused top 20
                                ▼
                         ┌─────────────┐
                         │Cross-Encoder│
                         │  Re-ranker  │
                         └──────┬──────┘
                                │ top 5 chunks
                                ▼
                         ┌─────────────┐
                         │   OpenAI    │
                         │    LLM      │
                         └──────┬──────┘
                                │ SSE stream
                                ▼
                         ┌─────────────┐
                         │   Client    │
                         └─────────────┘
```

---

## 7. CELERY + REDIS WORKFLOW EXPLANATION

### Why Celery?

File ingestion involves multiple slow operations: reading the file, calling the OpenAI Embeddings API, writing to Qdrant. Doing this synchronously inside the HTTP request would:
- Block the web server thread for 10–60+ seconds
- Risk timeouts at the client or load balancer
- Make the system unscalable under concurrent uploads

Celery solves this by executing tasks in separate worker processes.

### How Celery + Redis Work Together

```
Web Process (FastAPI)          Redis              Worker Process (Celery)
─────────────────────          ─────              ──────────────────────
1. Receive file upload
2. Save file to disk
3. Create task message ─────► Queue: [task1]
4. Return job_id ◄───────────  Store: job1=pending

                                                   5. Dequeue task1
                                                   6. Execute ingestion
                                                   7. Update status ────► Store: job1=processing
                                                   8. Complete
                                                   9. Update status ────► Store: job1=completed

Client polling:
GET /status/job1 ──────────── Read: job1=completed ──► Return to client
```

### Key Celery Configuration

```python
# celery_app.py
from celery import Celery

celery = Celery(
    "rag_worker",
    broker="redis://localhost:6379/0",      # Task queue
    backend="redis://localhost:6379/1",     # Result storage
    include=["app.workers.tasks"]
)

celery.conf.update(
    task_serializer="json",
    result_expires=3600,           # Results expire after 1 hour
    task_acks_late=True,           # Only ack after task completes (safer)
    worker_prefetch_multiplier=1,  # One task per worker at a time
    task_max_retries=3,            # Retry failed tasks 3 times
    task_retry_backoff=True        # Exponential backoff on retry
)
```

### Task Definition Pattern

```python
# workers/tasks.py
@celery.task(bind=True, max_retries=3)
def ingest_document_task(self, file_path: str, job_id: str):
    try:
        redis_client.set(f"job:{job_id}:status", "processing")
        result = ingestion_pipeline.run(file_path)
        redis_client.set(f"job:{job_id}:status", "completed")
        redis_client.set(f"job:{job_id}:chunks", result["num_chunks"])
    except Exception as exc:
        redis_client.set(f"job:{job_id}:status", "failed")
        redis_client.set(f"job:{job_id}:error", str(exc))
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)
```

---

## 8. RETRIEVAL PIPELINE EXPLANATION

### Dense Retrieval (Qdrant)

At ingestion time, each chunk is converted to a high-dimensional vector using OpenAI's `text-embedding-3-small` (1536 dimensions). These vectors encode *semantic meaning* — chunks about "authentication" and "login" will have similar vectors even without shared keywords.

At query time:
1. Embed the user's query using the same model
2. Search Qdrant using cosine similarity
3. Return the 20 most similar chunks

```python
# pipelines/query_pipeline.py
def dense_retrieve(query: str, top_k: int = 20) -> list[ScoredChunk]:
    query_vector = embedding_service.embed(query)
    results = qdrant_client.search(
        collection_name="docs",
        query_vector=query_vector,
        limit=top_k,
        with_payload=True
    )
    return [ScoredChunk(id=r.id, text=r.payload["text"], score=r.score) for r in results]
```

### Sparse Retrieval (BM25)

BM25 (Best Match 25) is a classical information retrieval algorithm. It scores documents based on:
- **Term frequency (TF):** How often does the query term appear in the chunk?
- **Inverse document frequency (IDF):** How rare is that term across all chunks?
- **Document length normalization:** Penalizes very long chunks

BM25 excels at exact keyword matches — if a developer asks about `AuthMiddleware`, BM25 will find chunks containing that exact class name even if semantically dissimilar content is nearby.

```python
# utils/bm25_utils.py
from rank_bm25 import BM25Okapi

def build_index(chunks: list[str]) -> BM25Okapi:
    tokenized = [chunk.lower().split() for chunk in chunks]
    return BM25Okapi(tokenized)

def retrieve(index: BM25Okapi, corpus: list[str], query: str, top_k: int = 20) -> list[ScoredChunk]:
    tokens = query.lower().split()
    scores = index.get_scores(tokens)
    top_indices = scores.argsort()[-top_k:][::-1]
    return [ScoredChunk(id=i, text=corpus[i], score=scores[i]) for i in top_indices]
```

### Reciprocal Rank Fusion (RRF)

RRF is a rank aggregation algorithm. Instead of using raw scores (which are on different scales between dense and sparse), it uses the *rank position* of each result:

```
RRF_score(doc) = Σ 1 / (k + rank_in_retriever)
```

Where `k=60` is a constant that dampens the influence of top-ranked documents.

This is robust because it doesn't require score normalization and handles disagreements between retrievers gracefully.

```python
# utils/rrf.py
def reciprocal_rank_fusion(
    dense_results: list[ScoredChunk],
    sparse_results: list[ScoredChunk],
    k: int = 60
) -> list[ScoredChunk]:
    scores = {}
    for rank, chunk in enumerate(dense_results):
        scores[chunk.id] = scores.get(chunk.id, 0) + 1 / (k + rank + 1)
    for rank, chunk in enumerate(sparse_results):
        scores[chunk.id] = scores.get(chunk.id, 0) + 1 / (k + rank + 1)
    sorted_ids = sorted(scores, key=scores.get, reverse=True)
    # Return top 20 unique chunks
    all_chunks = {c.id: c for c in dense_results + sparse_results}
    return [all_chunks[id] for id in sorted_ids[:20]]
```

---

## 9. RE-RANKING EXPLANATION

### Why Re-rank?

Bi-encoder retrieval (the dense vector search) is fast because it pre-computes embeddings for all chunks. However, it encodes the query and document *separately* — there's no direct interaction between their tokens. This can miss subtle relevance signals.

A **cross-encoder** reads the query and document *together*, allowing full attention between all tokens. This is significantly more accurate but too slow to run on thousands of documents — which is why we use it only on the 20 fused candidates.

### Cross-Encoder Pipeline

```python
# utils/reranker.py
from sentence_transformers import CrossEncoder

class Reranker:
    def __init__(self):
        self.model = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

    def rerank(self, query: str, chunks: list[ScoredChunk], top_k: int = 5) -> list[ScoredChunk]:
        pairs = [(query, chunk.text) for chunk in chunks]
        scores = self.model.predict(pairs)  # Returns relevance scores for each pair
        ranked = sorted(zip(scores, chunks), key=lambda x: x[0], reverse=True)
        return [chunk for _, chunk in ranked[:top_k]]
```

### Two-Stage Retrieval Pattern

```
Stage 1 — Recall: Retrieve broadly (BM25 + Dense → RRF → 20 candidates)
  Goal: Don't miss anything relevant. High recall, lower precision.

Stage 2 — Precision: Re-rank precisely (Cross-Encoder → top 5)
  Goal: Surface the best 5 chunks for the LLM. High precision.
```

This mirrors how human information retrieval works: first scan broadly, then read carefully.

---

## 10. ERROR HANDLING STRATEGY

### Layer-by-Layer Error Handling

**API Layer (FastAPI):**
```python
@app.exception_handler(RequestValidationError)
async def validation_error_handler(request, exc):
    return JSONResponse(status_code=422, content={"detail": exc.errors()})

@app.exception_handler(Exception)
async def global_error_handler(request, exc):
    logger.error(f"Unhandled error: {exc}", exc_info=True)
    return JSONResponse(status_code=500, content={"detail": "Internal server error"})
```

**Celery Tasks:**
- `task_acks_late=True`: Tasks are only removed from the queue after completion. If a worker crashes mid-task, the task is re-queued.
- `max_retries=3` with exponential backoff: Transient failures (network blips, OpenAI rate limits) are retried automatically.
- On final failure: status set to "failed" with error message for client polling.

**External Service Failures:**

| Failure | Detection | Response |
|---|---|---|
| OpenAI API rate limit | 429 status code | Retry with backoff |
| OpenAI API unavailable | Connection error | Retry 3x, mark job failed |
| Qdrant timeout | Timeout exception | Retry, return partial results |
| Redis connection lost | ConnectionError | Fast-fail, return 503 |
| Invalid file type | Pydantic validation | Return 422 immediately |
| File too large | Content-Length check | Return 413 |

**Streaming Errors:**

```python
async def stream_response(query_pipeline):
    try:
        async for token in query_pipeline.astream():
            yield f"data: {json.dumps({'token': token})}\n\n"
    except Exception as e:
        yield f"data: {json.dumps({'error': str(e)})}\n\n"
    finally:
        yield f"data: {json.dumps({'token': '[DONE]'})}\n\n"
```

**Graceful Degradation:**
- If BM25 index fails to load: fall back to dense-only retrieval (log warning)
- If re-ranker fails: fall back to RRF-ranked results
- If conversation history is unavailable: proceed with no history (log warning)

---

## 11. SCALABILITY CONSIDERATIONS

### Horizontal Scaling of Celery Workers

The most straightforward scaling axis. When ingestion load increases, spin up additional worker instances. All workers share the same Redis queue — work is automatically distributed.

```bash
# Start 4 workers
celery -A app.workers.celery_app worker --concurrency=4 --loglevel=info
```

In production (Kubernetes): define a Celery worker Deployment and scale replicas.

### Qdrant Scalability

Qdrant supports:
- **Distributed mode**: Shard collections across multiple nodes
- **Collections**: Isolate documents per tenant/project
- **Filtering**: Add metadata filters to retrieval (e.g., only search `filename="README.md"`)

For very large corpora (millions of chunks), enable HNSW index tuning:
```python
qdrant_client.create_collection(
    collection_name="docs",
    vectors_config=VectorParams(size=1536, distance=Distance.COSINE),
    hnsw_config=HnswConfigDiff(m=16, ef_construct=100)  # Tune for recall/speed tradeoff
)
```

### BM25 Scalability Concern

The current in-memory BM25 approach doesn't scale to millions of documents. Solutions:
- **Short term**: Store serialized index in Redis, reload per-request
- **Medium term**: Use Elasticsearch or OpenSearch as a scalable sparse retrieval engine
- **Long term**: Qdrant's native sparse vector support (can replace BM25 entirely)

### FastAPI Scalability

- Run multiple Uvicorn workers: `uvicorn app.main:app --workers 4`
- Behind a load balancer (Nginx, AWS ALB)
- No shared in-memory state — all state in Redis/Qdrant

### Multi-Tenancy

Add a `namespace` or `user_id` field to all Qdrant payloads and filter on it at retrieval time. Use separate BM25 indices per namespace in Redis.

---

## 12. FUTURE IMPROVEMENTS

### Capability Enhancements

- **PDF and HTML support**: Add LangChain loaders for PDFs, HTML pages, and Jupyter notebooks
- **Incremental ingestion**: Detect file changes via hash comparison — only re-index changed chunks
- **Chunk deduplication**: Hash chunks before storing to prevent duplicates from re-uploads
- **Query expansion**: Use the LLM to generate related search queries before retrieval (HyDE — Hypothetical Document Embeddings)
- **Multi-modal support**: Ingest diagrams and screenshots using vision models, index their descriptions

### Retrieval Improvements

- **Contextual retrieval** (Anthropic's technique): Prepend chunk context summaries before embedding — dramatically improves retrieval accuracy
- **Parent-child chunking**: Store small chunks for retrieval precision but return surrounding context for LLM generation
- **Qdrant sparse vectors**: Replace BM25 with Qdrant's native sparse vector support for unified hybrid retrieval
- **Adaptive retrieval**: Dynamically tune `top_k` based on query complexity

### Infrastructure Improvements

- **Observability**: Add OpenTelemetry tracing to track latency across each pipeline stage
- **Evaluation pipeline**: Automated RAGAS evaluation on a golden question set after each deployment
- **Caching**: Cache embedding results for repeated queries; cache re-ranking results for popular questions
- **Async Qdrant client**: Use `AsyncQdrantClient` to avoid blocking the event loop during retrieval

---

## 13. SECURITY CONSIDERATIONS

### Authentication and Authorization

- Add API key authentication to all endpoints (FastAPI `Depends()` with header extraction)
- Namespace all Qdrant collections and BM25 indices by `user_id` to prevent cross-tenant data leakage
- JWT-based session management for the `/chat` endpoint

### File Upload Security

- Validate MIME type server-side (not just file extension — these can be spoofed)
- Enforce maximum file size (e.g., 10 MB)
- Scan for malicious content (especially for production deployments with untrusted users)
- Store files in isolated temp directories with randomized names
- Delete temp files after ingestion completes

### Secrets Management

- All API keys (OpenAI, Qdrant) in environment variables — never hardcoded
- Use Pydantic `BaseSettings` to load from `.env` with type validation
- In production: use AWS Secrets Manager, Vault, or Kubernetes Secrets

### Rate Limiting

- Apply per-IP rate limits on `/chat` (SSE connections are expensive)
- Apply per-user rate limits on `/ingest` to prevent abuse
- Use Redis-backed rate limiting (e.g., `slowapi` library)

### Data Privacy

- Do not log uploaded file contents in application logs
- Implement data retention policies — auto-delete old collections
- Consider end-to-end encryption for sensitive codebases

---

## 14. DEPLOYMENT ARCHITECTURE

### Local Development

```
docker-compose up
  ├── fastapi app (port 8000)
  ├── celery worker
  ├── redis (port 6379)
  └── qdrant (port 6333)
```

### Production (Kubernetes)

```
Internet
    │
    ▼
[Nginx Ingress / Load Balancer]
    │
    ├── [FastAPI Deployment]      ← 2-4 replicas, HPA on CPU/memory
    │       └── Service: ClusterIP
    │
    ├── [Celery Worker Deployment] ← 2-N replicas, scale by queue depth
    │
    ├── [Redis StatefulSet]        ← Redis Cluster or ElastiCache
    │
    └── [Qdrant StatefulSet]       ← Persistent volume, or Qdrant Cloud
```

### CI/CD Pipeline

```
Git Push
  → GitHub Actions / GitLab CI
    → Lint (ruff, mypy)
    → Unit Tests (pytest)
    → Build Docker Image
    → Push to ECR/GCR
    → Deploy to Staging
    → Integration Tests
    → Deploy to Production (blue/green)
```

### docker-compose.yml Skeleton

```yaml
version: "3.9"
services:
  api:
    build: .
    ports: ["8000:8000"]
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - REDIS_URL=redis://redis:6379/0
      - QDRANT_URL=http://qdrant:6333
    depends_on: [redis, qdrant]

  worker:
    build: .
    command: celery -A app.workers.celery_app worker --loglevel=info
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - REDIS_URL=redis://redis:6379/0
      - QDRANT_URL=http://qdrant:6333
    depends_on: [redis, qdrant]

  redis:
    image: redis:7-alpine
    volumes: [redis_data:/data]

  qdrant:
    image: qdrant/qdrant:latest
    volumes: [qdrant_data:/qdrant/storage]
    ports: ["6333:6333"]
```

---

## 15. STEP-BY-STEP IMPLEMENTATION ROADMAP

### Phase 1: Foundation (Days 1–3)

1. Initialize FastAPI project with `pyproject.toml` / `requirements.txt`
2. Set up `docker-compose.yml` with Redis and Qdrant
3. Implement `core/config.py` with Pydantic BaseSettings
4. Implement `core/logging.py` with structured JSON logging
5. Implement `GET /health` endpoint verifying Redis and Qdrant connectivity
6. Write smoke tests for all external connections

### Phase 2: Ingestion Pipeline (Days 4–7)

7. Implement `db/qdrant_client.py` — collection create, upsert, search
8. Implement `utils/chunker.py` — wrap LangChain RecursiveCharacterTextSplitter
9. Implement `services/embedding_service.py` — wrap OpenAI embeddings
10. Implement `pipelines/ingestion_pipeline.py` — end-to-end: load → chunk → embed → store
11. Test pipeline directly (no async yet) on a sample markdown file
12. Implement `utils/bm25_utils.py` — build and serialize BM25 index
13. Implement `workers/tasks.py` with Celery task wrapping the pipeline
14. Implement `POST /ingest` — save file, dispatch task, return job_id
15. Implement `GET /status/{job_id}` — read status from Redis
16. End-to-end test: upload file, poll status, verify chunks in Qdrant

### Phase 3: Query Pipeline (Days 8–12)

17. Implement `utils/rrf.py` — Reciprocal Rank Fusion
18. Implement `utils/reranker.py` — cross-encoder scoring
19. Implement `db/redis_client.py` — conversation history get/set
20. Implement `services/chat_history_service.py`
21. Implement `pipelines/query_pipeline.py` — full retrieval → rerank → generate
22. Test pipeline with direct Python calls, verify chunk quality
23. Implement `POST /chat` with SSE streaming
24. Test SSE: verify token-by-token streaming in curl and browser

### Phase 4: Hardening (Days 13–15)

25. Add input validation (file size, type, query length)
26. Add error handling at all layers (global handler, task retry logic)
27. Add rate limiting with `slowapi`
28. Write unit tests for `rrf.py`, `bm25_utils.py`, `reranker.py`
29. Write integration tests for ingest and query pipelines
30. Set up pre-commit hooks (ruff, mypy, pytest)

### Phase 5: Production Prep (Days 16–18)

31. Write `Dockerfile` with multi-stage build
32. Finalize `docker-compose.yml`
33. Write `README.md` with architecture diagrams and setup instructions
34. Add OpenTelemetry spans to ingestion and query pipelines
35. Document all API endpoints with OpenAPI/Swagger annotations
36. Run RAGAS evaluation on 10 golden Q&A pairs

---

## 16. SUGGESTED DEVELOPMENT ORDER

Build in this order to maximize testability at each step:

```
1. Infrastructure first (Redis, Qdrant, config, health)
   → Reason: All subsequent steps depend on these being stable.

2. Ingestion pipeline (synchronous, no Celery yet)
   → Reason: Test the core logic before adding async complexity.

3. Add Celery async wrapper around ingestion
   → Reason: Add one layer of complexity at a time.

4. Query pipeline (without streaming)
   → Reason: Verify retrieval quality before building the response layer.

5. Add SSE streaming
   → Reason: Streaming is a UI concern — get the logic right first.

6. Add conversation memory
   → Reason: Nice-to-have that builds on the stable query pipeline.

7. Error handling, tests, Docker
   → Reason: Harden what works before shipping.
```

**The principle:** At every step, the system should be runnable and testable. Avoid building multiple untested layers at once.

---

## 17. RESUME-WORTHY PROJECT HIGHLIGHTS

### What to Say on Your Resume

> *"Built a production-grade RAG system for codebase documentation Q&A. Implemented hybrid retrieval combining dense vector search (Qdrant) and BM25, fused with Reciprocal Rank Fusion and cross-encoder re-ranking. Designed asynchronous ingestion via Celery + Redis, enabling sub-second API responses for file uploads. Delivered streaming responses via SSE and maintained conversational memory across sessions."*

### Technical Differentiators Worth Calling Out

- **Custom RRF implementation** — shows you understand retrieval theory, not just library calls
- **Two-stage retrieval with re-ranking** — production pattern used by Cohere, Weaviate, and top RAG systems
- **Asynchronous job queue** — demonstrates real production engineering (Celery + Redis is a standard stack)
- **Streaming SSE responses** — shows frontend/backend integration knowledge
- **Hybrid retrieval** — most toy RAG demos only use dense search; this shows sophistication
- **Modular pipeline architecture** — shows software engineering maturity

### Portfolio Presentation Tips

- Include a live demo recording showing: upload → poll status → ask questions → stream answer
- Visualize retrieval: show which chunks were retrieved and their scores
- Include evaluation metrics (RAGAS scores) in the README
- Open-source it with a clean README explaining every architectural decision

---

## 18. INTERVIEW QUESTIONS FROM THIS PROJECT

### System Design Questions

1. **"Walk me through how you'd design a RAG system from scratch."**
   Use this project's architecture — two pipelines, async ingestion, hybrid retrieval.

2. **"Why use Celery for ingestion instead of just a background thread?"**
   Threads don't survive process restarts. Celery + Redis persists tasks across failures, enables scaling, and provides retry logic.

3. **"Why did you choose hybrid retrieval over pure vector search?"**
   Dense retrieval misses exact keyword matches. Sparse retrieval misses semantic similarity. Combining them maximizes recall before re-ranking.

4. **"What is Reciprocal Rank Fusion and why use it instead of score normalization?"**
   RRF uses rank positions rather than raw scores, avoiding the problem of incompatible score scales between dense (cosine similarity) and sparse (BM25) retrievers.

5. **"Why re-rank after RRF? Isn't RRF already a good ranking?"**
   RRF is a good ranking for combining two retrievers, but it doesn't directly model query-document relevance. A cross-encoder reads query + document together with full attention, giving much more accurate relevance scores.

6. **"How would you scale this system to handle 10,000 uploads per day?"**
   Scale Celery workers horizontally (Kubernetes HPA on queue depth), shard Qdrant, use ElasticSearch for BM25, add a CDN for file storage.

### Deep Dive Questions

7. **"How does BM25 work?"** Term frequency × inverse document frequency, with document length normalization and k1/b tuning parameters.

8. **"What's the difference between a bi-encoder and a cross-encoder?"**
   Bi-encoder: encode query and document separately — fast but lower accuracy. Cross-encoder: encode together — slower but higher accuracy. Use bi-encoder for recall, cross-encoder for precision.

9. **"How do you prevent the LLM from hallucinating?"**
   By including explicit instructions ("answer only from the provided context") and top-quality retrieved chunks. The quality of retrieval directly determines hallucination rate.

10. **"What metrics would you use to evaluate this RAG system?"**
    See Section 19 below — this is a great opportunity to discuss RAGAS.

---

## 19. METRICS TO EVALUATE THE RAG SYSTEM

### RAGAS Framework Metrics

RAGAS (Retrieval Augmented Generation Assessment) provides automated metrics using an LLM as judge.

| Metric | What it Measures | Formula |
|---|---|---|
| **Faithfulness** | Does the answer stick to the retrieved context? | Claims in answer that are supported by context / total claims |
| **Answer Relevancy** | Does the answer actually address the question? | LLM judge: 0–1 score |
| **Context Precision** | Are the retrieved chunks relevant to the question? | Relevant chunks retrieved / total chunks retrieved |
| **Context Recall** | Did retrieval find all necessary information? | Retrieved relevant chunks / all relevant chunks in corpus |

### Retrieval-Specific Metrics

- **MRR (Mean Reciprocal Rank):** Average of 1/rank of first relevant result across queries. Measures if the top result is the right one.
- **NDCG@5 (Normalized Discounted Cumulative Gain):** Measures the quality of the top 5 results, weighted by position.
- **Retrieval Hit Rate:** Percentage of queries where at least one relevant chunk appears in top-k results.

### System Performance Metrics

- **Ingestion latency:** Time from file upload to "completed" status
- **Query latency (P50/P95/P99):** End-to-end time to first token and to completion
- **Time to First Token (TTFT):** Critical for perceived streaming responsiveness
- **Celery queue depth:** If this grows, workers need scaling

### Evaluation Approach

Build a **golden dataset**: 20–50 (question, expected answer, source chunk) triples manually created from your documents. Run this dataset against the system after every major change to detect regressions. Track RAGAS metrics over time in a simple spreadsheet or MLflow.

---

## 20. COMMON FAILURE CASES AND SOLUTIONS

### Failure 1: Retrieval Returns Irrelevant Chunks

**Symptom:** LLM gives generic or hallucinated answers. Retrieved chunks don't match the question.

**Causes and Solutions:**
- **Bad chunking**: Chunks that split mid-concept miss context. Solution: Increase overlap (100 instead of 50), use semantic splitting, or implement parent-child chunking.
- **Wrong embedding model**: Small/weak embedding models lose semantic signal. Solution: Upgrade to `text-embedding-3-large`.
- **Query too short/ambiguous**: One-word queries don't retrieve well. Solution: Implement HyDE (generate a hypothetical answer and embed that instead).

### Failure 2: BM25 Index Out of Sync

**Symptom:** Sparse retrieval misses chunks that are clearly present in the corpus.

**Cause:** New chunks were added to Qdrant but the BM25 index was not rebuilt.

**Solution:** Rebuild the BM25 index atomically at the end of every ingestion task. Store the index alongside a corpus version hash in Redis. Validate on startup.

### Failure 3: Celery Task Stuck in "Pending"

**Symptom:** Job status never moves past "pending". No error in logs.

**Causes and Solutions:**
- Worker not running → ensure `celery worker` process is active
- Redis connection failure → check broker URL
- Task silently crashed on import → check `include=["app.workers.tasks"]` in Celery config
- Worker prefetch is holding the task → set `worker_prefetch_multiplier=1`

### Failure 4: SSE Connection Drops Mid-Stream

**Symptom:** Client receives partial response, then connection closes.

**Causes and Solutions:**
- Nginx proxy timeout → set `proxy_read_timeout 300s` for SSE routes
- OpenAI API timeout → wrap streaming in retry logic
- Client disconnects → wrap generator in try/except, detect `asyncio.CancelledError`

### Failure 5: LLM Context Window Overflow

**Symptom:** OpenAI API returns a `context_length_exceeded` error.

**Cause:** Conversation history + 5 chunks + system prompt exceeds the model's token limit.

**Solutions:**
- Truncate conversation history to the last N turns
- Limit chunk text to 300 tokens before passing to LLM
- Summarize older conversation history with a separate LLM call
- Use a model with a larger context window (GPT-4o: 128k tokens)

### Failure 6: Re-ranker Changes Slow Query Response

**Symptom:** P95 query latency is too high (>5 seconds).

**Cause:** Cross-encoder inference on CPU is slow for 20 candidates.

**Solutions:**
- Run the cross-encoder on GPU (dramatic speedup)
- Reduce candidates to 10 before re-ranking
- Cache re-ranking results for popular queries in Redis
- Use a smaller cross-encoder model (MiniLM is already small — consider quantization)
- Switch to a lighter re-ranking approach (ColBERT late interaction)

---

*This PRP was generated for a senior-level portfolio and system design interview context. Every decision is defensible and maps to real production engineering tradeoffs.*
