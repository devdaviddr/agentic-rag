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

## Every spec ships with an implementation plan

A spec says **what** and **why**. Its companion **implementation plan** (in
[`plans/`](plans/)) says **how** — it decomposes the spec into concrete,
agent-executable **coding tasks** tied to the tech stack: each task names the files to
create/modify, the libraries used, its dependencies on other tasks, and the tests that
prove it done.

- Location: `specs/plans/NNNN-slug.md`, mirroring the spec's number and slug.
- Template: [`plans/TEMPLATE.md`](plans/TEMPLATE.md).
- **A spec is not ready to build until its plan exists and is agreed.** The plan is what
  an assistant executes, task by task, on the feature branch.

## Writing a new spec (and its plan)

1. **Spec.** Copy [`TEMPLATE.md`](TEMPLATE.md) to the next number: `NNNN-short-slug.md`
   (zero-padded, e.g. `0007-hybrid-reranking.md`). Fill in the frontmatter (`id`,
   `title`, `status: Proposed`, `release`, dates) and sections. Keep it tight — Summary,
   Problem, Goals/Non-goals, Requirements, Design, Acceptance criteria, Dependencies,
   Open questions.
2. **Plan.** Copy [`plans/TEMPLATE.md`](plans/TEMPLATE.md) to `plans/NNNN-short-slug.md`
   and break the spec into ordered, dependency-aware coding tasks with file paths, libs,
   and tests.
3. Open a PR for the spec + plan (docs-only) into `develop`. Once agreed, set the spec
   `status: Accepted`.
4. Build the plan's tasks on a `feature/NNNN-slug` branch. On release, set the spec
   `status: Shipped` and note the version.
5. Keep spec **and** plan updated if reality diverges — together they are the source of
   truth.

## How specs relate to git (Gitflow)

- One `feature/NNNN-slug` branch (off `develop`) implements one spec by executing its
  plan; the PR merges back into `develop`.
- The PR references its spec and plan; reviewers check the diff against the plan's tasks
  and the spec's acceptance criteria.
- Shipping is a `release/vX.Y.Z` branch merged to `main` and tagged (see
  [`CONTRIBUTING.md`](../CONTRIBUTING.md)).

## Index

| Spec | Title | Plan | Status | Release |
| --- | --- | --- | --- | --- |
| [0001](0001-project-foundation.md) | Project foundation | [plan](plans/0001-project-foundation.md) | Proposed | v0.1.0 |
| [0002](0002-document-ingestion-pipeline.md) | Document ingestion pipeline | [plan](plans/0002-document-ingestion-pipeline.md) | Proposed | v0.2.0 |
| [0003](0003-multimodal-embeddings-and-vector-store.md) | Multimodal embeddings & vector store | [plan](plans/0003-multimodal-embeddings-and-vector-store.md) | Proposed | v0.3.0 |
| [0004](0004-agentic-retrieval-orchestration.md) | Agentic retrieval orchestration | [plan](plans/0004-agentic-retrieval-orchestration.md) | Proposed | v0.4.0 |
| [0005](0005-chat-and-citations.md) | Chat interface & citations | [plan](plans/0005-chat-and-citations.md) | Proposed | v0.5.0 |
| [0006](0006-self-hosting-and-deployment.md) | Self-hosting & deployment | [plan](plans/0006-self-hosting-and-deployment.md) | Proposed | v1.0.0 |

Open questions across specs are also tracked in the
[PRD](../docs/PRD.md#14-open-questions).
