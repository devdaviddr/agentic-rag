---
id: 0006
title: Self-hosting & deployment
status: Planned
release: v1.0.0
created: 2026-07-21
updated: 2026-07-21
---

# 0006 — Self-hosting & deployment

## Summary

Make Agentic RAG deployable to an operator's own hardware/domain in one guided
command: extend the boilerplate's Docker Compose + Cloudflare Tunnel setup with the
Python ingestion service and pgvector, add provider/model configuration (keys, choice),
cost controls, backups covering vectors and assets, and complete self-hosting docs.

## Problem / motivation

The product is only useful if a self-hosting user can stand it up without assembling
services by hand. The boilerplate already ships a clone-to-live path; this spec makes
that path work for the full Agentic RAG topology and documents exactly what leaves the
operator's box (the provider egress boundary).

## Goals

- One-command setup (`make setup`) brings a fresh clone to a live, question-answering
  instance behind HTTPS with no open ports.
- Admin/env configuration of LLM + embedding providers, models, and API keys.
- Cost visibility and caps; usage dashboard.
- Backups covering Postgres (incl. vectors) and MinIO assets, with a tested restore.
- Documentation: install, configure, provider setup, privacy/egress, operate, back up.

## Non-goals

- Managed/hosted SaaS offering (this is self-host).
- Local model inference (NG1 from the PRD).

## Requirements

### Functional

- **FR1 — Compose topology** — `web`, `ingestion`, `db` (pgvector), `minio`, `tunnel`
  compose cleanly; `make setup` / `make deploy` bring up/update the stack.
- **FR2 — Provider config** — Admin surface + env for selecting LLM and embedding
  providers/models and supplying keys; validated on save.
- **FR3 — Cost controls** — Usage dashboard (tokens, embeddings, estimated $) with
  configurable caps/alerts.
- **FR4 — Backups** — Extend the boilerplate's nightly backup to include pgvector data
  and MinIO assets; documented, tested restore runbook.
- **FR5 — Docs** — `docs/self-hosting.md` (and provider/privacy docs) covering prereqs,
  keys, first-run, corpus isolation model, backup/restore, upgrade.

### Non-functional

- **NFR1 — Time-to-live** — Fresh clone → live instance in < 30 min following docs,
  only API keys required.
- **NFR2 — Egress clarity** — Docs state exactly what is sent to providers (query text,
  retrieved chunk text, image crops during generation) and nothing else leaves the box.
- **NFR3 — Reproducible** — Pinned images/deps; a documented upgrade path.
- **NFR4 — Resilient** — Ingestion service restarts cleanly; in-flight jobs resume or
  fail safely.

## Design / approach

- Build on the boilerplate's `make setup`, Cloudflare Tunnel compose files, and macOS
  autostart scripts; add the `ingestion` service and pgvector to each relevant compose
  file and to the backup scripts.
- Provider config lives in `.env` (documented in `.env.example`) with an optional admin
  UI for runtime selection; keys never leave the server.
- Extend `scripts/backup-verify.sh` semantics to cover vector data and MinIO; add a
  restore drill to the docs.
- **Corpus isolation (OQ5):** finalize per-user vs. per-role model and document it.

## Acceptance criteria

- A fresh clone with only API keys set reaches a live instance answering a question
  about an ingested document in < 30 min.
- Switching providers/models via config takes effect without code changes.
- The usage dashboard reflects real token/embedding spend; a cap triggers as configured.
- A backup/restore cycle reproduces documents, chunks, vectors, and assets.
- Self-hosting docs are complete and followed end-to-end successfully.

## Dependencies

- Requires [0001](./0001-project-foundation.md)–[0005](./0005-chat-and-citations.md).
- Marks the v1.0 milestone.

## Open questions

- **OQ5** — Final per-user vs. per-role corpus isolation model (shared with 0004).
