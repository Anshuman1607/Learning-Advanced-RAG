# 📚 Codebase Documentation RAG System

## 🧠 Project Overview

This project is a **production-style Retrieval-Augmented Generation (RAG) system** designed to help developers query large codebases and documentation efficiently.

It uses:
- Hybrid retrieval (dense + sparse)
- Re-ranking for accuracy
- Asynchronous ingestion for scalability

---

# ⚙️ System Architecture

## 🔹 Two Pipelines

### 1. Ingestion Pipeline (Background)

- Upload docs (README, API reference, markdown)
- Chunk documents (512 tokens, overlap)
- Generate embeddings
- Store in:
  - Dense index → Qdrant
  - Sparse index → BM25
- Runs asynchronously using Celery

---

### 2. Query Pipeline (Real-time)

- User sends query
- Load chat history from Redis
- Retrieve results from:
  - Dense (Qdrant)
  - Sparse (BM25)
- Merge using RRF
- Re-rank using cross-encoder
- Send top chunks to LLM
- Stream response via SSE

---

# 🧩 Tech Stack

## Backend
- FastAPI → API layer
- Celery → async task processing
- Redis → queue + memory store

## Retrieval
- Qdrant → dense vector DB
- BM25 → keyword search
- RRF → result merging
- Cross-encoder → re-ranking

## LLM
- OpenAI / Anthropic

## Other
- SSE → streaming responses
- RAGAS → evaluation

---

# 🔄 Updated Architecture (with Celery)

---

# 🚀 Why Celery + Redis?

## Problem with BackgroundTasks ❌
- Runs in same server
- No retry
- Can crash on large docs

## Solution: Celery + Redis ✅

- Redis → Queue (stores jobs)
- Celery Worker → processes jobs
- Async + scalable + fault-tolerant

---

# ⏱️ Ingestion Time Estimates

| Case | Time |
|------|------|
| Small (README) | 2–8 sec |
| Medium (20–100 pages) | 3–6 min |
| Large (300 pages) | 10–15 min |

👉 Ingestion is heavy → runs in background  
👉 Query is fast → milliseconds

---

# 🧠 Key Design Principles

- Separate ingestion & query pipelines
- Precompute heavy work
- Optimize retrieval quality over speed
- Use async processing for scalability

---

# 📅 Project Timeline

## 🔹 Total Time: 15–25 Days

### Phase 1: Setup (Day 1–3)
- FastAPI setup
- Project structure

### Phase 2: Ingestion (Day 4–7)
- Chunking
- Embeddings
- Qdrant storage

### Phase 3: Retrieval (Day 8–11)
- Dense + BM25
- Hybrid retrieval

### Phase 4: Advanced (Day 12–15)
- RRF
- Re-ranking

### Phase 5: Async System (Day 16–18)
- Celery + Redis
- Job queue

### Phase 6: UX + Polish (Day 19–22)
- SSE streaming
- Status tracking

### Phase 7: Final (Day 23–25)
- Testing
- Resume prep

---

# 🎯 Learning Resources (YouTube Strategy)

## 1. RAG Basics
Search:
- "RAG explained"
- "RAG pipeline end to end"

## 2. LangChain
Search:
- "LangChain chunking"
- "RecursiveCharacterTextSplitter"

## 3. Vector DB
Search:
- "Qdrant tutorial"
- "vector database explained"

## 4. BM25
Search:
- "BM25 explained"
- "sparse vs dense retrieval"

## 5. Hybrid + RRF
Search:
- "Hybrid search RAG"
- "Reciprocal Rank Fusion"

## 6. Re-ranking
Search:
- "cross encoder vs bi encoder"
- "reranking in RAG"

## 7. Celery + Redis
Search:
- "Celery FastAPI tutorial"
- "Redis queue explained"

## 8. SSE
Search:
- "FastAPI streaming response"
- "Server Sent Events"

---

# ⚠️ Important Notes

## ❌ Don’t:
- Copy full project from YouTube
- Focus only on speed

## ✅ Do:
- Build component by component
- Understand each layer deeply
- Explain architecture clearly

---

# 🧠 Interview Key Points

- Async ingestion using Celery
- Hybrid retrieval improves accuracy
- Re-ranking boosts precision
- RAGAS used for evaluation
- System designed for scalability

---

# 🏁 Final Insight

> Ingestion is slow by design (minutes),  
> but query is fast (milliseconds).

> Focus on architecture, not raw speed.

---

# 🚀 Resume Line

"Built a scalable RAG system with hybrid retrieval (Qdrant + BM25), re-ranking, and asynchronous ingestion using Celery and Redis, enabling efficient querying over large codebases."

---

# System Architecture
"                         ┌──────────────────────────┐
                         │        Frontend          │
                         │  (React / Next.js UI)   │
                         └──────────┬──────────────┘
                                    │
                                    ▼
                         ┌──────────────────────────┐
                         │        FastAPI           │
                         │  (API Gateway Layer)     │
                         └───────┬───────┬─────────┘
                                 │       │
        ┌────────────────────────┘       └────────────────────────┐
        ▼                                                         ▼

┌──────────────────────┐                               ┌──────────────────────┐
│   Ingestion API      │                               │     Query API        │
│   POST /ingest       │                               │     POST /chat       │
│   GET /status/{id}   │                               │  (SSE Streaming)     │
└──────────┬───────────┘                               └──────────┬───────────┘
           │                                                      │
           ▼                                                      ▼

┌──────────────────────────┐                        ┌──────────────────────────┐
│      Redis (Broker)      │◄──────────────────────►│   Redis (Memory Store)   │
│   (Celery Task Queue)    │                        │  (Conversation History)  │
└──────────┬───────────────┘                        └──────────┬───────────────┘
           │                                                    │
           ▼                                                    ▼

┌──────────────────────────┐                        ┌──────────────────────────┐
│     Celery Workers       │                        │     Retrieval Layer      │
│  (Background Ingestion)  │                        │                          │
└──────────┬───────────────┘                        │  ┌────────────────────┐  │
           │                                        │  │      Qdrant       │  │
           ▼                                        │  │ (Dense Vectors)   │  │
┌──────────────────────────┐                        │  └────────────────────┘  │
│  Chunking + Processing   │                        │           ▲              │
│ (LangChain/LlamaIndex)   │                        │           │              │
└──────────┬───────────────┘                        │  ┌────────────────────┐  │
           │                                        │  │       BM25         │  │
           ▼                                        │  │ (Sparse Index)     │  │
┌──────────────────────────┐                        │  └────────────────────┘  │
│  Embedding Generation    │                        │           │              │
│ (OpenAI / HF Models)     │                        │           ▼              │
└──────────┬───────────────┘                        │  ┌────────────────────┐  │
           │                                        │  │   RRF Fusion       │  │
           ▼                                        │  └────────────────────┘  │
┌──────────────────────────┐                        │           │              │
│  Index Storage           │                        │           ▼              │
│ → Qdrant (Dense)         │                        │  ┌────────────────────┐  │
│ → BM25 (Sparse)          │                        │  │  Cross-Encoder     │  │
└──────────┬───────────────┘                        │  │   Re-Ranker        │  │
           │                                        │  └────────────────────┘  │
           ▼                                        │           │              │
┌──────────────────────────┐                        │           ▼              │
│ Job Status अपडेट         │                        │  ┌────────────────────┐  │
│ (Redis / DB)             │                        │  │   LLM (OpenAI /    │  │
└──────────────────────────┘                        │  │   Anthropic)       │  │
                                                    │  └────────────────────┘  │
                                                    └──────────┬───────────────┘
                                                               │
                                                               ▼
                                                    ┌──────────────────────┐
                                                    │   SSE Streaming      │
                                                    │  (Token-by-token)    │
                                                    └──────────────────────┘"
---
