# 🔬 Aura — Constrained-Cost RAG Research Assistant

> A full-stack Retrieval-Augmented Generation (RAG) assistant engineered around a **constrained-cost architecture with documented trade-offs**. Designed to survive free-tier hosting limits (512 MB RAM), vector embedding computation is offloaded to the client browser via Web Workers and ONNX Runtime Web, leaving the Node.js backend dedicated strictly to authenticated routing, pgvector retrieval, and SSE token streaming.

---

### 🌐 Deployment & Resources
- **Live Demo:** [aura-rag-research-assistant.vercel.app](https://aura-rag-research-assistant.vercel.app/)
- **Repository:** [github.com/Kulkarnisamruddh/Aura-RAG-Research-assistant](https://github.com/Kulkarnisamruddh/Aura-RAG-Research-assistant)
- **Technical Deep-Dive & Interview Guide:** [`Aura_Project_Report.md`](Aura_Project_Report.md)
- **Evaluation Benchmark:** [`eval/eval_report.md`](eval/eval_report.md)

---

## 🏗️ Architecture Overview

```
┌────────────────────────────────────────────────────────────────────────┐
│                        CLIENT BROWSER (React 19)                       │
│                                                                        │
│  [File Upload] ──► SHA-256 Hash Deduplication                          │
│  [Text Chunk]  ──► 300-word sliding window (80% step)                  │
│  [Web Worker]  ──► Xenova/all-MiniLM-L6-v2 (ONNX Web, 384-dim)         │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                       HTTPS + Supabase JWT Bearer
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     EXPRESS BACKEND (Node.js API)                      │
│                                                                        │
│  ├── 1. Rate Limiting (express-rate-limit: 50 req / 15 min / IP)       │
│  ├── 2. JWT Verification (auth.getUser() validates token on Supabase)  │
│  ├── 3. Early Exit (Zero tokens billed if no chunks pass threshold)    │
│  ├── 4. System Prompt Guardrails (Strict source attribution rules)     │
│  └── 5. Post-Stream Verification (Regex audit for [Source N] tags)     │
└───────────────────┬────────────────────────────────┬───────────────────┘
                    │                                │
    Vector Search (RPC)                   SSE Streaming (Tokens)
                    │                                │
                    ▼                                ▼
┌──────────────────────────────────────┐  ┌──────────────────────────────┐
│        SUPABASE (PostgreSQL)         │  │          GROQ LPU            │
│                                      │  │                              │
│  • pgvector HNSW Cosine Index        │  │  • openai/gpt-oss-120b       │
│  • match_document_chunks RPC         │  │  • Streaming completion      │
│  • Row-Level Security (RLS) policies │  │  • ~200+ tokens/sec          │
└──────────────────────────────────────┘  └──────────────────────────────┘
```

---

## ⚖️ Core Engineering Decisions & Trade-Offs

### 1. Browser-Side Embeddings (`@xenova/transformers`)
Instead of running an embedding model server-side or paying for external embedding APIs, Aura downloads a quantized `all-MiniLM-L6-v2` model (~23 MB) directly into the browser and runs inference inside a dedicated Web Worker thread.

* **Why this choice?**
  Free-tier cloud environments (e.g., Render free tier) enforce a strict 512 MB RAM ceiling. Hosting an ONNX or PyTorch runtime inside Node.js creates memory pressure that triggers out-of-memory (OOM) crashes under concurrent requests. Browser offloading reduces server RAM usage to baseline Node.js levels (~40 MB) and scales embedding compute client-side at zero operational cost.
* **Documented Trade-Offs:**
  - **Initial Bandwidth Penalty:** First-time visitors must download the ~23 MB ONNX model bundle into browser cache.
  - **Hardware Dependency:** Embedding generation speed depends on client CPU/GPU performance.
  - **Coupled Deployment:** Swapping the embedding model requires updating client bundle assets rather than performing a seamless backend-only migration.

### 2. Approximate Nearest Neighbor Search via pgvector HNSW
Document embeddings are indexed with PostgreSQL's `pgvector` extension using a **Hierarchical Navigable Small World (HNSW)** index configured with `vector_cosine_ops`.

* **Similarity Threshold Calibration:**
  A similarity cutoff of `0.20` is enforced inside the `match_document_chunks` RPC. In `all-MiniLM-L6-v2`'s normalized 384-dimensional cosine space, natural query-to-chunk pairs consistently score between 0.20 and 0.45. A 0.70+ threshold (typical for larger OpenAI embeddings) would drop valid matches.
* **Early-Exit Token Protection:**
  If no document chunks score above the 0.20 cutoff, the server bypasses the LLM API call entirely. It returns a standardized refusal message via SSE, preventing unnecessary token consumption.

### 3. Layered Guardrails Against Hallucination
* **System Prompt Constraints:** Strict system prompt rules instructing the model to rely solely on provided context and answer with explicit `[Source N]` citations or state: `"I cannot find the answer in the uploaded documents."`
* **Post-Stream RegEx Audit:** As the SSE stream finishes, the server audits the accumulated text for `[Source \d+]` tags. If relevant context chunks were retrieved but no citation was produced, the server dispatches a `guardrail: { unbacked: true }` event to alert the frontend.

---

## 🧪 Evaluation & Benchmark Suite

Aura includes an automated evaluation harness designed as a **directional regression smoke test** across five curated test cases testing factual extraction, security boundaries, database optimizations, out-of-domain refusals, and upload constraints.

Run the evaluation locally:
```bash
npm run eval
```

### Last Run Benchmark Results (`openai/gpt-oss-120b`, 2026-09-12)

| Metric | Score | Target | Interpretation |
| :--- | :--- | :--- | :--- |
| **Faithfulness / Grounding** | **5.00 / 5.0** | $\ge 4.5$ | No unsupported claims detected |
| **Answer Relevance** | **5.00 / 5.0** | $\ge 4.5$ | Responses directly answer the prompt |
| **Citation Compliance** | **100.0%** | $\ge 90\%$ | Correct source attribution tags |
| **Out-of-Domain Refusal** | **100.0%** | $100\%$ | Successfully rejects ungrounded queries |
| **Average Latency** | **1,332 ms** | $< 1,500$ ms | End-to-end response generation |

### 🔍 Methodology & Known Limitations

1. **Directional Smoke Test ($N=5$), Not Statistical Validation:**
   The five test cases in `eval/dataset.json` serve as an automated smoke test for CI/CD to detect prompt regressions and formatting breakages. They do not represent a statistically significant benchmark across long-tail domain distributions.
2. **Decoupled Synthetic Context:**
   The evaluation feeds curated synthetic context directly into the generator rather than executing live pgvector queries against Supabase. This intentionally decouples *generator behavior* (faithfulness, refusal handling, citation adherence) from *retrieval performance* (Recall@K, chunk noise).
3. **LLM-as-Judge Self-Preference Bias:**
   The evaluation judge uses the same model as the generator (`openai/gpt-oss-120b`). While practical for a zero-cost CI sanity check, this introduces known self-preference bias (models favor their own stylistic and reasoning traits). A production-scale benchmark would decouple the judge with a higher-tier frontier model or human annotation and measure retrieval metrics (Recall@K, MRR) via Ragas.

Detailed breakdown and individual test outputs are recorded in [`eval/eval_report.md`](eval/eval_report.md).

---

## 🛡️ Security & Isolation

- **Server-Side JWT Verification:** Every API call validates the Supabase auth token via `userSupabase.auth.getUser()`. The `userId` is extracted directly from the verified token—never accepted from request body parameters.
- **Row-Level Security (RLS):** Enabled across all database tables (`documents`, `document_chunks`, `chat_messages`). Database policies ensure queries can only read and write records where `user_id = auth.uid()`.
- **Client-Side SHA-256 Deduplication:** Files are hashed in the browser using the Web Crypto API prior to upload. If a matching hash exists for that user, the server rejects the upload with a `409 Conflict`.
- **Rate Limiting:** `express-rate-limit` enforces a maximum of 50 requests per 15-minute window per IP on `/api/*`.
- **Environment Isolation:** Backend API keys (`GROQ_API_KEY`) remain strictly server-side. Only public `VITE_` parameters are bundled into the client build.

---

## 💻 Tech Stack

| Component | Technology | Rationale |
| :--- | :--- | :--- |
| **Frontend** | React 19, Vite | Responsive UI, streaming token display, worker orchestration |
| **In-Browser ML** | `@xenova/transformers` (ONNX Web) | Edge vectorization without server RAM consumption |
| **Backend API** | Node.js, Express 5 | Lightweight API orchestration, SSE token streaming |
| **Vector Database** | Supabase (PostgreSQL + pgvector) | Multi-tenant HNSW indexing with native RLS |
| **Inference Engine** | Groq (`openai/gpt-oss-120b`) | Ultra-low latency streaming inference via LPUs |
| **Evaluation** | Custom Node.js Harness | Directional CI smoke test for citation and faithfulness |
| **Containerization**| Docker, Docker Compose | Multi-stage build for local testing and reproducible deployment |

---

## 🚀 Getting Started

### 1. Prerequisites
- Node.js 20+
- A [Supabase](https://supabase.com/) project
- A [Groq Cloud](https://console.groq.com/) API key

### 2. Clone & Install
```bash
git clone https://github.com/Kulkarnisamruddh/Aura-RAG-Research-assistant.git
cd Aura-RAG-Research-assistant
npm install
```

### 3. Environment Configuration
Create a `.env` file in the project root:
```bash
cp .env.example .env
```

Fill in your configuration:
```env
PORT=3001
VITE_API_URL=http://localhost:3001/api/chat
GROQ_API_KEY=gsk_your_groq_api_key_here
GROQ_MODEL=openai/gpt-oss-120b
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=eyJyour_anon_key_here
```

### 4. Database Setup
Copy the contents of [`setup.sql`](setup.sql) and execute it in your Supabase SQL Editor. This script:
- Enables the `vector` extension.
- Provisions `documents`, `document_chunks`, and `chat_messages` tables.
- Creates the HNSW index (`embedding vector_cosine_ops`).
- Configures the `match_document_chunks` RPC function.
- Sets up Row Level Security (RLS) policies.

### 5. Running the Application

```bash
# Run both Frontend (Vite) and Backend (Express) concurrently
npm run dev:full

# Run Backend only
npm start

# Run Frontend only
npm run dev

# Run Evaluation Suite
npm run eval
```

- Frontend interface: `http://localhost:5173`
- Backend API: `http://localhost:3001`

---

## 🐳 Docker Deployment

A multi-stage [`Dockerfile`](Dockerfile) compiles the Vite frontend in Stage 1 and serves it via the Express backend in an Alpine Node.js runner in Stage 2.

```bash
# Build and run container stack
docker compose up --build

# Access containerized app at http://localhost:3001
```

---

## 🚫 What This Project Is Not

To maintain honest, clear technical expectations:
- **Not Agentic:** Aura follows a deterministic RAG pipeline (chunk → embed → vector search → generate). The LLM does not execute tool calls, plan recursive sub-tasks, or dynamically route search strategies.
- **Not Hybrid Search:** Retrieval relies entirely on dense vector cosine similarity. Sparse lexical search (BM25) and reciprocal rank fusion (RRF) are not implemented.
- **Not an Enterprise Cluster:** The server is configured as a single stateless Node.js instance suitable for cost-constrained environments; it has not been benchmarked under high-concurrency enterprise load.

---

## 👤 Author

**Samruddh Kulkarni**  
Final Year AI & Data Science — MIT Chhatrapati Sambhajinagar  
- **GitHub:** [@Kulkarnisamruddh](https://github.com/Kulkarnisamruddh)  
- **LinkedIn:** [Samruddh Kulkarni](https://www.linkedin.com/in/samruddhi-kulkarni-31a653261)
