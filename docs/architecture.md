---
title: Architecture
status: Draft
created: 2026-07-21
updated: 2026-07-21
---

# Architecture

This document describes how Agentic RAG is put together: the components, how a
document flows through ingestion, and how a question flows through the agent to a
cited answer. It complements the [PRD](./PRD.md) and the numbered
[specs](../specs/).

## Design principles

1. **Cloud APIs, no GPU.** All model inference (LLM + embeddings) is a network call
   to a provider endpoint. The operator supplies keys; nothing about the runtime
   assumes local accelerators.
2. **Provider-agnostic by construction.** Callers depend on interfaces
   (`LlmProvider`, `EmbeddingProvider`), never a vendor SDK directly. Adding a
   provider is an adapter + config change.
3. **Structure-preserving ingestion.** Documents are not flattened to text. Tables
   and figures survive as first-class, individually retrievable units with layout
   metadata.
4. **The agent does retrieval, not a fixed pipeline.** Retrieval is a set of tools
   the agent calls in a bounded loop, not a hard-coded top-k fetch.
5. **Citations are data, not prose.** The model emits structured references; the UI
   renders and resolves them. Groundedness is verifiable.
6. **Reuse the platform.** Auth, RBAC, storage, PWA, Docker, CI, and deployment come
   from the Next.js boilerplate and are extended, not reinvented.

## Component overview

| Component | Responsibility | Tech |
| --- | --- | --- |
| **Web app** | UI (ingestion, chat, source panel), API, agent orchestration | Next.js 16 (TS), from boilerplate |
| **Provider layer** | Uniform LLM + embedding access across vendors | TS interfaces + adapters |
| **Ingestion service** | Parse → chunk → embed; layout-aware, sandboxed | Python (FastAPI worker) |
| **Database** | Relational data + vectors in one store | Postgres 17 + pgvector, Drizzle |
| **Object storage** | Original docs + extracted image assets | MinIO (S3-compatible) |
| **Model providers** | Generation + embeddings (the only egress) | LLM: Claude / OpenAI / Gemini · Embeddings + rerank: NVIDIA NIM (default) — NV-CLIP, nv-embedqa, nv-rerankqa |

### Why Python for ingestion

Layout-aware document parsing, table extraction, and image cropping are best served
by the Python ecosystem. Isolating this in a separate service also (a) sandboxes
untrusted document input away from the web app, (b) lets ingestion scale/retry
independently, and (c) keeps heavy dependencies out of the Next.js image. The web
app enqueues jobs and reads results; it never parses documents itself.

## Ingestion data flow

```
upload ─▶ MinIO (original)
   │
   └─▶ ingestion_jobs (queued)
          │
          ▼  Python ingestion service
   ┌──────────────────────────────────────────────┐
   │ 1. Fetch original from MinIO                  │
   │ 2. Parse: text blocks, tables, figures        │
   │    + layout metadata (page, bbox, order)      │
   │ 3. Render table/figure crops ─▶ MinIO         │
   │ 4. Chunk: semantic text units; each table /   │
   │    figure = one unit (+ caption/description)   │
   │ 5. Embed: text→text-embeddings;               │
   │    table/figure crop→multimodal embeddings    │
   │ 6. Persist chunks + vectors + metadata        │
   └──────────────────────────────────────────────┘
          │
          ▼
   documents.status = ready   (or failed + reason)
```

Ingestion is **idempotent and re-runnable** per document. Deleting a document
cascades to its chunks, vectors, and MinIO assets. Concretely, step 2 parses with
**Docling** (PyMuPDF fallback), and step 5 embeds via **NVIDIA NIM** (free tier) —
`llama-3.2-nv-embedqa-1b-v2` for text (Matryoshka → 1024-dim) and **NV-CLIP** for
crops (1024-dim) — so all vectors share one `vector(1024)` column type.

## Query (agent) data flow

```
user question
   │
   ▼
Agent orchestrator (Next.js)
   │  bounded reason → act → observe loop
   │
   ├─▶ tool: search_corpus(query, filters, modality)
   │       → embed query per space (NV-CLIP + nv-embedqa), k-NN in pgvector,
   │         fuse with Postgres FTS, rerank (NIM) → ranked chunks (scoped to user)
   ├─▶ tool: get_chunk(id) / get_page_image(doc, page)
   ├─▶ tool: list_documents()
   │
   ▼  (agent judges sufficiency or hits step budget)
Grounded generation (vision-capable LLM)
   │  given retrieved text + table/figure crops
   ▼
Answer with structured citations  ─▶ streamed to UI
   │
   ▼
Source panel resolves each citation → doc + page + highlighted region
```

Key points:

- The agent **reformulates** queries and may issue **several** retrievals for one
  question (decomposition, follow-up, cross-reference).
- Retrieval is **multimodal**: text, table, and figure vectors are all searchable;
  results self-describe their modality.
- Retrieval is **hybrid + reranked**: per-space vector k-NN (crop space via NV-CLIP,
  text space via `nv-embedqa`) is fused with Postgres full-text search, then a **NIM
  reranker** (`nv-rerankqa`) produces the final cross-space ordering (cross-space
  vector scores are never compared directly).
- Retrieval is **scoped**: every tool applies a per-user ownership predicate (admin
  override), plus any conversation-level document filter.
- Generation is **vision-capable**: table/figure crops are passed to the model as
  images, so it can read a chart or table directly.
- **Citations** are emitted as structured references (chunk id → document, page,
  region, modality), persisted on the message, and rendered inline + in the source
  panel.

## Provider abstraction

Two interfaces isolate vendor differences:

- **`LlmProvider`** — chat/completion with tool-use and vision; streaming; exposes
  capability flags (vision, tool-use, native-citations) so orchestration can adapt.
- **`EmbeddingProvider`** — `embedText(...)` and `embedMultimodal(...)`; reports the
  model id and vector dimensions.
- **`Reranker`** — `rerank(query, candidates)`; produces the final relevance ordering
  over the fused candidate pool.

Every stored vector records its **embedding model + dimensions**, so a provider
switch is detectable and a re-embedding path is well-defined. The default LLM adapters
are Claude, OpenAI, and Gemini; the default embedding + rerank provider is **NVIDIA
NIM** (free tier — NV-CLIP for crops, `nv-embedqa` for text, `nv-rerankqa` for
reranking). Everything is selected by environment/admin config and swappable (e.g. to
Voyage/Cohere) without touching callers.

## Storage model

- **Postgres 17 + pgvector** holds relational data (`documents`, `chunks`,
  `conversations`, `messages`, `ingestion_jobs`) and the vector column(s) on chunks.
  One database → one thing to run, back up, and restore (reusing the boilerplate's
  backup tooling).
- **MinIO** holds binary assets: original uploads, rendered page images, and
  table/figure crops referenced by chunks.

## Deployment topology

Reuses the boilerplate's Docker Compose + Cloudflare Tunnel model, adding the Python
ingestion service:

```
docker-compose:
  web        → Next.js app (agent orchestration, API, UI)
  ingestion  → Python FastAPI worker (parse/chunk/embed)
  db         → Postgres 17 + pgvector
  minio      → S3-compatible object storage
  tunnel     → Cloudflare Tunnel (HTTPS, no open ports)
```

`make setup` takes a fresh clone to a live instance; the operator supplies model API
keys and picks providers. See [`specs/v1.0.0`](../specs/v1.0.0/spec.md).

## Security & trust boundaries

- **Untrusted input:** uploaded documents are parsed only inside the sandboxed
  ingestion service, with resource limits and strict type/size validation.
- **Egress boundary:** the *only* data leaving the operator's infrastructure is what
  is sent to the chosen model API endpoints — chunk text/crops sent to NVIDIA NIM for
  embedding, query text + candidates sent for embedding/reranking, and query text +
  retrieved chunk text + image crops sent to the LLM during generation. This is
  documented so operators understand exactly what each provider sees.
- **Corpus isolation:** documents are per-user; every retrieval tool applies an
  ownership predicate (admin role overrides), so a user can only ever retrieve their
  own corpus.
- **Platform security** (auth, RBAC, hashing, tokens, quotas) is inherited from the
  boilerplate.

## What this document does not fix

The key library/vendor choices are now decided (Docling for parsing; NVIDIA NIM for
embeddings + rerank; per-user corpus isolation) and recorded in the release specs.
Exact schema DDL, migrations, and the agent's tool signatures remain to be defined in
those specs and their plans, which are the source of truth for implementation.
