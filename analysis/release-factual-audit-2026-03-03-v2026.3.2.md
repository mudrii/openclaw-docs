# Release Factual Integrity Audit and Execution Plan (v2026.3.2)

## Objective

Keep `openclaw-docs` aligned to the latest stable OpenClaw tag `v2026.3.2` with reproducible evidence from source and release metadata only.

## Scope

- Source of truth (code): `~/src/openclaw` (release tags only)
- Source of truth (release metadata): GitHub release `openclaw/openclaw` tag `v2026.3.2`
- Target docs: `~/src/openclaw-docs`
- Priority: update release snapshot markers, release summaries, and release-verified audit artifacts.

## Detailed Plan

1. Pull latest repo state in both trees.
2. Validate the `v2026.3.2` release metadata online and capture publish date.
3. Calculate release-window code deltas and repo-wide counts.
4. Derive tag-scoped source metrics for `v2026.3.2`.
5. Align top-level docs metadata (README, CHANGELOG, ARCHITECTURE, AGENT_README) and analysis headers.
6. Document executed corrections in a release audit artifact.
7. Commit, push, and publish a matching tag/release in `openclaw-docs`.

## Execution Evidence

- `git pull --rebase --autostash` completed cleanly for both repos.
- Release metadata (GitHub API): `v2026.3.2`, published `2026-03-03T04:43:07Z`, URL `https://github.com/openclaw/openclaw/releases/tag/v2026.3.2`.
- Release window stats: `git rev-list --count v2026.3.1..v2026.3.2` -> `862` commits.
- Release diff stats: `git diff --shortstat v2026.3.1 v2026.3.2` -> `2119 files changed, +121,000 / -55,228 lines`.
- Top-level changed-file distribution from tag diff:
  - `src: 1371`, `extensions: 326`, `apps: 201`, `docs: 98`, `ui: 41`, `scripts: 35`, `test: 9`.
- Tag-scoped source metrics for `v2026.3.2` (`src/extensions/ui/vendor/test/scripts`):
  - TypeScript files: `5,414`
  - Total TypeScript lines: `997,106`
  - Extensions directories: `40`
  - Extensions `package.json`: `33`

## Applied Documentation Changes

- `README.md`
  - Updated current release to `v2026.3.2`, release notes pointer, versioning section, and TS coverage counts.
- `CHANGELOG.md`
  - Added `OpenClaw v2026.3.2` as the latest documented release with highlights, distribution, behavior shifts, and upgrade checklist.
  - Shifted `v2026.3.1` to historical context.
- `ARCHITECTURE.md`
  - Updated architecture snapshot header and current package reference from `2026.3.1` to `2026.3.2`.
- `AGENT_README.md`
  - Marked legacy module additions as historical and added a v2026.3.2 behavioral note.
- `analysis/*.md`
  - Updated release headers from `2026.03-02 / v2026.3.1` to `2026-03-03 / v2026.3.2` where used as current snapshot markers.
- `analysis/release-factual-audit-2026-03-03-v2026.3.2.md`
  - Added this execution plan and evidence artifact.

## Residual Validation

- Historical release summaries remain intact for `v2026.3.1` and earlier for context and do not represent the active current snapshot.
- No file-level content diff against each changed file in the 862-commit release window is yet in scope for this pass; this pass focuses on documentation consistency and release-factual metadata synchronization.
