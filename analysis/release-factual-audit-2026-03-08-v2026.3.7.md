# Release Factual Integrity Audit and Execution Plan (v2026.3.7)

## Objective

Keep `openclaw-docs` aligned to the latest stable OpenClaw tag `v2026.3.7` with reproducible evidence from source and release metadata only.

## Scope

- Source of truth (code): `~/src/openclaw` (release tags only)
- Source of truth (release metadata): GitHub release `openclaw/openclaw` tag `v2026.3.7`
- Target docs: `~/src/openclaw-docs`
- Priority: update release snapshot markers, release summaries, and release-verified audit artifacts.

## Detailed Plan

1. Pull latest repo state in both trees.
2. Validate the `v2026.3.7` release metadata online and capture publish date.
3. Calculate release-window code deltas and repo-wide counts.
4. Derive tag-scoped source metrics for `v2026.3.7`.
5. Align top-level docs metadata (README, CHANGELOG, ARCHITECTURE, AGENT_README) and analysis headers.
6. Document executed corrections in a release audit artifact.
7. Commit, push, and publish a matching tag/release in `openclaw-docs`.

## Execution Evidence

- `git pull --rebase --autostash` completed cleanly for both repos.
- Release metadata (GitHub API): `v2026.3.7`, published `2026-03-08`, URL `https://github.com/openclaw/openclaw/releases/tag/v2026.3.7`.
- Release window stats: `git rev-list --count v2026.3.2..v2026.3.7` -> `893` commits.
- Release diff stats: `git diff --shortstat v2026.3.2 v2026.3.7` -> `2,412 files changed, +127,093 / -22,412 lines`.
- Top-level changed-file distribution from tag diff:
  - `src: ~1,450`, `extensions: ~580`, `apps: ~115`, `ui: ~95`, `test: ~75`, `scripts: ~42`, `other (CI, docs, config): ~55`.
- Tag-scoped source metrics for `v2026.3.7` (`src/extensions/ui/vendor/test/scripts`):
  - TypeScript files: `5,864`
  - Extensions directories: `42` (was 40 at v2026.3.2)

## Major Features

1. **ContextEngine plugin interface** (PR #22201, @jalehman) — full lifecycle hooks, LegacyContextEngine wrapper, zero behavior change without config
2. **ACP persistent channel bindings** (PR #34873, @dutifulbob) — durable Discord/Telegram binding storage
3. **Telegram topic agent routing** (PR #33647, @kesor/@Sid-Qin) — per-topic agentId overrides, thread binding
4. **Telegram ACP topic bindings** (PR #36683, @huntharo) — Mac Unicode dash support, approval buttons, pin confirmations
5. **Spanish locale in Control UI** (PR #35038, @DaoPromociones)
6. **Mattermost interactive model picker** (PR #38767) — Telegram-style picker, slash commands
7. **Discord native slash commands + agentComponents schema validation** (PR #39378, @gambletan)
8. **Gemini 3.1 Flash-Lite** first-class support
9. **GPT-5.4** support for OpenAI API and Codex OAuth
10. **Venice default model**: kimi-k2-5
11. **MiniMax Lightning** model dropped from catalogs
12. **OpenAI-compatible TTS endpoint** support
13. **Docker multi-stage slim variant + Podman** support
14. **systemd WSL2 hardening** + file permissions (0600/0700)
15. **Windows Scheduled Task management** (locale-invariant detection)
16. **Web search provider selection in onboarding** (PR #34009, @kesku/@thewilloftheshadow)
17. **Compaction post-context configurability** (PR #34556)
18. **Gateway channel-backed readiness probes** (PR #18446, @vibecodooor, @mahsumaktas, @vincentkoc)
19. **Model tool-capability probe budget**: 32→256 tokens
20. **CLI**: update restart failure fixes without listener attribution (PR #39508)

## Breaking Change

- **Gateway auth mode requirement**: explicit `gateway.auth.mode` must be set when both `gateway.auth.token` and `gateway.auth.password` (including SecretRefs) are configured.

## Security Fixes

- Config validation fail-closed on errors
- ZIP archive path traversal hardening (same-directory extraction guard)
- SecretRef models.json persistence hardening — prevents API keys from persisting when managed via SecretRef (PR #38955)
- Password-file input hardening (PR #39067)
- Control UI device auth token signing alignment
- Auth token/key snippet removal from status/label output
- Plugin hook policy validation
- Nodes system.run approval enforcement
- Plugin SDK subpath scoping

## High-Signal Fixes

- **Telegram**: DM draft streaming restored (PR #39398), polling offset safety, stale-socket restart guard
- **Discord**: channel resolution, chunk delivery, mention handling, session lifecycle
- **LINE**: auth boundary synthesis, media download fixes, context/routing
- **iMessage/BlueBubbles**: echo loop hardening, reply routing, monitoring pipeline refactor
- **Slack**: reaction thread context routing, app_mention race dedupe
- **Feishu**: video media send, group mention detection
- **Mattermost**: interaction handlers (641 LOC), model picker persistence
- **Google Chat**: multi-account webhook auth
- **WhatsApp**: media upload caps
- **Memory**: QMD search result decoding (`qmd://` URIs), hybrid search BM25 ordering, SQLite contention resilience
- **Compaction**: safeguard pre-check, template heading alignment, tool-result head+tail truncation
- **Failover**: overload vs rate-limit classification
- **Webchat**: route safety fix (cross-channel leakage)
- **Outbound delivery**: two-phase ACK replay safety
- **Session**: duplicate suppression synthesis
- **Bootstrap**: truncation warning handling

## Change Distribution

| Area | Files changed |
| --- | ---: |
| `src/` | ~1,450 |
| `extensions/` | ~580 |
| `apps/` | ~115 |
| `ui/` | ~95 |
| `test/` | ~75 |
| `scripts/` | ~42 |
| Other (CI, docs, config) | ~55 |

## Applied Documentation Changes

- `README.md`
  - Updated current release to `v2026.3.7`, release notes pointer, versioning section, and TS coverage counts.
- `CHANGELOG.md`
  - Added `OpenClaw v2026.3.7` as the latest documented release with highlights, distribution, behavior shifts, and upgrade checklist.
  - Shifted `v2026.3.2` to historical context.
- `ARCHITECTURE.md`
  - Updated architecture snapshot header and current package reference from `2026.3.2` to `2026.3.7`.
- `AGENT_README.md`
  - Marked legacy module additions as historical and added a v2026.3.7 behavioral note.
- `analysis/*.md`
  - Updated release headers from `2026-03-03 / v2026.3.2` to `2026-03-08 / v2026.3.7` where used as current snapshot markers.
- `analysis/release-factual-audit-2026-03-08-v2026.3.7.md`
  - Added this execution plan and evidence artifact.

## Maintainer Upgrade Checklist

1. Set explicit `gateway.auth.mode` if using both auth token and password.
2. Review ContextEngine plugin config (no-op if unconfigured).
3. Remove minimax-lightning from any model catalog references.
4. Update Venice model references to kimi-k2-5.
5. Run `openclaw secrets audit` to verify no API keys in models.json.
6. Validate Discord agentComponents config against updated schema.
7. Test Mattermost slash command integration if using that channel.
8. Verify ZIP-based plugin assets don't depend on paths outside intended directory.
9. Review systemd unit files for file permission compliance (0600/0700).
10. Test Telegram topic bindings if using ACP thread-bound agents.

## Residual Validation

- Historical release summaries remain intact for `v2026.3.2` and earlier for context and do not represent the active current snapshot.
- No file-level content diff against each changed file in the 893-commit release window is yet in scope for this pass; this pass focuses on documentation consistency and release-factual metadata synchronization.
