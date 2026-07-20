# Contributing

Thanks for your interest in Agentic RAG. This project follows **spec-driven
development** and **Gitflow** with feature branching. Every unit of work is planned
before it is built, and every spec ships with an **implementation plan** that breaks
the work into concrete, agent-executable coding tasks tied to the tech stack.

> **Note:** the project is at the planning stage — only docs, specs, and plans exist so
> far. The setup/test commands below describe the intended system (per the specs) and
> apply once the corresponding code lands, starting with release
> [`specs/v0.1.0`](specs/v0.1.0/spec.md) and its
> [plan](specs/v0.1.0/plan.md).

## The loop: spec → plan → branch → PR

Work is organised **one folder per release** under `specs/`, named by semver, each
holding a `spec.md` and a `plan.md`.

1. **Create the release folder.** For a new slice of work, `cp -r specs/_template
   specs/vX.Y.Z` (next semver). Fill in `spec.md`, open it as `Proposed`, get it to
   `Accepted`.
2. **Write its implementation plan** in the same folder's `plan.md`. It enumerates the
   exact code tasks (files, libs, tests) an agent or contributor executes to satisfy the
   spec. **A release is not ready to build until its `plan.md` exists.** See
   [`specs/README.md`](specs/README.md).
3. **Branch** off `develop` (see Gitflow below) and execute the plan's tasks.
4. **Open a PR into `develop`**, referencing the spec and plan. CI must pass; reviewers
   check the diff against the plan's tasks and the spec's acceptance criteria.

## Gitflow branching model

Two long-lived branches, three kinds of supporting branches. For the full project-specific
walkthrough — the SDD→Gitflow flow, a worked example shipping `v0.1.0`, a hotfix flow, and
a command cheat sheet — see **[`docs/gitflow.md`](docs/gitflow.md)**.

| Branch | Base | Merges into | Purpose |
| --- | --- | --- | --- |
| **`main`** | — | — | Production-ready. Every commit is a tagged release (`vX.Y.Z`). |
| **`develop`** | `main` | — | Integration branch; the latest delivered development changes. |
| **`feature/<slug>`** | `develop` | `develop` | One spec (or a coherent part). Named after the spec slug. |
| **`release/vX.Y.Z`** | `develop` | `main` **and** `develop` | Stabilize a release: version bump, changelog, spec→`Shipped`, bugfixes only. |
| **`hotfix/vX.Y.Z`** | `main` | `main` **and** `develop` | Urgent production fix; tag a new patch. |

```
main      ●──────────────────────●──────────────────●────▶   (tags: v0.1.0 … v1.0.0)
           \                    ↗ \                 ↗
develop     ●────●────●────●───●   ●────●────●────●          (integration)
             \        \       /         \       /
 feature/     f/0001   f/0002 …          f/0003 …            (branch off develop → PR into develop)
                                    release/v0.2.0  ─────────┘  (branch off develop → merge to main + develop)
```

### Rules

- **Feature work** always branches from `develop` and PRs back into `develop`. Never
  commit features straight to `develop` or `main`.
- **Branch names** follow the spec: `feature/0002-document-ingestion-pipeline`. Small
  chores may use `feature/chore-<slug>`.
- **Releases:** when `develop` holds a shippable set, cut `release/vX.Y.Z` off
  `develop`. On it: bump version, update `CHANGELOG.md`, set shipped specs to `Shipped`.
  Merge into `main`, **tag `vX.Y.Z`**, then merge back into `develop`. Pushing the tag
  is what triggers deploy. Pre-1.0 until spec 0006 ships.
- **Hotfixes:** branch `hotfix/vX.Y.Z` off `main`, fix, merge to `main` (tag) and
  `develop`.
- `main` and `develop` are protected; changes arrive only via reviewed PRs (features →
  `develop`; `release/*` and `hotfix/*` → `main` + `develop`).

## Local development gate

Run before pushing:

```bash
# web app
pnpm lint && pnpm typecheck && pnpm test && pnpm build
# ingestion service
ruff check && pytest
```

## Commit messages

[Conventional Commits](https://www.conventionalcommits.org), enforced by a commitlint
`commit-msg` hook. Keep body lines ≤ 100 characters. Examples:

```
feat(ingestion): extract tables as first-class chunks
fix(retrieval): filter vectors by document scope
docs(spec): accept 0003 multimodal embeddings
chore(release): v0.2.0
```

## Code style

- TypeScript strict mode; no `any` without justification. Prettier + ESLint are the
  source of truth.
- Python formatted/linted with `ruff`; typed where practical.
- Provider access goes through the `LlmProvider` / `EmbeddingProvider` interfaces —
  never import a vendor SDK directly in callers.
- Keep untrusted document parsing inside the isolated Python ingestion service.

## Attribution

Do **not** add AI/assistant trailers (`Co-Authored-By: Claude`, "Generated with…",
etc.) to commits or PRs. Commits are authored solely by the contributor.
