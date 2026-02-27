# Release Factual Integrity Audit and Execution Plan (v2026.2.26-4, multi-agent)

## Objective

Perform a released-version-only factual audit of `openclaw-docs` against `openclaw` source and release metadata, then implement verified documentation corrections and publish a docs release.

## Scope

- Source of truth (code): `~/src/openclaw` (stable tags only)
- Source of truth (release metadata):
  - GitHub releases (`openclaw/openclaw`)
  - npm package metadata (`openclaw`)
- Docs target: `~/src/openclaw-docs`

## Detailed Plan

1. Pull latest changes in both repos.
2. Build release baseline online (latest stable tag + publish date + npm latest).
3. Run parallel subagent audits by concern area.
4. Consolidate only high-confidence mismatches with exact file references.
5. Patch factual drift in docs.
6. Re-run docs consistency checks.
7. Commit, push, and publish a new `openclaw-docs` release.

## Parallel Subagent Sessions

- `Dalton` (`019c9f04-c3bb-7962-9524-fecaf6e67142`): release/date/version consistency audit.
- `Nietzsche` (`019c9f04-c3fc-7163-9a52-873d9bd6c4a7`): command/path validity audit.
- `Volta` (`019c9f04-c47f-7873-9c26-cf5b0ca15be1`): metrics/counts/stats audit vs `v2026.2.26`.
- `Boyle` (`019c9f04-c504-7d23-b04e-795fcfacaa07`): internal consistency / stale release wording audit.

## Release Baseline (Online)

- `gh api repos/openclaw/openclaw/releases/latest` => latest stable `v2026.2.26`, published `2026-02-27T00:01:43Z`.
- `npm view openclaw version --userconfig "$(mktemp)"` => `2026.2.26`.

## High-Confidence Findings Applied

1. **AGENTS command mismatch**
   - `pnpm format` was described as check mode but writes changes.
   - Fixed to `pnpm format:check`.

2. **Maintainer workflow source path mismatch**
   - Local path `.agents/skills/PR_WORKFLOW.md` was not valid in this repo checkout.
   - Fixed to canonical maintainers URL.

3. **CLI command reference drift**
   - `openclaw gateway` table entry listed `dev` as subcommand; clarified as `--dev` option on gateway run flow.
   - `openclaw nodes` table entry listed `pair`; updated to current pairing flow commands (`pending/approve/reject`).

4. **Stale analysis summary count**
   - `analysis/cli-config-infra.md` summary updated from `952` to `1,216` TypeScript files across analyzed modules.

5. **ARCHITECTURE cross-module stats drift**
   - Updated stale overall metrics and top-10 module source/test file counts to current `v2026.2.26` values.

6. **Forward-looking release wording in release-only docs**
   - Replaced `Ships in next npm release` with explicit released-tag wording (`Released in v2026.2.24`) in:
     - `analysis/cli-tools-media.md`
     - `analysis/security-web-browser.md`
     - `analysis/security-plugins.md`

7. **Supersession marker for historical audit snapshot**
   - Added explicit superseded-note header in `analysis/release-factual-audit-2026-02-27.md` to prevent contradiction with newer v2026.2.26 audit artifacts.

## Integrity Status

- Released-version-only baseline and metadata are aligned with current stable (`v2026.2.26` / `2026.2.26`).
- Applied corrections remove high-confidence factual drift identified in this pass.
- Residual low-priority editorial consistency items (for example section ordering style) can be handled in a future cleanup pass.
