---
plan_for: 0004
title: Agentic retrieval orchestration — implementation plan
status: Proposed
branch: feature/0004-agentic-retrieval-orchestration
created: 2026-07-21
updated: 2026-07-21
---

# 0004 — Implementation plan

> Implements [`specs/v0.4.0/spec.md`](./spec.md).
> This plan turns that spec into ordered, agent-executable coding tasks. Execute tasks in
> order, respecting `depends`.

## Objective

Build the TypeScript/Next.js agent that answers questions over the 0003 corpus: real
`LlmProvider` adapters for Claude, OpenAI, and Gemini behind the 0001 interface; a bounded
reason→act→observe orchestration loop with retrieval **tools** over the 0003 primitives;
grounded, vision-aware generation that emits **structured, validated citations**; refusal
when unsupported; and per-request token/cost tracking. This slice exposes an orchestration
**API** and citation **data** — the chat UI and streaming rendering are spec 0005.

## Prerequisites

- Specs [0001](../v0.1.0/spec.md)–[0003](../v0.3.0/spec.md)
  `Shipped`: provider interfaces + selector (0001), ingested `documents`/`chunks` with
  crops in MinIO (0002), pgvector retrieval primitives `searchByVector` + filters + hybrid
  (0003).
- API keys for at least one provider (Anthropic / OpenAI / Google) in `.env`. Tests run
  against a **mock** provider and need no keys.
- 0003 retrieval module exported for server use (e.g. `src/lib/retrieval/index.ts`).

## Tech stack for this slice

- **Web/agent:** Next.js 16 (App Router, Route Handlers), TypeScript 5.9 (strict), `pnpm`.
- **Provider SDKs:** `@anthropic-ai/sdk` (Claude), `openai` (OpenAI), `@google/genai`
  (Gemini). All isolated behind `LlmProvider` — no vendor SDK imported outside adapters.
- **Default models (CLAUDE guidance, provider-agnostic):** Claude `claude-opus-4-8`
  (Opus 4.8) as sensible default; OpenAI/Gemini defaults configurable. Claude adapter uses
  **adaptive thinking** (`thinking: {type: "adaptive"}`) — never `budget_tokens` (400 on
  Opus 4.8). `output_config: {effort: "high"}` for the answer step.
- **Validation:** Zod for tool-input schemas + API request/response bodies.
- **DB:** Drizzle + Postgres 17 (agent-trace / usage persistence, OQ5 scoping columns).
- **Test:** Vitest with a recorded-tool-call **MockLlmProvider** for deterministic CI.

## Touchpoints (files & modules)

- Providers (LLM adapters): `src/lib/providers/llm/anthropic.ts`,
  `src/lib/providers/llm/openai.ts`, `src/lib/providers/llm/gemini.ts`,
  `src/lib/providers/llm/mock.ts`; update `src/lib/providers/index.ts` selector.
- Agent core: `src/lib/agent/orchestrator.ts`, `src/lib/agent/tools.ts`,
  `src/lib/agent/citations.ts`, `src/lib/agent/prompt.ts`, `src/lib/agent/types.ts`,
  `src/lib/agent/usage.ts`, `src/lib/agent/scope.ts` (OQ5).
- Retrieval bridge (0003 consumer): `src/lib/agent/retrieval.ts` (thin wrapper only).
- Route handlers: `src/app/api/agent/query/route.ts` (+ optional
  `src/app/api/agent/tools/route.ts` for introspection).
- DB: `src/db/schema.ts` (conversations/messages/citations/usage/trace, corpus-scope
  columns); new `drizzle/NNNN_*.sql` migration.
- Config/docs: `src/lib/env.ts`, `.env.example`, `CHANGELOG.md`.

## Task breakdown

### T1 — Extend LLM provider interface for orchestration

- **Goal:** a precise `LlmProvider` contract the loop depends on: chat, tool-use, vision,
  streaming, capability flags, usage reporting.
- **Files:** `src/lib/providers/llm/types.ts` (or extend `src/lib/providers/types.ts`),
  `src/lib/agent/types.ts` (new — shared agent domain types).
- **Libs/tech:** TypeScript only.
- **Depends:** —
- **Details:** Define provider-neutral types: `LlmMessage` (system/user/assistant/tool
  roles), `ContentPart` (text | image), `ToolSpec` (name/description/JSON-schema),
  `ToolCall`, `ToolResult`, `LlmResponse` ({ content, toolCalls, stopReason, usage }),
  `Usage` ({ inputTokens, outputTokens, cachedTokens?, estimatedCostUsd? }). `LlmProvider`
  gains `generate(req)` and `stream(req)` where `req` carries messages, tools, `toolChoice`,
  vision images, and generation opts. `capabilities` reports `{ vision, toolUse, streaming,
  nativeCitations }` so orchestration adapts (e.g. vision off ⇒ skip crop attachment).
- **Tests:** type-level (compiles under strict TS); unit test asserting capability shape
  and that a message with an image part round-trips through the neutral types.
- **Done when:** interface compiles; `getLlmProvider()` still returns 0001 stub against it.

### T2 — Mock LLM provider (recorded tool calls)

- **Goal:** deterministic provider for CI that replays scripted tool calls + final answers.
- **Files:** `src/lib/providers/llm/mock.ts`, `src/lib/agent/__fixtures__/*.json`.
- **Libs/tech:** TypeScript only.
- **Depends:** T1
- **Details:** `MockLlmProvider` takes an ordered script of turns; each turn yields either
  `toolCalls` (with fixed inputs) or a final `content` + `citations` payload, plus a
  synthetic `usage`. It records every request it received for assertions. Supports a
  "refusal" turn and a "cite-unretrieved-chunk" turn to exercise the validator (T7). No
  network. Selectable via `LLM_PROVIDER=mock`.
- **Tests:** unit — driving a two-turn script (search → answer) returns expected calls;
  request log captures the tool results fed back.
- **Done when:** mock drives the loop end-to-end in later tasks with stable output.

### T3 — Retrieval tools over 0003 primitives

- **Goal:** the four agent tools, each a typed function + JSON schema, built on 0003.
- **Files:** `src/lib/agent/tools.ts`, `src/lib/agent/retrieval.ts`.
- **Libs/tech:** Zod (schemas), 0003 `searchByVector`/filters/hybrid, Drizzle, MinIO client.
- **Depends:** T1
- **Details:** Implement `search_corpus(query, filters, modality)` — embeds query via
  `EmbeddingProvider`, calls 0003 retrieval, returns ranked `{chunk_id, document, page,
  region, modality, score, snippet}`; `get_chunk(id)` — full text/caption + metadata;
  `get_page_image(doc, page)` — returns a page render reference/bytes for vision;
  `list_documents()` — scoped document set. Each tool: Zod input schema, exported JSON
  schema for `ToolSpec`, and a handler taking a `ScopeContext` (T4). A per-request
  retrieval cache (NFR2) memoizes identical tool calls; a `RetrievalLedger` records every
  chunk id actually returned this request (feeds T7 validator).
- **Tests:** unit against a seeded test DB — `search_corpus` returns ranked filtered
  results; `get_chunk` resolves; `get_page_image` returns a crop for a table chunk; ledger
  accumulates returned ids.
- **Done when:** all four tools callable with typed inputs; ledger populated; tests green.

### T4 — Corpus scope enforcement at the tool/query layer (OQ5)

- **Goal:** per-user/per-role isolation enforced where retrieval happens, not in the UI.
- **Files:** `src/lib/agent/scope.ts`, `src/db/schema.ts` (owner/role columns), tools (T3).
- **Libs/tech:** Auth.js session (from boilerplate), Drizzle filters.
- **Depends:** T3
- **Details:** `resolveScope(session, requestedDocIds?)` produces a `ScopeContext`
  ({ ownerId, roleScope, allowedDocumentIds }). **Decision (OQ5):** corpora are per-user by
  default; an admin/role may be granted a wider `roleScope`. Every tool query is
  constrained by `allowedDocumentIds` — a caller cannot widen scope by passing arbitrary
  `filters.document`. `list_documents()` returns only in-scope docs. Requested doc subset
  (conversation scoping, PRD FR15) intersects with the allowed set. Document the model in
  a `docs/` note and cross-link 0006.
- **Tests:** unit — user A cannot retrieve user B's chunks via `search_corpus` or
  `get_chunk`; role-scoped user sees the wider set; requested subset never exceeds allowed.
- **Done when:** cross-tenant retrieval is impossible at the tool layer; tests prove it.

### T5 — System prompt & planning contract

- **Goal:** the prompt that instructs decomposition, retrieval-first behaviour, citation
  discipline, and refusal.
- **Files:** `src/lib/agent/prompt.ts`.
- **Libs/tech:** TypeScript template; no vendor specifics.
- **Depends:** T1, T3
- **Details:** Compose the system prompt: decompose multi-part questions into sub-queries;
  **reformulate** search terms (do not embed the raw question); prefer tools before
  answering; pass table/figure crops as images when reading them; every substantive claim
  must carry a citation referencing a retrieved `chunk_id`; refuse clearly when retrieval
  is insufficient. Keep the prompt stable/deterministic (prompt-cache friendly; volatile
  bits last). Export the tool descriptions used in `ToolSpec` from one source of truth.
- **Tests:** snapshot test of the rendered prompt; assertion that refusal + citation rules
  are present and tool names match `tools.ts`.
- **Done when:** prompt renders deterministically and is consumed by the orchestrator.

### T6 — Orchestration loop (reason→act→observe, bounded)

- **Goal:** the bounded tool-use loop that plans, retrieves, judges sufficiency, answers.
- **Files:** `src/lib/agent/orchestrator.ts`, `src/lib/agent/usage.ts`.
- **Libs/tech:** `LlmProvider` (T1) only — provider-agnostic (NFR1); Zod for tool dispatch.
- **Depends:** T2, T3, T4, T5
- **Details:** `runAgent({ question, scope, provider, budget })`: seed messages with system
  prompt + question; loop — call `provider.generate` with tools; if `stopReason === "toolUse"`
  dispatch each `ToolCall` through T3 handlers (scope-checked), append `ToolResult`s
  (attaching page/crop images when vision-capable), and continue; stop when the model
  produces a final answer or the **budget** (max steps AND token ceiling from config) is
  hit. Accumulate `Usage` across turns (T tracking) into a per-request total. On budget
  exhaustion without grounding, return a bounded/graceful refusal (NFR2). No unbounded
  loops; retrieval cache active within the request. Return `{ answer, citations, trace,
  usage }` where `trace` is the ordered step log.
- **Details (Claude adapter note):** the answer step uses adaptive thinking +
  `effort: "high"`; the loop treats thinking as opaque and never sets `budget_tokens`.
- **Tests:** unit with `MockLlmProvider` — multi-part script triggers ≥2 retrievals then a
  single grounded answer; budget cap terminates a runaway script; usage totals sum turns.
- **Done when:** the loop drives a full question→answer with the mock, deterministically.

### T7 — Structured citations + validator

- **Goal:** typed citations `{chunk_id, document, page, region, modality}` with a validator
  that rejects any citation not corresponding to an actually-retrieved chunk.
- **Files:** `src/lib/agent/citations.ts`.
- **Libs/tech:** Zod, the `RetrievalLedger` from T3.
- **Depends:** T3, T6
- **Details:** `Citation` Zod schema; `extractCitations(answer)` (from model output /
  structured field); `validateCitations(citations, ledger)` — every `chunk_id` MUST be in
  the request's `RetrievalLedger`; unknown ids are rejected (fabricated-reference guard).
  Reconcile `document/page/region/modality` against the ledger's stored metadata (repair or
  reject mismatches per policy). Expose `resolveCitation(chunk_id)` returning source
  metadata for 0005's source panel. Invalid citations either drop the offending claim or
  force a refusal per config — never surface unvalidated citations.
- **Tests:** unit — valid citation passes; a citation to an un-retrieved id is rejected;
  metadata mismatch is repaired/rejected; NFR4 spot-check that ≥95% of claims in the eval
  fixture carry a resolvable citation.
- **Done when:** no citation escapes that isn't backed by a retrieved chunk.

### T8 — Real Claude / OpenAI / Gemini adapters

- **Goal:** production adapters implementing `LlmProvider` for all three vendors.
- **Files:** `src/lib/providers/llm/anthropic.ts`, `src/lib/providers/llm/openai.ts`,
  `src/lib/providers/llm/gemini.ts`; update `src/lib/providers/index.ts`.
- **Libs/tech:** `@anthropic-ai/sdk`, `openai`, `@google/genai`.
- **Depends:** T1
- **Details:** Map neutral types → each SDK and back: tool-use (tool specs, tool_use /
  tool_result round-trip), vision (image content parts), streaming, and `usage` extraction
  ({inputTokens, outputTokens, cached}). **Claude:** `claude-opus-4-8` default,
  `thinking: {type: "adaptive"}`, `output_config: {effort: "high"}`, **no** sampling params
  or `budget_tokens` (400 on Opus 4.8); parse tool `input` via JSON (never raw-string
  match). **OpenAI/Gemini:** equivalent tool-call + vision mappings; set capability flags
  honestly (`nativeCitations` where offered, else false). Selector reads `LLM_PROVIDER` +
  per-provider model env. Each adapter parses tool JSON defensively and surfaces refusals.
- **Tests:** contract test suite runs the **same** cases against `mock` always and, gated
  behind env keys, live against each vendor: a tool round-trip and a vision read return the
  neutral shape; capability flags correct. No invented results — live tests skip without
  keys.
- **Done when:** switching `LLM_PROVIDER` changes the answering model with no orchestration
  change (NFR1); mock + at least one live adapter pass the contract suite.

### T9 — Per-request usage & cost tracking

- **Goal:** record token/embedding usage and estimated cost per request, attributable to a
  conversation.
- **Files:** `src/lib/agent/usage.ts`, `src/db/schema.ts` (usage table), migration.
- **Libs/tech:** Drizzle; a small model→price table in config.
- **Depends:** T6, T8
- **Details:** Sum LLM turns + embedding calls into a `RequestUsage`; multiply by a
  configurable per-model price table → `estimatedCostUsd`. Persist one usage row per request
  keyed to `{conversationId, messageId, provider, model}`. Surface totals in the API
  response for 0005/0006 display (NFR: cost transparency). Prices are config, not hardcoded.
- **Tests:** unit — usage sums across a multi-turn mock run; cost = tokens × configured
  price; missing price falls back gracefully (records tokens, null cost).
- **Done when:** every request yields a persisted usage row with tokens and (when priced)
  cost.

### T10 — Agent-trace persistence (scope of retention, OQ)

- **Goal:** persist enough agent trace for replay/debugging without unbounded cost.
- **Files:** `src/db/schema.ts` (trace columns/table), `src/lib/agent/orchestrator.ts`
  (emit trace), migration.
- **Libs/tech:** Drizzle (JSONB), config flag.
- **Depends:** T6
- **Details:** **Decision (open question):** persist a *structured summary* trace by
  default — ordered steps of `{type: reason|tool_call|tool_result, toolName, inputSummary,
  returnedChunkIds, tokens}` — not full raw model text or full chunk bodies (which live in
  `chunks`). A `AGENT_TRACE_VERBOSE` flag can store fuller payloads in dev. Trace attaches
  to the message row so 0005 can render the optional "thinking" trace (PRD FR12). Cap trace
  size; truncate with a marker when exceeded.
- **Tests:** unit — a mock run persists a step-ordered trace referencing the retrieved
  chunk ids; verbose flag stores more; size cap truncates.
- **Done when:** each answered request stores a replayable, bounded trace.

### T11 — Orchestration API route handler

- **Goal:** the HTTP seam 0005 consumes — accepts a question, returns grounded answer +
  structured citations + usage (+ trace).
- **Files:** `src/app/api/agent/query/route.ts`,
  optional `src/app/api/agent/tools/route.ts`.
- **Libs/tech:** Next.js Route Handler, Zod (request/response), Auth.js session, `server-only`.
- **Depends:** T6, T7, T9, T10
- **Details:** `POST /api/agent/query` — auth-gated; body `{ question, documentIds?,
  provider?, model? }` validated by Zod. Resolve `ScopeContext` from the session (T4), run
  `runAgent`, and return `{ answer, citations: Citation[], usage, traceId }`. Keep the
  contract **streaming-agnostic** so 0005 can add SSE without changing the data shape (leave
  a clean seam: a single non-streaming JSON response now; note where a stream would attach).
  Errors map to typed HTTP codes; refusals return a normal 200 with a refusal answer + empty
  citations. Never expose out-of-scope documents.
- **Tests:** integration — authed request with the mock provider returns a grounded answer
  with validated citations and a usage block; an out-of-corpus question returns a clear
  refusal; unauthenticated request is rejected; scope isolation holds end-to-end.
- **Done when:** the API answers grounded questions, refuses unsupported ones, and exposes
  citations + usage as stable data for 0005.

## Data model / migrations

One migration adding (extending PRD §10):

- `conversations` / `messages` — chat threads (created here for the agent to write into;
  0005 owns the UI). `messages.answer` text, `messages.provider`/`model`.
- `citations` — `{message_id, chunk_id (FK chunks), document_id, page, region (jsonb bbox),
  modality}`; FK to `chunks` guarantees a citation cannot reference a non-existent chunk at
  the DB layer (belt-and-suspenders to T7's ledger check).
- `request_usage` — `{message_id, provider, model, input_tokens, output_tokens,
  embedding_tokens, estimated_cost_usd (nullable)}`.
- `agent_traces` (or `messages.trace` JSONB) — bounded structured step log (T10).
- Corpus-scope columns: ensure `documents.owner_id` (+ optional role-scope table) exist for
  T4; add if 0002/0003 didn't.

No new vector columns — retrieval reuses 0003's indexes.

## Testing strategy

- **Unit:** provider adapters (neutral↔SDK mapping via mock), tools + scope isolation,
  citation validator, usage summation, prompt snapshot, trace shape.
- **Integration:** orchestration loop end-to-end on `MockLlmProvider` (multi-part →
  multiple retrievals → single grounded answer; budget cap; refusal); tools against a
  seeded pgvector test DB; API route with auth + scope.
- **Contract:** one suite run against `mock` (always) and each live vendor (skipped without
  keys) — no invented results (NFR3).
- **Eval:** on the labelled multimodal fixture, assert ≥95% of answer claims carry a
  resolvable citation (NFR4) and a table/figure question is answered from the crop and cited
  to it.
- **Determinism:** all CI-required tests use the mock provider — no network, stable output.

## Definition of done

- [ ] Every task complete; all listed tests green.
- [ ] Local gate passes (`pnpm lint && pnpm typecheck && pnpm test && pnpm build`).
- [ ] Spec acceptance criteria met: multi-part → multiple retrievals + one grounded answer;
      table/figure question answered from the crop and cited; every citation resolves to a
      retrieved chunk; out-of-corpus question declined; switching provider via config
      changes the model with no code change.
- [ ] Orchestration API + citation data exposed as a clean, streaming-agnostic seam for
      0005 (chat UI/streaming not built here).
- [ ] OQ5 resolved: per-user/per-role isolation enforced at the tool/query layer and
      documented; agent-trace retention decision recorded.
- [ ] `.env.example` documents `LLM_PROVIDER` + per-provider model/key vars and budget
      config; `CHANGELOG.md` updated.
- [ ] PR merged into `develop`; spec 0004 set to `Shipped` at the v0.4.0 release.

## Risks / notes

- **Provider capability drift** — vision/tool-use/citation support differs per vendor;
  orchestration must branch on `capabilities`, never assume. Keep vendor SDKs strictly
  inside adapters (provider abstraction is sacred).
- **Claude API shape** — Opus 4.8 rejects `budget_tokens` and sampling params (400); use
  adaptive thinking + `effort`. Parse tool `input` as JSON, never string-match.
- **Fabricated citations** — the ledger + validator (T7) and the `chunks` FK (migration)
  are two independent guards; keep both.
- **Budget correctness** — an under-tight step/token cap can truncate legitimate multi-part
  answers; make caps config and test both truncation and completion paths.
- **OQ5 vs 0006** — the per-user/per-role model set here must stay consistent with 0006's
  deployment/RBAC; cross-link both docs.
- **Trace cost** — verbose traces can balloon storage; default to summary, gate full
  payloads behind a dev flag (T10).
- **Seam for 0005** — resist adding streaming/rendering here; keep the response shape stable
  so 0005 layers SSE + inline citation rendering without touching orchestration.
