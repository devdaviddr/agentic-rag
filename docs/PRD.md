---
title: Product Requirements Document — Agentic RAG
status: Draft
version: 0.1.0
created: 2026-07-21
updated: 2026-07-21
owner: Daniel Ruffolo
---

# Agentic RAG — Product Requirements Document

## 1. Overview

**Agentic RAG** is a self-hostable, full-stack application that lets a user ingest
their own documents and ask natural-language questions about them. Rather than a
single-shot "embed the question → fetch top-k → stuff into a prompt" pipeline, an
**agent** plans and executes retrieval: it decides what to search for, searches
across **text, tables, and images**, reads what it finds, decides whether it has
enough, and composes an answer with **inline citations** back to the exact source
passage, table, or figure.

The application runs entirely against **cloud model API endpoints** (LLMs and
embeddings) — no local GPU required — behind a **provider-agnostic** abstraction so
the operator chooses their provider(s). It is built on top of the
[`nextjs-fullstack-boilerplate`](https://github.com/devdaviddr/nextjs-fullstack-boilerplate)
(Next.js 16, Auth.js v5, Postgres + Drizzle, MinIO, Docker, CI) for the web app and
platform, with **Python services** for the compute-heavy ingestion and parsing work.

### One-line pitch

> Point it at your documents, ask questions in plain language, and get answers an
> agent has actually researched — grounded in your text, tables, and figures, with
> citations you can click.

## 2. Problem statement

Naïve RAG is brittle in the ways that matter most to real documents:

- **It ignores non-text content.** Most RAG pipelines flatten a PDF to plain text,
  silently dropping tables and figures — often exactly where the answer lives
  (financials, lab results, schematics, charts).
- **It is single-shot.** One embedding of one question, one retrieval, one
  generation. Multi-part questions, questions that require cross-referencing, and
  questions whose best search terms differ from the user's phrasing all degrade.
- **It cannot be trusted without citations.** Users can't tell a grounded answer
  from a hallucinated one, and can't verify against the source.
- **It is hard to self-host.** Many solutions assume a managed vector DB, a specific
  model vendor, or a GPU. Operators who want to own their data and their bill are
  left to assemble it themselves.

Agentic RAG exists to fix all four: multimodal retrieval, an agent loop, first-class
citations, and a single-command self-host story on commodity hardware using cloud
model APIs.

## 3. Goals & non-goals

### Goals

- **G1 — Multimodal answers.** Retrieve and reason over text, tables, and images;
  cite whichever modality the answer came from.
- **G2 — Agentic retrieval.** An agent decomposes questions, issues multiple
  targeted retrievals, judges sufficiency, and iterates before answering.
- **G3 — Verifiable citations.** Every claim in an answer links to a specific source
  chunk (document + page + region), rendered inline and clickable.
- **G4 — Self-hostable in one command.** Clone → configure keys → `make setup` →
  live app on the operator's own hardware/domain, reusing the boilerplate's
  Docker + Cloudflare Tunnel deployment.
- **G5 — Provider-agnostic.** LLM and embedding providers are pluggable; Claude,
  OpenAI, and Gemini are all first-class from day one, selected by config.
- **G6 — Full ingestion UX.** A first-class interface to upload documents, watch
  ingestion progress, browse the corpus, and manage/delete sources.

### Non-goals (v1)

- **NG1 —** No local/on-device model inference (no GPU story). Cloud APIs only in v1.
- **NG2 —** No real-time collaboration / multi-user shared workspaces beyond the
  boilerplate's existing auth + RBAC. Corpora are per-user (or per-role) — not a
  shared team knowledge base with granular sharing in v1.
- **NG3 —** No fine-tuning or model training. Retrieval-augmented only.
- **NG4 —** No web crawling / live external data sources in v1 — the corpus is
  user-uploaded documents.
- **NG5 —** Not a general chatbot: out-of-corpus questions are answered only insofar
  as the agent can ground them, and are otherwise declined with a clear message.

## 4. Target users

| Persona | Needs | Why Agentic RAG |
| --- | --- | --- |
| **Self-hosting individual** (the primary user) | Own their documents and their bill; ask questions across a personal library of PDFs/reports | One-command self-host, bring-your-own API key, data never leaves their box except to the chosen model API |
| **Small team / operator** | A private Q&A layer over internal docs (handbooks, specs, contracts) | RBAC from the boilerplate, provider choice, auditable citations |
| **Developer / tinkerer** | A clean, hackable agentic-RAG reference to extend | Provider-agnostic abstractions, documented specs, typed schema |

## 5. Key user journeys

1. **Ingest.** User drags a folder of PDFs into the upload interface. Each document
   is queued; the UI shows per-document status (queued → parsing → chunking →
   embedding → ready) and surfaces failures with reasons. Tables and images are
   extracted and previewable.
2. **Ask.** User types a question. The agent shows its work (optional "thinking"
   trace: sub-questions, searches issued, what it retrieved), then streams a final
   answer with inline citation markers.
3. **Verify.** User clicks a citation. A source panel opens to the exact page with
   the cited text/table/figure highlighted.
4. **Manage.** User browses the corpus, inspects a document's extracted chunks and
   figures, re-runs ingestion, or deletes a source (and its vectors).

## 6. Functional requirements

### 6.1 Document ingestion

- **FR1 — Upload.** Accept PDF, DOCX, PPTX, XLSX, Markdown, HTML, TXT, and common
  image formats (PNG/JPG). Uploads go to S3-compatible object storage (MinIO), with
  size/type validation and per-user quota (reuse boilerplate file-upload).
- **FR2 — Parse (Python service).** Extract a structured representation per document:
  text blocks, **tables** (as structured data + rendered image), and **figures/images**
  (as cropped image regions), each with **layout metadata** (page number, bounding
  box, reading order).
- **FR3 — Chunk.** Produce retrieval units that respect structure: text is chunked
  semantically; each table and figure is its own retrievable unit with a text
  description/caption alongside the image crop.
- **FR4 — Embed.** Generate embeddings via the configured provider — **text
  embeddings** for text chunks and **multimodal embeddings** (image + text) for
  table/figure crops — and persist vectors + metadata to pgvector.
- **FR5 — Status & lifecycle.** Every document exposes ingestion state and errors;
  ingestion is idempotent and re-runnable; deleting a document removes its chunks,
  vectors, and stored assets.
- **FR6 — Ingestion interface.** A web UI to upload, monitor progress in real time,
  browse ingested documents, preview extracted tables/figures, and delete sources.

### 6.2 Agentic retrieval & answering

- **FR7 — Query planning.** The agent decomposes a question into one or more
  retrieval steps and reformulates search queries (it does not just embed the raw
  question).
- **FR8 — Multimodal retrieval.** Retrieval spans text, table, and figure vectors;
  results carry their modality and source metadata. Hybrid retrieval (per-space
  vector k-NN fused with Postgres full-text search) is **reranked** into one ordering,
  with metadata filtering (by document, page, type).
- **FR9 — Tool-use loop.** The agent runs a bounded reason→retrieve→observe loop
  (search corpus, fetch a full chunk, fetch a page image, list documents) until it
  judges it has sufficient grounding or hits a step budget.
- **FR10 — Grounded generation.** The final answer is generated by a vision-capable
  LLM given the retrieved text, tables, and image crops. If the corpus does not
  support an answer, the agent says so rather than inventing one.
- **FR11 — Citations.** Every substantive claim carries a citation to a specific
  source unit (document + page + region/modality). Citations are structured data,
  rendered inline in the answer and resolvable to a source-panel highlight.
- **FR12 — Streaming & trace.** Answers stream token-by-token; the agent's steps
  (sub-questions, searches, retrieved sources) are optionally surfaced as a
  collapsible trace.

### 6.3 Chat & interface

- **FR13 — Conversational sessions.** Persisted chat threads with history; follow-up
  questions retain context and can reuse prior retrievals.
- **FR14 — Source panel.** A side panel renders cited sources with the relevant
  region highlighted; clicking an inline citation scrolls/opens to it.
- **FR15 — Corpus scoping.** The user can scope a conversation to the whole corpus or
  a selected subset of documents.

### 6.4 Platform (inherited / extended from boilerplate)

- **FR16 — Auth & RBAC.** Auth.js v5 credentials + optional OAuth; roles gate admin
  functions (model config, all-corpus management). Inherited from boilerplate.
- **FR17 — Provider configuration.** An admin surface (and env config) to select LLM
  and embedding providers and models, and to supply API keys.
- **FR18 — Observability.** Per-request logging of agent steps, token usage, and cost
  estimates; basic usage dashboard.

## 7. Non-functional requirements

- **NFR1 — Self-hostable.** Runs on a single commodity host via Docker Compose;
  reuses the boilerplate's `make setup` clone-to-live and Cloudflare Tunnel (HTTPS,
  no open ports). No managed cloud services required beyond the chosen model APIs.
- **NFR2 — Provider-agnostic.** LLM and embedding access sits behind interfaces; a
  provider is added by implementing an adapter + config, with no changes to callers.
- **NFR3 — Privacy.** Documents and vectors stay on the operator's infrastructure.
  The only egress is to the operator-chosen model API endpoints. This boundary is
  documented explicitly.
- **NFR4 — Performance.** Interactive latency target: first token < 3 s for a typical
  question on a warm corpus; ingestion throughput sufficient for a few hundred pages
  in minutes on commodity hardware (bounded by provider embedding throughput).
- **NFR5 — Cost transparency.** Token and embedding usage is tracked and surfaced so
  the operator can see and cap spend.
- **NFR6 — Security.** Inherit the boilerplate's security posture (Argon2id, edge
  guards, single-use tokens, quotas). Uploaded documents are untrusted input —
  parsing runs in an isolated service with resource limits.
- **NFR7 — Extensibility.** Ingestion (parsers), retrieval (strategies), and agent
  tools are modular and independently testable.
- **NFR8 — Tested.** Unit + integration tests for ingestion and retrieval; E2E for the
  ask→cite→verify journey; green in CI (inherit boilerplate's Vitest + Playwright +
  Actions, add Python `pytest`).

## 8. System architecture (summary)

See [`docs/architecture.md`](./architecture.md) for detail. In brief:

```
┌────────────────────────────────────────────────────────────────┐
│  Browser (PWA)                                                  │
│  • Ingestion UI  • Chat + streaming answers  • Source panel     │
└───────────────┬────────────────────────────────────────────────┘
                │  HTTP / SSE
┌───────────────▼────────────────────────────────────────────────┐
│  Next.js app (from boilerplate)                                 │
│  • Auth.js v5 + RBAC   • Server Actions / Route Handlers        │
│  • Agent orchestration (reason→retrieve→answer loop)            │
│  • Provider abstraction: LLM + embeddings adapters              │
└───┬───────────────┬───────────────────────┬────────────────────┘
    │               │                       │
    │ enqueue       │ SQL + vector          │ model APIs (egress)
    ▼               ▼                       ▼
┌─────────┐   ┌──────────────┐      ┌────────────────────────┐
│ Python  │   │ Postgres 17  │      │ Cloud providers        │
│ ingest  │   │ + pgvector   │      │ Claude / OpenAI /      │
│ service │   │ (chunks,     │      │ Gemini (LLM),          │
│ (parse, │   │ vectors,     │      │ NVIDIA NIM (embeds)    │
│ chunk,  │   │ metadata)    │      └────────────────────────┘
│ embed)  │   └──────────────┘
└────┬────┘
     │ read/write assets
     ▼
┌──────────────┐
│ MinIO (S3)   │  original docs, table/figure image crops, page renders
└──────────────┘
```

## 9. Technology stack

| Layer | Choice | Notes |
| --- | --- | --- |
| Web app / API / agent orchestration | **Next.js 16** (App Router, RSC, Server Actions) + TypeScript | From `nextjs-fullstack-boilerplate` |
| Auth / RBAC / PWA / CI / Docker | Auth.js v5, Tailwind/shadcn, Playwright, GitHub Actions | Inherited from boilerplate |
| Database + vectors | **PostgreSQL 17 + pgvector**, Drizzle ORM | Single store for relational data and vectors |
| Object storage | **MinIO** (S3-compatible) | Original docs + extracted image assets |
| Ingestion / parsing | **Python service** (FastAPI worker) + **Docling** parser (PyMuPDF fallback) | Layout-aware parsing, chunking, embedding calls |
| LLM (generation + agent) | **Provider-agnostic**: Claude, OpenAI, Gemini | Vision-capable; behind an adapter interface |
| Embeddings + rerank | **NVIDIA NIM** (free tier) — NV-CLIP for crops, `llama-3.2-nv-embedqa` for text, `nv-rerankqa` for reranking; provider-agnostic | Crops embedded directly; text + crops both 1024-dim |
| Retrieval | pgvector k-NN (per space) + **Postgres FTS** hybrid, reranked | Single store; no dedicated vector/search engine |
| Deployment | Docker Compose + Cloudflare Tunnel | Boilerplate `make setup` / `make deploy` |

> Key library/vendor choices are decided — **Docling** (parsing) and **NVIDIA NIM**
> (embeddings + rerank) — with full detail and rationale in
> [`specs/v0.2.0`](../specs/v0.2.0/spec.md) and
> [`specs/v0.3.0`](../specs/v0.3.0/spec.md).

## 10. Data model (overview)

Extends the boilerplate schema (`users`, `roles`, `files`) with:

- **`documents`** — one per uploaded source; owner, storage key, status, error, page
  count, timestamps.
- **`chunks`** — retrieval units; FK to document, `modality` (text | table | figure),
  page, bounding box, text/caption, storage key for image crops.
- **`embeddings`** — pgvector column(s) per chunk; model + dimensions recorded so
  re-embedding on a provider change is detectable.
- **`conversations`** / **`messages`** — chat threads; messages store the answer plus
  structured **citations** (chunk references) and the agent trace.
- **`ingestion_jobs`** — queue/state for the Python worker; retries and status.

Full schema in [`specs/v0.3.0`](../specs/v0.3.0/spec.md)
and [`specs/v0.5.0`](../specs/v0.5.0/spec.md).

## 11. Roadmap → releases

Delivery is sliced one folder per release under [`specs/`](../specs/) — each with a
`spec.md` and a `plan.md`. Each release is independently shippable and testable.

| Release | Title | Delivers |
| --- | --- | --- |
| [v0.1.0](../specs/v0.1.0/spec.md) | Project foundation | Fork boilerplate, add Python service scaffold, pgvector, provider-abstraction skeleton, CI |
| [v0.2.0](../specs/v0.2.0/spec.md) | Document ingestion pipeline | Upload → parse → chunk → status; ingestion UI |
| [v0.3.0](../specs/v0.3.0/spec.md) | Multimodal embeddings & vector store | Text + multimodal embeddings, pgvector schema, retrieval primitives |
| [v0.4.0](../specs/v0.4.0/spec.md) | Agentic retrieval orchestration | Provider-agnostic LLM layer, agent tool-use loop, grounded generation |
| [v0.5.0](../specs/v0.5.0/spec.md) | Chat interface & citations | Conversations, streaming, inline citations, source panel |
| [v1.0.0](../specs/v1.0.0/spec.md) | Self-hosting & deployment | Compose topology, provider config, one-command setup, docs |

## 12. Success metrics

- **Answer groundedness** — ≥ 95% of answer claims carry a resolvable citation
  (measured on an internal eval set).
- **Retrieval quality** — measurable recall@k on a labelled multimodal QA set,
  including table/figure questions.
- **Time-to-first-token** — < 3 s p50 on a warm corpus.
- **Self-host success** — a fresh clone reaches a live, question-answering instance
  in < 30 min following the docs, with only API keys required.
- **Multimodal coverage** — table/figure questions answered correctly at parity with
  text questions on the eval set.

## 13. Risks & mitigations

| Risk | Mitigation |
| --- | --- |
| PDF layout parsing is the hardest, flakiest part | Isolate in the Python service behind a stable `parse()` interface (Docling default, PyMuPDF fallback); make ingestion re-runnable; fail loudly per-document |
| Provider abstraction leaks (capabilities differ, e.g. citations, vision, embedding dims) | Define a capability-based interface; record embedding model+dims per vector; document a re-embedding path on provider switch |
| Multimodal embedding vendor lock-in / cost | Default to NVIDIA NIM (free tier); keep embeddings behind a swappable adapter; store crops so re-embedding is possible; surface cost |
| Agent loops are slow or expensive | Bounded step budget, caching of retrievals within a conversation, cost tracking with caps |
| Untrusted document input (parser exploits) | Sandboxed parsing service, resource limits, type/size validation |
| Scope creep into a general chatbot | Enforce grounding; decline unsupported questions; NG5 |

## 14. Key decisions (resolved)

All initial open questions are now decided; each is recorded in its release's `spec.md`
and tracked in the [roadmap](./roadmap.md#key-decisions-open-questions).

- **OQ1 (v0.2.0) — PDF parser:** ✅ **Docling** (default), PyMuPDF fallback, behind a
  swappable `parse()` interface. Fully self-hosted; no extra egress.
- **OQ2 (v0.3.0) — Embeddings:** ✅ **NVIDIA NIM** (free tier) — **NV-CLIP**
  (`nvidia/nvclip`, 1024-dim) for crops, **`llama-3.2-nv-embedqa-1b-v2`** (Matryoshka →
  1024-dim) for text; both 1024-dim, behind `EmbeddingProvider` (swappable).
- **OQ3 (v0.3.0) — Hybrid search:** ✅ **Postgres full-text search + a free NIM reranker**
  (`llama-3.2-nv-rerankqa-1b-v2`) over merged candidates; in-database, no dedicated
  engine. The reranker also merges the two embedding spaces into one ordering.
- **OQ4 (v0.5.0) — Citation UX:** ✅ **Full-page render with the region highlighted**
  (bbox) as primary, standalone crop as inset/fallback.
- **OQ5 (v0.4.0/v1.0.0) — Corpus isolation:** ✅ **Per-user ownership + admin override**,
  enforced by an ownership predicate on every retrieval tool.

Remaining smaller decisions live in the individual specs' Open-questions sections (e.g.
caption generation strategy, agent-trace persistence depth).

## 15. Out of scope (restated)

Local/on-device inference; model fine-tuning; web crawling / live sources; shared
multi-tenant team knowledge bases with granular sharing; mobile-native apps (the PWA
covers mobile web). These may be considered post-v1.
