<div align="center">

# Agentic RAG

A self-hostable, full-stack **agentic RAG** — ingest your own documents and ask
questions in plain language. An agent plans the retrieval, searches across **text,
tables, and images**, and answers with **inline citations** you can click. Runs on
**cloud model APIs** (no GPU), behind a **provider-agnostic** layer.

![Next.js](https://img.shields.io/badge/Next.js-16-000000?logo=nextdotjs&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-5.9-3178c6?logo=typescript&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.12-3776ab?logo=python&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-17-4169e1?logo=postgresql&logoColor=white)
![pgvector](https://img.shields.io/badge/pgvector-enabled-4169e1?logo=postgresql&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-ready-2496ed?logo=docker&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green.svg)

**Status: 📐 Planning** — this repo currently contains the PRD, architecture, and
specs. No application code yet.

</div>

---

## What it is

Naïve RAG flattens your documents to text (losing tables and figures), retrieves once,
and gives you answers you can't verify. **Agentic RAG** fixes all four:

- 🧠 **Agentic retrieval** — an agent decomposes your question, issues multiple targeted
  searches, judges whether it has enough, and iterates before answering.
- 🖼️ **Multimodal** — text, **tables**, and **figures** are retrieved as first-class
  units via multimodal embeddings, and read by a vision-capable model.
- 🔗 **Verifiable citations** — every claim links to a specific document + page + region;
  click a citation to see the highlighted source.
- 🏠 **Self-hostable** — one guided command (`make setup`) takes a clone to a live app
  behind HTTPS on your own hardware. Your documents and vectors stay on your box; the
  only egress is to the model API you choose.
- 🔌 **Provider-agnostic** — Claude, OpenAI, and Gemini are first-class from day one,
  selected by config. No GPU required.

Built on the [`nextjs-fullstack-boilerplate`](https://github.com/devdaviddr/nextjs-fullstack-boilerplate)
(Next.js 16, Auth.js v5, Postgres + Drizzle, MinIO, Docker, CI) for the web app and
platform, with **Python services** for document ingestion and parsing.

## How it works

```
                 ┌─────────────── Browser (PWA) ───────────────┐
                 │  Ingestion UI · Chat · Clickable citations   │
                 └───────────────────────┬──────────────────────┘
                                         │ HTTP / SSE
                 ┌───────────────────────▼──────────────────────┐
   Next.js app   │  Auth + RBAC · Agent loop (reason→retrieve→   │
   (boilerplate) │  answer) · Provider layer (LLM + embeddings)  │
                 └───┬───────────────┬───────────────┬──────────┘
                     │ enqueue       │ SQL + vectors │ model APIs (only egress)
                     ▼               ▼               ▼
              ┌──────────┐   ┌──────────────┐   ┌──────────────────────┐
   Python ──▶ │ ingest   │   │ Postgres 17  │   │ LLM: Claude/OpenAI/  │
   service    │ Docling  │   │ + pgvector   │   │ Gemini · NVIDIA NIM  │
   (Docling)  │ parse ·  │   │ + FTS        │   │ (embeds + rerank,    │
              │ chunk ·  │   └──────────────┘   │ free tier)           │
              │ embed    │          ▲           └──────────────────────┘
              └────┬─────┘          │
                   ▼                │
              ┌──────────┐          │
              │ MinIO    │──────────┘  originals + table/figure crops
              └──────────┘
```

See **[docs/architecture.md](docs/architecture.md)** for the full picture.

## Documentation

| Doc | What's inside |
| --- | --- |
| 📄 **[PRD](docs/PRD.md)** | Product vision, goals/non-goals, users, functional & non-functional requirements, roadmap, risks |
| 🏛️ **[Architecture](docs/architecture.md)** | Components, ingestion & query data flows, provider abstraction, trust boundaries |
| 🗺️ **[Roadmap](docs/roadmap.md)** | What each release delivers, capability clusters, key decisions |
| 🗂️ **[Specs](specs/)** | One folder per release (semver); each holds a `spec.md` + `plan.md` (see below) |

### Specs & plans

Work is organised one folder per release under [`specs/`](specs/) — each contains the
**spec** (what & why) and its **implementation plan** (agent-executable coding tasks).

| Release | Title | Spec | Plan |
| --- | --- | --- | --- |
| v0.1.0 | Project foundation | [spec](specs/v0.1.0/spec.md) | [plan](specs/v0.1.0/plan.md) |
| v0.2.0 | Document ingestion pipeline | [spec](specs/v0.2.0/spec.md) | [plan](specs/v0.2.0/plan.md) |
| v0.3.0 | Multimodal embeddings & vector store | [spec](specs/v0.3.0/spec.md) | [plan](specs/v0.3.0/plan.md) |
| v0.4.0 | Agentic retrieval orchestration | [spec](specs/v0.4.0/spec.md) | [plan](specs/v0.4.0/plan.md) |
| v0.5.0 | Chat interface & citations | [spec](specs/v0.5.0/spec.md) | [plan](specs/v0.5.0/plan.md) |
| v1.0.0 | Self-hosting & deployment | [spec](specs/v1.0.0/spec.md) | [plan](specs/v1.0.0/plan.md) |

## Tech stack

**Web / API / agent:** Next.js 16 (App Router, RSC, Server Actions), TypeScript ·
**Auth / RBAC / PWA / CI:** Auth.js v5, Tailwind/shadcn, Playwright, GitHub Actions
(from the boilerplate) · **Data + vectors:** PostgreSQL 17 + pgvector, Drizzle ·
**Storage:** MinIO (S3-compatible) · **Ingestion:** Python (FastAPI worker) +
**Docling** parser (PyMuPDF fallback) · **Models:** provider-agnostic LLM
(Claude / OpenAI / Gemini) · embeddings + rerank via **NVIDIA NIM** free tier
(NV-CLIP, `nv-embedqa`, `nv-rerankqa`) — all swappable · **Retrieval:** pgvector +
Postgres FTS hybrid · **Deploy:** Docker Compose + Cloudflare Tunnel.

## Status & roadmap

This is a planning repository. The build is sliced into the six specs above, each
independently shippable, culminating in a `v1.0` one-command self-host. Progress and
decisions live in the specs; open questions are tracked in the
[PRD](docs/PRD.md#14-open-questions).

## License

[MIT](LICENSE) © 2026 Daniel Ruffolo
