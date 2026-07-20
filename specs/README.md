# Specs

This project is **spec-driven**: non-trivial work is described in a spec *before* it is
built. A spec is the single source of truth for what a slice of the system should do and
why — decisions, requirements, and open questions live here, not scattered across issues
and commit messages.

## Layout: one folder per release

Work is organised by **release**, using [semantic versioning](https://semver.org). Each
release is a folder under `specs/` containing two files:

```
specs/
  _template/              # copy this to start a new release
    spec.md
    plan.md
  v0.1.0/
    spec.md               # WHAT & WHY  — the specification
    plan.md               # HOW         — the implementation plan (agent-executable tasks)
  v0.2.0/
    spec.md
    plan.md
  …
  v1.0.0/
```

- **`spec.md`** — the specification: summary, problem, goals/non-goals, requirements,
  design, acceptance criteria, dependencies, open questions.
- **`plan.md`** — the implementation plan: the spec broken into concrete,
  agent-executable **coding tasks** tied to the tech stack — each task names the files to
  create/modify, the libraries used, its dependencies on other tasks, and the tests that
  prove it done.

## Why spec-driven

- **Think before building.** Requirements and trade-offs are settled on paper, cheaply.
- **Reviewable intent.** A PR can be judged against an agreed spec, not guessed at.
- **Durable rationale.** Six months later, the *why* is still here.
- **AI-friendly.** Assistants ground their work in the spec + plan rather than inferring
  intent from source.

## Lifecycle

Every `spec.md` carries a `status` in its frontmatter:

| Status | Meaning |
| --- | --- |
| **Proposed** | Drafted, under discussion. Requirements may still change. |
| **Accepted** | Agreed and ready to build. Changes now update the spec deliberately. |
| **Shipped** | Delivered and merged; released under the folder's version. |
| **Superseded** | Replaced by a later release (link to it). |

Typical flow: `Proposed → Accepted → Shipped`.

## Every release ships spec **and** plan together

**A release is not ready to build until its `plan.md` exists and is agreed.** The plan is
what an assistant executes, task by task, on the feature branch.

## Starting a new release

1. **Create the folder.** `cp -r specs/_template specs/vX.Y.Z` (pick the next semver).
2. **Spec.** Fill in `spec.md` frontmatter (`id`, `title`, `status: Proposed`, `release`,
   dates) and sections.
3. **Plan.** Fill in `plan.md` — break the spec into ordered, dependency-aware coding
   tasks with file paths, libs, and tests.
4. Open a PR for the release folder (docs-only) into `develop`. Once agreed, set the
   spec `status: Accepted`.
5. Build the plan's tasks on a `feature/<slug>` branch (see below). On release, set the
   spec `status: Shipped`.
6. Keep spec **and** plan updated if reality diverges — together they are the source of
   truth.

## How releases relate to git (Gitflow)

- One `feature/<slug>` branch (off `develop`) implements one release by executing its
  `plan.md`; the PR merges back into `develop`.
- The PR references the release's `spec.md` and `plan.md`; reviewers check the diff
  against the plan's tasks and the spec's acceptance criteria.
- Shipping is a `release/vX.Y.Z` branch merged to `main` and tagged (see
  [`docs/gitflow.md`](../docs/gitflow.md) for the full workflow, or
  [`CONTRIBUTING.md`](../CONTRIBUTING.md) for the concise rules).

## Index

| Release | Title | Spec | Plan | Status |
| --- | --- | --- | --- | --- |
| **v0.1.0** | Project foundation | [spec](v0.1.0/spec.md) | [plan](v0.1.0/plan.md) | Proposed |
| **v0.2.0** | Document ingestion pipeline | [spec](v0.2.0/spec.md) | [plan](v0.2.0/plan.md) | Proposed |
| **v0.3.0** | Multimodal embeddings & vector store | [spec](v0.3.0/spec.md) | [plan](v0.3.0/plan.md) | Proposed |
| **v0.4.0** | Agentic retrieval orchestration | [spec](v0.4.0/spec.md) | [plan](v0.4.0/plan.md) | Proposed |
| **v0.5.0** | Chat interface & citations | [spec](v0.5.0/spec.md) | [plan](v0.5.0/plan.md) | Proposed |
| **v1.0.0** | Self-hosting & deployment | [spec](v1.0.0/spec.md) | [plan](v1.0.0/plan.md) | Proposed |

Open questions across releases are also tracked in the
[PRD](../docs/PRD.md#14-open-questions).
