# Release Factual Integrity Audit and Execution Plan (v2026.2.26-3)

## Objective

Validate that `openclaw-docs` remains factually aligned with released OpenClaw artifacts, focusing on release-sensitive claims, and publish an updated docs release after corrections.

## Scope

- Source code baseline: `~/src/openclaw` (release tags only)
- Docs baseline: `~/src/openclaw-docs`
- Online sources of truth:
  - GitHub releases (`openclaw/openclaw`)
  - npm registry package metadata (`openclaw`)

## Detailed Execution Plan

1. Sync repos and confirm clean baselines.
2. Build authoritative release baseline from tags + online release metadata.
3. Extract release-sensitive claims from `openclaw-docs` (version headers, changelog windows, stats, release policy lines).
4. Verify those claims against released OpenClaw tags (`v2026.2.23`..`v2026.2.26`) and runtime package metadata.
5. Patch factual drift only.
6. Re-scan docs for residual stale release-state statements.
7. Commit, push, and publish a new `openclaw-docs` release.

## Evidence Collected

### Repository Sync

- `git -C ~/src/openclaw pull --rebase --stat` => already up to date
- `git -C ~/src/openclaw-docs pull --rebase --stat` => already up to date

### Online Release Baseline

- `npm view openclaw version --userconfig "$(mktemp)"` => `2026.2.26`
- `gh api repos/openclaw/openclaw/releases/latest` =>
  - `tag_name`: `v2026.2.26`
  - `published_at`: `2026-02-27T00:01:43Z`
- `gh release list -R openclaw/openclaw --limit 8` confirms `v2026.2.26` as latest stable release.

### Release Window Verification (source tags)

- `v2026.2.25..v2026.2.26`: `340` commits, `856 files changed`, `+58881/-7073`
- `v2026.2.24..v2026.2.25`: `159` commits, `376 files changed`, `+15083/-5552`
- `v2026.2.23..v2026.2.24`: `228` commits, `458 files changed`, `+20743/-4989`

### Snapshot Metrics Verification (`v2026.2.26`)

- TypeScript files (`src|extensions|ui|vendor|test|scripts`, `.ts/.tsx`): `4875`
- TypeScript lines (same scope): `887472`
- `extensions/*` directories: `39`
- `extensions/*/package.json`: `32`
- Bundled skills (`skills/**/SKILL.md`): `52`

## Findings

1. Top-level release summaries/stats in `README.md` and `CHANGELOG.md` matched verified source-tag data.
2. One factual drift remained in `AGENTS.md`: plugin post-check guidance still stated current stable release line as `2026.2.25`.
3. The release-state correction was straightforward and isolated.

## Corrections Applied

- `AGENTS.md`
  - Updated plugin post-check stable line from `2026.2.25` to `2026.2.26`.

## Integrity Status

- Release-sensitive docs claims checked in this pass are aligned with released OpenClaw state as of 2026-02-27.
- No mismatches were found in verified release windows/stats.
- One stale stable-version literal was corrected.

## Release Step

- Planned release tag: `v2026.2.26-3`
- Purpose: publish factual consistency correction after full release-focused verification run.
