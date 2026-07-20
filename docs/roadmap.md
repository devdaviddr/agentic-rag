---
title: Roadmap
status: Draft
created: 2026-07-21
updated: 2026-07-21
---

# Roadmap

What the six releases deliver, and what you can do after each. Every release is a
folder under [`specs/`](../specs/) with a `spec.md` (what & why) and a `plan.md`
(agent-executable tasks). Delivery is strictly linear — each release builds on the one
before.

> **Outcome at v1.0.0:** a self-hostable, full-stack **agentic RAG** — upload your
> documents, ask questions in plain language, and an agent researches across **text,
> tables, and figures** and returns a **streamed, cited** answer you can verify in one
> click. Runs on **cloud model APIs** (no GPU), every provider **swappable by config**,
> deployable to your own box behind HTTPS in one command.

## Releases

| Release | Title | Status | Adds | After it, the system can… |
| --- | --- | --- | --- | --- |
| [v0.1.0](../specs/v0.1.0/spec.md) | Project foundation | Proposed | Rebranded Next.js app + Python FastAPI service, pgvector, provider interfaces (stub), compose, CI | boot, authenticate, run both runtimes green — no RAG yet |
| [v0.2.0](../specs/v0.2.0/spec.md) | Document ingestion pipeline | Proposed | Upload → **Docling** layout parse → text/table/figure chunks + crops in MinIO; ingestion UI | turn documents into structure-preserving, previewable units |
| [v0.3.0](../specs/v0.3.0/spec.md) | Multimodal embeddings & vector store | Proposed | **NVIDIA NIM** text + multimodal embeddings, pgvector schema, `searchByVector` + filters + Postgres-FTS hybrid + NIM rerank | retrieve across all three modalities by meaning, filterably |
| [v0.4.0](../specs/v0.4.0/spec.md) | Agentic retrieval orchestration | Proposed | Claude/OpenAI/Gemini adapters, bounded agent loop, grounded generation, validated citations | actually *answer* a question with grounded, cited output |
| [v0.5.0](../specs/v0.5.0/spec.md) | Chat interface & citations | Proposed | Persisted chat, SSE streaming, agent trace, inline citations, source panel, scoping | be *used* by a human — the end-to-end product journey |
| [v1.0.0](../specs/v1.0.0/spec.md) | Self-hosting & deployment | Proposed | `make setup`, provider/key admin, cost dashboard + caps, backups/restore, docs | be *deployed and operated* by a self-hoster in < 30 min |

## Capability clusters at v1.0

- **Ingest anything structured** — PDF/DOCX/PPTX/XLSX/MD/HTML/TXT/PNG/JPG; tables and
  figures preserved as first-class units, with a real-time upload/browse/delete UI.
- **Multimodal retrieval** — questions match text chunks *and* table/figure crops (each
  embedded per space via NVIDIA NIM); filter by document, page, or modality; keyword
  hybrid via Postgres FTS, fused and reranked into one ordering.
- **Agentic answering** — decompose multi-part questions, reformulate, issue multiple
  retrievals, read image crops with a vision model, iterate within a budget.
- **Verifiable trust** — every claim carries a citation validated against actually
  retrieved chunks; inline + resolvable to a highlighted source region; unsupported
  questions declined, not hallucinated.
- **Own it** — one-command self-host behind Cloudflare Tunnel, bring-your-own keys,
  provider choice by config, cost caps, backups of vectors + assets, documented egress.

## Cross-cutting guarantees

Provider-agnostic (LLM + embeddings behind interfaces; vectors record model + dims) ·
privacy by architecture (data stays on the operator's box; one documented egress
boundary) · sandboxed ingestion · tested throughout (parser golden files, retrieval
recall@k, deterministic agent tests via a mock provider, E2E ask→cite→verify).

## Out of scope for v1

No local/on-device inference · no fine-tuning · no web crawling / live sources · no
shared multi-tenant team knowledge bases with granular sharing · no native mobile app
(PWA covers mobile web).

## Key decisions (open questions)

Resolved decisions are recorded in each release's `spec.md`; see the
[PRD](./PRD.md#14-open-questions) for the running list.

| # | Decision | Resolved in | Status |
| --- | --- | --- | --- |
| OQ1 | PDF layout-parsing library | v0.2.0 | ✅ Docling (PyMuPDF fallback) |
| OQ2 | Multimodal embedding vendor + dimensions | v0.3.0 | ✅ NVIDIA NIM (NV-CLIP + nv-embedqa, 1024-dim) |
| OQ3 | Hybrid search: Postgres FTS vs. dedicated engine | v0.3.0 | ✅ Postgres FTS + NIM reranker (in-database) |
| OQ4 | Citation UX for image/table sources | v0.5.0 | ✅ Page render + bbox highlight |
| OQ5 | Per-user vs. per-role corpus isolation | v0.4.0 / v1.0.0 | ✅ Per-user + admin override |

## How this maps to Gitflow

Each release is built on a `feature/<slug>` branch off `develop`, integrated via PR,
then shipped as a `release/vX.Y.Z` merged to `main` and tagged. See
[`CONTRIBUTING.md`](../CONTRIBUTING.md).
