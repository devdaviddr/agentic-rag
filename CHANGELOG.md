# Changelog

All notable changes to this project are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Releases map to the folders under [`specs/`](specs/): each `vX.Y.Z` ships the
corresponding release's `spec.md` + `plan.md`. No code has been released yet — the
project is at the planning stage.

## [Unreleased]

### Added

- Product requirements ([`docs/PRD.md`](docs/PRD.md)) and architecture
  ([`docs/architecture.md`](docs/architecture.md)).
- Specifications and implementation plans for all six planned releases
  (v0.1.0–v1.0.0) under [`specs/`](specs/).
- Spec-driven development process ([`specs/README.md`](specs/README.md)) and release
  templates ([`specs/_template/`](specs/_template/)).
- Contributor workflow and Gitflow branching model
  ([`CONTRIBUTING.md`](CONTRIBUTING.md)).
- Assistant guidance ([`CLAUDE.md`](CLAUDE.md)).
- Roadmap ([`docs/roadmap.md`](docs/roadmap.md)) and this changelog.

### Changed

- Adopted classic Gitflow (main + develop + feature/release/hotfix) with feature
  branching, and made every spec ship with a companion implementation plan.
- Reorganised `specs/` into one folder per release (semantic versioning), each holding
  `spec.md` and `plan.md`.

### Decided

Resolved all five initial open questions (recorded in each release's `spec.md`; see the
[roadmap](docs/roadmap.md#key-decisions-open-questions)):

- **OQ1 — PDF parser:** Docling (default), PyMuPDF fallback, behind a swappable interface.
- **OQ2 — Embeddings:** NVIDIA NIM (free tier) — NV-CLIP (crops) + `llama-3.2-nv-embedqa`
  (text), both 1024-dim, behind `EmbeddingProvider`.
- **OQ3 — Hybrid search:** Postgres full-text search + a free NIM reranker over merged
  candidates; in-database, no dedicated engine.
- **OQ4 — Citation UX:** full-page render with bbox highlight (primary), crop as fallback.
- **OQ5 — Corpus isolation:** per-user ownership + admin override, enforced at the query
  layer.

<!--
Release sections are added as each vX.Y.Z ships, e.g.:

## [0.1.0] - YYYY-MM-DD
### Added
- Project foundation: ...
-->
