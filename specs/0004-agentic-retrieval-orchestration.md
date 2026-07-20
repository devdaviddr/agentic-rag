---
id: 0004
title: Agentic retrieval orchestration
status: Proposed
release: v0.4.0
created: 2026-07-21
updated: 2026-07-21
---

# 0004 — Agentic retrieval orchestration

## Summary

Implement the provider-agnostic LLM layer and the agent that answers questions: a
bounded reason→retrieve→observe loop that decomposes questions, issues multiple
multimodal retrievals, judges sufficiency, and produces a grounded answer with
structured citations — passing table/figure crops to a vision-capable model.

## Problem / motivation

Single-shot RAG fails on multi-part questions, on questions whose best search terms
differ from the user's phrasing, and on evidence spread across documents or
modalities. An agent that plans retrieval, uses tools, and iterates fixes this. This
spec also fills in the real LLM adapters behind the interface from 0001.

## Goals

- Real `LlmProvider` adapters for **Claude, OpenAI, and Gemini** (chat, tool-use,
  vision, streaming), selected by config, exposing capability flags.
- An agent loop with retrieval **tools** over the 0003 primitives.
- Query decomposition and reformulation (not raw-question embedding).
- Grounded, vision-aware generation that emits **structured citations** and declines
  when the corpus doesn't support an answer.
- Bounded cost/steps with per-request usage + cost tracking.

## Non-goals

- Chat persistence, streaming UI, and citation rendering (0005) — this spec exposes
  the orchestration API and citation data; 0005 renders it.
- Ingestion/embedding internals (0002/0003).

## Requirements

### Functional

- **FR1 — LLM adapters** — Claude/OpenAI/Gemini adapters implement `LlmProvider`
  (tool-use, vision, streaming); capability flags drive orchestration differences.
- **FR2 — Agent tools** — `search_corpus(query, filters, modality)`, `get_chunk(id)`,
  `get_page_image(doc, page)`, `list_documents()`, built on 0003.
- **FR3 — Planning** — The agent decomposes a question into sub-queries and reformulates
  search terms; may issue multiple retrievals per question.
- **FR4 — Loop** — Bounded reason→act→observe loop with a step/token budget; stops when
  grounding is sufficient or budget is hit.
- **FR5 — Vision generation** — Retrieved table/figure crops are passed to the model as
  images so it can read them directly.
- **FR6 — Citations** — The answer carries structured citations `{chunk_id, document,
  page, region, modality}` for each substantive claim.
- **FR7 — Refusal** — If retrieval doesn't support an answer, the agent says so instead
  of inventing one.
- **FR8 — Usage** — Per-request token/embedding usage and estimated cost recorded.

### Non-functional

- **NFR1 — Provider-agnostic** — Orchestration depends only on `LlmProvider`; adding a
  provider needs no orchestration change.
- **NFR2 — Bounded & safe** — Hard caps on steps/tokens; no unbounded loops; retrieval
  results cached within a request.
- **NFR3 — Deterministic tests** — Agent logic tested with a mock provider (recorded
  tool calls) for stable CI.
- **NFR4 — Groundedness** — On the eval set, ≥ 95% of answer claims carry a resolvable
  citation.

## Design / approach

- **Provider layer:** thin adapters mapping the common interface to each vendor SDK.
  Where a vendor offers native citation/grounding features, the adapter may use them;
  otherwise citations are produced via the tool-use contract (the model references
  retrieved chunk ids, validated against what was actually retrieved).
- **Agent:** a tool-use loop. System prompt instructs decomposition, retrieval-first
  behaviour, citation discipline, and refusal. A validator rejects citations that don't
  correspond to retrieved chunks, preventing fabricated references.
- **Budgets:** configurable max steps and token ceiling; usage tracked per request and
  attributed to a conversation for cost display (0005/0006).
- **Corpus scoping (OQ5):** tools accept document filters so a conversation can be scoped
  to a subset; per-user/per-role isolation enforced at the query layer.

## Acceptance criteria

- A multi-part question triggers multiple retrievals and a single grounded answer.
- A question answerable only from a table/figure is answered from the crop, cited to it.
- Every citation resolves to a chunk that was actually retrieved (validator passes).
- An out-of-corpus question is declined with a clear message.
- Switching provider via config changes the answering model with no code change.

## Dependencies

- Requires [0001](./0001-project-foundation.md)–[0003](./0003-multimodal-embeddings-and-vector-store.md).
- Feeds [0005](./0005-chat-and-citations.md).

## Open questions

- **OQ5** — Per-user vs. per-role corpus isolation model (also touches 0006).
- How much of the agent trace to persist for replay/debugging vs. cost.
