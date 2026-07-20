---
title: Gitflow for this project
status: Draft
created: 2026-07-21
updated: 2026-07-21
---

# Gitflow for Agentic RAG

How this project uses **classic Gitflow** with feature branching, and how it maps onto
the **release-per-folder** spec-driven workflow (each release is `specs/vX.Y.Z/` with a
`spec.md` + `plan.md`). This is the practical, project-specific companion to the concise
rules in [`CONTRIBUTING.md`](../CONTRIBUTING.md) and the process in
[`specs/README.md`](../specs/README.md).

## The core idea

One **release folder** ≙ one **feature branch** ≙ one **version tag**. The spec's `id`
(e.g. `0002`) and the release folder (`v0.2.0`) refer to the same slice of work; the
feature branch that builds it is named after it (`feature/0002-document-ingestion-pipeline`).

## Long-lived branches

| Branch | Role | Rule |
| --- | --- | --- |
| **`main`** | Production. Every commit is a tagged `vX.Y.Z` release. | Only `release/*` and `hotfix/*` merge here; each merge is tagged. |
| **`develop`** | Integration. The latest delivered development work. | Feature branches merge here via PR. Always ahead of (or equal to) `main`. |

**Current state:** `main` holds the initial scaffold; `develop` is ahead with all the
planning docs (PRD, architecture, specs + plans, roadmap). The first production release
will be `v0.1.0`. From here on, all non-trivial work goes through the branches below.

## Supporting branches

| Branch | Cut from | Merges into | Purpose |
| --- | --- | --- | --- |
| **`feature/<NNNN-slug>`** | `develop` | `develop` | Build one release by executing its `plan.md`. |
| **`docs/<slug>`** | `develop` | `develop` | Docs-only changes (authoring/editing a spec + plan, PRD, this doc, etc.). |
| **`release/vX.Y.Z`** | `develop` | `main` **and** `develop` | Stabilise a release: version bump, `CHANGELOG.md`, spec → `Shipped`, bugfixes only. |
| **`hotfix/vX.Y.Z`** | `main` | `main` **and** `develop` | Urgent production fix; tags a new patch. |

### Branch naming for this project

- Implementation: `feature/0002-document-ingestion-pipeline` — the spec `id` + slug, so a
  branch maps unambiguously to a release folder. (The chosen name is recorded in each
  release's `plan.md` frontmatter as `branch:`.)
- Docs/spec authoring: `docs/roadmap-update`, `docs/0007-hybrid-reranking-spec`, etc.
- Small chores: `feature/chore-<slug>`.

## The end-to-end flow (SDD → Gitflow)

```
1. PLAN     docs/<slug> off develop → write specs/vX.Y.Z/{spec.md,plan.md}
            → PR into develop → spec status: Proposed → Accepted
                                   │
2. BUILD    feature/<NNNN-slug> off develop → execute plan.md task by task
            → PR into develop (CI green; reviewed vs. plan + acceptance criteria)
                                   │
3. RELEASE  release/vX.Y.Z off develop → bump version, update CHANGELOG.md,
            set spec status: Shipped, bugfixes only
            → merge to main + TAG vX.Y.Z → merge back to develop
                                   │
4. (if needed) HOTFIX  hotfix/vX.Y.Z off main → fix → merge to main (tag) + develop
```

A release may bundle one feature (the common case here — one release folder per version)
or several, if more than one `feature/*` has landed on `develop` since the last tag.

## How the six planned releases flow

```
main     ●───────────────────●──────────────●─────────── … ──────●
          \                 ↗ \            ↗                      ↗
           \        tag v0.1.0  \  tag v0.2.0                tag v1.0.0
            \               ↗    \       ↗                        ↗
develop  ●───●────●────●───●──────●──────●──── … ────────────────●
              \        ↗          \      ↗
   feature/    0001-foundation     0002-ingestion  …  1.0.0-self-hosting
   (off develop, PR back into develop; release/* branches cut from develop to main)
```

Release → branch → tag mapping:

| Release folder | Feature branch | Release branch | Tag |
| --- | --- | --- | --- |
| `specs/v0.1.0/` | `feature/0001-project-foundation` | `release/v0.1.0` | `v0.1.0` |
| `specs/v0.2.0/` | `feature/0002-document-ingestion-pipeline` | `release/v0.2.0` | `v0.2.0` |
| `specs/v0.3.0/` | `feature/0003-multimodal-embeddings-and-vector-store` | `release/v0.3.0` | `v0.3.0` |
| `specs/v0.4.0/` | `feature/0004-agentic-retrieval-orchestration` | `release/v0.4.0` | `v0.4.0` |
| `specs/v0.5.0/` | `feature/0005-chat-and-citations` | `release/v0.5.0` | `v0.5.0` |
| `specs/v1.0.0/` | `feature/0006-self-hosting-and-deployment` | `release/v1.0.0` | `v1.0.0` |

## Worked example — shipping v0.1.0

```bash
# 1. Build the release on a feature branch off develop
git switch develop && git pull
git switch -c feature/0001-project-foundation
#    … execute specs/v0.1.0/plan.md task by task, committing as you go …
git push -u origin feature/0001-project-foundation
#    open a PR into develop; CI (TS + Python) must be green; reviewed vs.
#    plan.md tasks + spec.md acceptance criteria; squash/merge into develop.

# 2. Cut the release from develop
git switch develop && git pull
git switch -c release/v0.1.0
#    - bump the version (package.json)
#    - update CHANGELOG.md: move items from [Unreleased] into a [0.1.0] section
#    - set specs/v0.1.0/spec.md frontmatter: status: Shipped
#    - release-only bugfixes if any
git commit -am "chore(release): v0.1.0"

# 3. Merge to main and tag (the tag is what defines/triggers the release)
git switch main && git pull
git merge --no-ff release/v0.1.0
git tag -a v0.1.0 -m "v0.1.0 — project foundation"
git push origin main --tags

# 4. Merge the release branch back into develop, then delete it
git switch develop
git merge --no-ff release/v0.1.0
git push origin develop
git branch -d release/v0.1.0 && git push origin --delete release/v0.1.0
```

## Hotfix example (production bug after a release)

```bash
git switch main && git pull
git switch -c hotfix/v0.1.1
#    … fix, with a test …
git commit -am "fix: <the bug>"
git switch main && git merge --no-ff hotfix/v0.1.1
git tag -a v0.1.1 -m "v0.1.1 — <fix summary>" && git push origin main --tags
git switch develop && git merge --no-ff hotfix/v0.1.1 && git push origin develop
git branch -d hotfix/v0.1.1
```

## Docs & spec changes

Documentation and spec/plan authoring are **docs-only PRs into `develop`** on a
`docs/<slug>` branch — they don't need a release branch. A new spec opens as `Proposed`
and moves to `Accepted` when the docs PR merges; it becomes `Shipped` only when the
implementation ships in a `release/*`.

> Note: during the initial planning bootstrap, the planning docs were committed directly
> to `develop` for speed. That's a one-time exception — from the first `feature/*`
> onward, use branches + PRs as above.

## Commit conventions

- **[Conventional Commits](https://www.conventionalcommits.org)** (enforced by a
  commitlint `commit-msg` hook once tooling lands): `feat`, `fix`, `docs`, `chore`,
  `refactor`, `test`, `ci`, with optional scopes — e.g. `feat(ingestion):`,
  `fix(retrieval):`, `docs(spec):`, `chore(release):`. Body lines ≤ 100 chars.
- **No AI attribution.** Never add `Co-Authored-By: Claude`, "Generated with…", or any
  assistant trailer to commits or PRs. Commits are authored solely by the contributor.

## Recommended branch protection

On GitHub, protect `main` and `develop`: require a PR + passing CI before merge, disallow
direct pushes, and restrict who can push tags. `main` should additionally require the
change to arrive only via `release/*` or `hotfix/*`.

## Cheat sheet

| Task | Command |
| --- | --- |
| Start a feature | `git switch develop && git switch -c feature/<NNNN-slug>` |
| Start docs/spec work | `git switch develop && git switch -c docs/<slug>` |
| Cut a release | `git switch develop && git switch -c release/vX.Y.Z` |
| Tag on main | `git switch main && git merge --no-ff release/vX.Y.Z && git tag -a vX.Y.Z` |
| Back-merge to develop | `git switch develop && git merge --no-ff release/vX.Y.Z` |
| Start a hotfix | `git switch main && git switch -c hotfix/vX.Y.Z` |

## See also

- [`CONTRIBUTING.md`](../CONTRIBUTING.md) — concise contributor rules
- [`specs/README.md`](../specs/README.md) — spec-driven process and lifecycle
- [`docs/roadmap.md`](./roadmap.md) — release plan and status
