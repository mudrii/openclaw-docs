# Docs Correction Release Plan: v2026.5.7 to v2026.5.7-1

## Scope

- Docs repo: `/Users/mudrii/src/open_claw/openclaw-docs`
- Source repo: `/Users/mudrii/src/open_claw/openclaw`
- Baseline docs release: `v2026.5.7`
- Target docs correction release: `v2026.5.7-1`
- Upstream OpenClaw baseline remains `v2026.5.7`
- No source behavior changes are introduced.

## Assumptions

- `~/src/open_claw/opeonclaw` means `~/src/open_claw/openclaw`; no separate `opeonclaw` checkout exists.
- The docs repository documents stable releases only. Unreleased `main` and beta releases remain out of scope except for explicit exclusion notes.
- A docs-only correction should use a suffixed tag (`v2026.5.7-1`) instead of retagging `v2026.5.7` or inventing an upstream version.

## Parallel Review Split

1. Release and changelog validation
   - Verify newest stable upstream tag.
   - Compare official GitHub release metadata with `README.md` and `CHANGELOG.md`.
   - Identify stale prerelease exclusion notes and release timestamp drift.

2. Docs inventory validation
   - Compare current docs inventory against `openclaw@v2026.5.7`.
   - Count stable TypeScript files, extension directories, extension packages, and released skill entrypoints.
   - Identify missing mirror pages and broken links as residual work.

3. Stable source surface validation
   - Review CLI, channel, provider, plugin, gateway, and config docs against stable `v2026.5.7`.
   - Exclude beta-only and unreleased `main` behavior from stable docs changes.

4. Release integrity validation
   - Confirm `openclaw-docs` has `v2026.5.7` but no `v2026.5.7-1` tag.
   - Define docs-only validation, commit, tag, push, and release commands.

## Evidence

- `gh release view v2026.5.7 --repo openclaw/openclaw` reports published `2026-05-07T20:57:43Z`.
- `gh release view v2026.5.5 --repo openclaw/openclaw` reports published `2026-05-06T09:00:55Z`.
- Newer upstream releases are prereleases only through `v2026.5.10-beta.3`.
- `openclaw-docs` has `v2026.5.7`, but no `v2026.5.7-1` tag or release.
- `git rev-list --count v2026.5.5..v2026.5.7` returns `93`.
- `git diff --shortstat v2026.5.5..v2026.5.7` returns `372 files changed, 9668 insertions(+), 1163 deletions(-)`.
- `v2026.5.7` has 14,201 scoped TypeScript files, 125 top-level `extensions/` directories, 119 extension package manifests, and 87 released `SKILL.md` entrypoints.

## Implemented Corrections

1. Updated current docs snapshot metadata to `v2026.5.7-1`.
2. Corrected the `v2026.5.5` GitHub release published timestamp to `2026-05-06 09:00:55 UTC`.
3. Added a top changelog correction section for `v2026.5.7-1`.
4. Refreshed prerelease exclusion notes through `v2026.5.10-beta.3`.
5. Corrected stable inventory counts that previously reflected unreleased branch state.
6. Corrected bounded stable-doc mismatches for Slack `SLACK_USER_TOKEN`, SDK export examples, `openclaw channels` subcommands, `config patch`, and Gateway `usage-cost`/`stability`/`diagnostics export` subcommands.

## Residual Findings

- `openclaw-docs/docs` is a partial docs mirror, not a full byte-for-byte copy of `openclaw@v2026.5.7:docs`; the source tag contains hundreds more docs pages.
- `docs/docs.json` advertises many routes whose local Markdown pages are not present in this repository.
- Many internal links resolve only when the full docs source/publish pipeline exists. A full portal mirror sync should be a separate release because it would add or update hundreds of files.
- Several analysis files still carry `v2026.5.5` headers. Updating those responsibly requires a broader analysis-doc refresh rather than metadata-only correction.

## Validation Plan

```bash
cd /Users/mudrii/src/open_claw/openclaw-docs
git status --short --branch
git diff --check
node -e 'JSON.parse(require("fs").readFileSync("docs/docs.json", "utf8")); console.log("docs/docs.json OK")'
rg -n "2026-05-06 08:12:30|Current validated docs snapshot: v2026\\.5\\.7 \\(|Current docs version: v2026\\.5\\.7 \\(|132 extension directories|211 released skill|211 bundled skill|userToken.*config-only|plugin-sdk/feishu|plugin-sdk/zalo" README.md AGENT_README.md ARCHITECTURE.md docs/channels/slack.md docs/plugins/architecture.md
```

## Release Commands

```bash
cd /Users/mudrii/src/open_claw/openclaw-docs
git add README.md CHANGELOG.md AGENT_README.md ARCHITECTURE.md docs/channels/slack.md docs/cli/channels.md docs/cli/index.md docs/plugins/architecture.md docs/superpowers/plans/2026-05-11-docs-correction-v2026.5.7-1.md
git commit -m "docs: publish v2026.5.7-1 correction metadata"
git tag -a v2026.5.7-1 -m "docs correction: v2026.5.7-1 metadata fix for upstream v2026.5.7"
git push origin main
git push origin v2026.5.7-1
gh release create v2026.5.7-1 --repo mudrii/openclaw-docs --latest --title "OpenClaw Docs v2026.5.7-1" --notes-file /tmp/openclaw-docs-v2026.5.7-1-notes.md
```
