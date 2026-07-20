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
BM25, metadata filters) that the agent will use in [0004](./0004-agentic-retrieval-orchestration.md).

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
- **FR5 — Hybrid (optional)** — Combine vector similarity with Postgres full-text/BM25
  for keyword-heavy queries; behind a flag.
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

- **Multimodal vendor (OQ2):** pin the default multimodal embedding model and its
  dimensions here (e.g. a Voyage multimodal model), validated on sample table/figure
  crops. Keep it behind `EmbeddingProvider` so it is swappable.
- **Schema:** vectors on `chunks` (or a dedicated `embeddings` table keyed by chunk),
  carrying model + dimensions. If text and multimodal models differ in dimension,
  store them in separate vector columns/tables and union results at query time.
- **Indexing:** choose HNSW vs. IVFFlat based on corpus size and recall/latency; record
  the decision.
- **Hybrid (OQ3):** implement BM25 via Postgres FTS as an optional re-ranking signal;
  evaluate whether it materially helps before committing further.

## Acceptance criteria

- A `ready` document from 0002 has vectors for all three modalities, each tagged with
  model + dimensions.
- `searchByVector` returns correctly ranked, filterable results across modalities.
- A table/figure question retrieves the relevant crop chunk in top-k on the eval set.
- Changing the embedding model flags existing vectors and a re-embed job restores them.

## Dependencies

- Requires [0001](./0001-project-foundation.md), [0002](./0002-document-ingestion-pipeline.md).
- Feeds [0004](./0004-agentic-retrieval-orchestration.md).

## Open questions

- **OQ2** — Default multimodal embedding vendor + dimensions (validate on samples).
- **OQ3** — Is Postgres FTS/BM25 sufficient for hybrid search, or is a dedicated
  engine warranted later?
