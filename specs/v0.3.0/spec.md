---
id: 0003
title: Multimodal embeddings & vector store
status: Proposed
release: v0.3.0
created: 2026-07-21
updated: 2026-07-21
---

# 0003 — Multimodal embeddings & vector store

## Summary

Embed chunks — text via text embeddings, table/figure crops via **multimodal
embeddings** — and persist them in pgvector with the metadata needed for filtered,
multimodal retrieval. Expose retrieval primitives (vector search, optional hybrid
BM25, metadata filters) that the agent will use in [0004](../v0.4.0/spec.md).

## Problem / motivation

Text-only embeddings can't retrieve a chart or a table by its visual content. To
answer questions whose evidence is a figure or table, those regions must be embedded
in a shared space with the query. This spec makes retrieval multimodal and defines
the single-store (pgvector) schema and query primitives everything else depends on.

## Goals

- Text chunks embedded with a text-embedding model; table/figure crops embedded with
  a **multimodal** model (image + caption), all via the `EmbeddingProvider` interface.
- pgvector schema storing vectors with `{model, dimensions, modality}` and source
  metadata, so provider switches are detectable and re-embeddable.
- Retrieval primitives: k-NN vector search, metadata filtering (document, page,
  modality), and optional hybrid (vector + Postgres FTS/BM25) ranking.
- A re-embedding path when the configured embedding model changes.

## Non-goals

- The agent loop and answer generation (0004).
- Chat UI and citations rendering (0005).

## Requirements

### Functional

- **FR1 — Embed text** — Text chunks → text embeddings via `EmbeddingProvider.embedText`.
- **FR2 — Embed multimodal** — Table/figure crops (+ caption) → multimodal embeddings
  via `EmbeddingProvider.embedMultimodal`.
- **FR3 — Persist** — Store vectors in pgvector alongside `{chunk_id, document_id,
  modality, page, bbox, model, dimensions}`.
- **FR4 — Vector search** — `searchByVector(embedding, k, filters)` returning ranked
  chunks with scores and source metadata.
- **FR5 — Hybrid + rerank** — Combine vector similarity with **Postgres full-text
  search** for keyword-heavy queries, then rerank the merged candidates with a **NIM
  reranker** (`nvidia/llama-3.2-nv-rerankqa-1b-v2`); behind a flag. The reranker also
  merges the two embedding spaces (crop/text) into a single relevance ordering.
- **FR6 — Filters** — Restrict retrieval by document set, page range, and modality.
- **FR7 — Re-embed** — Detect vectors whose `model`/`dimensions` differ from the active
  config and provide a re-embedding job.

### Non-functional

- **NFR1 — Provider-agnostic** — All embedding calls go through `EmbeddingProvider`;
  no vendor SDK in callers.
- **NFR2 — Indexed** — Appropriate pgvector index (HNSW/IVFFlat) for interactive
  latency at expected corpus sizes; documented tradeoffs.
- **NFR3 — Dimension-safe** — Mixed dimensions across models handled explicitly
  (separate columns/tables or guarded queries); no silent dimension mismatches.
- **NFR4 — Tested** — Retrieval recall@k on a small labelled multimodal set, including
  table/figure questions.

## Design / approach

- **Embedding vendor (OQ2 — resolved: NVIDIA NIM, free tier).** Default to **NVIDIA
  NIM** (build.nvidia.com free tier) behind `EmbeddingProvider`: **NV-CLIP**
  (`nvidia/nvclip`, 1024-dim, image+text space) for table/figure crops, and
  **`nvidia/llama-3.2-nv-embedqa-1b-v2`** (Matryoshka, truncated to 1024-dim; 8192-token
  context, multilingual) for text chunks. Both are 1024-dim, so a single `vector(1024)`
  column type suffices; NIM exposes an OpenAI-compatible embeddings API, keeping the
  adapter thin. Validated on sample crops during the build. Fully swappable (e.g. to
  Voyage/Cohere) via config.
- **Schema:** vectors on `chunks` (or a dedicated `embeddings` table keyed by chunk),
  carrying model + dimensions + modality. Text and crop vectors are two logical spaces
  (different models) but with NIM both are 1024-dim, so one `vector(1024)` column type
  suffices, discriminated by `modality`; the query is embedded in each relevant space
  and results are unioned. Guard against dimension drift if a swapped provider changes
  dims (store `dimensions`; re-embed on mismatch).
- **Indexing:** choose HNSW vs. IVFFlat based on corpus size and recall/latency; record
  the decision.
- **Hybrid & rerank (OQ3 — resolved: Postgres FTS + NIM reranker).** Keyword signal via
  **Postgres full-text search** (in-database, no new service), unioned with vector
  results, then reranked by a **free NVIDIA NIM reranker**
  (`nvidia/llama-3.2-nv-rerankqa-1b-v2`) over the merged candidate set. The reranker also
  supplies the common relevance ordering across the two embedding spaces (crop vs. text)
  from OQ2. Kept behind the hybrid flag and measured on the recall@k eval; all models on
  the NIM free tier.

## Acceptance criteria

- A `ready` document from 0002 has vectors for all three modalities, each tagged with
  model + dimensions.
- `searchByVector` returns correctly ranked, filterable results across modalities.
- A table/figure question retrieves the relevant crop chunk in top-k on the eval set.
- Changing the embedding model flags existing vectors and a re-embed job restores them.

## Dependencies

- Requires [0001](../v0.1.0/spec.md), [0002](../v0.2.0/spec.md).
- Feeds [0004](../v0.4.0/spec.md).

## Open questions

- ~~**OQ2** — Default multimodal embedding vendor + dimensions.~~ **Resolved: NVIDIA
  NIM** (free tier) — NV-CLIP (crops) + `llama-3.2-nv-embedqa-1b-v2` (text), both
  1024-dim, behind `EmbeddingProvider`.
- ~~**OQ3** — Is Postgres FTS/BM25 sufficient for hybrid search?~~ **Resolved: Postgres
  full-text search + a free NIM reranker** (`llama-3.2-nv-rerankqa`) over merged
  candidates; in-database, no dedicated engine for v1.
