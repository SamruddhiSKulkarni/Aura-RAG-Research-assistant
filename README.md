# 🔬 Aura — Document RAG Research Assistant

A full-stack RAG (Retrieval-Augmented Generation) application that lets you upload documents and ask questions answered strictly from those documents.

To run this project on free cloud hosting without crashing the server, I designed the app so that **AI embeddings run directly inside the user's browser** instead of on the backend. This keeps the server lightweight, free to host, and fast.

---

### 🌐 Links
- **Live Demo:** [aura-rag-research-assistant.vercel.app](https://aura-rag-research-assistant.vercel.app/)
- **GitHub:** [github.com/Kulkarnisamruddh/Aura-RAG-Research-assistant](https://github.com/Kulkarnisamruddh/Aura-RAG-Research-assistant)
- **Technical Report & Interview Guide:** [`Aura_Project_Report.md`](Aura_Project_Report.md)

---

## 💡 Why In-Browser Embeddings? (The Main Design Decision)

Most RAG apps generate vector embeddings on the backend server or pay for an API like OpenAI. 

When hosting on a free platform like Render, you only get **512 MB of RAM**. Loading an AI model in Node.js easily exceeds this limit and crashes the server.

**How I solved it:**
- When a user opens the app, the browser downloads a lightweight model (`all-MiniLM-L6-v2`, ~23 MB) once.
- The model runs inside a background browser thread (Web Worker) using `@xenova/transformers`.
- The user's device creates the embeddings, so the server never runs out of memory.

**Trade-offs (Honest Pros & Cons):**
- ✅ **Pros:** Zero server RAM issues, zero embedding API costs, and scales easily across users.
- ⚠️ **Cons:** First-time load takes a few seconds to download the 23 MB model; processing speed depends on the user's computer.

---

## ⚙️ How It Works

```
1. Browser (React 19)
   ├── You upload a file (PDF, TXT, MD)
   ├── Checks file hash to prevent duplicate uploads
   ├── Splits text into 300-word chunks
   └── Web Worker converts chunks into 384-dimension vectors
         │
         ▼ (Sends vectors + Supabase Login Token)
2. Backend (Node.js & Express)
   ├── Verifies the user's login token with Supabase
   ├── Applies rate limits (50 requests per 15 minutes)
   └── Saves chunks and vectors to Supabase
         │
         ▼
3. Vector Search (Supabase PostgreSQL + pgvector)
   ├── Uses an HNSW cosine index to quickly find the top 5 relevant chunks
   └── Early exit: If no chunk scores above 0.20, stops early to save AI tokens
         │
         ▼
4. Answer Generation (Groq)
   ├── Groq LLM (gpt-oss-120b) generates the answer using ONLY the found text
   ├── Streams the answer back word-by-word (Server-Sent Events)
   └── Guardrail check: Warns if the answer didn't cite its sources [Source 1]
```

---

## 🧪 Evaluation & Testing

I built a simple automated test script to check if the app answers accurately and cites sources correctly.

Run the test locally:
```bash
npm run eval
```

Results are saved to [`eval/eval_report.md`](eval/eval_report.md).

### Latest Test Results (5 Sample Questions)

| Metric | Score | What It Means |
| :--- | :--- | :--- |
| **Faithfulness** | **5.0 / 5.0** | Answers only use facts from the text (no made-up facts) |
| **Relevance** | **5.0 / 5.0** | Answers directly answer the question |
| **Citation Check** | **100%** | Properly includes `[Source N]` tags |
| **Refusal Check** | **100%** | Correctly says "I cannot find the answer" for out-of-domain questions |
| **Speed / Latency** | **~1.3 sec** | Fast response streaming via Groq |

### 📌 Honest Notes on the Evaluation

- **Small Test Set (5 questions):** This is a quick sanity check to catch broken prompts or formatting issues when changing code. It is a directional smoke test, not a full scientific benchmark.
- **Tested with Prepared Context:** The test gives the AI pre-selected text chunks directly (`eval/dataset.json`) to test whether the AI follows instructions, separate from testing database search quality.
- **The AI Judges Itself:** The same model (`gpt-oss-120b`) generates the answer and scores it. While AI models can have a slight bias toward their own writing style, this is a practical and free way to catch obvious mistakes automatically.

---

## 🔒 Security & Protection

- **Secure Login:** Uses Supabase Auth (JWT). The backend verifies the token directly with Supabase—it never trusts user IDs sent in the request body.
- **Data Privacy (RLS):** Row-Level Security in PostgreSQL ensures each user can only read and write their own documents and chat history.
- **No Duplicate Files:** The app creates a SHA-256 hash of each file in the browser before uploading. If you already uploaded it, it rejects the file (HTTP 409).
- **Rate Limiting:** Limited to 50 requests every 15 minutes per IP address to prevent spam.
- **Hidden Keys:** Sensitive keys (like `GROQ_API_KEY`) stay on the backend and are never sent to the browser.

---

## 🛠️ Tech Stack

- **Frontend:** React 19, Vite
- **In-Browser AI:** `@xenova/transformers` (ONNX Web, Web Worker)
- **Backend:** Node.js, Express
- **Database:** Supabase (PostgreSQL with `pgvector` & HNSW indexing)
- **Fast LLM Inference:** Groq (`openai/gpt-oss-120b`)
- **Streaming:** Server-Sent Events (SSE)
- **Deployment:** Docker & Docker Compose

---

## 🚀 Quick Setup

### 1. Clone and Install
```bash
git clone https://github.com/Kulkarnisamruddh/Aura-RAG-Research-assistant.git
cd Aura-RAG-Research-assistant
npm install
```

### 2. Environment Variables
Create a `.env` file from the example:
```bash
cp .env.example .env
```

Add your keys:
```env
PORT=3001
VITE_API_URL=http://localhost:3001/api/chat
GROQ_API_KEY=gsk_your_groq_key
GROQ_MODEL=openai/gpt-oss-120b
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=eyJyour_anon_key
```

### 3. Database
Run the script in [`setup.sql`](setup.sql) in your Supabase SQL Editor to create the required tables, vector index, and functions.

### 4. Run Locally
```bash
# Run both Frontend and Backend
npm run dev:full

# Run Backend only
npm start

# Run Frontend only
npm run dev

# Run evaluation test
npm run eval
```

Open `http://localhost:5173` in your browser.

---

## 🐳 Docker

Run the entire app using Docker:
```bash
docker compose up --build
```
Open `http://localhost:3001` in your browser.

---

## 👨‍💻 Author

**Samruddh Kulkarni**  
AI & Data Science Graduate — MIT Chhatrapati Sambhajinagar  
- **GitHub:** [@Kulkarnisamruddh](https://github.com/Kulkarnisamruddh)  
- **LinkedIn:** [Samruddh Kulkarni](https://www.linkedin.com/in/samruddhi-kulkarni-31a653261)
