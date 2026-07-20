---
plan_for: 0006
title: Self-hosting & deployment — implementation plan
status: Proposed
branch: feature/0006-self-hosting-and-deployment
created: 2026-07-21
updated: 2026-07-21
---

# 0006 — Implementation plan

> Implements [`specs/0006-self-hosting-and-deployment.md`](../0006-self-hosting-and-deployment.md).
> This plan turns that spec into ordered, agent-executable coding tasks. Execute tasks
> in order, respecting `depends`.

## Objective

Make Agentic RAG deployable to an operator's own host in one guided command: a Docker
Compose topology (`web`, `ingestion`, `db` pgvector, `minio`, `tunnel`) wired to
`make setup` / `make deploy`, admin + env provider/model/key configuration validated on
save (keys never leave the server), a usage/cost dashboard with caps and alerts,
backups covering pgvector data and MinIO assets with a tested restore runbook, a
finalized corpus-isolation model (OQ5), and complete self-hosting/provider/privacy docs
— so a fresh clone reaches a live, question-answering instance in < 30 min. This is the
v1.0 milestone.

## Prerequisites

- Specs 0001–0005 **Shipped**: foundation + compose scaffold (0001), ingestion (0002),
  embeddings/vector store (0003), provider layer + per-request usage tracking (0004),
  chat + citations (0005).
- A host with Docker + Docker Compose, a Cloudflare account + a domain (for the tunnel).
- At least one LLM provider API key and one embedding provider API key for a live run.
- The boilerplate's `make setup`, Cloudflare Tunnel compose files, macOS autostart
  scripts, and `scripts/backup-verify.sh` present and working from the fork.

## Tech stack for this slice

- **Web:** Next.js 16, TypeScript 5.9 (strict), React 19, Server Actions, Drizzle ORM,
  Zod (env + admin-form validation), Tailwind v4 + shadcn/ui, `pnpm`. Turbopack for
  `dev` and `build`.
- **DB:** PostgreSQL 17 + `pgvector` (image `pgvector/pgvector:pg17`), Drizzle
  migrations. Usage/config tables from this slice + 0004.
- **Ingestion service:** Python 3.12 / FastAPI worker (folded into prod compose;
  no new Python code beyond a prod-ready Dockerfile/entrypoint).
- **Infra:** Docker Compose (base + `deploy/prod` + tunnel variants), `Makefile`
  (`setup`/`deploy`/`backup`/`restore` targets), `cloudflared` tunnel, macOS
  `launchd`/autostart scripts, `scripts/backup-verify.sh` (Postgres `pg_dump` +
  `mc mirror` for MinIO), `.env.example`.
- **Providers (config only):** existing `src/lib/providers/*` adapters from 0004 — this
  slice configures/validates them, it does not add SDKs.

## Touchpoints (files & modules)

- **Infra / deploy:** `docker-compose.yml`, `deploy/prod/docker-compose.yml`,
  `deploy/prod/docker-compose.tunnel.yml`, `deploy/prod/cloudflared/config.yml`,
  `deploy/macos/*` (autostart), `Makefile`, `.env.example`, `ingestion/Dockerfile`.
- **Backups:** `scripts/backup-verify.sh` (extend), `scripts/restore.sh` (new),
  `deploy/prod/backup.env.example`.
- **Provider config UI:** `src/app/(app)/admin/providers/page.tsx`,
  `src/app/(app)/admin/providers/actions.ts`,
  `src/app/(app)/admin/providers/provider-form.tsx`,
  `src/lib/providers/config.ts` (persist/read effective config), `src/lib/providers/validate.ts`.
- **Usage/cost dashboard:** `src/lib/usage/pricing.ts`, `src/lib/usage/aggregate.ts`,
  `src/lib/usage/caps.ts`, `src/app/(app)/admin/usage/page.tsx`,
  `src/app/(app)/admin/usage/actions.ts`.
- **Corpus isolation (OQ5):** `src/lib/corpus/scope.ts`, related query guards, Drizzle
  schema/migration for the chosen model, `docs/architecture.md` (isolation note).
- **DB:** `drizzle/NNNN_provider_config.sql`, `drizzle/NNNN_usage_caps.sql`,
  `drizzle/NNNN_corpus_isolation.sql`, `src/db/schema.ts`.
- **Docs:** `docs/self-hosting.md`, `docs/providers.md`, `docs/privacy.md`,
  `README.md` (docs table), `CHANGELOG.md`.

## Task breakdown

### T1 — Production compose topology + tunnel

- **Goal:** a single prod compose brings up `web` + `ingestion` + `db`(pgvector) +
  `minio` + `tunnel` with healthchecks, restart policies, and no open host ports.
- **Files:** `deploy/prod/docker-compose.yml` (new), `deploy/prod/docker-compose.tunnel.yml`
  (new), `deploy/prod/cloudflared/config.yml` (new), `ingestion/Dockerfile` (prod
  target), `docker-compose.yml` (align service names/networks with prod).
- **Libs/tech:** Docker Compose, `pgvector/pgvector:pg17`, `cloudflared`, `minio/minio`.
- **Depends:** — (builds on 0001's base compose)
- **Details:** Fold the Python `ingestion` service into prod compose on the shared
  network, reachable from `web` at `http://ingestion:8000`. Pin all image tags. Give
  `db`, `minio`, `ingestion`, `web` healthchecks + `depends_on: condition:
  service_healthy`. `restart: unless-stopped` on all. Ingestion set to restart cleanly
  and resume/fail in-flight jobs safely (NFR4). Tunnel routes the public hostname to
  `web` only; `db`/`minio` publish no host ports. Named volumes for `db` data and
  `minio` data.
- **Tests:** a smoke script (documented) that after `up`, `web` reaches
  `ingestion:8000/health`, `db` accepts a `SELECT`, and `minio` responds on its health
  endpoint; kill+restart `ingestion` and confirm it comes back healthy.
- **Done when:** `docker compose -f deploy/prod/docker-compose.yml up` yields all
  services healthy; public hostname serves the app over HTTPS via the tunnel; no
  `db`/`minio` ports exposed to the host.

### T2 — Makefile setup/deploy path

- **Goal:** `make setup` takes a fresh clone to a live instance; `make deploy` updates
  a running one — the operator supplies only API keys.
- **Files:** `Makefile` (extend), `.env.example` (add all prod knobs), `deploy/prod/*`
  referenced by targets.
- **Libs/tech:** GNU Make, Docker Compose, the boilerplate's existing `setup` flow.
- **Depends:** T1
- **Details:** `make setup` — copy `.env.example`→`.env` if absent, prompt/validate
  required keys, `docker compose ... build`, run DB migrations (`pnpm db:migrate` inside
  `web`), create the MinIO bucket, bring the stack up, and print the live URL + a
  next-steps note pointing at the admin provider page. `make deploy` — pull/rebuild
  changed images, run pending migrations, `up -d` with zero-downtime-ish restart.
  Add `make backup` and `make restore` delegating to the scripts (T6/T7). Fail fast
  with a clear message if a required env var is missing.
- **Tests:** documented dry-run on a clean checkout timing the flow; assert
  migrations run and the bucket exists idempotently (re-running `make setup` is safe).
- **Done when:** on a clean clone with only keys set, `make setup` completes and the app
  answers a question about an ingested doc in < 30 min (NFR1).

### T3 — Provider/model config: env schema + effective config

- **Goal:** a single source of truth for the active LLM + embedding provider, model,
  and keys — env-driven with an admin override, keys read only server-side.
- **Files:** `src/lib/env.ts` (extend Zod schema), `src/lib/providers/config.ts` (new),
  `.env.example`, `drizzle/NNNN_provider_config.sql` (new), `src/db/schema.ts`.
- **Libs/tech:** Zod, Drizzle, `server-only`.
- **Depends:** — (uses 0004's `LlmProvider`/`EmbeddingProvider` selectors)
- **Details:** Add env keys for LLM provider+model, embedding provider+model, and per-
  provider API keys with safe defaults. Add a `provider_config` table (singleton row:
  chosen providers/models, key **references** — keys themselves stay in env/secret store,
  never in a column that can be read client-side). `getEffectiveProviderConfig()` merges
  admin-set selection over env defaults; `getLlmProvider()`/`getEmbeddingProvider()`
  from 0004 read through it. All key access is `server-only`; keys are never serialized
  to any client payload or the config table.
- **Tests:** unit — effective-config precedence (admin over env, env over default);
  a guard test asserting no key value is present in the serialized config returned to
  the client.
- **Done when:** switching provider/model via env **or** admin selection changes the
  active provider with no code change; keys never leave the server (NFR/FR2).

### T4 — Provider config admin UI + validated save

- **Goal:** an admin surface to pick providers/models and enter keys, validated against
  the provider on save; keys write to the server secret store only.
- **Files:** `src/app/(app)/admin/providers/page.tsx` (new),
  `src/app/(app)/admin/providers/provider-form.tsx` (new),
  `src/app/(app)/admin/providers/actions.ts` (new),
  `src/lib/providers/validate.ts` (new).
- **Libs/tech:** Server Actions, Zod, RBAC guard (admin-only) from the boilerplate,
  existing provider adapters.
- **Depends:** T3
- **Details:** Admin-gated page lists available LLM + embedding providers/models from
  the adapter registry. Form fields for selection + API keys. On save, a Server Action
  runs `validateProvider()` — a minimal live call (e.g. a 1-token completion / a tiny
  embed) through the adapter to confirm the key + model work; on success persist the
  selection to `provider_config` and the key to the server secret store; on failure
  return a field-level error and persist nothing. Keys are write-only in the UI (never
  echoed back). Non-admins get 403.
- **Tests:** unit — action rejects invalid config and does not persist; integration with
  a mock provider — valid save persists selection, invalid key surfaces an error and
  leaves prior config intact; RBAC test — non-admin blocked. E2E (Playwright, mock
  provider): admin sets a provider, save succeeds, active provider reflects it.
- **Done when:** an admin can switch providers/models and supply keys through the UI,
  validated on save, with keys never returned to the client (FR2).

### T5 — Usage/cost dashboard with caps + alerts

- **Goal:** surface token/embedding spend and estimated $, with configurable caps and
  alerts, building on 0004's per-request usage records.
- **Files:** `src/lib/usage/pricing.ts` (new), `src/lib/usage/aggregate.ts` (new),
  `src/lib/usage/caps.ts` (new), `src/app/(app)/admin/usage/page.tsx` (new),
  `src/app/(app)/admin/usage/actions.ts` (new), `drizzle/NNNN_usage_caps.sql` (new),
  `src/db/schema.ts`.
- **Libs/tech:** Drizzle aggregate queries, Server Actions, Zod, a per-provider price
  table.
- **Depends:** T3
- **Details:** `pricing.ts` holds per-provider/model unit prices (tokens in/out,
  embeddings) — configurable/overridable in `.env`. `aggregate.ts` rolls up the 0004
  `usage` records by day/provider/model into tokens, embeddings, estimated $.
  `usage_caps` table stores a monthly $ cap + alert threshold %. `caps.ts` exposes
  `checkCap()` used at request time (from 0004's call sites) to warn/deny once the cap
  is hit, and computes whether an alert should fire. Dashboard page renders totals +
  a per-provider/model breakdown + current-period spend vs. cap. Alerts surface in-app
  (banner) and via server log; enforcement mode (warn vs. hard-stop) is configurable.
- **Tests:** unit — pricing math for known usage rows; cap trigger at the configured
  threshold; alert fires at the alert %. Integration — seed usage rows, assert the
  aggregate matches; assert a request is blocked once a hard cap is exceeded.
- **Done when:** the dashboard reflects real spend from live requests and a configured
  cap triggers as specified (FR3, acceptance).

### T6 — Backups covering pgvector data + MinIO assets

- **Goal:** the nightly backup captures the full Postgres DB (including vector data) and
  all MinIO assets, atomically and verifiably.
- **Files:** `scripts/backup-verify.sh` (extend), `deploy/prod/backup.env.example`
  (new), `Makefile` (`make backup`).
- **Libs/tech:** `pg_dump`/`pg_dumpall` (custom format), MinIO `mc mirror`, `gzip`,
  checksum (`sha256sum`), cron/launchd for scheduling.
- **Depends:** T1
- **Details:** Extend the existing script to (a) `pg_dump` the app DB in custom format
  (vectors are ordinary column data, captured by a full dump — verify the `vector`
  type round-trips), (b) `mc mirror` the MinIO bucket (originals, page renders, crops)
  to the backup target, (c) write a manifest with timestamps, row counts, object
  counts, and checksums. Keep the boilerplate's verify step and extend it to assert the
  dump restores into a scratch DB and object counts match the manifest. Retention +
  destination configurable via `backup.env`. Schedule via the macOS autostart/launchd
  path.
- **Tests:** a CI/integration run that backs up a seeded stack, restores the dump into a
  throwaway Postgres, and asserts document/chunk/vector counts + a k-NN query still work;
  asserts MinIO object count matches.
- **Done when:** a scheduled backup produces a verified artifact covering DB (incl.
  vectors) + assets, and the verify step passes (FR4).

### T7 — Restore runbook + tested round-trip

- **Goal:** a one-command restore that reproduces documents, chunks, vectors, and assets
  from a backup, with a documented, tested runbook.
- **Files:** `scripts/restore.sh` (new), `Makefile` (`make restore`),
  `docs/self-hosting.md` (restore section — cross-linked in T9).
- **Libs/tech:** `pg_restore`, MinIO `mc mirror` (reverse), checksum verification.
- **Depends:** T6
- **Details:** `restore.sh` takes a backup manifest/dir, verifies checksums, restores
  the Postgres custom-format dump into `db`, and mirrors assets back into MinIO. It
  refuses to run against a non-empty target without an explicit `--force`. After
  restore it runs a self-check (row counts + object counts vs. manifest, a sample k-NN
  query, a sample asset fetch). Document the exact steps as a runbook.
- **Tests:** an integration test performing a full backup→wipe→restore round-trip on a
  seeded stack, asserting documents, chunks, vectors (k-NN result identical), and MinIO
  assets are reproduced byte-for-count.
- **Done when:** a backup/restore round-trip reproduces documents, chunks, vectors, and
  assets, and the runbook has been followed end-to-end successfully (FR4, acceptance).

### T8 — Finalize corpus isolation model (OQ5)

- **Goal:** decide and implement per-user vs. per-role corpus isolation, enforced in
  data-access, and document it.
- **Files:** `src/lib/corpus/scope.ts` (new), query guards in retrieval/ingestion
  call sites, `drizzle/NNNN_corpus_isolation.sql` (new), `src/db/schema.ts`,
  `docs/architecture.md` (isolation note), `docs/privacy.md` (referenced from T9).
- **Libs/tech:** Drizzle, RBAC roles from the boilerplate, `server-only`.
- **Depends:** T3
- **Details:** Resolve OQ5 by adopting **per-user corpora by default with a per-role
  “all-corpus” admin scope** (consistent with PRD NG2 + FR16). Add/confirm an owner
  column on `documents` and enforce a `scopeForUser()` filter on every corpus read
  (retrieval, listing, chunk/asset fetch) and every ingestion write, so a user only
  sees their own documents unless their role grants all-corpus access. Ensure delete
  cascades stay owner-scoped. Record the decision in the spec (OQ5 → resolved) and in
  `docs/architecture.md`.
- **Tests:** integration — user A cannot retrieve, list, fetch, or delete user B's
  documents/chunks/assets; an admin/all-corpus role can; ingestion writes are stamped
  with the correct owner. Regression: existing single-user flows unaffected.
- **Done when:** corpus isolation is enforced at the query layer, tested, and the chosen
  model is documented; OQ5 marked resolved in the spec.

### T9 — Self-hosting, provider, and privacy docs

- **Goal:** complete operator docs that take a fresh clone to a live, backed-up instance,
  with an explicit egress boundary.
- **Files:** `docs/self-hosting.md` (new), `docs/providers.md` (new),
  `docs/privacy.md` (new), `README.md` (docs table), `.env.example` (final), `CHANGELOG.md`.
- **Libs/tech:** Markdown; cross-links to the compose/Makefile/scripts.
- **Depends:** T2, T4, T5, T7, T8
- **Details:** `self-hosting.md` — prereqs, obtaining keys, first-run via `make setup`,
  the < 30-min happy path, upgrade path (`make deploy`, pinned images), backup/restore
  runbook (from T7), and troubleshooting. `providers.md` — supported LLM + embedding
  providers/models, how to configure them (env + admin UI from T4), and the re-embedding
  note on provider switch. `privacy.md` — the **egress boundary**: exactly what is sent
  to providers (query text, retrieved chunk text, image crops during generation) and
  that nothing else leaves the box (NFR2), plus the corpus-isolation model from T8.
  Update the README docs table + `.env.example` for every knob added in this slice.
- **Tests:** docs lint / link check; a manual (documented) end-to-end walkthrough by a
  fresh operator following only these docs.
- **Done when:** the docs are complete, cross-linked, and were followed end-to-end to a
  live instance successfully (FR5, NFR2, acceptance).

## Data model / migrations

- **`provider_config`** — singleton row: selected LLM provider+model, embedding
  provider+model. API keys are **not** stored here (server secret store / env only).
- **`usage_caps`** — monthly $ cap, alert threshold %, enforcement mode (warn |
  hard-stop). Builds on 0004's `usage` records (no schema change to `usage`; read-only
  aggregation here).
- **Corpus isolation** — confirm/add an owner FK on `documents` (and ensure `chunks`/
  assets inherit it via document FK); index the owner column for scoped queries. No new
  vector columns; vector dimensions/indexes are owned by 0003.
- Migrations: `NNNN_provider_config.sql`, `NNNN_usage_caps.sql`,
  `NNNN_corpus_isolation.sql`, applied via `pnpm db:migrate` (run by `make setup`/`deploy`).

## Testing strategy

- **Unit:** effective provider-config precedence + key-leak guard (T3); provider
  validate action (T4); pricing/aggregation/cap math (T5).
- **Integration (mock providers for determinism):** validated provider save (T4);
  usage aggregation + cap enforcement against seeded rows (T5); backup verify (T6);
  full backup→restore round-trip incl. k-NN + assets (T7); corpus-isolation
  cross-user access (T8).
- **E2E (Playwright, mock provider):** admin sets a provider and it takes effect (T4);
  usage dashboard renders real seeded spend (T5).
- **System/manual (documented):** the < 30-min fresh-clone → live-instance run (T2/T9)
  and the restore runbook walkthrough (T7/T9).
- **CI:** existing TS jobs + Python `pytest`; compose smoke check from T1.

## Definition of done

- [ ] Every task complete; all listed tests green.
- [ ] Local gate passes (`pnpm lint && pnpm typecheck && pnpm test && pnpm build`;
      `ruff check && pytest`).
- [ ] `make setup` on a fresh clone with only API keys reaches a live instance that
      answers a question about an ingested document in < 30 min.
- [ ] Switching providers/models via env or admin config takes effect with no code
      change; keys never leave the server.
- [ ] Usage dashboard reflects real spend; a configured cap triggers as specified.
- [ ] A backup/restore round-trip reproduces documents, chunks, vectors, and assets.
- [ ] `docs/self-hosting.md`, `docs/providers.md`, `docs/privacy.md` complete;
      egress boundary + corpus-isolation model documented; `.env.example` + README
      docs table updated; `CHANGELOG.md` updated.
- [ ] OQ5 resolved in the spec (per-user default + per-role all-corpus admin scope).
- [ ] PR merged into `develop`; spec 0006 set to `Shipped` at the v1.0.0 release.

## Risks / notes

- **Egress doc must be exact.** `docs/privacy.md` is a trust artifact — it must state
  precisely what is sent to providers (query text, retrieved chunk text, image crops in
  generation) and nothing else; keep it in sync if 0004/0005 change what is sent.
- **Key handling.** Keys live in env / the server secret store only — never a DB column
  or any client payload; the T3/T4 guard tests enforce this. Redact keys from logs.
- **pgvector in dumps.** Confirm the `vector` type + indexes round-trip through
  `pg_dump`/`pg_restore` (extension must exist before restore); the T7 round-trip test
  is the guardrail.
- **Ingestion resilience (NFR4).** In-flight jobs must resume or fail safely across a
  restart — validate the restart behavior in T1's smoke test.
- **Resolves OQ5** (per-user vs. per-role corpus isolation) in T8; keep consistent with
  0004's usage/ownership model and PRD NG2.
