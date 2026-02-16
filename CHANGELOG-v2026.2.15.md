# Changelog: v2026.2.14 → v2026.2.15

> **Released:** 2026-02-15 | **Commits:** 880 | **Files changed:** 1,526 | **Lines changed:** ~113,000

---

## Security Fixes (12)

| Change | Impact |
|--------|--------|
| Harden sandbox Docker config validation | Prevents config injection in sandbox setup |
| Harden prompt path sanitization | Prevents directory traversal in prompt file paths |
| Restrict skill download target paths (`infra/install-safe-path.ts`) | Prevents writes outside allowed directories |
| Scope session tools and webhook secret fallback | Prevents cross-session tool/secret leakage |
| Preserve control-UI scopes in bypass mode | Scope enforcement even when bypass is active |
| Scope pairing stores by account (`pairing/pairing-store.ts`) | Prevents cross-account pairing data access |
| Account-scoped pairing allowlists (Telegram/WhatsApp) | Allowlists isolated per account |
| Redact Telegram bot tokens in errors | Prevents token leakage in error messages |
| Redact sensitive status details for non-admin scopes | Status API respects scope restrictions |
| Harden `chat.send` message input sanitization | Prevents injection via message tool |
| LINE webhook fail-closed when auth missing | Rejects unauthenticated LINE webhooks instead of passing through |
| Control UI XSS fix (JSON endpoint + CSP lockdown) | Prevents XSS via control UI |

## Features (11)

| Feature | Files | PR |
|---------|-------|----|
| Discord Component v2 UI tool support | `discord/components.ts`, `discord/components-registry.ts`, `discord/send.components.ts` | #17419 |
| Cross-platform skill install fallback for non-brew | `infra/install-safe-path.ts` | #17687 |
| Account selector for pairing commands (CLI) | `pairing/pairing-labels.ts`, CLI commands | — |
| Cron finished-run webhook | `cron/` service | #14535 |
| Plugin LLM input/output hook payloads | `plugins/hooks.ts`, `plugins/types.ts` | #16724 |
| Multi-image support in image tool | `agents/tools/image-tool.ts`, `agents/tools/image-tool.helpers.ts` | #17512 |
| Per-channel `ackReaction` config | `config/types.telegram.ts`, `types.discord.ts`, `types.slack.ts`, `types.whatsapp.ts` | #17092 |
| Nested subagent orchestration controls | `agents/subagent-depth.ts`, `agents/subagent-announce-queue.ts`, `agents/tools/subagents-tool.ts` | #14447 |
| `messages.suppressToolErrors` config option | `config/types.messages.ts` | #16620 |
| Gateway: preserve partial output on abort | `gateway/` | #15026 |
| Model fallback support for `sessions_spawn` | `agents/` spawn logic | #17197 |

## Memory / QMD (6)

- Isolate managed collections per agent
- Rebind drifted managed collection paths
- Harden context window cache collisions
- Support unicode tokens in FTS query builder
- Inject runtime date-time into memory flush prompt
- Verify QMD index artifact after manual reindex

## Cron (5)

- Normalize skill-filter snapshots
- Pass agent-level skill filter to isolated cron sessions
- Treat missing `enabled` as `true` in `update()`
- Infer payload kind for model-only update patches
- Preserve cron prompts for tagged interval events

## Telegram (8)

- Simplify send/dispatch/target handling (major refactor)
- Stop block streaming splitting messages when `streamMode` off
- Treat no-op `editMessage` as success
- Stream replies in-place without duplicate final sends
- Stop dropping voice messages on `getFile` network errors
- Include voice transcript in body text
- Restore `thread_id=1` handling for DMs
- Legacy `allowFrom` migration to account-scoped format

## Performance (6)

- Massive test consolidation (100+ test refactors)
- Lazy-load heavy deps in models list, onboarding, plugins, telegram
- Speed up Slack/Gateway/Telegram test suites
- Avoid async cron timer callbacks
- Skip skill scans for inline directives
- Isolate browser profile hot-reload

## Refactoring (Major)

- **300+ refactor commits** deduplicating code across all modules
- Centralized helpers: SHA-256, env snapshots, JSON file locks, etc.
- Shared test harnesses across modules
- Total file count reduced from ~3,051 to ~2,980 despite new features

## Other Notable Changes

- Preserve OpenAI reasoning replay IDs
- Fix Codex/PTY process spawning (#14257)
- Discord role-based allowlist fix (#16369)
- Session key normalization fix
- Gateway: keep boot sessions ephemeral
- Honor configured `contextWindow` overrides
- Gateway `config.patch` merge object arrays by ID

## Documentation Updates

- Updated `DEVELOPER-REFERENCE.md`: new high-blast-radius files, v2026.2.15 gotchas, new config keys
- Updated `ARCHITECTURE.md`: security hardening section, updated stats, Discord Component v2, account-scoped pairing
- Updated `README.md`: version bump, updated file counts

## Breaking Changes

None — all changes are backward compatible. Legacy `allowFrom` in Telegram is auto-migrated.

---

*Generated: 2026-02-16*
