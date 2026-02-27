# Release Factual Integrity Audit and Execution Plan (2026-02-27)

> Historical snapshot: this report captured pre-`v2026.2.26` synthesis state and is superseded by `analysis/release-factual-audit-2026-02-27-v2026.2.26.md` and `analysis/release-factual-audit-2026-02-27-v2026.2.26-3.md`.

## Objective

Validate that `openclaw-docs` release-facing documentation remains factually correct against released OpenClaw source tags, with explicit online verification of latest published releases.

## Scope

- Source of truth (code): `~/src/openclaw`, released tags (stable only)
- Source of truth (release metadata): GitHub releases for `openclaw/openclaw`
- Documents reviewed in this pass:
  - `README.md`
  - `CHANGELOG.md`
  - `ARCHITECTURE.md`
  - `DEVELOPER-REFERENCE.md`
  - `AGENTS.md`
  - `analysis/agent-system.md`
  - `analysis/channels.md`
  - `analysis/cli-config-infra.md`
  - `analysis/core-engine.md`
  - `analysis/security-plugins.md`
  - `analysis/security-web-browser.md`

## Detailed Plan

1. Sync repositories and release refs
2. Build release baseline from local tags + upstream release metadata
3. Extract release-sensitive and high-risk factual claims from docs
4. Verify each claim against released source tags (paths, symbols, file inventories, counts, dates)
5. Validate publish dates and latest stable release online
6. Patch incorrect or stale statements in docs
7. Re-check diffs for consistency and release-scope language
8. Commit, push, and publish a new `openclaw-docs` release

## Execution Evidence

### 1) Repository Sync

- `~/src/openclaw` pull status: already up to date (`git pull --rebase --autostash`)
- `~/src/openclaw-docs` pull status from `origin main`: already up to date
- Upstream tags refreshed for source validation:
  - `git -C ~/src/openclaw fetch upstream --tags`
  - New tags discovered locally: `v2026.2.26`, `v2026.2.26-beta.1`

### 2) Release Baseline (source + online)

- Source tag dates (git):
  - `v2026.2.15`: 2026-02-16
  - `v2026.2.22`: 2026-02-22 (tag commit date)
  - `v2026.2.23`: 2026-02-24
  - `v2026.2.24`: 2026-02-25
  - `v2026.2.25`: 2026-02-26
- Online release publish date check (GitHub release metadata):
  - `v2026.2.22` published at `2026-02-23T04:09:40Z`
  - Latest stable release is `v2026.2.26`, published `2026-02-27T00:01:43Z`

### 3) High-Signal Claim Verification Results

- `v2026.2.14..v2026.2.15`: `705` commits, `1492 files changed`, `+62497/-45534`
- `v2026.2.17..v2026.2.19`: `572` commits
- `v2026.2.23..v2026.2.24`: `228` commits
- `v2026.2.25` snapshot stats:
  - TypeScript files (`src|extensions|ui|vendor|test|scripts`, `.ts/.tsx`): `4679`
  - TypeScript lines (same scope): `842204`
  - `extensions/*` directories: `38`
  - `extensions/*/package.json`: `31`
- Code/symbol/path checks validated:
  - `fixSecurityFootguns` exists; `runSecurityFix` absent
  - `src/auto-reply/reply/{dispatcher-registry.ts,reply-dispatcher.ts,agent-runner-execution.ts}` paths valid
  - `src/channels/plugins/outbound/direct-text-media.ts` exists
  - `src/gateway/http-auth-helpers.ts` exists
  - `src/auto-reply/reply/commands-session.ts` uses `/session` (no `/sessions`)
  - `src/agents/subagent-spawn.ts` is `535` lines in `v2026.2.25`
  - Plugin install/update APIs now exported as `installPluginFrom*`, `uninstallPlugin`, `updateNpmInstalledPlugins`
  - Config key family uses `plugins.entries.*`
  - Module inventory checks aligned (examples):
    - `src/providers`: `6` source, `5` tests
    - `src/sessions`: `7` source, `1` test
    - `src/routing`: `5` source, `5` tests
    - `src/hooks`: `23` source, `15` tests
    - `src/agents`: `342` source, `318` tests (`660` total `.ts`)
    - `src/agents/tools`: `46` source files
    - `src/gateway`: `184` source, `101` tests (`285` total `.ts`)
    - `src/security`: `19` source, `9` tests (`28` total `.ts`)
    - `src/plugin-sdk`: `22` source, `8` tests (`30` total `.ts`)
    - `src/acp`: `11` source, `6` tests (`17` total `.ts`)

## Findings

1. Most pending doc edits in this branch were factual corrections and validated against release tags.
2. A release-state drift existed: docs were framing `v2026.2.25` as "current stable" while upstream published `v2026.2.26` on 2026-02-27 UTC.
3. This pass corrected wording so docs are explicit about:
   - Latest upstream stable release (`v2026.2.26`)
   - Latest fully validated/synthesized snapshot in this repo (`v2026.2.25`)

## Corrections Applied in This Pass

- `README.md`
  - Reframed release status to distinguish latest upstream stable vs current validated docs snapshot.
  - Updated changelog description wording to avoid implying full coverage through latest upstream release.
  - Added explicit upstream-latest note in versioning section.
- `CHANGELOG.md`
  - Renamed top heading to "Latest Documented Release Summary".
  - Added explicit upstream latest stable tag/publish timestamp note.
- `ARCHITECTURE.md`
  - Clarified metrics are from analyzed `v2026.2.25` snapshot.
  - Added upstream-latest note (`v2026.2.26` published; synthesis pending).

## Integrity Status

- `openclaw-docs` is now explicit about documentation coverage boundaries and latest upstream release state.
- Release-focused factual corrections in this branch are validated against released source tags and online GitHub release metadata.
- Remaining gap: full synthesis content for `v2026.2.26` has not been produced in this pass.
