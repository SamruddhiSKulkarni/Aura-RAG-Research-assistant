# 🔬 Aura — Document RAG Research Assistant

A full-stack RAG (Retrieval-Augmented Generation) application that lets users upload documents and ask questions answered strictly from those documents.

Built around a **constrained-cost architecture**: to run reliably on free cloud hosting (512 MB RAM limit) without paid embedding APIs, vector embeddings are generated directly inside the user's browser using Web Workers and ONNX Runtime Web.

---

### 🌐 Links
- **Live Demo:** [aura-rag-research-assistant.vercel.app](https://aura-rag-research-assistant.vercel.app/)
- **Repository:** [github.com/Kulkarnisamruddh/Aura-RAG-Research-assistant](https://github.com/Kulkarnisamruddh/Aura-RAG-Research-assistant)
- **Technical Report:** [`Aura_Project_Report.md`](Aura_Project_Report.md)

---

## ✨ Features

- 📄 **Multi-Format Ingestion:** Supports uploading `.pdf`, `.txt`, and `.md` files.
- ⚡ **In-Browser Embeddings:** Generates 384-dimensional dense vectors client-side via `@xenova/transformers` (`all-MiniLM-L6-v2`) inside a dedicated Web Worker.
- 🔍 **Vector Search:** Fast approximate nearest neighbor search via Supabase (PostgreSQL + `pgvector`) using an HNSW cosine distance index.
- 💬 **Streaming Responses:** Real-time token streaming using Server-Sent Events (SSE) powered by Groq (`openai/gpt-oss-120b`).
- 🛡️ **Citation Guardrails:** Strict prompt enforcement requiring `[Source N]` tags and a post-stream regex auditor to warn users of unbacked claims.
- 💰 **Early-Exit Token Protection:** Skips LLM inference entirely if no document chunks meet the 0.20 similarity threshold, saving API tokens.
- 🔐 **Multi-Tenant Security:** Server-side JWT validation (`auth.getUser()`) and PostgreSQL Row-Level Security (RLS) guarantee users only access their own documents.
- 🚫 **Duplicate Prevention:** Client-side SHA-256 hashing detects and rejects duplicate file re-uploads with a `409 Conflict`.

---

## 💡 Why In-Browser Embeddings?

Most RAG applications generate vector embeddings on the backend server or pay for an API like OpenAI. 

On free hosting tiers (such as Render's 512 MB RAM limit), loading an in-memory transformer model inside Node.js causes immediate out-of-memory crashes under concurrent traffic.

**The Solution:**
- On first visit, the client browser downloads a quantized `all-MiniLM-L6-v2` model (~23 MB) once and caches it.
- A background browser Web Worker computes embeddings on the user's device.
- The Node.js server stays lightweight (~40 MB RAM) and handles only routing, vector queries, and streaming.

**Trade-offs:**
- ✅ **Pros:** Zero server RAM pressure, zero embedding API costs, and scales client-side.
- ⚠️ **Cons:** First visit requires downloading the 23 MB model bundle; embedding speed depends on the user's device.

---

## ⚙️ Architecture & Data Flow

```
1. Browser (React 19)
   ├── File uploaded (PDF, TXT, MD)
   ├── SHA-256 hash generated to check for duplicates
   ├── Text split into 300-word sliding window chunks
   └── Web Worker generates 384-dim vectors (all-MiniLM-L6-v2)
         │
         ▼ (Sends vectors + Supabase JWT)
2. Express API (Node.js)
   ├── Rate limiting (50 requests / 15 minutes per IP)
   ├── Validates JWT with Supabase auth.getUser()
   └── Stores chunks and vectors in Supabase
         │
         ▼
3. Vector Database (Supabase PostgreSQL + pgvector)
   ├── HNSW index searches top 5 chunks (similarity threshold: 0.20)
   └── Early exit: Bypasses LLM if no chunks meet threshold
         │
         ▼
4. LLM Inference (Groq)
   ├── Groq generates response based ONLY on retrieved context
   ├── Streams tokens via Server-Sent Events (SSE)
   └── Post-stream regex check verifies [Source N] citation tags
```

---

## 🔌 API Endpoints

| Method | Endpoint | Description | Auth Required |
| :--- | :--- | :--- | :--- |
| `GET` | `/health` | Server health check | No |
| `POST` | `/api/upload` | Validates JWT, checks file hash, saves document chunks & embeddings | Yes (Bearer JWT) |
| `POST` | `/api/chat` | Runs pgvector search, queries Groq LLM, streams response via SSE | Yes (Bearer JWT) |

---

## 🧪 Evaluation

A lightweight automated test script verifies response faithfulness, relevance, citation compliance, and out-of-domain refusals.

```bash
npm run eval
```

Results are saved to [`eval/eval_report.md`](eval/eval_report.md).

### Latest Benchmark Results (5 Test Cases)

| Metric | Score | Target | Interpretation |
| :--- | :--- | :--- | :--- |
| **Faithfulness** | **5.0 / 5.0** | $\ge 4.5$ | Answers strictly rely on provided context |
| **Relevance** | **5.0 / 5.0** | $\ge 4.5$ | Answers directly address the user's query |
| **Citation Compliance** | **100%** | $\ge 90\%$ | Correctly includes `[Source N]` attribution |
| **Out-of-Domain Refusal** | **100%** | $100\%$ | Refuses when context is missing |
| **Average Latency** | **1,332 ms** | $< 1,500$ ms | Token streaming speed |

### Evaluation Methodology & Scope:
- **Directional Smoke Test ($N=5$):** Designed as a fast CI regression test to catch prompt formatting breakages, not a statistical benchmark.
- **Decoupled Context:** Evaluates generator adherence given controlled context chunks (`eval/dataset.json`), isolating generator behavior from retrieval variance.
- **Single-Model Judge:** The generator and judge use the same model (`openai/gpt-oss-120b`). This is a practical, zero-cost sanity check with acknowledged self-preference bias.

---

## 🗺️ Roadmap & Next Steps

- [ ] **Hybrid Search:** Combine dense vector search with PostgreSQL full-text search (`tsvector` / BM25) using Reciprocal Rank Fusion (RRF).
- [ ] **Live Retrieval Evaluation:** Measure Hit Rate and Recall@K against real database chunks using Ragas.
- [ ] **OCR Support:** Add client-side OCR via Tesseract.js for scanned PDF documents.

---

## 🚀 Getting Started

### 1. Clone & Install
```bash
git clone https://github.com/Kulkarnisamruddh/Aura-RAG-Research-assistant.git
cd Aura-RAG-Research-assistant
npm install
```

### 2. Environment Variables
Create a `.env` file in the root directory:
```bash
cp .env.example .env
```

Configure your credentials:
```env
PORT=3001
VITE_API_URL=http://localhost:3001/api/chat
GROQ_API_KEY=gsk_your_groq_key
GROQ_MODEL=openai/gpt-oss-120b
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=eyJyour_anon_key
```

### 3. Database Setup
Run the SQL script in [`setup.sql`](setup.sql) in your Supabase SQL Editor to create tables, indexes, RLS policies, and the `match_document_chunks` RPC function.

### 4. Run Locally
```bash
# Run both Frontend and Backend concurrently
npm run dev:full

# Run Backend only
npm start

# Run Frontend only
npm run dev

# Run evaluation test
npm run eval
```

- Frontend: `http://localhost:5173`
- Backend API: `http://localhost:3001`

---

## 🐳 Docker

Run the entire application in a container:
```bash
docker compose up --build
```
Access the application at `http://localhost:3001`.

---

## 👨‍💻 Author

**Samruddh Kulkarni**  
AI & Data Science Graduate — MIT Chhatrapati Sambhajinagar  
- **GitHub:** [@Kulkarnisamruddh](https://github.com/Kulkarnisamruddh)  
- **LinkedIn:** [Samruddh Kulkarni](https://www.linkedin.com/in/samruddhi-kulkarni-31a653261)
