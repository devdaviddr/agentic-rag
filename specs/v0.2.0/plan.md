---
plan_for: 0002
title: Document ingestion pipeline — implementation plan
status: Proposed
branch: feature/0002-document-ingestion-pipeline
created: 2026-07-21
updated: 2026-07-21
---

# 0002 — Implementation plan

> Implements [`specs/v0.2.0/spec.md`](./spec.md).
> This plan turns that spec into ordered, agent-executable coding tasks. Execute tasks in
> order, respecting `depends`.

## Objective

Turn an uploaded file into structure-preserving retrieval units. A user drags a document
into a first-class ingestion UI; the web app validates it, stores the original in MinIO,
and enqueues a job. The Python service parses it layout-aware (text blocks, tables,
figures with `{page, bbox, order}`), renders table/figure crops to MinIO, and writes
`chunks` (semantic text units + one unit per table/figure with a caption). Status moves
`queued → parsing → chunking → ready | failed(reason)` and is shown live in the UI, which
also browses the corpus, previews tables/figures, and deletes documents (cascade). This
slice **stops at chunk creation** — embedding is defined here but implemented in 0003; a
clear seam (`ready_for_embed` boundary + `embedding` status value reserved) is left open.

## Prerequisites

- **Spec 0001 shipped**: forked boilerplate web app, Python ingestion scaffold
  (`ingestion/` with FastAPI `/health`), pgvector enabled, provider-abstraction skeleton,
  extended Docker Compose (web + ingestion + db + minio), CI green on both runtimes.
- MinIO bucket + credentials wired from the boilerplate file-upload feature.
- No model API keys required for this slice (caption generation defaults to a heuristic;
  the vision-LLM caption path is behind a flag and stubbed — see T7 and Risks).

## Tech stack for this slice

- **Web app:** Next.js 16 (App Router, RSC, Server Actions, Turbopack), TypeScript 5.9
  (strict), Drizzle ORM, `@aws-sdk/client-s3` (boilerplate MinIO client), Zod, shadcn/ui
  (dropzone/card/badge/table/dialog), Vitest + Playwright, `pnpm`.
- **DB:** PostgreSQL 17 + pgvector; Drizzle migrations (`documents`, `chunks`,
  `ingestion_jobs` + status enums). No vector columns yet (0003 adds them).
- **Ingestion service:** Python 3.12, `fastapi` + `uvicorn`, `pydantic` (v2), `httpx`,
  `boto3` (MinIO/S3), `Pillow` (crop rendering), `ruff`, `pytest`. Format parsers:
  `python-docx` (DOCX), `python-pptx` (PPTX), `openpyxl` (XLSX), `markdown-it-py` (MD),
  `beautifulsoup4` (HTML). PDF layout parser is **`docling`** (OQ1 resolved — the chosen
  default), with **`pymupdf`** (fitz) as the lightweight fallback for simple/plain PDFs,
  both behind a stable `parse()` interface.
- **Infra:** existing Docker Compose; ingestion service gains the new deps above.

## Touchpoints (files & modules)

- **DB:** `src/db/schema.ts` (add `documents`, `chunks`, `ingestion_jobs` + enums);
  new `drizzle/NNNN_ingestion.sql` migration.
- **Web — upload/API:** `src/lib/storage/keys.ts` (asset key helpers),
  `src/app/(app)/ingest/actions.ts` (upload Server Action + validation/quota),
  `src/app/api/documents/route.ts` (list/create), `src/app/api/documents/[id]/route.ts`
  (get/delete), `src/app/api/documents/[id]/events/route.ts` (SSE status stream),
  `src/app/api/documents/[id]/reingest/route.ts`.
- **Web — enqueue/client:** `src/lib/ingestion/jobs.ts` (enqueue/claim helpers),
  `src/lib/ingestion/client.ts` (typed fetch to the Python service).
- **Web — UI:** `src/app/(app)/ingest/page.tsx`, `src/app/(app)/ingest/[id]/page.tsx`,
  `src/components/ingest/UploadDropzone.tsx`, `IngestStatusBadge.tsx`, `CorpusList.tsx`,
  `DocumentDetail.tsx`, `AssetPreview.tsx`.
- **Ingestion service:** `ingestion/app/models.py` (Pydantic `ParsedDoc`, `Block`,
  `TableBlock`, `FigureBlock`, `Chunk`), `ingestion/app/storage.py` (boto3 MinIO),
  `ingestion/app/parsers/base.py` (`parse()` protocol + registry),
  `ingestion/app/parsers/{pdf,docx,pptx,xlsx,markdown,html,image}.py`,
  `ingestion/app/render.py` (crop rendering), `ingestion/app/chunking.py`,
  `ingestion/app/captions.py`, `ingestion/app/pipeline.py` (orchestrator),
  `ingestion/app/main.py` (add `POST /ingest`, `GET /jobs/{id}`), `ingestion/tests/…`,
  `ingestion/tests/fixtures/` (sample docs + golden JSON).
- **Config/docs:** `src/lib/env.ts`, `.env.example`, `CHANGELOG.md`.

## Task breakdown

### T1 — Ingestion data model & migration

- **Goal:** `documents`, `chunks`, `ingestion_jobs` tables with a status lifecycle exist
  and migrate cleanly.
- **Files:** `src/db/schema.ts` (edit), `drizzle/NNNN_ingestion.sql` (new via
  `pnpm db:generate`).
- **Libs/tech:** Drizzle ORM `pgEnum` / `pgTable`, `pnpm db:migrate`.
- **Depends:** —
- **Details:** Add `documentStatus` enum `queued | parsing | chunking | embedding |
  ready | failed` (`embedding` reserved for 0003, never entered here) and `modality`
  enum `text | table | figure`. `documents`: `id`, `ownerId` (FK users), `filename`,
  `mimeType`, `byteSize`, `storageKey` (original in MinIO), `status`, `error` (nullable),
  `pageCount` (nullable), `createdAt`, `updatedAt`. `chunks`: `id`, `documentId` (FK,
  `onDelete: cascade`), `modality`, `pageNumber`, `bbox` (jsonb `{x,y,w,h}` nullable),
  `readingOrder` (int), `text` (nullable — caption/description for table/figure),
  `assetKey` (nullable — crop in MinIO), `createdAt`. `ingestion_jobs`: `id`,
  `documentId` (FK, cascade), `state` (`queued|running|done|failed`), `attempts`,
  `lastError`, `claimedAt`, `createdAt`, `updatedAt`. Indexes on `documents.ownerId`,
  `chunks.documentId`, `ingestion_jobs.state`.
- **Tests:** Vitest integration — migrate a test DB, insert a document + chunks, delete
  the document, assert chunks are gone (cascade); assert status enum rejects bad values.
- **Done when:** `pnpm db:migrate` applies; cascade + enum tests green.

### T2 — Upload: validation, quota, MinIO store, enqueue

- **Goal:** an authenticated user uploads a supported file; it lands in MinIO and creates
  `documents` + `ingestion_jobs` rows.
- **Files:** `src/lib/storage/keys.ts` (new), `src/app/(app)/ingest/actions.ts` (new),
  `src/lib/ingestion/jobs.ts` (new), `src/app/api/documents/route.ts` (new).
- **Libs/tech:** Next.js Server Actions, boilerplate S3 client (`@aws-sdk/client-s3`),
  Zod, Drizzle, `server-only`.
- **Depends:** T1
- **Details:** Accept `PDF, DOCX, PPTX, XLSX, MD, HTML, TXT, PNG, JPG`. Validate by
  extension + magic-byte sniff (not just client MIME); enforce a max size and per-user
  quota (reuse the boilerplate file-upload quota helper). Store original at
  `documents/{ownerId}/{documentId}/original.{ext}`; key helpers in `keys.ts` also define
  `assets/{documentId}/{chunkId}.png`. In one transaction insert `documents` (status
  `queued`) + `ingestion_jobs` (state `queued`). `GET /api/documents` lists the caller's
  documents (owner-scoped). Reject unsupported/oversized files with a typed error.
- **Tests:** Vitest — valid upload creates rows + object; oversized/wrong-type/quota-
  exceeded rejected; non-owner cannot list another user's docs.
- **Done when:** upload persists original + rows; validation/quota/ownership tests green.

### T3 — PDF layout parsing: Docling default + PyMuPDF fallback (OQ1 resolved)

- **Goal:** wire the chosen PDF layout parser — **Docling** — behind a swappable
  interface, with **PyMuPDF** as the lightweight fallback for simple/plain PDFs.
- **Files:** `ingestion/app/parsers/base.py` (new — `parse()` protocol + registry),
  `ingestion/app/parsers/pdf.py` (new — Docling adapter + PyMuPDF fallback),
  `docs/decisions/0002-pdf-parser.md` (new — short ADR recording the decision),
  `ingestion/pyproject.toml` (add `docling` + `pymupdf`).
- **Libs/tech:** `docling` (chosen default — layout-aware text/table/figure + bbox,
  fully self-hosted), `pymupdf`/`fitz` (lightweight fallback — light, fast,
  self-host-friendly).
- **Depends:** —
- **Details:** Define the stable seam first: `class DocumentParser(Protocol): def
  parse(self, path: str, mime: str) -> ParsedDoc` (do not rename this interface).
  **OQ1 resolved:** Docling is the default layout parser; PyMuPDF is the fallback for
  simple/plain PDFs, both behind `parse()`. Validate Docling's table/figure/bbox
  extraction on a 3–5 PDF fixture set (text order, table cell fidelity, figure bbox
  accuracy) and wire PyMuPDF as the fallback path selected for simple/plain PDFs. Record
  the decision + validation notes in the ADR. Implement `pdf.py` as a Docling adapter
  (with the PyMuPDF fallback) emitting the shared `ParsedDoc` (text `Block`s, `TableBlock`
  with structured cells + region bbox, `FigureBlock` with region bbox), each carrying
  `{page, bbox, order}`. Keep both vendor imports confined to `pdf.py` so swapping stays a
  one-file change.
- **Tests:** `pytest` golden-file test — parse a fixture PDF (tables + a figure) via
  Docling, assert block count, modalities, page numbers, and bbox tolerance against
  committed golden JSON; assert the PyMuPDF fallback parses a plain PDF fixture.
- **Done when:** ADR committed recording the Docling decision; `pdf.py` passes golden
  tests behind `DocumentParser` for both the Docling default and the PyMuPDF fallback.

### T4 — Parsed-doc model & non-PDF format parsers

- **Goal:** every supported non-PDF format maps into the same `ParsedDoc` shape.
- **Files:** `ingestion/app/models.py` (new — Pydantic types),
  `ingestion/app/parsers/{docx,pptx,xlsx,markdown,html,image}.py` (new), register all in
  `base.py`.
- **Libs/tech:** `python-docx`, `python-pptx`, `openpyxl`, `markdown-it-py`,
  `beautifulsoup4`, `Pillow`; Pydantic v2.
- **Depends:** T3
- **Details:** `models.py` defines `Block(kind, page, bbox|None, order, text|None)`,
  `TableBlock(cells: list[list[str]], page, bbox, order)`, `FigureBlock(page, bbox,
  order, image_ref)`, `ParsedDoc(page_count, blocks)`. Adapters: DOCX → paragraphs +
  tables (synthetic page/order, no bbox); PPTX → one page per slide, shapes → text/figure
  blocks; XLSX → each sheet a `TableBlock`; MD/HTML → headings/paragraphs as text blocks,
  `<table>` → `TableBlock`, `<img>`/embedded images → `FigureBlock`; PNG/JPG → single
  `FigureBlock` (page 1, full-image bbox). Where a format has no geometric bbox, use a
  documented sentinel and rely on reading `order`.
- **Tests:** `pytest` per format — a small fixture per type parses to the expected block
  kinds/counts; XLSX sheet → table cells; image → one figure.
- **Done when:** all 6 non-PDF parsers registered and passing their fixture tests.

### T5 — Render table/figure crops to MinIO

- **Goal:** each table/figure region becomes a PNG stored in MinIO, keyed for its chunk.
- **Files:** `ingestion/app/render.py` (new), `ingestion/app/storage.py` (new — boto3
  MinIO client).
- **Libs/tech:** `Pillow`, `boto3` (or `pymupdf` page rasterisation where the parser
  exposes it), MinIO via S3 API.
- **Depends:** T3, T4
- **Details:** `render.py` takes a page image (rasterised for PDF; the source image for
  image uploads; a rendered representation for office tables) + a `bbox` and crops it to
  PNG. `storage.py` wraps boto3 with `get_object(key)` (fetch original) and
  `put_object(key, bytes)` using the `assets/{documentId}/{chunkId}.png` scheme from T2.
  Return the stored key so chunking can attach it. Handle degenerate/off-page bboxes by
  clamping and logging, not crashing.
- **Tests:** `pytest` with a MinIO test double (or `moto`/local MinIO) — a bbox crop is
  produced, uploaded, and re-fetchable; an out-of-bounds bbox is clamped, not fatal.
- **Done when:** crops for tables/figures upload and round-trip from MinIO in tests.

### T6 — Structure-respecting chunking

- **Goal:** a `ParsedDoc` becomes ordered `Chunk`s — semantic text units; one unit per
  table and per figure.
- **Files:** `ingestion/app/chunking.py` (new); extend `models.py` with `Chunk`.
- **Libs/tech:** pure Python (token/char-window heuristic; no model calls here).
- **Depends:** T4
- **Details:** Group consecutive text blocks into semantic chunks with a target size and
  overlap, never merging across a table/figure boundary or a heading. Each `TableBlock`
  and `FigureBlock` becomes its own `Chunk` (`modality` table/figure) carrying its `bbox`,
  `page`, `order`, `asset` reference (set in T5/T7), and a `text` caption slot (filled in
  T7). `Chunk` fields mirror the DB `chunks` row (`modality`, `pageNumber`, `bbox`,
  `readingOrder`, `text`, `assetKey`). Reading order is preserved across modalities.
- **Tests:** `pytest` — a mixed `ParsedDoc` yields text chunks respecting size/overlap
  and boundaries, plus exactly one chunk per table and per figure, in correct order.
- **Done when:** chunker output matches a golden chunk layout for a mixed fixture.

### T7 — Captions & pipeline orchestrator (`POST /ingest`)

- **Goal:** one entrypoint runs fetch → parse → render → chunk → captions and reports
  status; embedding is left as an explicit seam for 0003.
- **Files:** `ingestion/app/captions.py` (new), `ingestion/app/pipeline.py` (new),
  `ingestion/app/main.py` (edit — add `POST /ingest`, `GET /jobs/{id}`).
- **Libs/tech:** FastAPI, the T3–T6 modules, provider layer stub (from 0001) for the
  optional vision-LLM caption path.
- **Depends:** T3, T4, T5, T6
- **Details:** `captions.py` exposes `caption(chunk, ctx) -> str` with a default
  **heuristic** (table → header row / nearby caption text; figure → nearest caption or
  alt text) and a flag-gated vision-LLM path that calls the provider stub (returns a
  placeholder until 0003/0004 wire real providers). `pipeline.py`: given a document +
  original key, fetch from MinIO, `parse()`, render crops, chunk, caption, and return an
  ordered `list[Chunk]` plus `page_count`; it advances a reported phase
  `parsing → chunking` and **stops at chunk creation** — it emits a `ready_for_embed`
  result and does **not** embed or write vectors. Malformed input raises a typed error
  that marks only that job failed (NFR1). `POST /ingest {documentId, storageKey}` runs
  the pipeline; `GET /jobs/{id}` returns phase/progress/error.
- **Tests:** `pytest` — end-to-end on the PDF fixture returns text/table/figure chunks
  with captions and crop keys, `page_count` set; a corrupt file returns a failed result
  with a reason and does not raise past the handler.
- **Done when:** `POST /ingest` produces chunks + captions for a real fixture and fails
  gracefully on a bad one; no embedding performed.

### T8 — Web ↔ ingestion wiring, persistence, idempotency & lifecycle

- **Goal:** the web app drives ingestion, persists chunks, and supports idempotent re-run
  and cascade delete.
- **Files:** `src/lib/ingestion/client.ts` (new), `src/lib/ingestion/jobs.ts` (edit),
  `src/app/api/documents/[id]/route.ts` (new — GET/DELETE),
  `src/app/api/documents/[id]/reingest/route.ts` (new).
- **Libs/tech:** Next.js Route Handlers, typed `fetch` to `http://ingestion:8000`,
  Drizzle transactions, S3 client for asset cleanup, `server-only`.
- **Depends:** T2, T7
- **Details:** A claim step (called by a Route Handler or worker tick) picks a `queued`
  job, sets `documents.status = parsing`, POSTs to `/ingest`, then on success writes
  `chunks` (mapping each returned `Chunk` → row, crop keys already in MinIO), sets
  `status = ready`; on error sets `failed` + `error`, increments `attempts` with backoff.
  **Idempotency:** re-ingest deletes existing `chunks` + their MinIO assets for the
  document inside a transaction before writing new ones, so no duplicates. `DELETE
  /api/documents/[id]` removes the row (DB cascade drops chunks) and deletes all MinIO
  assets under `documents/{id}/` and `assets/{id}/`. `reingest` re-enqueues a job and
  resets status to `queued`.
- **Tests:** Vitest integration (ingestion service mocked) — happy path persists chunks +
  sets `ready`; re-ingest yields no duplicate chunks; delete removes chunks + asset-delete
  calls fire; a failed `/ingest` sets `failed` + reason.
- **Done when:** persist, idempotent re-run, and cascade delete all pass; status
  transitions match the spec lifecycle.

### T9 — Live status: SSE endpoint + polling fallback

- **Goal:** the UI can watch a document's status change in real time.
- **Files:** `src/app/api/documents/[id]/events/route.ts` (new — SSE),
  `src/lib/ingestion/status.ts` (new — shared status DTO).
- **Libs/tech:** Next.js Route Handler streaming `text/event-stream`; client
  `EventSource`; polling fallback via `GET /api/documents/[id]`.
- **Depends:** T8
- **Details:** SSE handler emits `{status, phase, error?, updatedAt}` on change (poll the
  row on an interval or push on transition), closing on `ready|failed`. Define a single
  `IngestStatus` DTO reused by SSE, polling, and the UI. Owner-scoped; unauthorized
  callers get 401.
- **Tests:** Vitest — the SSE handler streams an initial event and a terminal event on
  status change; polling endpoint returns the same DTO shape.
- **Done when:** a status change is observable via SSE and via polling with one DTO.

### T10 — Ingestion UI: upload, live status, corpus, preview, delete (FR7)

- **Goal:** a first-class ingestion interface satisfying the spec's UX acceptance
  criteria.
- **Files:** `src/app/(app)/ingest/page.tsx`, `src/app/(app)/ingest/[id]/page.tsx`,
  `src/components/ingest/UploadDropzone.tsx`, `IngestStatusBadge.tsx`, `CorpusList.tsx`,
  `DocumentDetail.tsx`, `AssetPreview.tsx` (all new).
- **Libs/tech:** React 19, shadcn/ui (`dropzone`/`card`/`badge`/`table`/`dialog`),
  the T2 Server Action, `EventSource` from T9.
- **Depends:** T2, T9
- **Details:** `UploadDropzone` — drag-and-drop multi-file, client-side type/size hints,
  posts via the Server Action, shows per-file queued state. `CorpusList` — owner's
  documents with `IngestStatusBadge` (queued/parsing/chunking/ready/failed+reason),
  subscribing to SSE for live updates. `DocumentDetail` (`/ingest/[id]`) — metadata,
  chunk summary, and `AssetPreview` rendering table/figure crops from MinIO (signed URL
  or proxied route), plus a delete action (confirm dialog → `DELETE`) and a re-ingest
  button. Failed documents show the reason.
- **Tests:** Playwright E2E — upload a fixture, watch status reach `ready`, see extracted
  table/figure previews, delete it and confirm it disappears; a malformed fixture shows
  `failed` with a reason (regression guard for NFR1 at the UI).
- **Done when:** the full upload → live status → preview → delete journey passes E2E.

### T11 — Env, config, CI & docs

- **Goal:** every new knob is validated, documented, and gated in CI.
- **Files:** `src/lib/env.ts` (edit), `.env.example` (edit), `ingestion/pyproject.toml`
  (deps finalised), `.github/workflows/*` (ensure Python parser deps install),
  `CHANGELOG.md` (edit).
- **Libs/tech:** Zod (web env), ruff/pytest (CI).
- **Depends:** T7, T8
- **Details:** Add web env vars: `INGESTION_SERVICE_URL`, `UPLOAD_MAX_BYTES`,
  `UPLOAD_QUOTA_BYTES`, `MINIO_ASSET_BUCKET` (default to the boilerplate bucket) — to the
  Zod schema *and* `.env.example`, fail-fast on missing required vars. Add ingestion env:
  `MINIO_ENDPOINT/KEY/SECRET/BUCKET`, `CAPTION_MODE` (`heuristic|vision`, default
  `heuristic`). Ensure CI installs the new Python deps and runs the golden/parser tests;
  document the supported formats + the OQ1 decision link in `docs/`.
- **Tests:** Vitest env-schema test (valid passes; missing required throws); CI itself
  green on both runtimes with the new tests.
- **Done when:** app + service boot from `.env.example`; env test green; CI green.

## Data model / migrations

- One migration `drizzle/NNNN_ingestion.sql` adds:
  - **Enums:** `document_status` (`queued|parsing|chunking|embedding|ready|failed`),
    `chunk_modality` (`text|table|figure`). `embedding` is reserved for 0003 and is never
    entered by this slice.
  - **`documents`** — owner FK, `filename`, `mimeType`, `byteSize`, `storageKey`,
    `status`, `error?`, `pageCount?`, timestamps.
  - **`chunks`** — `documentId` FK `onDelete cascade`, `modality`, `pageNumber`, `bbox`
    (jsonb, nullable), `readingOrder`, `text?`, `assetKey?`, `createdAt`.
  - **`ingestion_jobs`** — `documentId` FK cascade, `state`, `attempts`, `lastError?`,
    `claimedAt?`, timestamps.
  - **Indexes:** `documents.ownerId`, `chunks.documentId`, `ingestion_jobs.state`.
- **No vector columns here.** 0003 adds `embeddings` (pgvector) keyed to `chunks`,
  recording `model` + `dimensions`; the reserved `embedding` status and the
  `ready_for_embed` pipeline result are the seam it plugs into.

## Testing strategy

- **Unit (web):** upload validation/quota/ownership; env schema; status DTO shape.
- **Unit (Python):** per-format parsers; chunker boundaries/overlap; caption heuristics;
  render/crop clamping.
- **Golden-file (Python):** fixture PDF (tables + figure) → committed golden `ParsedDoc`
  and chunk JSON with bbox tolerance; the Docling default (and PyMuPDF fallback) is
  validated against these so a swap is measurable.
- **Integration:** Drizzle migration + cascade delete; web ↔ ingestion with the Python
  service mocked (persist, idempotent re-run, delete, failure path); MinIO round-trip via
  `moto`/local MinIO.
- **E2E (Playwright):** upload → live status → table/figure preview → delete; malformed
  file → `failed` + reason.
- **Determinism:** no real model calls in CI — caption `vision` path uses the provider
  stub; embedding is out of scope for this slice.

## Definition of done

- [ ] Every task complete; all listed tests green.
- [ ] Local gate passes (`pnpm lint && pnpm typecheck && pnpm test && pnpm build`;
      `ruff check && pytest`).
- [ ] Uploading a PDF with tables and figures yields text/table/figure chunks with
      correct page/bbox metadata and stored MinIO crops.
- [ ] The UI shows live status and, on completion, previews extracted tables/figures.
- [ ] A malformed file marks only its own document `failed` with a reason.
- [ ] Re-running ingestion produces no duplicate chunks; deleting a document removes its
      chunks and MinIO assets.
- [ ] OQ1 resolved: Docling is the default PDF parser with PyMuPDF as the fallback,
      recorded in a committed ADR, behind the swappable `DocumentParser.parse()` interface.
- [ ] Pipeline stops cleanly at chunk creation with a documented `ready_for_embed` seam
      for 0003; no embedding/vectors introduced.
- [ ] Docs/`.env.example` updated for new config; `CHANGELOG.md` updated.
- [ ] PR merged into `develop`; spec 0002 set to `Shipped` at the v0.2.0 release.

## Risks / notes

- **OQ1 (resolved in T3 — Docling default, PyMuPDF fallback).** The PDF parser is the
  flakiest, heaviest dependency. Keep both vendor imports confined to `parsers/pdf.py`
  behind `DocumentParser` so the choice is a one-file swap; the golden fixtures make
  re-evaluation measurable.
- **Caption generation (spec open question, deferred).** Default is a heuristic; the
  vision-LLM path is flag-gated (`CAPTION_MODE=vision`) and stubbed until 0003/0004 wire
  real providers — this avoids adding provider SDKs or model calls in this slice.
- **Embedding seam.** This slice must *not* embed or add vector columns. The `embedding`
  status value and `ready_for_embed` pipeline result are the only forward references; do
  not implement them here.
- **Bbox fidelity across formats.** Office/MD/HTML formats lack real geometry; use the
  documented sentinel bbox and rely on `readingOrder`. Only PDF/image guarantee true
  bboxes — reflect this in the UI preview (crop-from-region vs. whole-file).
- **Sandboxing (NFR1).** All parsing runs in the Python service with resource limits; a
  malformed document fails its own job only and never the web app or the service.
- **MinIO cleanup.** Delete and re-ingest must remove orphaned crops; asset cleanup runs
  in the same transaction/flow as chunk removal to avoid leaks.
