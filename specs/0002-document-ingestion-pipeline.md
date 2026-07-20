---
id: 0002
title: Document ingestion pipeline
status: Proposed
release: v0.2.0
created: 2026-07-21
updated: 2026-07-21
---

# 0002 — Document ingestion pipeline

## Summary

Turn an uploaded document into structure-preserving retrieval units: parse text,
tables, and figures with layout metadata, render image crops to storage, chunk the
content, and expose real-time ingestion status through a first-class upload
interface. Embedding is defined here but implemented in [0003](./0003-multimodal-embeddings-and-vector-store.md).

## Problem / motivation

Answer quality is capped by ingestion quality. Flattening a document to plain text
discards tables and figures — often where the answer is. Ingestion must therefore
preserve structure, be observable (users need to see progress and failures), and be
re-runnable (parsing is the flakiest step in the system).

## Goals

- Layout-aware parsing that separates **text**, **tables**, and **figures**, each
  with page number, bounding box, and reading order.
- Table/figure regions rendered as image crops and stored in MinIO, referenced by
  their chunk.
- Structure-respecting chunking: semantic text chunks; each table and figure is its
  own retrieval unit with a text caption/description.
- An ingestion UI: upload, per-document live status, corpus browse, table/figure
  preview, delete.
- Idempotent, re-runnable ingestion with clear per-document failure reasons.

## Non-goals

- Embedding and vector storage (defined here, implemented in 0003).
- Retrieval and answering (0004).
- OCR of handwriting or exotic formats beyond the supported set.

## Requirements

### Functional

- **FR1 — Upload** — Accept PDF, DOCX, PPTX, XLSX, MD, HTML, TXT, PNG, JPG; validate
  type/size; store original in MinIO; create `documents` + `ingestion_jobs` rows.
- **FR2 — Parse** — Python service extracts text blocks, tables (structured + region),
  and figures (region) with `{page, bbox, order}` metadata.
- **FR3 — Render crops** — Each table/figure region is rendered to an image and stored
  in MinIO; the storage key is attached to its chunk.
- **FR4 — Chunk** — Produce `chunks` rows: semantic text units; one unit per table and
  per figure, each with a caption/description for text-side retrieval.
- **FR5 — Status** — `documents.status` transitions queued → parsing → chunking →
  (embedding, 0003) → ready | failed(reason); surfaced via API.
- **FR6 — Lifecycle** — Re-run ingestion for a document idempotently; delete cascades
  to chunks, vectors, and MinIO assets.
- **FR7 — Ingestion UI** — Drag-and-drop upload; live progress (SSE/polling); corpus
  list; document detail with extracted tables/figures preview; delete action.

### Non-functional

- **NFR1 — Sandboxed** — Parsing runs only in the ingestion service with resource
  limits; malformed input fails that document, never the service.
- **NFR2 — Throughput** — A few hundred pages ingest in minutes on commodity hardware
  (embedding throughput bounded by provider, per 0003).
- **NFR3 — Observability** — Job state, timings, and errors are logged and queryable.
- **NFR4 — Tested** — Golden-file parser tests on sample PDFs (incl. table/figure
  cases); E2E upload→ready.

## Design / approach

- **Parser choice (OQ1):** evaluate layout-aware libraries balancing table/figure
  fidelity against self-host footprint; pin the choice here with rationale. Wrap it
  behind a stable `parse(document) -> ParsedDoc` interface so it can be swapped.
- **Job model:** web app enqueues `ingestion_jobs`; the Python worker claims, processes,
  and updates status. Retries with backoff; poison jobs marked failed with reason.
- **Chunking:** text chunked semantically with overlap; tables/figures kept whole with
  generated captions (caption generation may use the vision LLM via the provider layer).
- **UI:** built on the boilerplate's file-upload + app shell; real-time status via SSE
  or polling.

## Acceptance criteria

- Uploading a PDF with tables and figures yields `chunks` of all three modalities with
  correct page/bbox metadata and stored crops.
- The UI shows live status and, on completion, previews extracted tables/figures.
- A deliberately malformed file marks only its own document `failed` with a reason.
- Re-running ingestion on a document produces no duplicates.
- Deleting a document removes its chunks and MinIO assets.

## Dependencies

- Requires [0001](./0001-project-foundation.md).
- Feeds [0003](./0003-multimodal-embeddings-and-vector-store.md) (embedding) and
  [0004](./0004-agentic-retrieval-orchestration.md) (retrieval).

## Open questions

- **OQ1** — Final PDF layout-parsing library (fidelity vs. footprint).
- Caption generation: heuristic vs. vision-LLM, and its cost/latency impact.
