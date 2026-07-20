---
plan_for: NNNN            # the spec this plan implements
title: <spec title> — implementation plan
status: Proposed          # Proposed → Accepted → Shipped
branch: feature/NNNN-slug # the Gitflow feature branch (off develop)
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

# NNNN — Implementation plan

> Implements [`specs/NNNN-slug.md`](../NNNN-slug.md). This plan turns that spec into
> ordered, agent-executable coding tasks. Execute tasks in order, respecting `depends`.

## Objective

One or two sentences: what a contributor/agent will have built when every task here is
done, and how that satisfies the spec's acceptance criteria.

## Prerequisites

- Specs/plans that must be `Shipped` first (e.g. requires 0001).
- Any environment/keys/services needed before starting.

## Tech stack for this slice

The specific libraries, packages, and versions this slice introduces or relies on —
grounded in the project stack (Next.js 16 / TS / Drizzle / Postgres+pgvector / MinIO;
Python 3.12 / FastAPI / ruff / pytest; provider SDKs). Name exact packages so tasks are
unambiguous.

## Touchpoints (files & modules)

List the files/directories this slice creates or changes, so reviewers and agents know
the blast radius up front. Group by area (web app / ingestion service / db / infra).

## Task breakdown

Each task is small enough to implement and review on its own. Keep them ordered.

### T1 — <short task name>

- **Goal:** what this task delivers.
- **Files:** `path/to/file.ts` (new), `path/to/other.py` (edit).
- **Libs/tech:** exact packages/APIs used.
- **Depends:** — (or T-prior).
- **Details:** concrete steps — interfaces to define, functions/endpoints to add,
  schema columns, config keys. Precise enough that an agent can implement without
  guessing structure.
- **Tests:** the unit/integration/E2E tests this task adds and what they assert.
- **Done when:** checkable outcome.

### T2 — <short task name>

- **Goal:** …
- **Files:** …
- **Libs/tech:** …
- **Depends:** T1.
- **Details:** …
- **Tests:** …
- **Done when:** …

<!-- add T3, T4, … as needed -->

## Data model / migrations

Schema additions or changes (Drizzle tables, pgvector columns, migrations) introduced by
this slice, if any. Note dimensions/indexes for vector columns.

## Testing strategy

How the slice is verified overall: unit vs. integration vs. E2E split, fixtures/sample
documents, mock providers for deterministic CI, and any eval measured.

## Definition of done

- [ ] Every task complete; all listed tests green.
- [ ] Local gate passes (`pnpm lint && pnpm typecheck && pnpm test && pnpm build`;
      `ruff check && pytest`).
- [ ] Spec acceptance criteria demonstrably met.
- [ ] Docs/`.env.example` updated for any new config.
- [ ] PR merged into `develop`; spec set to `Shipped` at release.

## Risks / notes

Anything that could trip up implementation, plus links to the spec's open questions this
plan resolves or defers.
