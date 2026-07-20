---
id: 0001
title: Project foundation
status: Proposed
release: v0.1.0
created: 2026-07-21
updated: 2026-07-21
---

# 0001 — Project foundation

## Summary

Stand up the repository on top of the Next.js full-stack boilerplate, add the Python
ingestion service scaffold, enable pgvector on Postgres, and lay down the
provider-abstraction skeleton and CI — so every later spec builds on a working,
tested base with no application features yet.

## Problem / motivation

Agentic RAG spans a TypeScript web app and a Python service, sharing one Postgres
(with vectors) and one object store. Before any feature work, the two runtimes,
the database extension, the provider seam, and CI must exist and be green, or later
specs will each re-litigate foundational decisions.

## Goals

- A running web app derived from `nextjs-fullstack-boilerplate`, rebranded to
  Agentic RAG, with auth/RBAC/PWA/storage intact.
- A Python ingestion service scaffold (FastAPI worker) that builds, runs, and is
  reachable from the web app, with its own tests and lint.
- Postgres with the **pgvector** extension enabled via a committed migration.
- The **provider-abstraction skeleton**: `LlmProvider` and `EmbeddingProvider`
  interfaces with a stub adapter, no real vendor calls yet.
- Docker Compose topology extended with the `ingestion` service; CI runs TS + Python
  checks and stays green.

## Non-goals

- No parsing, embedding, retrieval, or chat logic (later specs).
- No real provider adapters beyond a stub (0004 fills these in).

## Requirements

### Functional

- **FR1** — Repo is a fork/derivative of the boilerplate with app name, README, and
  metadata updated to Agentic RAG; existing boilerplate tests still pass.
- **FR2** — `ingestion/` Python service scaffold with FastAPI, a health endpoint, a
  Dockerfile, and `pytest` + `ruff` configured.
- **FR3** — Committed Drizzle migration enabling `CREATE EXTENSION vector;` and a
  smoke migration proving vector columns work.
- **FR4** — `src/lib/providers/` with `LlmProvider` and `EmbeddingProvider` TypeScript
  interfaces (capability flags, model/dimension reporting) and a no-op stub.
- **FR5** — `docker-compose.yml` gains an `ingestion` service wired to db + minio; the
  web app can reach it over the compose network.

### Non-functional

- **NFR1** — CI runs boilerplate TS checks (typecheck, lint, Vitest, Playwright) plus
  Python (`ruff`, `pytest`) and is green.
- **NFR2** — `make setup` still produces a live app (no regression to the
  boilerplate's clone-to-live path).
- **NFR3** — Secrets/keys via `.env` only; `.env.example` documents every new var.

## Design / approach

- Start from a clone of the boilerplate; keep its structure and conventions
  (`docs/`, `specs/`, Drizzle, MinIO, Docker, Actions). Add a top-level `ingestion/`
  Python package.
- Enable pgvector through a normal Drizzle migration so it participates in the
  existing migrate/backup workflow.
- Define provider interfaces in TS now, adapters later — later specs depend only on
  the interface, not vendors.
- Extend CI with a Python job (matrix or separate workflow) so both runtimes gate
  merges.

## Acceptance criteria

- Fresh clone + `make setup` → live app; `/` loads, auth works.
- `docker compose up` starts web, ingestion, db (pgvector), minio; ingestion health
  endpoint returns OK from within the network.
- A test inserts and queries a pgvector column successfully.
- CI is green on TS and Python jobs.

## Dependencies

- Upstream: `nextjs-fullstack-boilerplate`.
- Blocks: all subsequent specs.
