# Docs Correction Release Plan: v2026.5.7-1 to v2026.5.7-2

## Goal

Publish a docs-only correction release for `openclaw-docs` that remains validated against upstream stable `v2026.5.7` while restoring released documentation pages and correcting stale stable-docs details.

## Assumptions

- `~/src/open_claw/opeonclaw` means `~/src/open_claw/openclaw`.
- `openclaw-docs` publishes stable release documentation only; unreleased `main` changes and prereleases remain excluded from stable docs.
- `v2026.5.10-beta.4` exists as an upstream source tag, while GitHub releases currently list published prereleases through `v2026.5.10-beta.3`.
- The correct docs correction tag is `v2026.5.7-2`, because `v2026.5.7-1` already exists.

## Plan

1. Fetch and pull both local repositories.
   - Verify: `openclaw` fast-forwards to latest upstream `main`; `openclaw-docs` is clean and up to date with `origin/main`.
2. Confirm release boundaries from tags and GitHub release metadata.
   - Verify: latest stable upstream release is `v2026.5.7`, published `2026-05-07 20:57:43 UTC`; newer tags are prereleases/source tags only.
3. Audit released docs against the `v2026.5.7` source tree using parallel read-only agents.
   - Verify: findings identify missing or stale docs in channels, providers, plugins, CLI, reference, automation, concepts, gateway, tools, and web pages.
4. Restore missing and stale released docs from the `v2026.5.7` source tree.
   - Verify: copied docs come from `git archive v2026.5.7`, not from unreleased `main`.
5. Update docs-release metadata.
   - Verify: README, CHANGELOG, AGENT_README, and ARCHITECTURE identify `v2026.5.7-2` as the current docs snapshot, still validated against upstream `v2026.5.7`.
6. Run docs sanity checks.
   - Verify: `git diff --check`; docs inventory/path sanity; targeted searches for previously missing pages and stale facts.
7. Commit, tag, push, and create a GitHub release in `mudrii/openclaw-docs`.
   - Verify: commit and tag are pushed; GitHub release exists and is marked latest.

## Implemented Changes

1. Synced released `v2026.5.7` docs directories into the publish repo:
   - `docs/channels/`
   - `docs/providers/`
   - `docs/plugins/`
   - `docs/cli/`
   - `docs/reference/`
   - `docs/automation/`
   - `docs/concepts/`
   - `docs/gateway/`
   - `docs/tools/`
   - `docs/web/`
   - `docs/docs.json`
2. Added the `v2026.5.7-2` correction section to `CHANGELOG.md`.
3. Updated current snapshot metadata in `README.md`, `AGENT_README.md`, and `ARCHITECTURE.md`.
4. Refreshed the latest stable-docs prerelease exclusion note through source tag `v2026.5.10-beta.4`.

## Scope Guard

No upstream stable behavior changed. This release restores and corrects documentation for released `v2026.5.7` surfaces only.
