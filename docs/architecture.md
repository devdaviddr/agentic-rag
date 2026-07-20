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
| **Model providers** | Generation + embeddings (the only egress) | Claude / OpenAI / Gemini; multimodal embeddings vendor |

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
cascades to its chunks, vectors, and MinIO assets.

## Query (agent) data flow

```
user question
   │
   ▼
Agent orchestrator (Next.js)
   │  bounded reason → act → observe loop
   │
   ├─▶ tool: search_corpus(query, filters, modality)
   │       → embed query, vector search in pgvector (+ optional BM25),
   │         return ranked chunks with source metadata
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

Every stored vector records its **embedding model + dimensions**, so a provider
switch is detectable and a re-embedding path is well-defined. Adapters for Claude,
OpenAI, and Gemini (LLM) and the chosen text + multimodal embedding vendors ship in
the provider layer; selection is by environment/admin config.

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
  is sent to the chosen model API endpoints (query text, retrieved chunk text, and
  image crops during generation). This is documented so operators understand exactly
  what a provider sees.
- **Platform security** (auth, RBAC, hashing, tokens, quotas) is inherited from the
  boilerplate.

## What this document does not fix

Concrete library choices (PDF layout parser, multimodal embedding vendor), exact
schema DDL, and the agent's tool signatures are decided in the numbered specs, which
are the source of truth for implementation.
