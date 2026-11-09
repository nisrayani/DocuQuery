# DocuQuery — Ask Questions About Your Documents Using AI

Upload a PDF or text file, ask questions in plain English, and get AI-generated answers based on the document's content.

This project is built for **learning and portfolio purposes**. It uses real backend technologies used at companies like Google, Netflix, and Uber.

---

## What Does This Project Do?

Imagine you have a 50-page PDF report. Instead of reading it yourself, you upload it here and ask: *"What were the key findings in chapter 3?"* — and the system finds the relevant parts of the document and generates a clear answer.

This is called **RAG (Retrieval-Augmented Generation)**.

---

## Technologies Used — Explained Simply

| Technology | What it is | Why we use it |
|---|---|---|
| **FastAPI** | Python web framework | Handles HTTP requests (upload, query) |
| **Apache Kafka** | Message queue / event bus | Decouples file upload from heavy processing |
| **Redis** | In-memory key-value store | Caches query answers; rate-limits LLM calls |
| **ChromaDB** | Vector database (local) | Stores document chunks as searchable vectors |
| **Google Gemini** | Large Language Model (LLM) | Reads retrieved chunks and writes the answer |
| **PostgreSQL** | Relational database | Stores document metadata and query logs |
| **SQLAlchemy** | Python ORM | Talks to PostgreSQL without writing raw SQL |
| **Docker** | Container platform | Runs Kafka, Redis, Postgres locally in one command |

### Why can't we just send the whole document to the LLM?
LLMs have a **context window limit** — they can only read ~10,000–100,000 characters at once. A 100-page PDF is ~500,000 characters. RAG solves this by:
1. Splitting the document into small chunks (~500 chars each)
2. Converting each chunk into a **vector** (a list of numbers that captures meaning)
3. At query time, finding the 3 most *semantically similar* chunks to the question
4. Sending only those 3 chunks to the LLM

---

## How the System Works — Full Flow

```
User uploads document
        │
        ▼
POST /documents ──► Save file to disk
        │
        ▼
   PostgreSQL ◄── Record document (status: pending)
        │
        ▼
  Kafka Producer ──► Publish job to "document-processing" topic
        │
        ▼  (async, background)
  Kafka Consumer
        │
        ├──► Extract text from PDF/TXT
        ├──► Split into chunks (e.g., 500 chars with 50-char overlap)
        ├──► ChromaDB auto-generates embeddings (vectors) for each chunk
        ├──► Store chunks + vectors in ChromaDB
        └──► Update PostgreSQL status → "processed"

User asks a question
        │
        ▼
POST /query ──► Check Redis cache ──► (cache hit) ──► Return instantly
        │
        │ (cache miss)
        ▼
  ChromaDB: embed query → find 3 most similar chunks
        │
        ▼
  Gemini LLM: context (chunks) + question → answer
        │
        ▼
  Cache result in Redis (30 min TTL)
        │
        ▼
  Return answer + source chunks to user
```

---

## Prerequisites — What You Need Installed

Before starting, you need these on your computer. Click the links to install:

1. **Docker Desktop** — https://www.docker.com/products/docker-desktop/
   - After installing, open Docker Desktop and make sure it's running (whale icon in menu bar)

2. **Python 3.11+** — https://www.python.org/downloads/
   - Verify: open Terminal and run `python3 --version`

3. **A terminal** — Terminal on Mac, Command Prompt or PowerShell on Windows

That's it. You don't need to install Kafka, Redis, or PostgreSQL separately — Docker handles all of that.

---

## Setup — Step by Step

### Step 1: Go to the project folder

```bash
cd ~/Flipkart/docuquery
```

### Step 2: Create your configuration file

```bash
cp .env.example .env
```

This creates a `.env` file with all settings. The defaults work out of the box for local development.

### Step 3: (Optional but recommended) Add a Gemini API key

Without a key, the app uses **mock responses** — it still shows you how RAG works (retrieves chunks, formats response) but the "answer" is a template, not AI-generated.

To get a free key:
1. Go to https://aistudio.google.com/app/apikey
2. Sign in with Google
3. Click "Create API Key"
4. Copy the key

Then open `.env` in any text editor and set:
```
GEMINI_API_KEY=your_key_here
```

### Step 4: Start Kafka, Redis, and PostgreSQL

```bash
docker-compose up -d
```

This downloads and starts 4 containers (Zookeeper, Kafka, Redis, PostgreSQL) in the background. First run takes 2–3 minutes to download images.

Check that all containers are running:
```bash
docker-compose ps
```
You should see 4 containers with status `Up` or `healthy`.

Wait about **15 seconds** after this for Kafka to fully initialise.

### Step 5: Create a Python virtual environment

A virtual environment isolates your project's packages from your system Python.

```bash
python3 -m venv venv
source venv/bin/activate
```

On Windows:
```bash
python -m venv venv
venv\Scripts\activate
```

Your terminal prompt will show `(venv)` when active.

### Step 6: Install Python dependencies

```bash
pip install -r requirements.txt
```

This takes 2–5 minutes. ChromaDB downloads a small AI embedding model (~25 MB) on first use.

### Step 7: Start the application

```bash
uvicorn app.main:app --reload
```

You should see output like:
```
[App] Document processor thread started.
[Consumer] Starting document processor...
[Consumer] Connected to Kafka. Waiting for documents...
INFO:     Uvicorn running on http://127.0.0.1:8000
```

The app is now running. Open **http://localhost:8000/docs** in your browser to see the interactive API UI.

---

## Testing — Step by Step

### Method 1: Swagger UI (easiest — no terminal needed)

Open **http://localhost:8000/docs** — you get a web interface to call every API endpoint.

### Method 2: curl commands (copy-paste into terminal)

Open a **new terminal tab** (keep the app running in the first one).

---

#### Test 1: Health check
```bash
curl http://localhost:8000/health
```

Expected response:
```json
{
  "status": "ok",
  "redis": "connected",
  "kafka_topic": "document-processing",
  "llm_mode": "gemini-1.5-flash"
}
```

---

#### Test 2: Create a sample document to upload

```bash
cat > /tmp/sample.txt << 'EOF'
DocuQuery Sample Document

Introduction:
DocuQuery is a RAG-based document Q&A system. It uses Apache Kafka for async document processing,
ChromaDB for vector similarity search, Redis for caching query results, and Google Gemini for
generating answers from retrieved document chunks.

How it works:
1. The user uploads a document (PDF or TXT).
2. The Kafka consumer splits the document into chunks and stores embeddings in ChromaDB.
3. When a user asks a question, DocuQuery finds the most relevant chunks using vector search.
4. The top chunks are sent to the LLM along with the question to generate a grounded answer.

Key features:
- Async document processing via Kafka event streaming
- Semantic search using vector embeddings (no keyword matching)
- Redis cache prevents duplicate LLM calls for the same question
- Sliding window rate limiter protects the LLM API quota
- PostgreSQL tracks all documents and query history
EOF
```

---

#### Test 3: Upload the document

```bash
curl -X POST http://localhost:8000/documents \
  -F "file=@/tmp/sample.txt"
```

Expected response:
```json
{
  "doc_id": "abc-123-...",
  "filename": "sample.txt",
  "status": "pending",
  "message": "Document uploaded successfully. Check GET /documents/abc-123-... for processing status."
}
```

**Copy the `doc_id`** — you'll need it for the next steps.

---

#### Test 4: Check processing status

Replace `<doc_id>` with the value from the previous step:

```bash
curl http://localhost:8000/documents/<doc_id>
```

Wait until `status` changes to `"processed"`. This usually takes **2–5 seconds**.

```json
{
  "doc_id": "abc-123-...",
  "filename": "sample.txt",
  "status": "processed",
  "num_chunks": 6,
  "created_at": "...",
  "processed_at": "..."
}
```

In your app terminal, you'll see:
```
[Consumer] ✓ 'sample.txt' — 6 chunks embedded and stored.
```

---

#### Test 5: Ask a question

```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How does DocuQuery process documents?",
    "doc_id": "<doc_id>"
  }'
```

Expected response:
```json
{
  "query": "How does DocuQuery process documents?",
  "answer": "DocuQuery processes documents by splitting them into chunks and storing embeddings in ChromaDB...",
  "sources": [
    {
      "chunk_text": "The Kafka consumer splits the document into chunks...",
      "chunk_index": 2,
      "doc_id": "abc-123-...",
      "filename": "sample.txt",
      "relevance_score": 0.91
    }
  ],
  "model_used": "gemini-1.5-flash",
  "cached": false
}
```

---

#### Test 6: Ask the same question again — see Redis cache hit

Run the exact same curl command from Test 5 again:

```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How does DocuQuery process documents?",
    "doc_id": "<doc_id>"
  }'
```

This time you'll see `"cached": true` — Redis returned the answer instantly, no LLM call made.

---

#### Test 7: Query without specifying a document (search across all)

```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is the role of Redis in this system?"
  }'
```

---

#### Test 8: List all uploaded documents

```bash
curl http://localhost:8000/documents
```

---

#### Test 9: Upload a PDF

```bash
curl -X POST http://localhost:8000/documents \
  -F "file=@/path/to/your/document.pdf"
```

---

## What to Watch in the Terminal

While testing, watch your app terminal. You'll see the full flow in real time:

```
[Consumer] Processing 'sample.txt'...
[Consumer] ✓ 'sample.txt' — 6 chunks embedded and stored.

# On second identical query:
[LLM] Rate limit check passed.
# (or if cached — no LLM output at all, Redis handled it)
```

---

## Stopping the App

```bash
# Stop the FastAPI server
Ctrl+C

# Stop all Docker containers
docker-compose down

# Stop and delete all data (wipes PostgreSQL + Redis volumes)
docker-compose down -v
# Note: ChromaDB data is in ./chroma_db and ./uploads — delete manually if needed
rm -rf ./chroma_db ./uploads
```

---

## Project File Structure

```
docuquery/
├── docker-compose.yml        # Spins up Kafka, Redis, PostgreSQL
├── .env.example              # Config template — copy to .env
├── requirements.txt          # Python packages
└── app/
    ├── main.py               # App entry point, starts consumer thread
    ├── config.py             # Reads settings from .env
    ├── database.py           # PostgreSQL connection (SQLAlchemy)
    ├── models.py             # DB tables: Document, QueryLog
    ├── schemas.py            # API request/response shapes (Pydantic)
    ├── redis_client.py       # Cache helpers + sliding-window rate limiter
    ├── kafka_producer.py     # Publishes document jobs to Kafka
    ├── kafka_consumer.py     # Background processor: file → chunks → ChromaDB
    ├── vector_store.py       # ChromaDB: store and search chunk vectors
    ├── document_processor.py # Text extraction (PDF/TXT) and chunking
    ├── llm_service.py        # Calls Gemini API to generate answers
    ├── rag_service.py        # Orchestrates the full RAG pipeline
    └── routes/
        ├── documents.py      # POST /documents, GET /documents/{id}
        └── query.py          # POST /query
```

---

## Common Issues and Fixes

**"NoBrokersAvailable" error on startup**
Kafka isn't ready yet. Wait 15–20 seconds after `docker-compose up -d` and restart the app.

**"Connection refused" on PostgreSQL**
Run `docker-compose ps` — if postgres shows `starting`, wait 10 more seconds.

**ChromaDB downloads slowly on first query**
Normal — it's downloading the ONNX embedding model (~25 MB). Only happens once.

**"Document not found" on query**
Make sure to use the `doc_id` from the upload response, not the filename.

**Mock responses instead of AI answers**
Set `GEMINI_API_KEY` in your `.env` file. Get a free key at https://aistudio.google.com/app/apikey
# DocuQuery
