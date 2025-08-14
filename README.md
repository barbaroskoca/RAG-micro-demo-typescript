# RAG micro‑demo (TypeScript)

A tiny, batteries‑included Retrieval‑Augmented Generation (RAG) starter using **LangChainJS** with either **pgvector** (Postgres) or a simple **in‑memory** store. It ingests local Markdown files, creates embeddings, and answers questions with citations. Optional reranking via Cohere or Jina.

---

## Features

* **Ingest → chunk → embed** Markdown docs in `./docs/`
* **Query** with similarity search, optional **cross‑encoder reranker**
* **Switchable vector store**: `pgvector` (persistent) or `MemoryVectorStore` (demo)
* **Returns sources** (file + chunk) for transparency
* Minimal, readable code with clear extension points

---

## Project layout

```
.
├── docs/                      # Put your .md files here
├── src/
│   ├── config.ts              # Env + store selection
│   ├── loaders.ts             # Markdown → LangChain Documents
│   ├── vectorstore.ts         # PGVector or MemoryVectorStore
│   ├── ingest.ts              # Ingest CLI
│   ├── query.ts               # Query CLI (with optional reranker)
│   └── types.ts               # Shared types
├── package.json
├── tsconfig.json
├── .env.example
└── README.md
```

---

## Quickstart

### 1) Prereqs

* Node 18+
* For pgvector mode: Postgres 15+ with the `pgvector` extension.

### 2) Install

```bash
npm init -y
npm i typescript ts-node @types/node -D
npm i dotenv zod globby gray-matter
npm i @langchain/openai @langchain/community langchain
npm i pg # only used when DATABASE_URL is set (pgvector mode)
# Optional rerankers
npm i cohere-ai               # for COHERE_API_KEY reranker
npm i @jina-ai/jina-embeddings-client # (optional alt embeddings)
```

### 3) Env

Create `.env` (copy from `.env.example`):

```
# Embeddings (choose one; OpenAI shown)
OPENAI_API_KEY=sk-...
EMBEDDING_MODEL=text-embedding-3-small

# LLM for final answer (OpenAI shown)
OPENAI_CHAT_MODEL=gpt-4o-mini

# Vector store: leave unset for in-memory
DATABASE_URL=postgres://user:pass@localhost:5432/rag
PGVECTOR_TABLE=rag_documents

# Optional reranker
COHERE_API_KEY=
RERANKER=cohere   # or "none"
# Alternatively: RERANKER=jina with JINA_API_KEY=...
```

### 4) Prepare Postgres (only for pgvector)

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

The code will auto‑create the table if missing.

### 5) Add scripts

```json
{
  "scripts": {
    "ingest": "ts-node src/ingest.ts",
    "query": "ts-node src/query.ts"
  }
}
```

### 6) Put some docs

Drop Markdown files into `./docs/`. Frontmatter is supported but optional.

### 7) Ingest

```bash
npm run ingest
```

### 8) Query

```bash
npm run query -- "What does the deployment process say about rollbacks?"
```

---

## How it works (concepts)

### Chunking

We use `RecursiveCharacterTextSplitter` with sensible defaults (e.g., 800 tokens, 100 overlap). The goal is to keep chunks self‑contained while preserving context via overlap. Tweak for your domain density.

### Embeddings

Each chunk is embedded (default: OpenAI `text-embedding-3-small`) to produce vectors stored in pgvector or in memory. Smaller models are cheaper; larger ones may retrieve more accurately.

### Retrieval

We perform vector similarity search (top‑k). If a reranker is configured, we over‑fetch (e.g., k=12) then **rerank** down to a smaller k (e.g., 4) using a cross‑encoder (e.g., Cohere Rerank), which often improves precision.

### Caching

* **Persistent store**: pgvector keeps your embeddings so re‑ingests are incremental.
* **Stateless demo**: in-memory store loses state on process exit.
* For LLM response caching, you can plug in Redis via LangChain’s cache interface; for embeddings, rely on pgvector persistence or wrap your own disk cache if needed.

### Productionization (sketch)

* **Dataflow**: put ingestion behind a job/queue (e.g., BullMQ) and make it idempotent (hash content → upsert on change).
* **Schema**: maintain stable `id` for each doc + chunk; store metadata (source path, title, headings, checksum, timestamp).
* **Observability**: log retrieval sets; keep answer traces with prompt + context; add evaluations (hit rate, MRR, groundedness).
* **Safety**: return citations (we do); add content filtering; guardrails for hallucinations.
* **API**: expose `/ingest` and `/query` endpoints (Next.js / NestJS); stream tokens to clients.
* **Auth & secrets**: use a k/v or secret manager; never hardcode keys.
* **Scale**: shard by tenant; batch embeddings; use background backfills; set PG connection pool.
