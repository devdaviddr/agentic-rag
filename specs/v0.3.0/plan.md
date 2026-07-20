---
plan_for: 0003
title: Multimodal embeddings & vector store — implementation plan
status: Proposed
branch: feature/0003-multimodal-embeddings-and-vector-store
created: 2026-07-21
updated: 2026-07-21
---

# 0003 — Implementation plan

> Implements [`specs/v0.3.0/spec.md`](./spec.md).
> This plan turns that spec into ordered, agent-executable coding tasks. Execute tasks in
> order, respecting `depends`.

## Objective

When every task here is done, a `ready` document from 0002 has its text chunks and its
table/figure crops embedded — text via a text-embedding model, crops via a **multimodal**
model (NVIDIA NIM **NV-CLIP** by default) — all through the `EmbeddingProvider` interface.
Vectors live in pgvector tagged with `{model, dimensions, modality}` and source metadata,
so provider switches are detectable and re-embeddable. The TS layer exposes retrieval
primitives (`searchByVector`, metadata filters, optional FTS/BM25 hybrid) that 0004 will
drive, and a re-embedding job restores vectors when the configured model/dims change.

## Prerequisites

- Spec **0001** (`Shipped`): provider skeleton (`EmbeddingProvider` in
  `src/lib/providers/`), pgvector enabled via migration, env schema + selectors, CI.
- Spec **0002** (`Shipped`): `documents`, `chunks`, `ingestion_jobs` tables; parsed
  `chunks` of modality text | table | figure with `{page, bbox}` and crop storage keys;
  the Python ingestion pipeline that produces them.
- **API keys:** an NVIDIA NIM key (default multimodal + text vendor; free tier at
  build.nvidia.com) available to the ingestion service and the web app via env. CI uses a
  mock provider (no key required).
- Sample table/figure crops from 0002 fixtures available for NV-CLIP validation.

## Tech stack for this slice

- **Web (TS):** Next.js 16, TypeScript 5.9 (strict), `drizzle-orm` `vector` column type,
  Vitest. A `fetch`/OpenAI-compatible client (custom base URL + `NVIDIA_NIM_API_KEY`)
  behind `EmbeddingProvider` for query-time embedding. No vendor SDK in callers.
- **DB:** PostgreSQL 17 + `pgvector` (HNSW/IVFFlat indexes; `tsvector` + GIN for FTS).
  Drizzle migrations under `drizzle/`.
- **Ingestion service (Python 3.12):** `httpx` (NVIDIA NIM HTTP calls), `pydantic`,
  `ruff`, `pytest`. Embedding is the final pipeline step, writing vectors back to Postgres
  (via `psycopg`/existing DB layer from 0002).
- **Embeddings default (OQ2):** NVIDIA NIM **NV-CLIP** (`nvidia/nvclip`, 1024-dim) for
  crops; **`nvidia/llama-3.2-nv-embedqa-1b-v2`** (Matryoshka, truncated to 1024-dim) for
  text — both pinned here, both swappable behind the adapter. NIM is OpenAI-compatible
  (`/v1/embeddings`), so the adapter is thin. NV-CLIP and the text retriever are still
  **two logical embedding spaces** (CLIP space vs text-retriever space): the query is
  embedded in each relevant space with the matching model and results are unioned at
  query time (see T7). Cross-space score comparison isn't valid, so union/merge — not a
  single ANN search — is used even though both are 1024-dim.

### Where embedding runs — decision

Embedding runs in **two places**, split by lifecycle:

- **Ingestion-time (Python):** embedding text chunks and table/figure crops is the
  **final step of the ingestion pipeline** (architecture data-flow step 5). The crops and
  parsed chunks already live in the Python worker; embedding there avoids shipping images
  back to the TS layer, keeps the write path in one transaction, and lets ingestion
  retry/scale independently. The worker writes vectors + `{model, dimensions}` directly.
- **Query-time (TS):** the web app must embed the **user's query** at retrieval time
  (0004), so the TS layer also gets an `EmbeddingProvider` implementation for `embedText`
  (and `embedMultimodal` for future image queries). Both sides read the same env-selected
  model so query and corpus vectors share a space.

Both go through the **same `EmbeddingProvider` contract** (0001) — one in TS, one in
Python — so the vendor is swappable on either side without touching callers.

## Touchpoints (files & modules)

- **DB / schema:** `src/db/schema.ts` (add `embeddings` table + `chunks_fts` support);
  new `drizzle/NNNN_embeddings.sql`, `drizzle/NNNN_vector_indexes.sql`,
  `drizzle/NNNN_chunks_fts.sql`. Drop the 0001 `vector_smoke` table.
- **Providers (TS):** `src/lib/providers/embeddings/` (new) — `nvidia-nim.ts`, `mock.ts`,
  `index.ts`; extend the 0001 selector `getEmbeddingProvider()`.
- **Retrieval (TS):** `src/lib/retrieval/` (new) — `types.ts`, `searchByVector.ts`,
  `filters.ts`, `hybrid.ts`, `index.ts`.
- **Ingestion (Python):** `ingestion/app/embed.py` (new — `embedText`/`embedMultimodal`
  provider + pipeline step), `ingestion/app/providers/nvidia_nim.py`,
  `ingestion/app/providers/base.py`, wiring into the 0002 pipeline; `reembed.py` job.
- **Re-embed (TS side trigger):** `src/lib/retrieval/reembed.ts` (detect stale vectors),
  API route/server action to enqueue a re-embed `ingestion_job`.
- **Config/docs:** `.env.example`, `src/lib/env.ts` (embedding model/dims keys, NVIDIA
  NIM key + base URL), `CHANGELOG.md`, `docs/` note on dimension-safety + index choice.

## Task breakdown

### T1 — `embeddings` schema + drop smoke table

- **Goal:** a persistent, dimension-safe place to store vectors with provenance.
- **Files:** `src/db/schema.ts` (edit), `drizzle/NNNN_embeddings.sql` (new).
- **Libs/tech:** `drizzle-orm` `vector` type; `pgvector`.
- **Depends:** —
- **Details:** Add an `embeddings` table keyed by `chunk_id` (FK → `chunks`, cascade
  delete) with `document_id`, `modality` (text|table|figure), `page`, `bbox` (jsonb),
  `model` (text), `dimensions` (int), `created_at`. Store the vector in a column sized to
  the **default** dims (1024). To stay dimension-safe when text vs. multimodal models
  differ, add a **nullable second vector column** (e.g. `embedding_1536`) OR make the
  table generic with a `dim` bucket — pick the two-column approach: one column per active
  dimension, populated by modality/model; queries union matching columns (see T7). Drop
  the 0001 `vector_smoke` table in this migration.
- **Tests:** Vitest integration — migrate a test DB, insert an embedding row, read it back
  with `model`/`dimensions`/`modality` intact; `ON DELETE CASCADE` from `chunks` removes it.
- **Done when:** migration applies cleanly; smoke table gone; round-trip test green.

### T2 — Embedding provider contract (Python) + NVIDIA NIM adapter

- **Goal:** `embedText` and `embedMultimodal` available to the ingestion pipeline,
  vendor-swappable, reporting `model` + `dimensions`.
- **Files:** `ingestion/app/providers/base.py` (new — abstract `EmbeddingProvider`),
  `ingestion/app/providers/nvidia_nim.py` (new), `ingestion/app/providers/__init__.py`
  (selector reading env).
- **Libs/tech:** `httpx`, `pydantic`; NVIDIA NIM OpenAI-compatible `/v1/embeddings` API.
- **Depends:** —
- **Details:** Mirror the TS `EmbeddingProvider` shape (0001): `embed_text(texts) ->
  list[vector]`, `embed_multimodal(items: image+caption) -> list[vector]`, plus `model`
  and `dimensions` properties. NVIDIA NIM adapter calls
  `nvidia/llama-3.2-nv-embedqa-1b-v2` (text, Matryoshka truncated to 1024-dim) and
  `nvidia/nvclip` (NV-CLIP, image+text — text strings and base64 `data:image/...`
  inputs), hitting the OpenAI-compatible `/v1/embeddings` endpoint at
  `NVIDIA_NIM_BASE_URL`, batching inputs and respecting rate limits with backoff.
  Selector picks provider/model from env; a `MockProvider` returns deterministic
  fixed-dim vectors for CI (no network).
- **Tests:** `pytest` — mock provider returns correct-shaped/dim vectors; NVIDIA NIM
  adapter request/response shape asserted against a recorded fixture (respx/httpx mock),
  no live call.
- **Done when:** `ruff check` clean; provider selector returns mock in CI; unit tests green.

### T3 — OQ2: validate NV-CLIP multimodal on sample crops

- **Goal:** confirm the pinned multimodal vendor + dimensions actually retrieve
  table/figure crops well before committing the rest of the slice.
- **Files:** `ingestion/tests/test_multimodal_validation.py` (new, marked `@live`/skipped
  in CI); short findings note appended to `docs/` (dimension-safety section).
- **Libs/tech:** NVIDIA NIM NV-CLIP (`nvidia/nvclip`); 0002 sample crops.
- **Depends:** T2.
- **Details:** Embed a handful of labelled table/figure crops (+captions) and a set of
  matching/mismatching text queries via NV-CLIP's shared image+text space; assert the
  correct crop ranks top-k by cosine similarity. Record observed vector dimensions
  (expect 1024) and confirm they match the `dimensions` the adapter reports. If retrieval
  is poor or dims differ, flag before T4.
- **Tests:** the validation test itself (behind a live marker); documents pass/fail.
- **Done when:** sample crops retrieved in top-k; dims confirmed; OQ2 resolved in the doc.

### T4 — Embedding as the final ingestion step

- **Goal:** a `ready` document gets vectors for all three modalities, written with
  `{model, dimensions, modality}`.
- **Files:** `ingestion/app/embed.py` (new — pipeline step), wiring into the 0002 pipeline
  (status → `embedding` → `ready`).
- **Libs/tech:** T2 provider; DB write layer from 0002 (`psycopg`/Drizzle-generated SQL).
- **Depends:** T1, T2, T3.
- **Details:** After chunking, for each chunk: text chunks → `embed_text`; table/figure
  chunks → `embed_multimodal` (fetch crop from MinIO by storage key + caption). Batch by
  modality; write rows to `embeddings` with the correct dimension column (T1) and the
  provider's reported `model`/`dimensions`. Idempotent: re-running replaces a chunk's
  vectors rather than duplicating (upsert on `chunk_id`+`model`). Advance
  `documents.status` `embedding → ready`; on provider failure, mark `failed` with reason.
- **Tests:** `pytest` integration — run the pipeline (mock provider) on a fixture doc with
  text+table+figure; assert one `embeddings` row per chunk, correct modality/dims; re-run
  produces no duplicates; provider error → `failed` + reason.
- **Done when:** fixture doc reaches `ready` with all-modality vectors; idempotent re-run green.

### T5 — Embedding provider (TS) for query-time

- **Goal:** the web app can embed a query through `EmbeddingProvider`, same model as corpus.
- **Files:** `src/lib/providers/embeddings/nvidia-nim.ts` (new),
  `src/lib/providers/embeddings/mock.ts` (new),
  `src/lib/providers/embeddings/index.ts` (new); extend `getEmbeddingProvider()` (0001).
- **Libs/tech:** TypeScript; NVIDIA NIM OpenAI-compatible `/v1/embeddings` API (`fetch`
  or OpenAI-style client with a custom base URL); `server-only` guard.
- **Depends:** T2 (parity of contract).
- **Details:** Implement `embedText` (and `embedMultimodal` stub/impl for future image
  queries) against NVIDIA NIM, reading model/dims/base URL from env so it matches the
  ingestion side. Mock provider returns deterministic vectors for tests. Guard with
  `server-only`; never imported into client components.
- **Tests:** Vitest — mock returns correct-dim vector; NVIDIA NIM adapter request shape
  asserted (fetch mocked); selector returns mock under test env.
- **Done when:** `getEmbeddingProvider().embedText(q)` returns a vector; strict TS compiles.


### T6 — `searchByVector` + metadata filters

- **Goal:** `searchByVector(embedding, k, filters)` returning ranked chunks with scores
  and source metadata, filterable by document/page/modality.
- **Files:** `src/lib/retrieval/types.ts`, `src/lib/retrieval/searchByVector.ts`,
  `src/lib/retrieval/filters.ts`, `src/lib/retrieval/index.ts` (all new).
- **Libs/tech:** Drizzle + pgvector `<=>` (cosine) operator; SQL.
- **Depends:** T1.
- **Details:** Define `SearchFilters { documentIds?, pageRange?, modality? }` and a
  `SearchResult { chunkId, documentId, modality, page, bbox, score, ... }`. `searchByVector`
  runs a k-NN query (`ORDER BY embedding <=> $q LIMIT k`) with a `WHERE` built from filters,
  joining `chunks` for text/caption + crop key. Query the vector column matching the query
  embedding's dimension. Return cosine similarity as `score`.
- **Tests:** Vitest integration — seed embeddings across docs/pages/modalities; assert
  ordering by similarity, `k` respected, and each filter (documentIds, pageRange, modality)
  narrows results correctly.
- **Done when:** filtered k-NN returns correctly ranked, typed results; tests green.

### T7 — Multi-space multi-model query (union)

- **Goal:** the two embedding spaces (NV-CLIP vs text-retriever) handled explicitly — no
  invalid cross-space score comparison, no silent dimension mismatches.
- **Files:** `src/lib/retrieval/searchByVector.ts` (edit), `src/lib/retrieval/filters.ts`
  (edit); note in `docs/`.
- **Libs/tech:** SQL `UNION`; the vector column(s) from T1.
- **Depends:** T1, T6.
- **Details:** Even though both default models are 1024-dim, NV-CLIP space and the
  text-retriever space are **distinct** — cosine scores across them aren't comparable, so
  a single ANN search over the merged set is invalid. Implement query fan-out: embed the
  query with each active model, run a per-space k-NN (discriminated by `modality`), and
  **union + re-rank** results by normalized score rather than one combined search. Also
  guard against comparing a query vector to a column of a different dimension (throw,
  never coerce), in case a swapped-in alternative model changes dims. With the default
  1024-dim pair the queries hit one `vector(1024)` column but stay two separate searches
  merged at the end.
- **Tests:** Vitest — seed both spaces; a text query hits text-retriever vectors, a
  multimodal query hits NV-CLIP crop vectors; results union/re-rank correctly and a
  deliberate dimension mismatch throws a clear error.
- **Done when:** multi-space retrieval unions correctly; mismatch guarded + tested.

### T8 — OQ3: hybrid vector + Postgres FTS/BM25 (optional, flagged)

- **Goal:** an optional keyword signal fused with vector similarity, and a judgement on
  whether Postgres FTS is sufficient.
- **Files:** `drizzle/NNNN_chunks_fts.sql` (new — `tsvector` column + GIN index on
  chunk text/caption), `src/lib/retrieval/hybrid.ts` (new), `src/db/schema.ts` (edit).
- **Libs/tech:** Postgres `to_tsvector`/`ts_rank_cd` (FTS/BM25-style), GIN index; env flag.
- **Depends:** T6.
- **Details:** Add a generated `tsvector` over each chunk's text/caption + GIN index.
  `hybrid.ts` runs vector k-NN and FTS `ts_rank_cd` in parallel and fuses with Reciprocal
  Rank Fusion (RRF), behind `RETRIEVAL_HYBRID=on`. Evaluate on the T9 eval set whether
  hybrid materially beats vector-only on keyword-heavy queries; record the OQ3 verdict
  (FTS sufficient for v1, or flag a dedicated engine for later) in `docs/`.
- **Tests:** Vitest integration — a keyword-exact query ranks the lexically-matching chunk
  higher under hybrid than under vector-only; flag off = vector-only path unchanged.
- **Done when:** hybrid retrieval works behind the flag; OQ3 verdict documented.

### T9 — pgvector index choice (HNSW vs IVFFlat) + eval

- **Goal:** an appropriate, justified vector index for interactive latency, backed by a
  recall@k eval on a small labelled multimodal set.
- **Files:** `drizzle/NNNN_vector_indexes.sql` (new — chosen index per vector column);
  eval fixtures + `src/lib/retrieval/__tests__/recall.test.ts` (new); note in `docs/`.
- **Libs/tech:** pgvector `hnsw` / `ivfflat` with cosine ops.
- **Depends:** T6, T8.
- **Details:** Build a small labelled multimodal QA set (text + table/figure questions →
  expected chunk ids), reusing 0002 fixtures. Measure recall@k for HNSW vs IVFFlat at the
  expected corpus size; **default to HNSW** (better recall/latency, no train step at small
  scale) unless the eval shows IVFFlat is materially better — record the decision +
  tradeoffs. Emit the chosen index in a committed migration (one per active vector column).
- **Tests:** the recall@k eval asserts a table/figure question retrieves its crop in top-k
  and recall ≥ an agreed threshold; index migration applies.
- **Done when:** index committed; recall@k eval green; index decision + tradeoffs documented.

### T10 — Re-embedding job on model/dims change

- **Goal:** detect vectors whose `model`/`dimensions` differ from active config and
  restore them via a re-embed job.
- **Files:** `src/lib/retrieval/reembed.ts` (new — stale detection + enqueue),
  `ingestion/app/reembed.py` (new — worker job), an API route/server action to trigger it.
- **Libs/tech:** `ingestion_jobs` queue (0002); T2/T4 embedding step.
- **Depends:** T4, T6.
- **Details:** `reembed.ts` queries `embeddings` for rows where `model` ≠ active model or
  `dimensions` ≠ active dims and reports the stale set. Enqueue an `ingestion_job` of type
  `reembed` (whole corpus or per-document); the Python `reembed.py` re-runs the embedding
  step (T4) for affected chunks, writing to the correct dimension column and upserting on
  `chunk_id`+`model`, then removes superseded rows. Idempotent and resumable.
- **Tests:** integration — seed vectors under an "old" model; change active config; stale
  detection flags them; run re-embed (mock provider) → vectors carry the new model/dims,
  no duplicates, retrieval works against the refreshed set.
- **Done when:** changing the model flags existing vectors and the job restores them; green.

### T11 — Env, config, and docs

- **Goal:** every new knob validated and documented; dimension-safety + index decision
  written down.
- **Files:** `.env.example`, `src/lib/env.ts` (Zod), `ingestion/app/*` env read,
  `docs/` (embeddings/retrieval note), `CHANGELOG.md`.
- **Libs/tech:** Zod; shared env between TS + Python.
- **Depends:** T5, T7, T8, T9, T10.
- **Details:** Add `EMBEDDING_PROVIDER`, `EMBEDDING_TEXT_MODEL` (default
  `nvidia/llama-3.2-nv-embedqa-1b-v2`), `EMBEDDING_MULTIMODAL_MODEL` (default
  `nvidia/nvclip`), `EMBEDDING_DIMENSIONS` (default `1024`), `NVIDIA_NIM_API_KEY`,
  `NVIDIA_NIM_BASE_URL` (default `https://integrate.api.nvidia.com/v1`),
  `RETRIEVAL_HYBRID` (on/off), and a re-embed toggle to the Zod schema **and**
  `.env.example`, with safe defaults (mock provider, hybrid off). Fail-fast on missing
  required vars when a real provider is selected. Document the OQ2/OQ3 verdicts, the index
  choice, and the dimension-safety strategy.
- **Tests:** env-schema unit test — valid config passes; missing `NVIDIA_NIM_API_KEY`
  with NVIDIA NIM selected throws; mock default requires no key.
- **Done when:** app + ingestion start from `.env.example`; schema test green; docs updated.

## Data model / migrations

- **`embeddings`** (new): PK/FK `chunk_id` → `chunks` (cascade delete), `document_id`,
  `modality` (text|table|figure), `page`, `bbox` (jsonb), `model` (text),
  `dimensions` (int), `created_at`. Vector stored in a per-dimension column (default
  `embedding vector(1024)`; a nullable second column for a differing model dimension).
  Unique on (`chunk_id`, `model`) for idempotent upsert.
- **`chunks_fts`** (T8): generated `tsvector` over chunk text/caption + **GIN** index.
- **Vector indexes** (T9): **HNSW** (cosine) per active vector column by default; IVFFlat
  only if the eval justifies it — committed migration either way.
- **Migrations:** `NNNN_embeddings.sql`, `NNNN_chunks_fts.sql`, `NNNN_vector_indexes.sql`;
  the 0001 `vector_smoke` table is dropped in `NNNN_embeddings.sql`.

## Testing strategy

- **Unit:** TS + Python provider adapters (mock deterministic, NVIDIA NIM
  request/response mocked); env schema; filter builder.
- **Integration (test DB):** `embeddings` round-trip + cascade; `searchByVector` ordering
  and each filter; multi-space union + mismatch guard; hybrid RRF; re-embed flow;
  ingestion embedding step end-to-end on a fixture doc (mock provider).
- **Eval:** recall@k on a small labelled multimodal set (text + table/figure questions),
  used both to choose HNSW vs IVFFlat (T9) and to judge OQ3 hybrid value (T8).
- **Live (skipped in CI):** OQ2 NV-CLIP multimodal validation on sample crops (T3),
  behind a `@live` marker requiring a real key.
- **CI:** deterministic — mock providers only; both TS and Python jobs must be green.

## Definition of done

- [ ] Every task complete; all listed tests green.
- [ ] Local gate passes (`pnpm lint && pnpm typecheck && pnpm test && pnpm build`;
      `ruff check && pytest`).
- [ ] A `ready` 0002 document has vectors for all three modalities, each tagged with
      `model` + `dimensions`.
- [ ] `searchByVector` returns correctly ranked, filterable results across modalities; a
      table/figure question retrieves the relevant crop in top-k on the eval set.
- [ ] Changing the embedding model flags existing vectors and the re-embed job restores them.
- [ ] OQ2 (multimodal vendor + dims) and OQ3 (FTS/BM25 sufficiency) resolved and documented.
- [ ] `.env.example` documents every new var; `CHANGELOG.md` updated.
- [ ] PR merged into `develop`; spec 0003 set to `Shipped` at the v0.3.0 release.

## Risks / notes

- **Space and dimension mismatch are the core hazards.** NV-CLIP space and the
  text-retriever space are distinct even at a shared 1024-dim, so scores across them
  aren't comparable — the union path (T7) merges per-space searches rather than running
  one ANN over both, and it must throw on any dimension mismatch, never coerce.
- **OQ2 gates the slice.** If NV-CLIP under-retrieves crops or reports unexpected dims
  (T3), pause before T4 and revisit the vendor — it is behind the adapter precisely so
  this is a config change (swap to Voyage/Cohere), not a rewrite.
- **OQ3 is a measure-then-decide.** Ship FTS/BM25 hybrid behind a flag; only justify a
  dedicated search engine if the eval (T8/T9) shows FTS is materially insufficient.
- **Two provider implementations (TS + Python)** must stay in lockstep on model/dims via
  shared env — a query embedded with a different model than the corpus silently degrades
  recall. The env schema (T11) is the single source of truth.
- **Idempotency** across ingestion (T4) and re-embed (T10) relies on the
  (`chunk_id`, `model`) unique upsert — verify no duplicate vectors on re-runs.
- **Index build cost:** HNSW build time grows with corpus size; acceptable at v1 scale,
  but note it for the self-host docs (0006) so large-corpus operators aren't surprised.
