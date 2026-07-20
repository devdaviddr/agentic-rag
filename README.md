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
   Python ──▶ │ ingest   │   │ Postgres 17  │   │ Claude / OpenAI /     │
   service    │ parse ·  │   │ + pgvector   │   │ Gemini + multimodal   │
              │ chunk ·  │   └──────────────┘   │ embeddings vendor     │
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
| 🗂️ **[Specs](specs/)** | Numbered, independently-shippable delivery slices (see below) |

### Specs

| Spec | Title |
| --- | --- |
| [0001](specs/0001-project-foundation.md) | Project foundation |
| [0002](specs/0002-document-ingestion-pipeline.md) | Document ingestion pipeline |
| [0003](specs/0003-multimodal-embeddings-and-vector-store.md) | Multimodal embeddings & vector store |
| [0004](specs/0004-agentic-retrieval-orchestration.md) | Agentic retrieval orchestration |
| [0005](specs/0005-chat-and-citations.md) | Chat interface & citations |
| [0006](specs/0006-self-hosting-and-deployment.md) | Self-hosting & deployment |

## Tech stack

**Web / API / agent:** Next.js 16 (App Router, RSC, Server Actions), TypeScript ·
**Auth / RBAC / PWA / CI:** Auth.js v5, Tailwind/shadcn, Playwright, GitHub Actions
(from the boilerplate) · **Data + vectors:** PostgreSQL 17 + pgvector, Drizzle ·
**Storage:** MinIO (S3-compatible) · **Ingestion:** Python (FastAPI worker) ·
**Models:** provider-agnostic LLM (Claude / OpenAI / Gemini) + text & multimodal
embeddings · **Deploy:** Docker Compose + Cloudflare Tunnel.

## Status & roadmap

This is a planning repository. The build is sliced into the six specs above, each
independently shippable, culminating in a `v1.0` one-command self-host. Progress and
decisions live in the specs; open questions are tracked in the
[PRD](docs/PRD.md#14-open-questions).

## License

[MIT](LICENSE) © 2026 Daniel Ruffolo
