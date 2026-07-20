# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

**Agentic RAG** — a self-hostable, full-stack agentic RAG application. Users ingest
their own documents and ask questions; an agent plans retrieval across **text, tables,
and images** and answers with **inline, clickable citations**. It runs on **cloud model
APIs** (no GPU) behind a **provider-agnostic** layer, and is built on the
[`nextjs-fullstack-boilerplate`](https://github.com/devdaviddr/nextjs-fullstack-boilerplate)
with **Python services** for document ingestion.

> **Status: planning.** This repo currently contains only planning docs — the
> [PRD](docs/PRD.md), [architecture](docs/architecture.md), and the release
> [specs](specs/). **There is no application code yet.** Commands and structure below
> describe the *intended* system defined by the specs; treat them as the target, not
> as things that exist today. Build work proceeds release by release, starting with
> [`specs/v0.1.0`](specs/v0.1.0/spec.md).

Read [`docs/PRD.md`](docs/PRD.md) and [`docs/architecture.md`](docs/architecture.md)
before any substantive work.

## Key product decisions (locked)

These are settled; don't re-litigate them without updating the PRD/specs first:

- **Provider-agnostic LLM** — Claude, OpenAI, and Gemini are first-class behind an
  `LlmProvider` interface, selected by config. No vendor SDK in callers.
- **True multimodal embeddings** — table/figure crops are embedded directly (not just
  captioned) and read by a vision-capable model. Behind an `EmbeddingProvider` interface.
- **pgvector on the existing Postgres** — one store for relational data and vectors;
  no separate vector DB in v1.
- **Python for ingestion** — layout-aware parsing/chunking/embedding runs in an isolated
  Python (FastAPI) service; the web app enqueues jobs and never parses documents itself.
- **Cloud APIs only** — no local/on-device inference in v1 (PRD non-goal NG1).

## Intended stack

TypeScript web app: Next.js 16 (App Router, RSC, Server Actions) · React 19 ·
TypeScript 5.9 (strict) · Auth.js v5 · Drizzle + **Postgres 17 + pgvector** ·
Tailwind v4 + shadcn/ui · Vitest + Playwright · pnpm. **Turbopack for both `dev` and
`build`** (inherited constraint — no webpack-only tooling).

Python ingestion service: Python 3.12 · FastAPI worker · `ruff` + `pytest`.

Storage: MinIO (S3-compatible). Deploy: Docker Compose + Cloudflare Tunnel.

## Working here (now, during planning)

- Edits are to Markdown only: `docs/`, `specs/` (each release's `spec.md` + `plan.md`),
  `README.md`, this file.
- Keep the **house style**: every release folder under `specs/` holds a `spec.md` and a
  `plan.md` with YAML frontmatter (seed a new one from
  [`specs/_template/`](specs/_template/)); docs linked from the README table; badge
  header; ≤ 100-char body lines in prose where practical.
- When a decision is made, record it in the relevant release's **`spec.md`** (source of
  truth) — and its **`plan.md`** if it changes the tasks — and reflect cross-cutting
  product changes in the **PRD**. Keep spec, plan, and PRD in sync.

## Working here (once code exists — target conventions)

Mirror the boilerplate's conventions, plus Python:

- **Provider abstraction is sacred.** Callers depend on `LlmProvider` /
  `EmbeddingProvider`, never a vendor SDK. Every stored vector records its embedding
  `model` + `dimensions` so provider switches are detectable and re-embeddable.
- **Ingestion is isolated.** Untrusted document parsing lives only in the Python
  service with resource limits. The web app talks to it over the compose network.
- **DB changes:** edit the Drizzle schema → generate → commit the migration → migrate.
  pgvector is enabled via a committed migration.
- **Env** is validated and fails fast; add new vars to the schema *and* `.env.example`.
- **Before pushing (TS):** `pnpm lint && pnpm typecheck && pnpm test && pnpm build`.
- **Before pushing (Python):** `ruff check && pytest`.
- **server-only** guards DB/secret code — never import into client components.

## Spec-driven development (every release folder holds a spec + a plan)

Work is organised **one folder per release** under [`specs/`](specs/), named by semver
(e.g. `specs/v0.2.0/`). Each folder holds two files: `spec.md` (what & why) and
`plan.md` (how — agent-executable coding tasks). See [`specs/README.md`](specs/README.md)
for the full process. In short:

1. Create the release folder: `cp -r specs/_template specs/vX.Y.Z`.
2. Fill in `spec.md` — open it as **Proposed**; move to **Accepted** when agreed.
3. Fill in `plan.md` — it decomposes the spec into concrete, agent-executable **coding
   tasks** tied to this tech stack: each task names the files to create/modify, the
   libraries used, its dependencies, and the tests that prove it. **A release is not
   ready to build until its `plan.md` exists.**
4. Build by executing the plan's tasks on a `feature/` branch; keep spec + plan updated
   as the source of truth. When merged and released, set the spec to **Shipped**.

When you (an assistant) implement a release, work through its `plan.md` task by task, in
order, respecting the stated dependencies — don't improvise structure the plan doesn't
define.

## Git & workflow (Gitflow + feature branching)

- **Gitflow.** Two long-lived branches: **`main`** (production; every commit is a tagged
  `vX.Y.Z` release) and **`develop`** (integration). Supporting branches:
  - `feature/<slug>` — branch off **`develop`**, PR back into `develop`. Named after the
    release being built, e.g. `feature/0002-document-ingestion-pipeline` (spec `0002` →
    release `v0.2.0`); see the release's `plan.md` frontmatter for its `branch`. **All
    feature work lives here** — never commit features directly to `develop` or `main`.
  - `release/vX.Y.Z` — off `develop`; version bump, `CHANGELOG.md`, spec→`Shipped`,
    bugfixes only; merge to `main` (tag) **and** back to `develop`.
  - `hotfix/vX.Y.Z` — off `main`; merge to `main` (tag) and `develop`.
  - Full model, diagram, and rules in [`CONTRIBUTING.md`](CONTRIBUTING.md).
- **Conventional Commits** (e.g. `feat(ingestion):`, `fix(retrieval):`, `docs(spec):`,
  `chore(release):`). Keep commit **body lines ≤ 100 characters**.
- **Do NOT credit AI in git.** Never add `Co-Authored-By: Claude`, "Generated with…",
  or any AI/assistant trailer to commits or PR descriptions. Author is the repo owner.

## Documentation map

- [`docs/PRD.md`](docs/PRD.md) — product requirements, roadmap, open questions
- [`docs/architecture.md`](docs/architecture.md) — components, data flows, trust boundaries
- [`specs/README.md`](specs/README.md) — the spec-driven process: release folders,
  lifecycle (`Proposed → Accepted → Shipped`), and how it maps to Gitflow
- [`specs/`](specs/) — one folder per release (`v0.1.0`–`v1.0.0`), each holding a
  `spec.md` (what & why) and a `plan.md` (agent-executable tasks)
- [`specs/_template/`](specs/_template/) — `spec.md` + `plan.md` seed for a new release
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — Gitflow branching + contributor workflow
