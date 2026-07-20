---
plan_for: 0005
title: Chat interface & citations — implementation plan
status: Proposed
branch: feature/0005-chat-and-citations
created: 2026-07-21
updated: 2026-07-21
---

# 0005 — Implementation plan

> Implements [`specs/v0.5.0/spec.md`](./spec.md). This
> plan turns that spec into ordered, agent-executable coding tasks. Execute tasks in
> order, respecting `depends`.

## Objective

Build the user-facing question-answering product over the 0004 orchestration API:
persisted conversations (create/list/continue/delete) with follow-up context, SSE token
streaming with progressive render, a collapsible agent trace, inline clickable citations
that resolve to a source panel highlighting the exact region (text passage on page;
table/figure crop with bbox), conversation-level corpus scoping, and clean refusals — on
the boilerplate's responsive PWA shell (light/dark), first token < 3 s p50.

## Prerequisites

- **Requires 0004 `Shipped`** — the agent orchestration API (`runAgent`-style entry
  producing streamed tokens, trace events, and validated structured citations) and its
  retrieval tools that accept a document-scope filter (0004 OQ5). Transitively 0001–0003.
- 0003 chunk metadata (`document`, `page`, `bbox`, `modality`) and MinIO page renders /
  crop assets must exist and be resolvable by chunk id.
- No new external services or keys beyond 0004's provider config.

## Tech stack for this slice

- **Web:** Next.js 16 (App Router, RSC, Server Actions), React 19, TypeScript 5.9
  (strict), Tailwind v4, shadcn/ui. **Turbopack** for `dev` and `build`.
- **Streaming:** Route Handlers returning `ReadableStream` as `text/event-stream`
  (SSE); browser `EventSource` / `fetch`+`ReadableStream` reader on the client.
- **Persistence:** Drizzle ORM + Postgres 17 (existing store) — `conversations`,
  `messages` tables; `citations[]` and `trace` as `jsonb`.
- **Assets:** MinIO (S3-compatible) for page renders and table/figure crops, served via
  an existing signed-URL / asset route.
- **Testing:** Vitest (unit/integration), Playwright (E2E), a mock 0004 agent for
  deterministic streaming in CI.

## Touchpoints (files & modules)

- **DB:** `src/db/schema.ts` (add `conversations`, `messages`); new `drizzle/NNNN_*.sql`
  migration.
- **Server data layer:** `src/server/conversations.ts` (CRUD + scope), `server-only`.
- **Route handlers (API):**
  - `src/app/api/conversations/route.ts` (GET list, POST create).
  - `src/app/api/conversations/[id]/route.ts` (GET one, DELETE).
  - `src/app/api/conversations/[id]/messages/route.ts` (POST → SSE stream).
  - `src/app/api/assets/[chunkId]/route.ts` (page render / crop + bbox metadata).
- **Pages:** `src/app/(app)/chat/layout.tsx`, `src/app/(app)/chat/page.tsx`,
  `src/app/(app)/chat/[id]/page.tsx`.
- **Components:** `src/components/chat/` — `conversation-list.tsx`, `message-list.tsx`,
  `streaming-answer.tsx`, `citation-marker.tsx`, `agent-trace-panel.tsx`,
  `source-panel.tsx`, `scope-selector.tsx`, `composer.tsx`, `refusal.tsx`.
- **Client hooks/lib:** `src/lib/chat/use-sse-stream.ts`, `src/lib/chat/citations.ts`
  (types + parse/anchor helpers).
- **Tests:** `src/**/*.test.ts(x)`, `e2e/chat.spec.ts`.
- **Config/docs:** `.env.example`, `CHANGELOG.md` if new knobs.

## Task breakdown

### T1 — Data model: conversations & messages

- **Goal:** persist threads and messages with answer, citations, and optional trace.
- **Files:** `src/db/schema.ts` (edit), new `drizzle/NNNN_*.sql` migration.
- **Libs/tech:** Drizzle ORM (`pgTable`, `jsonb`, `uuid`, `timestamp`, FKs).
- **Depends:** —
- **Details:** `conversations` — `id` (uuid pk), `userId` (FK users), `title`,
  `scopeDocumentIds jsonb` (null/[] = whole corpus), `createdAt`, `updatedAt`.
  `messages` — `id`, `conversationId` (FK, cascade delete), `role` (`user`|`assistant`),
  `content text`, `citations jsonb` (array of `{chunkId, document, page, region, modality}`),
  `trace jsonb` (nullable serialized agent steps), `refused boolean default false`,
  `createdAt`. Index `messages(conversationId, createdAt)`. Generate + commit migration.
- **Tests:** Vitest integration — insert conversation + messages, cascade delete removes
  messages; `citations`/`trace` round-trip as typed JSON.
- **Done when:** `pnpm db:migrate` applies cleanly; round-trip + cascade tests green.

### T2 — Citation & trace types + server data layer

- **Goal:** shared types and `server-only` CRUD/scope functions for conversations.
- **Files:** `src/lib/chat/citations.ts` (new), `src/server/conversations.ts` (new).
- **Libs/tech:** TypeScript, Drizzle, `server-only`, Zod for input validation.
- **Depends:** T1
- **Details:** Define `Citation` (`chunkId, document{id,title}, page, region{bbox?},
  modality: 'text'|'table'|'figure'`) and `TraceStep` (`kind`, `label`, `detail`,
  `chunkIds?`) — reuse 0004's citation shape, don't redefine divergently. Implement
  `createConversation`, `listConversations(userId)`, `getConversation(id, userId)`,
  `deleteConversation`, `appendMessage`, and `setScope`. All ownership-checked by
  `userId`. Zod schemas for create/scope payloads.
- **Tests:** Vitest — ownership isolation (user B can't read/delete A's thread); scope
  set/get; Zod rejects malformed payloads.
- **Done when:** functions typecheck under strict TS; isolation + validation unit tests
  green.

### T3 — Conversation CRUD route handlers

- **Goal:** REST endpoints for list/create/get/delete threads.
- **Files:** `src/app/api/conversations/route.ts`,
  `src/app/api/conversations/[id]/route.ts`.
- **Libs/tech:** Next.js Route Handlers, Auth.js v5 session, T2 data layer.
- **Depends:** T2
- **Details:** `GET /api/conversations` → user's threads (id, title, updatedAt).
  `POST` → create (optional initial `scopeDocumentIds`) → 201 with id. `GET
  /api/conversations/[id]` → thread + messages. `DELETE` → cascade remove. All gated by
  session; 401 unauth, 404 on non-owned id (no existence leak).
- **Tests:** Vitest route tests — auth gating, create→list→get→delete lifecycle, 404 for
  another user's id.
- **Done when:** lifecycle passes; unauthorized and cross-user access rejected.

### T4 — Streaming message endpoint (SSE) over 0004

- **Goal:** POST a question → stream answer tokens, trace events, and final citations.
- **Files:** `src/app/api/conversations/[id]/messages/route.ts` (new).
- **Libs/tech:** Route Handler returning `ReadableStream` as `text/event-stream`; 0004
  agent entrypoint; T2 data layer.
- **Depends:** T2, T3
- **Details:** Persist the user message; load prior messages for follow-up context; pass
  the conversation's `scopeDocumentIds` into the agent's retrieval tools (0004 OQ5).
  Stream named SSE events on separate channels: `token` (answer delta), `trace` (agent
  step), `citations` (final validated array), `refusal`, `done`, `error`. On completion
  persist the assistant message (content + citations + trace + `refused`). Flush headers
  immediately so first `token` reaches the client fast (NFR1). Guard partial writes on
  client disconnect (abort signal).
- **Tests:** Vitest integration with a **mock 0004 agent** — asserts event ordering
  (`trace*`→`token*`→`citations`→`done`), that scope is forwarded to tools, that a
  refusal path emits `refusal`+empty citations and persists `refused=true`, and that the
  assistant message is persisted with citations.
- **Done when:** SSE stream emits ordered events; message + citations persisted; scope
  forwarded; refusal handled.

### T5 — Client SSE hook + composer

- **Goal:** consume the stream and render answer/trace/citations progressively.
- **Files:** `src/lib/chat/use-sse-stream.ts` (new), `src/components/chat/composer.tsx`
  (new).
- **Libs/tech:** React 19 client component, `fetch`+`ReadableStream` reader (POST body),
  `AbortController`.
- **Depends:** T4
- **Details:** `useSseStream` POSTs the question, parses SSE frames, and exposes
  reactive `tokens` (accumulated), `traceSteps`, `citations`, `status`
  (`idle|streaming|done|refused|error`), plus `cancel()`. Composer handles submit,
  disabled-while-streaming, and cancel. No layout shift as tokens arrive.
- **Tests:** Vitest — hook parses a mocked SSE `ReadableStream` into ordered state;
  cancel aborts the request.
- **Done when:** hook yields progressive state from a mocked stream; cancel works.

### T6 — Message list & streaming answer render

- **Goal:** render the thread with a progressively streamed assistant answer.
- **Files:** `src/components/chat/message-list.tsx`,
  `src/components/chat/streaming-answer.tsx`, `src/components/chat/refusal.tsx` (all new).
- **Libs/tech:** React 19, Tailwind v4, shadcn/ui, T5 hook.
- **Depends:** T5
- **Details:** `message-list` renders persisted + in-flight messages. `streaming-answer`
  renders accumulated markdown-safe text with inline citation-marker placeholders (T7)
  and a typing affordance while `status==='streaming'`. `refusal` renders the
  out-of-corpus message cleanly with **no citation markers** (FR/AC: no fake citations).
- **Tests:** Vitest RTL — tokens append in order; refusal renders text with zero markers.
- **Done when:** answers render progressively; refusals render clean, marker-free.

### T7 — Inline citation markers

- **Goal:** clickable inline markers anchored into `citations[]`.
- **Files:** `src/components/chat/citation-marker.tsx` (new),
  `src/lib/chat/citations.ts` (edit — anchor/parse helpers).
- **Libs/tech:** React 19, Tailwind, shadcn/ui, T2 types.
- **Depends:** T6
- **Details:** Parse citation references embedded in the answer (per 0004's contract —
  e.g. `[n]` or a marker token) into anchor elements keyed by `chunkId`. Each marker is a
  button that dispatches a `selectCitation(chunkId)` to open/scroll the source panel
  (T9). Accessible (role, aria-label with source doc/page), keyboard-focusable. Ignore /
  drop any marker whose id is not in the validated `citations[]` (defensive vs. fake
  refs).
- **Tests:** Vitest RTL — marker click fires selection with the right chunkId; unknown
  ids are not rendered as active citations.
- **Done when:** markers render inline, are keyboard-accessible, and select the correct
  citation.

### T8 — Collapsible agent-trace panel

- **Goal:** show the agent's steps when present; hidden by default (FR3).
- **Files:** `src/components/chat/agent-trace-panel.tsx` (new).
- **Libs/tech:** React 19, shadcn/ui `Collapsible`/`Accordion`, Tailwind.
- **Depends:** T6
- **Details:** Render `traceSteps` (sub-questions, searches issued, retrieved sources) in
  a collapsed disclosure with a "Show reasoning" toggle. Retrieved-source rows link to
  their citation (reuse T7 selection). Renders nothing if no trace. Persisted `trace` on
  historical messages renders the same way.
- **Tests:** Vitest RTL — collapsed by default; expands on toggle; absent when no trace.
- **Done when:** trace hidden by default, toggles open, and is absent when empty.

### T9 — Source panel: page-render + bbox highlight (text/table/figure)

- **Goal:** resolve a citation to its page render/crop with the region highlighted.
- **Files:** `src/components/chat/source-panel.tsx` (new),
  `src/app/api/assets/[chunkId]/route.ts` (new).
- **Libs/tech:** React 19, Tailwind, MinIO asset route, T2 types, absolute/overlay
  positioning for bbox.
- **Depends:** T7
- **Details:** Asset route returns, for a chunk, the page-render URL, crop URL (if
  table/figure), page dimensions, and normalized `bbox`. Panel lazy-loads on selection,
  scrolls to the citation, and draws a highlight overlay from the bbox scaled to the
  rendered page. **OQ4 (resolved):** for `table`/`figure` sources the **primary view is
  the full page render + bbox highlight** for spatial context, with the **standalone crop
  as an inset/fallback** (inset alongside the page; fallback when no page render exists).
  For `text`, highlight the passage region on the page. Handle missing assets gracefully
  (message, no crash).
- **Tests:** Vitest RTL + mocked asset route — text citation renders page + overlay at
  bbox; table/figure citation renders page-render+highlight and exposes crop fallback;
  missing asset shows a graceful state.
- **Done when:** clicking a citation opens the panel to the correct page with the region
  highlighted for all three modalities; OQ4 primary page-render + crop inset/fallback
  implemented.

### T10 — Corpus scope selector

- **Goal:** scope a conversation to all documents or a chosen subset (FR7).
- **Files:** `src/components/chat/scope-selector.tsx` (new); wires T2 `setScope`.
- **Libs/tech:** React 19, shadcn/ui multi-select/command, `list_documents` data.
- **Depends:** T3
- **Details:** Selector lists the user's documents; "All documents" default or a subset.
  Persists `scopeDocumentIds` on the conversation; T4 already forwards it to the agent's
  retrieval tools. Show current scope on the thread header.
- **Tests:** Vitest RTL — selecting a subset persists it; Vitest integration — scoped
  conversation forwards only selected doc ids to the mock agent's tools.
- **Done when:** scope is settable, persisted, and forwarded to retrieval.

### T11 — Chat pages & responsive PWA shell

- **Goal:** assemble the pages (list/new/continue) on the boilerplate PWA shell.
- **Files:** `src/app/(app)/chat/layout.tsx`, `src/app/(app)/chat/page.tsx`,
  `src/app/(app)/chat/[id]/page.tsx`, `src/components/chat/conversation-list.tsx` (all
  new).
- **Libs/tech:** Next.js App Router (RSC for initial load), Tailwind v4, shadcn/ui, T3
  endpoints.
- **Depends:** T6, T7, T8, T9, T10
- **Details:** `layout` = conversation sidebar (list, new, delete) + main pane +
  source panel; responsive (sidebar collapses to drawer on mobile), light/dark via the
  boilerplate theme. `page` = new conversation; `[id]/page` = continue (RSC loads
  persisted messages, then client hook streams follow-ups). Scope selector in the
  thread header.
- **Tests:** Vitest RTL smoke for layout composition; covered end-to-end by T12.
- **Done when:** create/continue/delete flows work; layout is responsive and themed.

### T12 — E2E: ask → stream → cite → verify (incl. table/figure)

- **Goal:** prove the full journey including a table/figure citation and a refusal.
- **Files:** `e2e/chat.spec.ts` (new); Playwright fixtures / mock agent seam.
- **Libs/tech:** Playwright; a deterministic mock 0004 agent (recorded stream with a
  text citation and a table/figure citation) so CI is stable.
- **Depends:** T11
- **Details:** Sign in → new conversation → ask → assert answer streams (first token
  appears quickly) with inline markers → click a **text** citation → source panel opens
  to the page with a highlight → click a **table/figure** citation → panel shows the
  page-render+highlight (OQ4 primary) and the crop inset/fallback is reachable → toggle the
  agent trace → ask a follow-up (context retained) → ask an out-of-corpus question →
  assert a clean refusal with **no** citation markers. Optionally a scoped-conversation
  case asserting retrieval is limited to selected docs.
- **Tests:** the spec itself; asserts render, highlight, trace, follow-up, refusal.
- **Done when:** E2E passes deterministically in CI with the mock agent.

## Data model / migrations

- One migration adds `conversations` and `messages`:
  - `conversations(id uuid pk, user_id fk, title text, scope_document_ids jsonb,
    created_at, updated_at)`.
  - `messages(id uuid pk, conversation_id fk cascade, role text, content text,
    citations jsonb, trace jsonb null, refused boolean default false, created_at)`,
    index on `(conversation_id, created_at)`.
- No new vector columns (retrieval lives in 0003/0004). `citations`/`trace` are `jsonb`
  mirroring 0004's structured citation and trace shapes — do not diverge.

## Testing strategy

- **Unit (Vitest/RTL):** citation types + anchor helpers, SSE hook parsing/cancel,
  citation markers, trace panel toggle, source-panel highlight for all three modalities,
  scope selector, refusal render.
- **Integration (Vitest):** conversation CRUD + ownership isolation; SSE endpoint event
  ordering, scope forwarding, refusal persistence — all against a **mock 0004 agent** for
  determinism.
- **E2E (Playwright):** the ask→stream→cite→verify journey incl. a table/figure citation,
  trace toggle, follow-up context, and a clean refusal, with the mock agent so CI is
  stable.
- **Perf note:** first-token < 3 s p50 (NFR1) is a manual/observed check against a warm
  real 0004 agent; CI asserts prompt first-token *behaviour* via the mock, not wall-time.

## Definition of done

- [ ] Every task complete; all listed tests green.
- [ ] Local gate passes (`pnpm lint && pnpm typecheck && pnpm test && pnpm build`).
- [ ] Spec acceptance criteria met: streaming answer with inline citations; clicking a
      citation opens the source panel highlighted on the correct page; a table/figure
      answer shows the crop/region; follow-ups retain context; scoped conversations
      retrieve only selected docs; refusals render with no fake citations.
- [ ] OQ4 resolved in code: page render + bbox highlight primary, crop inset/fallback.
- [ ] Trace hidden by default; responsive PWA shell in light/dark.
- [ ] `.env.example`/docs updated for any new config; `CHANGELOG.md` updated.
- [ ] PR merged into `develop`; spec 0005 set to `Shipped` at the v0.5.0 release.

## Risks / notes

- **Resolves OQ4** (citation UX for image/table): primary view is the full-page render +
  bbox highlight, standalone crop as an inset/fallback — implemented in T9.
- **Depends on 0004's citation contract.** If 0004's inline marker token or citation
  shape differs from T2's assumptions, reconcile by reusing 0004's types rather than
  redefining — a mismatch silently breaks marker→panel resolution.
- **SSE proxying.** Cloudflare Tunnel / any buffering proxy must not buffer
  `text/event-stream`; disable buffering and flush early to protect NFR1 first-token.
- **Turbopack constraint** — no webpack-only streaming/markdown tooling; keep deps
  Turbopack-compatible for `dev` and `build`.
- **Scope isolation** ties into 0004/0006 OQ5 (per-user vs per-role). Enforce ownership
  at the query layer here; defer the broader role model to 0006.
- **Trace persistence cost** (0004 open question): store the serialized trace as `jsonb`;
  if size is a concern, cap/trim steps before persisting rather than storing raw.
