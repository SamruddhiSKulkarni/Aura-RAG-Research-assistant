# 🔬 Aura — Browser-Embedded RAG Research Assistant

A full-stack Retrieval-Augmented Generation (RAG) application that lets users upload documents and ask questions answered strictly from those documents. Embeddings run in the browser via Web Workers; the Node.js server handles only routing, vector search, and LLM calls.

---

### Live Demo: [aura-rag-research-assistant.vercel.app](https://aura-rag-research-assistant.vercel.app/)
### Backend: [aura-rag-research-assistant.onrender.com](https://aura-rag-research-assistant.onrender.com/health)

---

## How It Works

```
Browser (React 19)
  └── Web Worker (ONNX Runtime Web)
        all-MiniLM-L6-v2 → 384-dim vectors
              │
              ▼  HTTPS + Supabase JWT
Express API (Node.js / server.js)
  ├── Validate JWT server-side (auth.getUser)
  ├── Rate-limit (50 req / 15 min per IP)
  ├── POST /api/upload  → Supabase document_chunks
  └── POST /api/chat    → pgvector search → Groq → SSE stream
              │
              ▼
Supabase PostgreSQL + pgvector
  ├── HNSW cosine index on embedding column
  └── Row Level Security on all tables
```

---

## Key Technical Decisions

### Browser-Side Embeddings
`all-MiniLM-L6-v2` runs inside a **Web Worker** using `@xenova/transformers` (ONNX Runtime Web). This keeps the Node.js server stateless and lightweight enough for a free hosting tier.

**Trade-offs:**
- Each new client downloads the model (~23 MB) on first use
- Embedding quality cannot be swapped server-side without a frontend redeploy
- No server-side control over model versions

### Vector Search
pgvector with an **HNSW cosine index** (`vector_cosine_ops`). Similarity threshold is 0.20 — appropriate for the all-MiniLM-L6-v2 embedding space where natural Q&A pairs score between 0.20 and 0.45.

The search is **vector-only** (not hybrid). Keyword search (BM25) is not implemented.

### Guardrail
After the LLM stream completes, the server checks whether the response contains `[Source N]` citation tags. If context was retrieved but no citations appear, it emits a `guardrail` SSE event and the UI shows a warning. This catches obvious cases of the model ignoring its context; it does not prevent subtler hallucinations.

---

## Evaluation

An automated evaluation harness runs the full RAG pipeline against five ground-truth QA pairs and uses the same LLM as a judge. Run it with:

```bash
npm run eval
```

Results are written to [`eval/eval_report.md`](eval/eval_report.md).

**Last run results (openai/gpt-oss-120b, 2026-09-12):**

| Metric | Score |
| :--- | :--- |
| Faithfulness / Grounding | 5.00 / 5.00 |
| Answer Relevance | 5.00 / 5.00 |
| Citation Compliance | 100% |
| Out-of-Domain Refusal | 100% |
| Average Latency | 1,332 ms |

> Note: All five test cases use synthetic context from `eval/dataset.json`, not live Supabase data. Scores reflect the LLM's behaviour under controlled input, not end-to-end retrieval accuracy.

---

## Docker

A multi-stage `Dockerfile` builds the Vite frontend in Stage 1 and copies the output into a `node:20-alpine` runner in Stage 2. The Express server serves the static `dist/` folder when it exists.

```bash
# Build and run the full stack in a container
docker compose up --build
# App available at http://localhost:3001
```

---

## Setup

### 1. Clone and install
```bash
git clone https://github.com/Kulkarnisamruddh/Aura-RAG-Research-assistant.git
cd Aura-RAG-Research-assistant
npm install
```

### 2. Environment variables
```bash
cp .env.example .env
```

```env
VITE_API_URL=http://localhost:3001/api/chat
GROQ_API_KEY=gsk_...
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=eyJ...
```

### 3. Database
Run [`setup.sql`](setup.sql) in your Supabase SQL Editor. It creates the tables, HNSW index, RLS policies, and the `match_document_chunks` RPC function.

### 4. Run locally

```bash
# Frontend + backend with hot reload
npm run dev:full

# Backend only
npm start

# Evaluation harness
npm run eval
```

---

## Security Notes

- **JWT validated server-side**: `auth.getUser()` extracts the user ID from the token; the request body `userId` field is ignored.
- **Rate limiting**: `express-rate-limit` — 50 requests per 15-minute window per IP.
- **Row Level Security**: PostgreSQL RLS ensures each user can only read and write their own rows across `documents`, `document_chunks`, and `chat_messages`.
- **No secret keys in frontend**: Only `VITE_` prefixed variables reach the browser bundle.

---

## Tech Stack

| Layer | Technology |
| :--- | :--- |
| Frontend | React 19, Vite |
| Embeddings | `@xenova/transformers` (ONNX Web, Web Worker) |
| Backend | Node.js, Express 5 |
| Vector DB | Supabase PostgreSQL + pgvector (HNSW) |
| Auth | Supabase Auth (email/password, JWT) |
| LLM | Groq (`openai/gpt-oss-120b`, configurable via `GROQ_MODEL`) |
| Streaming | Server-Sent Events (SSE) |
| Containerisation | Docker (multi-stage), Docker Compose |

---

## What This Is Not

- **Not agentic**: The system follows a fixed pipeline (embed → retrieve → generate). The LLM does not call tools, plan multi-step actions, or decide when to search.
- **Not hybrid search**: Retrieval is vector-similarity only. Keyword (BM25) search is not implemented.
- **Not a managed deployment**: The Dockerfile is provided for local containerisation and as a starting point for cloud deployment. It has not been load-tested.

---

## Author

**Samruddh Kulkarni** — Final year AI & DS, MIT Chhatrapati Sambhajinagar  
[LinkedIn](https://www.linkedin.com/in/samruddhi-kulkarni-31a653261) · [GitHub](https://github.com/Kulkarnisamruddh)
