# Contributing

Thanks for your interest in Agentic RAG. This repo is **spec-driven** and
**trunk-based**; the short version of the workflow is below.

> **Note:** the project is at the planning stage — only docs and specs exist so far.
> The setup/test commands below describe the intended system (per the specs) and will
> apply once the corresponding code lands, starting with
> [`specs/0001`](specs/0001-project-foundation.md).

## Workflow

1. **Start with a spec.** For anything non-trivial, write or update a spec first —
   copy [`specs/TEMPLATE.md`](specs/TEMPLATE.md), open it as `Proposed`, and get it to
   `Accepted` before building. See [`specs/README.md`](specs/README.md).
2. **Branch off `main`:** `feature/<slug>`. There is **no `develop` branch** — `main`
   is the only long-lived branch (trunk-based, matching the upstream boilerplate).
3. **Build with tests** where it makes sense — Vitest/Playwright for the web app,
   `pytest` for the Python ingestion service.
4. **Run the gate locally before pushing:**

   ```bash
   # web app
   pnpm lint && pnpm typecheck && pnpm test && pnpm build
   # ingestion service
   ruff check && pytest
   ```

5. **Open a PR into `main`**, referencing its spec. CI must pass; reviewers check the
   diff against the spec's acceptance criteria.
6. **Cut a release** by bumping the version, setting the shipped spec(s) to `Shipped`,
   updating `CHANGELOG.md`, and pushing a `vX.Y.Z` tag on `main`. The tag defines the
   release. Pre-1.0 until spec 0006 ships.

## Branch & release model (trunk-based)

```
main ───●───●───────●───────●────●──▶     (only long-lived branch)
         \         /         \    \
          feature/x          feature/y   → PR back into main
                                    │
                              tag vX.Y.Z  → release
```

Not classic Gitflow: no `develop`, no `release/*` or `hotfix/*` branches. A hotfix is
just a short `feature/`-style branch off `main` and a new patch tag.

## Commit messages

[Conventional Commits](https://www.conventionalcommits.org). Keep body lines
≤ 100 characters. Examples:

```
feat(ingestion): extract tables as first-class chunks
fix(retrieval): filter vectors by document scope
docs: add multimodal embedding rationale to spec 0003
chore(deps): bump next to 16.4
```

## Code style

- TypeScript strict mode; no `any` without justification. Prettier + ESLint are the
  source of truth.
- Python formatted/linted with `ruff`.
- Provider access goes through the `LlmProvider` / `EmbeddingProvider` interfaces —
  never import a vendor SDK directly in callers.
- Keep untrusted document parsing inside the isolated ingestion service.

## Attribution

Do **not** add AI/assistant trailers (`Co-Authored-By: Claude`, "Generated with…",
etc.) to commits or PRs. Commits are authored solely by the contributor.
