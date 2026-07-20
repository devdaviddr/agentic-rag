---
id: 0005
title: Chat interface & citations
status: Proposed
release: v0.5.0
created: 2026-07-21
updated: 2026-07-21
---

# 0005 — Chat interface & citations

## Summary

Deliver the user-facing question-answering experience: persisted conversations,
streaming answers with the agent's optional trace, **inline clickable citations**,
and a **source panel** that resolves each citation to its document page with the cited
text/table/figure highlighted. Users can scope a conversation to a subset of the corpus.

## Problem / motivation

The agent (0004) produces grounded answers and structured citations, but they are only
trustworthy if the user can *see* the reasoning stream and *verify* each claim against
the source in one click. This spec turns the orchestration API into the product.

## Goals

- Persisted chat threads with history and follow-up context.
- Streaming answers (token-by-token) with an optional collapsible agent trace
  (sub-questions, searches, retrieved sources).
- Inline citation markers that resolve to a source panel highlighting the exact region.
- Conversation-level corpus scoping (all documents or a selected subset).
- Rendering that handles all three citation modalities (text, table, figure).

## Non-goals

- The agent loop and provider layer (0004).
- Ingestion UI (0002).

## Requirements

### Functional

- **FR1 — Conversations** — Create/list/continue/delete threads; messages persisted
  with their answer, citations, and (optionally) agent trace.
- **FR2 — Streaming** — Answers stream via SSE; UI renders progressively.
- **FR3 — Trace** — A collapsible panel shows the agent's steps when available; hidden
  by default.
- **FR4 — Inline citations** — Citation markers rendered inline in the answer; clicking
  one opens/scrolls the source panel to that reference.
- **FR5 — Source panel** — Renders the cited source: for text, the passage on its page;
  for table/figure, the crop and its page context, with the region highlighted.
- **FR6 — Follow-ups** — Follow-up questions retain conversation context and may reuse
  prior retrievals.
- **FR7 — Scoping** — A conversation can be scoped to a chosen document subset; scope is
  passed to the agent's retrieval tools.

### Non-functional

- **NFR1 — Latency** — First token < 3 s p50 on a warm corpus (end-to-end with 0004).
- **NFR2 — Accessible & responsive** — Works on the boilerplate's PWA shell, mobile→
  desktop, light/dark.
- **NFR3 — Tested** — E2E for the ask→stream→cite→verify journey, including a
  table/figure citation.

## Design / approach

- **Data model:** `conversations` and `messages`; a message stores the answer text, a
  structured `citations[]` (chunk references from 0004), and an optional serialized
  trace. Citation references resolve via chunk metadata (document, page, bbox, modality)
  to a rendered page/crop from MinIO.
- **Rendering:** inline citation markers are anchors into `citations[]`. The source panel
  lazy-loads the page render or crop and draws the highlight from the bbox.
- **Citation UX (OQ4):** for image/table sources, decide between highlighting the bbox on
  a full-page render vs. showing the standalone crop — resolve with a quick UX check;
  default to page render + highlight for spatial context, crop as fallback.
- **Streaming:** built on the boilerplate's route handlers + SSE; trace events interleave
  with answer tokens on separate channels.

## Acceptance criteria

- Asking a question streams an answer with inline citations; clicking a citation opens
  the source panel highlighted on the correct page.
- A table/figure-sourced answer shows the crop/region correctly in the panel.
- Follow-up questions retain context; a scoped conversation only retrieves from its
  selected documents.
- Refusals (out-of-corpus) render clearly with no fake citations.

## Dependencies

- Requires [0004](./0004-agentic-retrieval-orchestration.md) (and transitively 0001–0003).
- Completes the core end-to-end product; feeds polish/deploy in [0006](./0006-self-hosting-and-deployment.md).

## Open questions

- **OQ4** — Citation UX for image/table sources (bbox-on-page vs. standalone crop).
