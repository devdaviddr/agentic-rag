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
> [PRD](docs/PRD.md), [architecture](docs/architecture.md), and numbered
> [specs](specs/). **There is no application code yet.** Commands and structure below
> describe the *intended* system defined by the specs; treat them as the target, not
> as things that exist today. Build work proceeds spec by spec, starting with
> [`specs/0001`](specs/0001-project-foundation.md).

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

- Edits are to Markdown only: `docs/`, `specs/`, `README.md`, this file.
- Keep the **boilerplate house style**: numbered specs with frontmatter, docs linked
  from the README table, badge header, ≤ 100-char body lines in prose where practical.
- When a decision is made, record it in the relevant **spec** (source of truth for
  implementation) and reflect cross-cutting product changes in the **PRD**.

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

## Spec-driven development

Non-trivial work starts with a spec. See [`specs/README.md`](specs/README.md) for the
full process. In short:

1. Copy [`specs/TEMPLATE.md`](specs/TEMPLATE.md) to the next `NNNN-slug.md`.
2. Open it as **Proposed**; move to **Accepted** when agreed, **Shipped** when merged.
3. Build against the spec; keep it updated as the source of truth. Decisions and open
   questions live in the spec, not in scattered commit messages.

## Git & workflow

- **Trunk-based** (matches the boilerplate — there is **no `develop` branch**). `main`
  is the only long-lived branch. Feature work branches off `main` as
  `feature/<slug>` and PRs back into `main`. A **release is a `vX.Y.Z` tag on `main`**:
  bump the version, set the shipped spec(s) to `Shipped`, update `CHANGELOG.md`, tag.
  Pre-1.0 until spec 0006 lands.
- **Conventional Commits** (e.g. `feat:`, `fix(auth):`, `docs:`, `chore(deps):`).
  Keep commit **body lines ≤ 100 characters**.
- **Do NOT credit AI in git.** Never add `Co-Authored-By: Claude`, "Generated with…",
  or any AI/assistant trailer to commits or PR descriptions. Author is the repo owner.

## See also

- [`docs/PRD.md`](docs/PRD.md) — product requirements, roadmap, open questions
- [`docs/architecture.md`](docs/architecture.md) — components, data flows, trust boundaries
- [`specs/`](specs/) — numbered delivery slices (0001–0006)
- [`CONTRIBUTING.md`](CONTRIBUTING.md) — contributor workflow
