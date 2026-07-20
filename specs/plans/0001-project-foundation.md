---
plan_for: 0001
title: Project foundation — implementation plan
status: Proposed
branch: feature/0001-project-foundation
created: 2026-07-21
updated: 2026-07-21
---

# 0001 — Implementation plan

> Implements [`specs/0001-project-foundation.md`](../0001-project-foundation.md). This
> plan turns that spec into ordered, agent-executable coding tasks. Execute tasks in
> order, respecting `depends`.

## Objective

Stand up a working, tested monorepo derived from `nextjs-fullstack-boilerplate`: the
web app rebranded to Agentic RAG, a Python ingestion service scaffold, pgvector enabled
on Postgres, the provider-abstraction interfaces (no vendors yet), the compose topology
extended, and CI green across TypeScript and Python — with **no product features**.

## Prerequisites

- Upstream template `nextjs-fullstack-boilerplate` accessible.
- Docker + pnpm + Python 3.12 available locally.
- No API keys required for this slice (no real provider calls yet).

## Tech stack for this slice

- **Web:** Next.js 16, TypeScript 5.9 (strict), Drizzle ORM, `pnpm`. Existing
  Vitest + Playwright + ESLint/Prettier from the boilerplate.
- **DB:** PostgreSQL 17 + `pgvector` extension (`drizzle-orm` `vector` column type).
- **Ingestion service:** Python 3.12, `fastapi`, `uvicorn`, `pydantic`, `ruff`,
  `pytest`, `httpx` (test client). Packaged with `pyproject.toml`.
- **Infra:** Docker Compose (extend boilerplate files), GitHub Actions.

## Touchpoints (files & modules)

- Web app: `package.json`, `README.md`, PWA manifest/branding, `src/lib/env.ts`.
- Providers: `src/lib/providers/` (new) — `types.ts`, `stub.ts`, `index.ts`.
- DB: `drizzle/` new migration; `src/db/schema.ts` (smoke table).
- Ingestion: `ingestion/` (new) — `pyproject.toml`, `app/main.py`, `app/health.py`,
  `Dockerfile`, `tests/`, `ruff.toml`.
- Infra: `docker-compose.yml`, `.env.example`, `.github/workflows/` (Python job).

## Task breakdown

### T1 — Fork boilerplate & rebrand

- **Goal:** Agentic RAG web app boots with all boilerplate features intact.
- **Files:** `package.json` (name/description), `README.md`, PWA manifest + `public/`
  branding, app metadata/title.
- **Libs/tech:** none new.
- **Depends:** —
- **Details:** Import the boilerplate as the app baseline; rename package, update app
  title/manifest, keep auth/RBAC/PWA/storage untouched.
- **Tests:** existing boilerplate Vitest + Playwright suites still pass unchanged.
- **Done when:** `pnpm dev` serves the app; auth/login works; existing tests green.

### T2 — Enable pgvector via migration

- **Goal:** Postgres has the `vector` extension, provable end to end.
- **Files:** new `drizzle/NNNN_*.sql` migration; `src/db/schema.ts` (temporary
  `vector_smoke` table with a `vector(3)` column); `drizzle.config.ts` if needed.
- **Libs/tech:** `drizzle-orm` `vector` type, `CREATE EXTENSION IF NOT EXISTS vector;`.
- **Depends:** T1
- **Details:** Emit `CREATE EXTENSION` in a committed migration so it runs via
  `pnpm db:migrate`. Add a minimal vector column to prove the type + a k-NN query work.
- **Tests:** Vitest integration test inserts rows and runs an `ORDER BY embedding <->`
  nearest-neighbour query against a test DB.
- **Done when:** migration applies cleanly; the k-NN test passes.

### T3 — Provider abstraction skeleton

- **Goal:** stable interfaces every later spec depends on; no vendor SDKs yet.
- **Files:** `src/lib/providers/types.ts`, `src/lib/providers/stub.ts`,
  `src/lib/providers/index.ts`.
- **Libs/tech:** TypeScript only.
- **Depends:** T1
- **Details:** Define `LlmProvider` (chat/stream, tool-use, vision, `capabilities`
  flags) and `EmbeddingProvider` (`embedText`, `embedMultimodal`, reports
  `model` + `dimensions`). Provide a `StubProvider` that throws `NotImplemented`, and a
  config-driven `getLlmProvider()` / `getEmbeddingProvider()` selector reading env.
- **Tests:** unit tests asserting the selector returns the stub and capability flags
  are shaped correctly.
- **Done when:** interfaces compile under strict TS; selector + stub unit-tested.

### T4 — Python ingestion service scaffold

- **Goal:** a runnable FastAPI worker with health, lint, and tests.
- **Files:** `ingestion/pyproject.toml`, `ingestion/app/main.py`,
  `ingestion/app/health.py`, `ingestion/Dockerfile`, `ingestion/ruff.toml`,
  `ingestion/tests/test_health.py`.
- **Libs/tech:** `fastapi`, `uvicorn`, `pydantic`, `ruff`, `pytest`, `httpx`.
- **Depends:** —
- **Details:** FastAPI app exposing `GET /health` → `{status: "ok"}`. Multi-stage
  Dockerfile on `python:3.12-slim`. `ruff` config; `pytest` with an `httpx`/TestClient
  health test. No parsing logic yet (that is spec 0002).
- **Tests:** `pytest` health test asserts 200 + payload.
- **Done when:** `uvicorn` serves `/health`; `ruff check` clean; `pytest` green;
  image builds.

### T5 — Extend Docker Compose topology

- **Goal:** one `docker compose up` runs web + ingestion + db(pgvector) + minio.
- **Files:** `docker-compose.yml` (+ deploy/prod variants as needed).
- **Libs/tech:** compose; a pgvector-capable Postgres image (e.g. `pgvector/pgvector:pg17`).
- **Depends:** T2, T4
- **Details:** Swap the Postgres image for a pgvector-enabled one; add an `ingestion`
  service on the compose network reachable from `web`; healthchecks and depends_on.
- **Tests:** a smoke check (script or documented) that `web` can reach
  `http://ingestion:8000/health` on the compose network.
- **Done when:** the full stack starts; ingestion health reachable from web.

### T6 — CI: add a Python job

- **Goal:** both runtimes gate merges into `develop`.
- **Files:** `.github/workflows/ci.yml` (add a Python job) or a new
  `.github/workflows/ingestion.yml`.
- **Libs/tech:** GitHub Actions, `setup-python`, `ruff`, `pytest`.
- **Depends:** T4
- **Details:** Keep the boilerplate's TS jobs (lint/typecheck/unit/E2E/Docker); add a
  parallel job running `ruff check` + `pytest` for `ingestion/`.
- **Tests:** CI itself; both jobs must be green.
- **Done when:** a PR into `develop` shows green TS and Python checks.

### T7 — Env, config, and docs

- **Goal:** every new knob is documented and validated.
- **Files:** `.env.example`, `src/lib/env.ts` (Zod schema), `docs/` note on the
  two-runtime setup, `CHANGELOG.md`.
- **Libs/tech:** Zod.
- **Depends:** T3, T5
- **Details:** Add provider-selection env keys (LLM/embedding provider + model,
  ingestion service URL) to the Zod schema *and* `.env.example`, with safe defaults
  (stub provider). Fail-fast on missing required vars.
- **Tests:** env-schema unit test (valid config passes; missing required var throws).
- **Done when:** app + service start from `.env.example`; schema test green.

## Data model / migrations

- One migration enabling `pgvector` and a temporary `vector_smoke` table proving the
  `vector` type and k-NN operator. Real schema (`documents`, `chunks`, `embeddings`,
  …) arrives in specs 0002–0005; the smoke table may be dropped once 0003 lands.

## Testing strategy

- **Unit:** provider selector/stub, env schema.
- **Integration:** pgvector migration + k-NN query against a test DB; Python health.
- **E2E:** unchanged boilerplate Playwright auth flow still green (regression guard).
- **CI:** TS jobs + new Python job both required.

## Definition of done

- [ ] All tasks complete; all listed tests green.
- [ ] `pnpm lint && pnpm typecheck && pnpm test && pnpm build` pass; `ruff check` +
      `pytest` pass.
- [ ] `docker compose up` runs web + ingestion + db(pgvector) + minio; ingestion health
      reachable from web.
- [ ] Fresh clone + `make setup` still reaches a live app (no boilerplate regression).
- [ ] `.env.example` documents every new var; `CHANGELOG.md` updated.
- [ ] PR merged into `develop`; spec 0001 set to `Shipped` at the v0.1.0 release.

## Risks / notes

- Swapping to a pgvector Postgres image must not break the boilerplate's existing
  migrations/backup scripts — verify `pnpm db:migrate` and backup/restore still work.
- Keep the provider layer vendor-free here; adapters are spec 0004. Resist adding SDKs.
- No spec open questions are resolved by this slice; it only lays foundations.
