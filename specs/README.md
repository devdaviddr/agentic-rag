# Specs

This project is **spec-driven**: non-trivial work is described in a numbered spec
*before* it is built. A spec is the single source of truth for what a slice of the
system should do and why — decisions, requirements, and open questions live here, not
scattered across issues and commit messages.

## Why spec-driven

- **Think before building.** Requirements and trade-offs are settled on paper, cheaply.
- **Reviewable intent.** A PR can be judged against an agreed spec, not guessed at.
- **Durable rationale.** Six months later, the *why* is still here.
- **AI-friendly.** Assistants working in this repo ground their work in the spec rather
  than inferring intent from source.

## Lifecycle

Every spec carries a `status` in its frontmatter:

| Status | Meaning |
| --- | --- |
| **Proposed** | Drafted, under discussion. Requirements may still change. |
| **Accepted** | Agreed and ready to build. Changes now update the spec deliberately. |
| **Shipped** | Delivered and merged to `main`; released under the noted version. |
| **Superseded** | Replaced by a later spec (link to it). |

Typical flow: `Proposed → Accepted → Shipped`. A spec may be `Superseded` if a later
one replaces its approach.

## Writing a new spec

1. Copy [`TEMPLATE.md`](TEMPLATE.md) to the next number: `NNNN-short-slug.md`
   (zero-padded, e.g. `0007-hybrid-reranking.md`).
2. Fill in the frontmatter (`id`, `title`, `status: Proposed`, `release`, dates) and
   sections. Keep it tight — Summary, Problem, Goals/Non-goals, Requirements, Design,
   Acceptance criteria, Dependencies, Open questions.
3. Open a PR for the spec (docs-only). Once agreed, set `status: Accepted`.
4. Build against it on a `feature/<slug>` branch. On merge, set `status: Shipped` and
   note the release version.
5. Keep the spec updated if reality diverges — it stays the source of truth.

## How specs relate to git

- One `feature/<slug>` branch implements one spec (or a coherent part of it).
- The PR references its spec; reviewers check the diff against the spec's acceptance
  criteria.
- Shipping is a `vX.Y.Z` tag on `main` (see [`CONTRIBUTING.md`](../CONTRIBUTING.md)).

## Index

| Spec | Title | Status | Release |
| --- | --- | --- | --- |
| [0001](0001-project-foundation.md) | Project foundation | Proposed | v0.1.0 |
| [0002](0002-document-ingestion-pipeline.md) | Document ingestion pipeline | Proposed | v0.2.0 |
| [0003](0003-multimodal-embeddings-and-vector-store.md) | Multimodal embeddings & vector store | Proposed | v0.3.0 |
| [0004](0004-agentic-retrieval-orchestration.md) | Agentic retrieval orchestration | Proposed | v0.4.0 |
| [0005](0005-chat-and-citations.md) | Chat interface & citations | Proposed | v0.5.0 |
| [0006](0006-self-hosting-and-deployment.md) | Self-hosting & deployment | Proposed | v1.0.0 |

Open questions across specs are also tracked in the
[PRD](../docs/PRD.md#14-open-questions).
