# OpenClaw Changelog

Consolidated changelog assembled from versioned changelog files in this repository.

---

## OpenClaw v2026.2.24 — Current Released Summary

> **Released:** 2026-02-25 | **Policy note:** latest released section stays at top.

## Highlights

- Auto-reply stop controls significantly expanded: multilingual standalone stop phrases, punctuation-tolerant matching, and exact `do not do that` stop trigger support.
- Heartbeat safety hardened: default delivery target shifted to `none`, direct/DM heartbeat targets are now blocked, and blocked-delivery prompts are internalized.
- Cross-channel shared-session routing now fails closed and preserves route metadata for followup/overflow delivery to prevent channel hijacking.
- Security hardening wave across sandbox, exec, workspace FS, and ingress authorization boundaries (including container namespace-join default block for sandbox).
- Broad reliability fixes landed across Discord voice/DAVE, typing keepalive, WhatsApp reconnect behavior, model fallback traversal, and allowlisted-model selection.

For full detail, see the v2026.2.24 synthesis section below.

---

# Synthesis Review: v2026.2.23 → v2026.2.24 (Released)

> **Window analyzed:** `v2026.2.23..v2026.2.24`
> **Release date:** 2026-02-25
> **Scan stats:** 228 commits, 458 files changed, +20,743 / -4,989 lines

## Change Distribution (By Top-Level Area)

| Area | Files changed |
| --- | ---: |
| `src/` | 247 |
| `apps/` | 86 |
| `extensions/` | 66 |
| `docs/` | 28 |
| `ui/` | 12 |
| `scripts/` | 5 |
| `test/` | 4 |

## Breaking / Behavior Shifts

1. Heartbeat delivery now blocks direct/DM targets; heartbeat still runs but only non-DM destinations can receive outbound delivery.
2. Sandbox Docker `network: "container:<id>"` namespace-join mode is blocked by default; break-glass enablement now requires explicit config (`dangerouslyAllowContainerNamespaceJoin: true`).
3. Heartbeat default target changed from `last` to `none`, making external heartbeat delivery explicit opt-in.

## High-Signal Runtime Fixes

- Shared-session routing hardened to fail closed for cross-channel replies and preserve originating channel metadata through followup/queue paths.
- Typing keepalive lifecycle improved across core and extension dispatch paths so indicators survive long inference runs.
- Discord voice path reliability improved (DAVE dependency/runtime handling, decrypt-failure tolerance and recovery logic, stale listener cleanup).
- Fallback traversal on model fallback chains fixed to continue through configured fallbacks when already on fallback models.
- Gateway model allowlist behavior fixed so explicit allowlisted refs remain selectable even when bundled catalog metadata is stale.

## Security Hardening Themes

- Exec approval/display binding tightened so shell-wrapper payloads cannot drift from approved command text.
- Workspace and sandbox path guards tightened (including `@` path normalization and tmp-media hardlink alias rejection).
- Ingress authorization hardened across channel-specific paths (for example Telegram DM media write authorization and Synology Chat DM fail-closed allowlists).
- Safe-bin trust defaults tightened to immutable system directories with stronger audit/doctor guidance for risky trusted-dir configurations.

## Maintainer Upgrade Checklist (v2026.2.24)

1. Verify heartbeat targets: if external delivery is expected, set explicit non-DM targets; default `none` and DM blocking are now enforced.
2. Audit sandbox Docker settings for namespace-join assumptions; enable break-glass key only where intentionally required.
3. Re-test shared-session multi-channel routes and followup behavior (especially Discord/Webchat mixed sessions).
4. Re-run security audit and review safe-bin trusted-dir findings after upgrade.
5. Validate channel-specific reliability paths you depend on (Discord voice, WhatsApp reconnect, typing keepalive).

---

# Changelog: v2026.2.14 -> v2026.2.15

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

---

# Synthesis Review: v2026.2.15 → v2026.2.17

> **Window analyzed:** `v2026.2.15..v2026.2.17` (979 commits)
> **Method:** prioritize behavior and operator/developer impact; ignore low-signal test churn

---

## Executive synthesis

This release window is best understood as a **runtime-hardening and orchestration-stability cycle** with four dominant themes:

1. **Subagent execution became more deterministic** (spawn/announce semantics, routing, completion reliability).
2. **Cron scheduling became safer and more explicit** (default stagger behavior + operator controls).
3. **Security boundaries tightened** (notably config `$include` confinement and sandbox/input hardening).
4. **Tool/runtime behavior shifted toward guarded streaming + loop control** (Z.AI tool-stream default, stronger loop detection, read-context guard behavior).

The most important takeaway for maintainers is not a new feature checklist; it is that **assumptions that used to be “soft defaults” are now enforced behaviors**.

---

## What changed in practice (high signal)

### 1) Subagents: from best-effort to managed flow

Subagent workflows now lean heavily into push-based lifecycle semantics:

- `/subagents spawn` adds deterministic command-path spawning.
- `sessions_spawn` user experience now explicitly sets expectation that completion is auto-announced.
- Completion routing/retry/output selection has been hardened.

**Operational implication:**
Do not design runbooks around aggressive polling of spawned work. Treat spawned runs as async jobs with eventual callback/announce.

### 2) Cron: schedules now carry intent, not just expression

Top-of-hour recurring jobs now receive deterministic stagger by default and persist it as `schedule.staggerMs`; operators can force exact timing via `--exact` / `staggerMs: 0`.

**Operational implication:**
If your mental model is “`0 * * * *` means exact wall-clock fire”, that is no longer universally true. Exactness must be explicit.

### 3) Security posture: config/file boundaries tightened

The most consequential hardening in this window is strict confinement of config `$include` paths to config-root boundaries (with traversal/symlink guards), plus additional sandbox/env sanitization work.

**Operational implication:**
Older include layouts that reached outside config root may fail after upgrade; this is expected security behavior, not parser regression.

### 4) Tool/runtime behavior: safer defaults, stricter failure handling

- Z.AI tool-call streaming defaults to enabled.
- Loop detection now escalates/no-progress blocks on repeated poll/log patterns.
- Read-context handling emphasizes bounded incremental reads over full-dump retries.

**Operational implication:**
Agent/tool orchestration prompts and automation scripts need explicit progress checks, bounded retries, and chunked-read recovery patterns.

---

## Recommended maintainer workflow updates

1. **Before blaming runtime bugs, validate config confinement constraints first** (`$include` path containment, symlink behavior).
2. **When debugging cron timing, inspect `schedule.staggerMs`**, not only cron expression.
3. **Treat subagent completion as announce-driven**, and only poll for intervention/debug.
4. **For tool loops, require no-progress detection + backoff in agent logic docs/tests.**
5. **For large file analysis flows, enforce paged reads (`offset`/`limit`) in prompts and examples.**

---

## Docs in this repo updated for this window

- `README.md` — version/index updates for v2026.2.17 docs.
- `DEVELOPER-REFERENCE.md` — concrete gotchas + workflow guidance for this release window.
- `CHANGELOG.md` — this synthesized review (consolidated section for v2026.2.15 → v2026.2.17).

---

*Generated: 2026-02-18*
---

# Synthesis Review: v2026.2.17 → v2026.2.19

> **Window analyzed:** `v2026.2.17..v2026.2.19` (~900+ commits)
> **Method:** behavior and operator/developer impact focus; ignore low-signal test/style churn
> **Package version confirmed:** `2026.2.19-2` (npm)

---

## Executive synthesis

This release window is a **security-surge + runtime-stability cycle** defined by four major themes:

1. **Security hardening across virtually every surface** — 40+ distinct security fixes spanning SSRF, plugin integrity, exec sandboxing, gateway auth, canvas session scoping, and protocol injection vectors. This is the most security-dense release in the observed history of these docs.
2. **Gateway auth semantics shifted to secure-by-default** — token auth is now the default; explicit `"none"` is required for intentional open loopback setups, and a new security audit fires when `no_auth` is detected with remote exposure risk.
3. **Cron/heartbeat delivery fixed in multiple important ways** — explicit Telegram topic targets now work, heartbeat skips when no content exists, cron announces route correctly, and spin-loop prevention is now solid.
4. **iOS/APNs platform matured** — full Apple Watch companion, APNs wake-before-invoke, share extension, and major pairing/auth stabilization. Low impact for this setup but shows platform direction.

The most important operational takeaway: **security boundaries that were advisory in 2026.2.17 are now enforced or audited at boot**. Review gateway auth, plugin integrity, hook token uniqueness, and exec safeBins settings before upgrading.

---

## What changed in 2026.2.19 for us

### 1) Gateway auth is now token-mode by default

Gateway auth now defaults to token mode with auto-generation and persistence of `gateway.auth.token`. Explicit `gateway.auth.mode: "none"` is required for intentional open loopback setups.

Additionally, the new security audit (`Security/Audit`) emits `gateway.http.no_auth` findings when `mode="none"` leaves gateway HTTP APIs reachable, with loopback-only → warning and remote-exposure → critical severity.

**Impact for this setup:** If the current config relies on implicit open auth, expect audit warnings after upgrade. Review `openclaw security audit` output post-upgrade.

### 2) `hooks.token` must differ from `gateway.auth.token` — startup failure if equal

New startup validation rejects configs where `hooks.token` matches `gateway.auth.token`. The gateway fails to start if these values are identical.

**Impact:** If these were previously set to the same value, the gateway will refuse to start after upgrade. Separate them before upgrading.

### 3) Heartbeat behavior: skip when HEARTBEAT.md missing or empty

Interval heartbeats are now skipped automatically when `HEARTBEAT.md` is missing or empty and no tagged cron events are queued. The cron-event fallback for queued tagged reminders is preserved.

**Impact for this setup:** This is the correct behavior — less noise, fewer empty heartbeat runs. No action needed unless you expected heartbeats to fire on empty HEARTBEAT.md.

### 4) Cron/heartbeat Telegram topic delivery now works

Explicit Telegram topic targets in cron and heartbeat delivery (`<chatId>:topic:<threadId>`) now correctly route scheduled sends into the configured topic instead of defaulting to the last active thread.

**Impact:** If heartbeat/cron sends were previously landing in the wrong Telegram thread, this is the fix. No config change required — existing `<chatId>:topic:<threadId>` targets now work correctly.

### 5) YAML frontmatter uses YAML 1.2 core schema — `on`/`off` are now strings

Frontmatter YAML parsing now uses YAML 1.2 core schema to avoid implicit coercion. Previously, `on`/`off`/`yes`/`no`/`true`/`false` in quoted or unquoted frontmatter could be auto-coerced to booleans.

**Impact:** Any AGENTS.md, HEARTBEAT.md, or cron prompt frontmatter that relied on `on`/`off` being booleans must now be explicit. Use `true`/`false` as YAML boolean literals.

### 6) Browser/Relay requires gateway-token auth on both `/extension` and `/cdp`

The Chrome extension relay endpoint and the CDP endpoint now require `gateway.auth.token` authentication. The Chrome extension setup aligns to a single `gateway.auth.token` input.

**Impact for this setup:** If the browser relay was previously accessible without auth, it now requires the token. The `browser` tool in agent context handles this automatically, but any manual or external relay clients must supply the token.

### 7) `read` tool auto-pages based on model context window

The `read` tool now auto-pages across chunks when no explicit `limit` is provided, scaling its per-call output budget from the model's `contextWindow`. Larger-context models can read more before context guards kick in.

**Impact:** Less need for manual `offset`/`limit` for small-to-medium files. Context compaction (`[compacted: tool output removed to free context]`) in large reads now recovers more gracefully.

### 8) exec tool has preflight guard for shell env var injection

A new preflight guard detects likely shell env var injection patterns (e.g., `$DM_JSON`, `$TMPDIR`) in Python/Node scripts before execution, preventing recurring cron failures when models emit mixed shell+language source.

**Impact:** Cron jobs that previously silently failed from env-var-in-code patterns now get early warnings instead of wasted tokens and failed runs.

### 9) SSRF hardening: NAT64/6to4/Teredo/octal/hex IPv4 now blocked

SSRF bypass vectors via IPv6 transition addresses (NAT64 `64:ff9b::/96`, 6to4 `2002::/16`, Teredo `2001:0000::/32`) and non-standard IPv4 forms (octal, hex, short, packed — e.g., `0177.0.0.1`, `127.1`, `2130706433`) are now blocked.

**Impact:** Low operational impact for normal use, but important for security posture. Any automation that targets private addresses via these forms will be blocked.

### 10) macOS LaunchAgent SQLite fix: `TMPDIR` now forwarded to service environment

`TMPDIR` is now forwarded into installed service environments, resolving `SQLITE_CANTOPEN` failures in macOS LaunchAgent gateway runs when SQLite can't write temp/journal files.

**Impact for this setup:** If the gateway daemon was experiencing intermittent SQLite errors under macOS, this is the fix. No config change needed.

### 11) Canvas session capabilities now node-scoped (not shared-IP fallback)

The `/__openclaw__/canvas/*` and `/__openclaw__/a2ui/*` endpoints now use node-scoped session capability URLs instead of shared-IP fallback auth, failing closed when trusted-proxy requests omit forwarded client headers.

**Impact:** Canvas/A2UI automation that relied on shared-IP auth will need to use scoped session URLs. Gateway-proxied canvas calls via normal tool paths are unaffected.

### 12) Security/Exec: `safeBins` now reject PATH-hijacked binaries

`tools.exec.safeBins` binaries must now resolve from trusted bin directories (system defaults plus gateway startup `PATH`). A trojan binary in a user-added PATH directory with the same name as a safe bin will be rejected.

**Impact:** Better security posture. No impact unless you have non-standard PATH layouts during gateway startup.

### 13) Cron webhook delivery now SSRF-guarded

Cron webhook POST delivery now routes through SSRF-guarded outbound fetch (`fetchWithSsrFGuard`), blocking private/metadata destinations before dispatch.

**Impact:** Webhook targets pointing to private addresses (localhost, RFC1918) will be rejected. Use only publicly reachable webhook endpoints.

### 15) Plaintext `ws://` connections blocked to non-loopback hosts

Plaintext `ws://` WebSocket connections to non-loopback hosts are now rejected. Secure `wss://` transport is required for remote WebSocket endpoints.

**Impact:** Any automation or integration connecting to remote WebSocket endpoints via `ws://` must switch to `wss://`. Loopback (`ws://localhost`, `ws://127.0.0.1`) is still allowed.

### 16) Control-plane write RPCs are now rate-limited

`config.apply`, `config.patch`, and `update.run` RPCs are rate-limited to 3 requests per minute per `deviceId+clientIp`. Gateway restarts are coalesced with a 30-second cooldown, and config change audit details (actor, device, IP, changed paths) are now logged.

**Impact:** Automation that rapidly applies config changes (e.g., scripted `config.patch` loops) will hit 429 rate limits. Space out config writes or batch changes into single `config.apply` calls.

### 17) Discord moderation actions require trusted-sender guild permissions

Moderation actions (`timeout`, `kick`, `ban`) now enforce guild permission checks on the trusted sender and ignore untrusted `senderUserId` parameters. This prevents privilege escalation via tool-driven flows.

**Impact:** Discord operators using moderation tools must ensure the bot/sender has appropriate guild permissions. Untrusted sender IDs in tool parameters are now rejected.

### 14) Sub-agent context guard + compacted-output recovery guidance

Accumulated tool-result context is now guarded before model calls, truncating oversized outputs and compacting oldest tool-result messages to avoid context-window overflow crashes. Explicit guidance added to recover from `[compacted: ...]` / `[truncated: ...]` markers by re-reading with smaller chunks.

**Impact for this setup:** Sub-agent runs that previously crashed on large tool outputs now degrade more gracefully. Agent prompts in AGENTS.md that handle compacted output are already correct.

---

## Recommended operational checks after upgrading to 2026.2.19

1. **Run `openclaw security audit`** — check for `gateway.http.no_auth` and any new plugin/hook findings.
2. **Verify `hooks.token` ≠ `gateway.auth.token`** — gateway will refuse to start if they match.
3. **Check `gateway.auth.mode`** — if previously relying on implicit open mode, you may need to add `mode: "none"` explicitly (or configure a token).
4. **Review YAML frontmatter in agent prompts** — switch `on`/`off` to `true`/`false` where boolean semantics are intended.
5. **Test cron/heartbeat Telegram topic routing** — if using `<chatId>:topic:<threadId>` targets, verify sends now land in the correct topic.
6. **Check browser relay clients** — any non-standard browser relay clients need gateway-token auth on `/extension` and `/cdp`.
7. **Review cron webhook targets** — ensure all webhook URLs are publicly reachable (not localhost/RFC1918).
8. **Switch remote WebSocket connections to `wss://`** — plaintext `ws://` to non-loopback hosts is now blocked.
9. **Check config automation pacing** — control-plane RPCs (`config.apply`, `config.patch`, `update.run`) are rate-limited to 3/min per device+IP.
10. **Verify Discord moderation bot permissions** — moderation actions now enforce guild permission checks on the trusted sender.

---

## Docs in this repo updated for this window

- `README.md` — version bump to v2026.2.19, changelog table updated.
- `DEVELOPER-REFERENCE.md` — v2026.2.19 workflow additions, new gotchas (items 33–45).
- `CHANGELOG.md` — this synthesized review (consolidated section for v2026.2.17 → v2026.2.19).

---

*Generated: 2026-02-20*

---

# Synthesis Review: v2026.2.19 → v2026.2.21

> **Window analyzed:** `v2026.2.19..v2026.2.21` (~290 commits, 832 files changed, +45,078/-9,997 lines)
> **Method:** behavior and operator/developer impact focus; ignore low-signal test/style churn
> **Package version confirmed:** `2026.2.21` (npm)

---

## Executive synthesis

This release window is a **provider expansion + Discord maturity + security breadth cycle** defined by four major themes:

1. **New model providers and per-channel routing controls** — Gemini 3.1 and Volcengine/Byteplus (Doubao) land as first-class providers; per-channel model overrides via `channels.modelByChannel` and per-account/channel `defaultTo` outbound routing give operators fine-grained delivery control without rewriting agent configs.
2. **Discord platform matured to production-grade** — thread-bound subagents, voice channel support, stream preview mode, configurable lifecycle status reactions, ephemeral slash-command defaults, forum tag management, and channel topic metadata land as a cohesive batch. Discord operators should expect significant new surface area to configure.
3. **Security breadth across execution, sandbox, networking, and prompt surfaces** — 40+ discrete security fixes covering heredoc allowlist bypass, shell startup-file injection, WhatsApp JID auth, sandbox browser hardening, prototype-chain traversal in webhook templates, compaction retry amplification, TTS provider-hop injection, and many more. Less concentrated than v2026.2.19's security surge but broader in scope.
4. **Telegram, memory, and cron reliability** — Telegram streaming reworked (simplified config, split reasoning/answer lanes, draft cleanup), QMD memory search diversified for multi-source ranking, heartbeat behavior regression partially reversed, and cron `maxConcurrentRuns` actually honored.

The most important operational takeaway: **this release is safe to upgrade to for macOS/Telegram setups but requires attention for any deployment using Discord, WhatsApp, BlueBubbles, sandbox browser, or advanced exec/shell tooling**. Security fixes in particular affect exec allowlists, heredoc parsing, and startup-file injection — review the operational checks section before upgrading in production.

---

## What changed in 2026.2.21 for us

### 1) Telegram streaming config simplified — `channels.telegram.streaming` boolean

Telegram preview streaming is now configured as a single boolean `channels.telegram.streaming: true/false`. The legacy `streamMode` values (`partial`, `block`, etc.) are auto-mapped on load, so existing configs continue to work without changes. The internal partial/block branching has been removed.

Additionally, streaming previews are now split into separate reasoning and answer lanes, preventing cross-lane overwrites when models emit interleaved reasoning + answer blocks. The first-preview 30-char debounce is restored, and `NO_REPLY` prefix suppression is scoped to partial sentinel fragments only.

**Impact for this setup (@MudrionBot):** No config migration required — legacy `streamMode` values auto-map. The reasoning/answer lane split is a quality improvement: if you were seeing garbled streaming output when using thinking-capable models (e.g., `kimi-coding/k2p5`), this release fixes it. Draft cleanup is now guaranteed even when dispatch throws.

### 2) Per-channel model overrides via `channels.modelByChannel`

Operators can now configure different models for different channels/accounts using `channels.modelByChannel`. The active model override is surfaced in `/status` output so it is visible during diagnostic sessions.

Also added: per-account/channel `defaultTo` outbound routing fallback so `openclaw agent --deliver` can send without requiring an explicit `--reply-to` when a default target is configured for the account/channel.

**Impact for this setup:** Useful if you want to use a different model for Telegram bot responses vs. gateway-direct sessions. Add `channels.modelByChannel` entries per channel ID to override the primary model on a per-source basis.

### 3) New model providers — Gemini 3.1 and Volcengine/Byteplus (Doubao)

`google/gemini-3.1-pro-preview` is now a first-class model in the catalog. Volcengine (Doubao) and BytePlus providers are added with coding variants, interactive and non-interactive onboarding flows, and documentation aligned to `volcengine-api-key`.

The `kimi-coding` implicit provider template is also fixed — previously, `kimi-coding/k2p5` returned 403 errors due to a missing provider template with the correct `anthropic-messages` API type and base URL. This is a direct fix for the current workaround in use.

**Impact for this setup:** The Kimi-coding fix is directly relevant — `kimi-coding/k2p5` now works without the manual `models.generated.js` patch workaround documented in MEMORY.md. When upgrading to v2026.2.21, the 403 errors from Kimi should be resolved natively. Gemini 3.1 is available as a fallback option if needed.

### 4) Cron `maxConcurrentRuns` now actually honored

The cron timer loop now respects `cron.maxConcurrentRuns` so due jobs can execute up to the configured parallelism instead of always running serially. Previously, this config key was accepted but had no effect in the timer loop.

**Impact for this setup:** If you have multiple cron jobs with overlapping schedules and had set `maxConcurrentRuns` expecting parallel execution, it now works. If you prefer serial cron execution, verify your `maxConcurrentRuns` value (default is 1 if unset, which preserves serial behavior).

### 5) Heartbeat behavior regression — missing `HEARTBEAT.md` no longer suppresses runs

The v2026.2.19 behavior change that skipped heartbeats when `HEARTBEAT.md` was missing or empty has been partially reversed. Missing `HEARTBEAT.md` no longer suppresses runs; only effectively empty files now skip. Prompt-driven and tagged-cron execution paths are preserved regardless.

**Impact for this setup:** If you removed or never created `HEARTBEAT.md` expecting heartbeats to fire anyway, they now will again. If you had an empty `HEARTBEAT.md` and want heartbeats to fire, add actual content. The tagged-cron fallback is unchanged.

### 6) Compaction safeguard now loads in production builds

The embedded compaction safeguard/context-pruning extension failed to load in production builds due to runtime file-path resolution failure. It is now wired into the bundled extension factory so it activates correctly in installed (non-dev) deployments.

**Impact for this setup:** If running a production-installed `openclaw` (not `pnpm dev`), the compaction safeguard was silently absent in v2026.2.19. It is now active. This means long-running agent sessions will have proper context compaction behavior and be less likely to hit context-window exhaustion.

### 7) Security: exec and shell allowlist hardening

Multiple exec-path security fixes in this release that affect any deployment using `tools.exec.safeBins` or shell tooling:

- **Heredoc allowlist bypass blocked** — unquoted heredoc body expansion tokens are now blocked in shell allowlist analysis; unterminated heredocs are rejected; explicit approval is required for allowlisted heredoc execution on gateway hosts.
- **Shell startup-file injection blocked** — `BASH_ENV`, `ENV`, `BASH_FUNC_*`, `LD_*`, `DYLD_*` are now stripped across config env ingestion, node-host inherited environment sanitization, and macOS exec host runtime.
- **macOS `system.run` allowlists** — evaluated per shell segment in macOS node runtime and companion exec host, including chained operators (`&&`, `||`, `;`). Shell/process substitution parsing fails closed. Unsafe parse cases require explicit approval.
- **Prototype-chain traversal in webhook templates blocked** — `__proto__`, `constructor`, `prototype` keys are blocked in `getByPath` webhook template path resolution.

**Impact for this setup:** If using `tools.exec` with any shell-based tool runs or cron scripts that use heredocs, test your exec paths after upgrading. Legitimate heredoc use that was previously passing through implicitly now requires explicit gateway approval.

### 8) Security: sandbox browser defaults hardened

The sandbox browser container no longer runs with `--no-sandbox` by default. The new default is sandboxed; opt-in requires `OPENCLAW_BROWSER_NO_SANDBOX=1` / `CLAWDBOT_BROWSER_NO_SANDBOX=1` env vars. Additionally:

- noVNC observer sessions now require password auth and use one-time observer tokens.
- Sandbox browser containers default to a dedicated Docker network (`openclaw-sandbox-browser`).
- `openclaw security audit` warns when browser sandbox runs on bridge network without source-range limits.

**Impact for this setup:** No sandbox browser in use on this setup. No action needed.

### 9) Security: WhatsApp JID allowlist enforcement hardened

WhatsApp reaction actions now enforce allowlist JID authorization so authenticated callers cannot target non-allowlisted chats by forging `chatJid` + valid `messageId` pairs. Outbound auth is also centralized and returns HTTP 403 on tool auth errors.

**Impact for this setup:** No WhatsApp in use on this setup. No action needed.

### 10) Security: TTS model-driven provider switching now opt-in

Model-driven TTS provider switching is now opt-in by default (`messages.tts.modelOverrides.allowProvider=false` unless explicitly enabled). Voice/style overrides remain available. This prevents prompt-injection-driven provider hops and unexpected TTS cost escalation.

**Impact for this setup:** No TTS in use on this setup. If TTS is ever enabled, note that `allowProvider=true` must be explicitly set to permit model-driven TTS provider selection.

### 11) Security: compaction retry amplification blocked

The overflow compaction retry budgeting is now global across tool-result truncation recovery. Previously, successful truncation could reset the overflow retry counter and amplify retry/cost cycles — a potential prompt-injection vector. This is now fixed.

**Impact for this setup:** Directly relevant for production agent runs. Long-running sessions with large tool outputs will no longer have their retry counter reset, preventing unexpected cost amplification.

### 12) Security: inbound metadata stripped from chat surfaces

Untrusted inbound user context metadata blocks are now stripped at display boundaries across TUI, webchat, and macOS surfaces, preventing injected metadata from leaking into chat history. User-authored content is preserved.

**Impact for this setup:** Relevant for TUI-based interactions. If you use the TUI (`openclaw` interactive mode), injected metadata from untrusted senders no longer appears in the chat log.

### 13) Auto-updater and `openclaw update --dry-run` added

An optional built-in auto-updater (`update.auto.*`) is now available, default-off. The `openclaw update --dry-run` command lets you check what would be updated without applying changes. Prerelease suffixes are also now correctly ignored in release-check version comparisons.

**Impact for this setup:** `update.auto` is off by default — no change to current behavior. Use `openclaw update --dry-run` to check for available updates without committing to them.

### 14) BlueBubbles (iMessage) channel added

BlueBubbles is now a supported channel for iMessage routing. Authentication is required for all webhook requests (no passwordless fallback). A dedicated `xurl` skill is also added.

**Impact for this setup:** Not currently in use. If iMessage support is wanted in the future, BlueBubbles is the integration path.

### 15) Discord platform features (not in use but changes to monitor)

Discord received significant new surface area: thread-bound subagents (#21805), stream preview mode, configurable lifecycle status reactions with emoji/timing overrides, voice channel join/leave/status via `/vc`, configurable ephemeral defaults, forum `available_tags` management, and channel topics in trusted metadata. Discord message handlers now properly await processing instead of fire-and-forget.

**Impact for this setup:** Discord is not in use. However, if Discord is ever added, the v2026.2.21 config surface for Discord is substantially larger than previous versions. These items require no action now.

### 16) Memory/QMD search reliability improvements

QMD memory search received a broad reliability fix: per-agent `memorySearch.enabled=false` is now respected at gateway startup; multi-collection searches split into per-collection queries to avoid sparse-term drops; collection-hinted doc resolution preferred to avoid stale-hash collisions; boot updates retry on transient lock/timeout failures; `qmd embed` skipped in BM25-only `search` mode; embed runs are serialized globally with failure backoff to prevent CPU storms on multi-agent hosts.

Mixed-source ranking is also fixed so session transcript hits no longer crowd out durable memory-file matches in top results.

**Impact for this setup:** If using QMD memory backend (`memory.backend: qmd`), this is a significant reliability improvement. Multi-agent setups on the same host will no longer cause CPU storms from concurrent embed runs.

### 17) Gateway auth improvements for trusted-proxy and custom bind-host

Several gateway auth edge cases are fixed:

- `trusted-proxy` mode with `bind="loopback"` now requires `gateway.trustedProxies` to include a loopback address, preventing same-host proxy misconfiguration from silently blocking auth.
- `gateway.customBindHost` is now accepted in strict config validation when `bind="custom"`, fixing startup failures for valid custom bind configurations.
- `health` method is now accessible to all authenticated clients across roles/scopes while still enforcing role/scope for other methods.

**Impact for this setup (gateway on port 18789, loopback bind):** If you run a local reverse proxy in front of the gateway, the trusted-proxy loopback requirement now enforces correct config. Standard loopback-direct setups are unaffected.

### 18) CLI and config fixes

Several CLI/config improvements:

- `openclaw config unset <path>` no longer re-introduces defaulted keys (e.g. `commands.ownerDisplay`) through schema normalization after a write.
- `openclaw -v` is preserved as root-only version alias so subcommand `-v, --verbose` flags no longer intercept it globally.
- `config set --strict-json` is now canonical; `--json` is kept as a legacy alias.
- Bootstrap agent files with missing/invalid paths are skipped with a warning instead of crashing the agent session.
- `pairing list` and `pairing approve` now default to the sole available pairing channel when omitted.

**Impact for this setup:** The `config unset` fix is directly relevant — if you have previously used `openclaw config unset` and noticed keys reappearing, this is fixed. The `-v` alias restoration prevents flag parsing confusion in verbose subcommand flows.

### 19) Docker build fixed

Two Docker build issues are resolved: `ownerDisplay` is now included in `CommandsSchema` object-level defaults (fixing `TS2769` build failures), and Playwright Chromium is installed into `/home/node/.cache/ms-playwright` with correct `node:node` ownership so browser binaries are available to the runtime user.

**Impact for this setup:** Not running Docker. No action needed.

### 20) SHA-1 → SHA-256 migration for synthetic IDs

Gateway lock and tool-call synthetic IDs are migrated from SHA-1 to SHA-256 with unchanged truncation length. The owner-ID obfuscation now uses a dedicated `ownerDisplaySecret` HMAC key from configuration, decoupled from the gateway token.

**Impact for this setup:** Transparent upgrade — no config change required. If you have external tooling that parses synthetic IDs, the hash basis changes but the format/length does not. The `ownerDisplaySecret` field is new; if not configured, a derived default is used.

---

## Recommended operational checks after upgrading to 2026.2.21

1. **Verify Kimi-coding works natively** — the `kimi-coding` implicit provider template is now correct. If you applied the `models.generated.js` patch workaround from MEMORY.md, verify it is not needed after upgrade and test `kimi-coding/k2p5` responds without 403 errors.
2. **Check Telegram streaming behavior** — if you had `streamMode` configured explicitly (e.g. `streamMode: "partial"`), the auto-map should handle it, but verify streaming previews still behave as expected in @MudrionBot after upgrade.
3. **Test cron job concurrency** — if you have multiple cron jobs and rely on serial execution, confirm `cron.maxConcurrentRuns` is explicitly set to `1` in config (it was previously the effective behavior regardless of config).
4. **Verify heartbeat fires if `HEARTBEAT.md` is absent** — v2026.2.19 suppressed heartbeats on missing `HEARTBEAT.md`; v2026.2.21 restores firing behavior when the file is absent. Confirm this is your intended behavior.
5. **Test compaction in long agent sessions** — the compaction safeguard now loads correctly in production builds. After upgrading, run a long agent session and confirm context compaction activates without crashing.
6. **Review exec/shell tool runs for heredoc patterns** — if any cron scripts or agent tools use heredoc syntax, test them after upgrade. Unquoted heredoc body expansion is now blocked; legitimate use requires explicit gateway approval.
7. **Check `openclaw config unset` side effects** — if you previously worked around the key-reappearance bug by not using `config unset`, verify your current config has no unwanted defaulted keys after upgrade.
8. **Run `openclaw update --dry-run`** — use the new dry-run flag to confirm the updater sees the expected version and does not attempt unexpected upgrades.
9. **Check `ownerDisplaySecret`** — if owner-ID obfuscation is in use, a new dedicated `ownerDisplaySecret` HMAC key is now expected. If not explicitly set, a derived default is used. Set explicitly if you require stable obfuscated owner IDs across gateway restarts.
10. **Review QMD memory search if enabled** — if using `memory.backend: qmd`, the serialized embed behavior and per-collection search split may change result ordering. Verify memory recall quality after upgrade.

---

## Docs in this repo updated for this window

- `README.md` — version bump to v2026.2.21, changelog table updated, versioning section updated.
- `CHANGELOG.md` — this synthesized review (consolidated section for v2026.2.19 → v2026.2.21).

---

*Generated: 2026-02-23*

---

# OpenClaw v2026.2.22 — Synthesis Review

**Cycle:** v2026.2.21 → v2026.2.22  
**Released:** 2026-02-24  
**Scope:** Platform expansion, runtime hardening, multilingual memory, and a major security cycle  
**Breaking changes:** 4  
**Security fixes:** 30+  

---

## Executive Summary

v2026.2.22 is the largest single release in recent OpenClaw history. The release adds two new channel plugins (Synology Chat), a new AI provider (Mistral), web search grounding via Gemini, full Control UI cron edit parity, and an optional auto-updater. Simultaneously it ships over 30 security fixes — including critical exec-approval bypasses, SSRF hardening, symlink escape prevention, and auth hardening across nearly every channel. Four breaking changes affect streaming config, device auth, DM scope defaults, and tool-failure reply verbosity.

---

## Breaking Changes

### 1. Google Antigravity Provider Removed
`google-antigravity/*` model and profile configs no longer work. The `google-antigravity-auth` bundled plugin is removed.

**Migration:** Switch to `google-gemini-cli` or another supported Google provider.

```json
// Before
{ "model": "google-antigravity/gemini-2.0-flash" }
// After  
{ "model": "google-gemini-cli/gemini-2.0-flash" }
```

### 2. Tool-Failure Replies Hide Raw Errors by Default
Detailed error suffixes (provider/runtime messages, local path fragments) are now hidden by default.

**Impact:** Agents still send failure summaries; enable `/verbose on` or `/verbose full` to restore raw details.

### 3. `session.dmScope` Defaults to `per-channel-peer`
CLI local onboarding now sets `session.dmScope` to `per-channel-peer` for new/implicit DM scope configurations.

**Migration:** If you depend on shared DM continuity across senders, explicitly set:
```json
{ "session": { "dmScope": "main" } }
```

### 4. Unified Streaming Config + Device Auth v1 Removed
- Channel preview-streaming now uses `channels.<channel>.streaming` with enum values `off | partial | block | progress`
- Slack native stream toggle moved to `channels.slack.nativeStreaming`
- Legacy keys (`streamMode`, Slack boolean `streaming`) still read and migrated by `openclaw doctor --fix`
- **Device auth signature v1 removed.** Clients must sign `v2` payloads with `connect.challenge` nonce; nonce-less connects are rejected.

---

## Major New Features

### New Channel: Synology Chat
Native Synology Chat channel plugin with webhook ingress, direct-message routing, outbound send/media support, per-account config, and DM policy controls.

**Config key:** `channels.synologychat.*`

### New Provider: Mistral
Full Mistral provider support including memory embeddings and voice support.

**Usage:** Set `model: "mistral/<model-id>"` in agent config.

### Grounded Web Search via Gemini
Grounded Gemini provider support for web search with provider auto-detection.

**Config:** Enable via `tools.webSearch.provider: "gemini"` with appropriate API key.

### Full Control UI Cron Edit Parity
Complete web cron management with:
- Clone and rich validation/help text
- All-jobs run history with pagination, search, sort, and multi-filter controls
- Improved cron page layout for scheduling and failure triage

### Memory FTS Multilingual Expansion
Full-text search query expansion now supports:
- **Spanish + Portuguese** stop-word filtering
- **Japanese** tokenization with mixed-script (ASCII + katakana) support  
- **Korean** particle-aware extraction with mixed Korean/English stems
- **Arabic** stop-word filtering

### Optional Auto-Updater
Built-in auto-updater for package installs (`update.auto.*`), default-off.
- Stable rollout delay + jitter
- Beta hourly cadence
- `openclaw update --dry-run` previews actions without mutating config

### Control UI: Tools Panel Data-Driven
Tools panel now driven from runtime `tools.catalog` with per-tool provenance labels (`core` / `plugin:<id>`). Static fallback list when runtime catalog is unavailable.

### Control UI: Version Status Pill
Web header now shows a version status pill before the Health indicator.

---

## Security Fixes (30+)

This release is a major security hardening cycle. Fixes organized by subsystem:

### Exec Approval System (Critical)
- **Safe-bin PATH hijacking:** Stop trusting `PATH`-derived directories for safe-bin allowlist checks; add `tools.exec.safeBinTrustedDirs` and pin to resolved absolute paths.
- **Wrapper-path bypass:** When users choose `allow-always` for shell-wrapper commands, persist allowlist patterns for the inner executable, not the wrapper shell binary.
- **Shell line continuations:** Fail closed on `\\\n`/`\\\r\n` line continuations and treat shell-wrapper execution as approval-required in allowlist mode.
- **`env` wrapper transparency:** Treat `env` and shell-dispatch wrappers as transparent during allowlist analysis so policy checks match the effective executable.
- **Safe-bin profiles:** Require explicit safe-bin profiles for `tools.exec.safeBins` entries; add `tools.exec.safeBinProfiles` for custom binaries.
- **`sort --compress-program` bypass:** Block `sort --compress-program` in allowlist mode.
- **macOS app basename matching:** Enforce path-only allowlist matching; migrate legacy basename entries to resolved paths; harden shell-chain handling.
- **Sandbox fail-closed:** Fail closed when `tools.exec.host=sandbox` is configured but sandbox runtime is unavailable.
- **Shell exec env:** Block request-scoped `HOME`/`ZDOTDIR` overrides; block `SHELLOPTS`/`PS4`; restrict shell-wrapper env to explicit allowlist.
- **Shell startup injection:** Validate login-shell executable paths; block `SHELL`/`HOME`/`ZDOTDIR` in config env ingestion.

### SSRF Hardening
- Expand IPv4 fetch guard to RFC special-use/non-global ranges (benchmarking `198.18.0.0/15`, TEST-NET, multicast, reserved/broadcast).
- Normalize IPv6 dotted-quad transition literals (`::127.0.0.1`, `64:ff9b::8.8.8.8`).
- Enable `autoSelectFamily` on pinned undici dispatchers so IPv6-unreachable environments fall back to IPv4.
- MSTeams: enforce allowlist checks for SharePoint URLs and redirect targets across full redirect chains.

### Symlink Escape Prevention
- Browser uploads: accept in-root paths when uploads directory is a symlink alias.
- Security/Archive: block zip symlink escapes during extraction.
- Media sandbox: enforce symlink-escape checks before sandbox-validated reads.
- Control UI: block symlink-based out-of-root static file reads via realpath containment.
- Gateway avatars: block symlink traversal during local avatar resolution.
- Hooks transforms: enforce symlink-safe containment for webhook transform module paths.

### Auth and Identity
- **Elevated scope bypass:** Match `tools.elevated.allowFrom` against sender identities only (not recipient `ctx.To`).
- **Feishu display-name collision:** Enforce ID-only allowlist matching; ignore mutable display names.
- **Group policy toolsBySender:** Require explicit sender-key types (`id:`, `e164:`, `username:`, `name:`).
- **Discord allowlist:** Canonicalize resolved names to IDs; add audit warnings for name/tag-based entries.
- **Discord: block node-role connections** when device identity metadata is missing.
- **Gateway insecure-config warning:** Emit startup warning when dangerous flags are enabled.
- **Owner display secret:** Auto-generate dedicated `commands.ownerDisplaySecret`; remove gateway token fallback.
- **Prototype pollution:** Block `__proto__`, `constructor`, `prototype` traversal in config merge helpers.
- **Hook auth rate limits:** Normalize IPv4/IPv4-mapped IPv6 addresses to one throttle bucket.

### Session and Credential Security
- **`sessions_history` redaction:** Redact sensitive token patterns from output; surface `contentRedacted` metadata.
- **`openclaw config get` redaction:** Redact sensitive values before printing paths.
- **Voice call WebSocket:** Strict pre-start timeouts; pending/per-IP connection limits; total connection caps for streaming endpoints.
- **Media inbound byte limits:** Enforce limits during download/read across Discord, Telegram, Zalo, MSTeams, BlueBubbles.
- **WhatsApp `allowFrom`:** Enforce for direct-message outbound targets in all send modes.
- **Channels/Security:** Fail closed on missing provider group policy config — runtime defaults to `allowlist` when `channels.<provider>` is absent.

---

## Channels

### Telegram
- WSL2: disable `autoSelectFamily` by default; memoize WSL2 detection.
- DNS: default Node 22+ to `ipv4first` for Telegram fetch paths; add `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER` override.
- Forward bursts: coalesce forwarded text+media through a dedicated forward lane debounce window.
- Streaming: preserve archived draft preview mapping after flush; clean superseded reasoning preview bubbles.
- Replies: scope messaging-tool text/media dedupe to same-target sends only.
- Polling: persist safe update-offset watermark bounded by pending updates; force-restart stuck runner instances.
- Media: send user-facing reply when media download fails (non-size errors).
- Webhook: keep monitors alive until gateway abort signals fire; add `channels.telegram.webhookPort` config.

### Slack
- Threading: keep parent-session forking and thread-history context active beyond first turn.
- Threading: respect `replyToMode` when Slack auto-populates top-level `thread_ts`.
- Extension: forward `message read` `threadId` to `readMessages`; use delivery-context `threadId` as outbound fallback.
- Upload: resolve bare user IDs to DM channel IDs via `conversations.open` before `files.uploadV2`.
- Slash commands: preserve Bolt app receiver when registering external select options handlers.
- Queue routing: preserve string `thread_ts` values through collect-mode queue drain.

### Discord
- Allowlist: canonicalize resolved Discord allowlist names to IDs.
- Allowlist: add security audit warnings for name/tag-based entries.

### BlueBubbles
- DM history: restore DM backfill context with account-scoped rolling history.
- Private API cache: treat unknown (`null`) cache status as disabled.
- Webhooks: accept payloads when BlueBubbles omits `handle` but provides DM `chatGuid`.

### Webchat
- Apply assistant `final` payload messages directly to chat state (no history refresh wait).
- For out-of-band final events, append final assistant payloads directly.
- Preserve external session routing metadata when internal `chat.send` turns run under `webchat`.
- Preserve existing session `label` across `/new` and `/reset` rollovers.

### Browser Extension Relay
- MV3 worker: preserve debugger attachments across relay drops; auto-reconnect with bounded backoff+jitter.
- Persist and rehydrate attached tab state via `chrome.storage.session`.
- Recover from `target_closed` navigation detaches; enforce per-tab operation locks.
- Remote CDP: extend stale-target recovery to reuse the sole available tab.

---

## Agents

### Reasoning
- Avoid classifying provider reasoning-required errors as context overflows.
- OpenRouter: map `/think` levels to `reasoning.effort` in embedded runs.
- OpenRouter: default reasoning to enabled when model advertises `reasoning: true`.

### Compaction
- Count auto-compactions only after non-retry `auto_compaction_end`.
- Restore embedded compaction safeguard/context-pruning extension loading in production builds.
- Strip stale assistant usage snapshots from pre-compaction turns to prevent immediate re-triggering.
- Pass model metadata through embedded runtime for safeguard summarization.
- Cancel safeguard compaction when summary generation cannot run.

### Context Overflow
- Treat HTTP 502/503/504 as failover-eligible transient timeouts.
- Kimi: classify Moonshot `token limit` failures as context overflows for auto-compaction.

### Provider-Specific
- **Moonshot/Kimi:** Force `supportsDeveloperRole=false`; mark K2.5 as image-capable; enable consecutive-user turn merging.
- **OpenRouter:** Inject `cache_control` on system prompts for Anthropic models; preserve `openrouter/` prefix during normalization.
- **Google:** Sanitize non-base64 `thought_signature` values from assistant replay transcripts.
- **Mistral:** Sanitize tool-call IDs; generate strict provider-safe pending tool-call IDs.
- **Ollama:** Preserve unsafe integer tool-call arguments as exact strings during NDJSON parsing.
- **Anthropic:** Default missing `api` fields to `anthropic-messages` during config validation.

### Subagents
- Honor `tools.subagents.tools.alsoAllow` when resolving built-in subagent deny defaults.
- Make announce call timeouts configurable via `agents.defaults.subagents.announceTimeoutMs` (default 60s).

### Workspace/Exec
- Guard `resolveUserPath` against undefined/null input.
- Honor explicit agent context when resolving `tools.exec` defaults.
- Map sandbox container-workdir file-tool paths to host workspace roots before workspace-only validation.

### Transcripts/Tool-calls
- Validate assistant tool-call names before persistence and during replay sanitization.
- Emit default `✅ Done.` acknowledgement only for direct/private tool-only completions.

---

## Cron

### Major Fixes
- **`maxConcurrentRuns`:** Now enforced in the timer loop (was always serializing).
- **Manual run timeout:** Apply same per-job timeout guard for `cron.run` executions.
- **Startup catch-up:** Enforce per-job timeout guards for startup replay runs.
- **Isolated sessions:** Force fresh session IDs for `sessionTarget="isolated"` executions.
- **Schedule timing:** For `every` jobs, prefer `lastRunAtMs + everyMs` when in future.

### Status/Visibility
- Split execution outcome (`lastRunStatus`) from delivery outcome (`lastDeliveryStatus`) in persisted state.
- Persist `delivered` state in cron job records.
- Clean up settled per-path run-log write queue entries.

### Auth/Delivery
- Propagate auth-profile resolution to isolated cron sessions.
- Pass resolved `agentDir` through isolated cron and queued follow-up embedded runs.
- Route text-only announce jobs with explicit thread/topic targets through direct outbound delivery.
- Telegram: validate cron `delivery.to` with shared target parsing; resolve legacy `@username`/`t.me` targets.

---

## Gateway

### Auth Unification
- Unify call/probe/status/auth credential-source precedence on shared resolver helpers.
- Refactor gateway credential resolution and websocket auth handshake paths.
- Add explicit `auth.deviceToken` support in connect frames.

### Pairing
- Treat `operator.admin` as satisfying other `operator.*` scope checks.
- Auto-approve loopback `scope-upgrade` pairing requests.
- Include `operator.read` and `operator.write` in default operator connect scope bundles.
- Treat `operator.admin` pairing tokens as satisfying `operator.write` requests.

### Stability
- Fix restart-loop edge cases: explicit bootstrap detection, lock reacquisition.
- Use optional gateway-port reachability as primary stale-lock liveness signal.
- Retry short-lived missing config snapshots during reload.
- Compare array-valued config paths structurally during diffing.
- Verify gateway health after daemon restart.

---

## Memory

### QMD
- Migrate legacy unscoped collection bindings to per-agent scoped names during startup.
- Normalize Han-script BM25 search queries for mixed CJK+Latin prompts.
- Add optional `memory.qmd.mcporter` search routing through mcporter keep-alive flows.
- On Windows, resolve bare `qmd`/`mcporter` command names to npm shim executables (`.cmd`).

### Embeddings
- Centralize remote memory HTTP calls behind shared guarded helper.
- Apply configured remote-base host pinning across OpenAI/Voyage/Gemini embedding requests.
- Enforce per-input 8k safety cap before embedding batching; 2k fallback for local providers.

### Indexing
- Detect memory source-set changes and trigger full reindex automatically.

---

## Plugins/Extensions

### Install/Discovery
- Strip `workspace:*` devDependency entries from copied plugin manifests before `npm install`.
- Ignore scanned extension backup/disabled directory patterns (`.backup-*`, `.bak`, `.disabled*`).
- Move updater backup directories under `.openclaw-install-backups`.
- Make `openclaw plugins enable` update allowlists via shared plugin-enable policy.
- Auto-enable built-in channels by writing `channels.<id>.enabled=true` (not `plugins.entries.<id>`).
- When `plugins.allow` is active, also allowlist configured built-in channels automatically.

### Feishu
- Restore bundled Feishu SDK availability for global installs.
- Prefer `file_key` over `image_key` when both are present in video messages.
- In group chats, command authorization falls back to top-level `channels.feishu.allowFrom`.

### Hooks
- Run legacy `before_agent_start` once per agent turn; reuse across model-resolve and prompt-build.
- Suppress duplicate main-session events for delivered hook turns.
- Avoid redundant hook-module recompilation on gateway restart using stable file metadata keys.

---

## Logging and Delivery

- Cap single log-file size with `logging.maxFileBytes` (default 500 MB) to prevent disk exhaustion.
- Quarantine queue entries immediately on permanent delivery errors — move to `failed/` instead of retrying.
- Classify undici `TypeError: fetch failed` as transient in unhandled-rejection detection.

---

## TUI

- Enable multiline-paste burst coalescing on macOS Terminal.app and iTerm.
- Isolate right-to-left script lines (Arabic/Hebrew) with Unicode bidi isolation marks.
- Request immediate renders after setting `sending`/`waiting` activity states.
- Arm Ctrl+C exit timing when clearing non-empty composer text; add SIGINT fallback.

---

## iOS

- Prefetch TTS segments and suppress expected speech-cancellation errors for smoother talk playback.

---

## Removed

- Bundled `food-order` skill removed from core — install from ClawHub instead.
- Google Antigravity provider and `google-antigravity-auth` plugin removed.

---

## Config Changes

| Key | Change |
|-----|--------|
| `channels.<channel>.streaming` | Unified streaming config (replaces `streamMode`) |
| `channels.slack.nativeStreaming` | Replaced Slack boolean `streaming` |
| `session.dmScope` | Now defaults to `per-channel-peer` on new installs |
| `update.auto.*` | New: optional auto-updater |
| `agents.defaults.subagents.announceTimeoutMs` | New: configurable announce timeout |
| `tools.exec.safeBinTrustedDirs` | New: explicit trusted directories for safe-bin |
| `tools.exec.safeBinProfiles` | New: profiles for custom safe binaries |
| `memory.qmd.mcporter` | New: optional mcporter search routing |
| `agents.defaults.memorySearch.provider` | Now accepts `"mistral"` |
| `logging.maxFileBytes` | New: log file size cap (default 500 MB) |
| `channels.modelByChannel` | New: per-channel model overrides (allowlisted) |
| `config.bindings[].comment` | New: optional annotation field |

---

## Upgrade Notes

1. **Run `openclaw doctor --fix`** after upgrading — migrates legacy streaming config keys automatically.
2. **Review device auth:** Device-auth signature v1 is removed; all clients must sign `v2` payloads.
3. **Check DM scope:** If your setup relies on shared DM history across multiple senders, set `session.dmScope: "main"` explicitly.
4. **Google Antigravity configs:** Replace with `google-gemini-cli` or another supported Google provider.
5. **Security:** Review `tools.exec.safeBins` — safe-bin path resolution and profiles changed significantly.

---


# OpenClaw v2026.2.23 — Synthesis Review (Released)

**Cycle:** v2026.2.22 → v2026.2.23 (Release Cycle)
**Status:** RELEASED — tag `v2026.2.23` (2026-02-24)
**Scope:** Provider normalization, session hardening, reasoning fixes, security sweep  

---

## Executive Summary

v2026.2.23 shipped on 2026-02-24. The primary feature addition is Vercel AI Gateway Claude shorthand normalization. The bulk of the release is fixes: session key canonicalization, Telegram stability (reactions, polling), agent reasoning improvements, context overflow detection expansion (including Chinese patterns), and a significant security sweep covering exec obfuscation detection, XSS prevention in skill HTML output, OTEL credential redaction, and Python skill packaging hardening.

---

## New Features

### Vercel AI Gateway Claude Shorthand Normalization
Accept Claude shorthand model refs (`vercel-ai-gateway/claude-*`) by normalizing to canonical Anthropic-routed model IDs.

**Before:** Claude shorthand refs would fail or route incorrectly through Vercel AI Gateway.  
**After:** `vercel-ai-gateway/claude-opus-4-6` normalizes automatically.

Thanks @sallyom, @markbooch, and @vincentkoc. (PR #23985)

---

## Session Fixes

### Session Key Canonicalization
Canonicalize inbound mixed-case session keys for metadata and route updates. Migrate legacy case-variant entries to a single lowercase key.

**Impact:** Prevents duplicate sessions and missing TUI/WebUI history for users with legacy case-variant session keys. (PR #9561)

---

## Telegram Fixes

### Reactions: Soft-Fail + snake_case Support
- Soft-fail reaction action errors (policy/token/emoji/API)
- Accept snake_case `message_id` in reaction payloads
- Fallback to inbound message-id context when explicit `messageId` is omitted

**Impact:** DM reactions stay stable without regeneration loops. (PR #20236, #21001)

### Polling: Per-Bot Offset Scoping
Scope persisted polling offsets to bot identity and reuse a single awaited runner-stop path on abort/retry.

**Impact:** Prevents cross-token offset bleed and overlapping pollers during restart/error recovery. (PR #10850, #11347)

---

## Agent Fixes

### Reasoning: Thinking-Block Leak Prevention
When model-default thinking is active (e.g., `thinking=low`), keep auto-reasoning disabled unless explicitly enabled.

**Impact:** Prevents `Reasoning:` thinking-block leakage in channel replies. (PR #24335, #24290)

### Reasoning: Error Classification
Avoid classifying provider reasoning-required errors as context overflows.

**Impact:** These failures no longer trigger compaction-style overflow recovery. (PR #24593)

### Model Config Boundary
Codify `agents.defaults.model` / `agents.defaults.imageModel` config-boundary input as `string | {primary, fallbacks}`.

**Impact:** Fixes `models status --agent` source attribution — defaults-inherited agents labeled as `defaults`. (PR #24210)

### Compaction: Agent Directory Scoping
- Pass `agentDir` into manual `/compact` command runs for correct auth/profile resolution.
- Pass model metadata through embedded runtime so safeguard summarization can run when `ctx.model` is unavailable.
- Cancel safeguard compaction when summary generation cannot run (missing model/API key), preserving history.

(PR #24133, #3479, #10711)

### Context Overflow: Extended Detection
- Detect additional provider context-overflow error shapes (`input length` + `max_tokens` exceed-context variants).
- Add Chinese context-overflow pattern detection in `isContextOverflowError` for localized provider errors.
- Treat HTTP 502/503/504 errors as failover-eligible transient timeouts.
- Groq: avoid classifying Groq TPM limit errors as context overflow.

(PR #9951, #22855, #20999, #16176)

---

## Auto-Reply Fix

### Direct-Chat Metadata Handling
Hide direct-chat `message_id`/`message_id_full` and sender metadata only from normalized chat type (not sender-id sentinels).

**Impact:** Preserves group metadata visibility; prevents sender-id spoofed direct-mode classification. (PR #24373)

---

## Slack Fix

### Group Policy Inheritance
Move Slack account `groupPolicy` defaulting to provider-level schema defaults so multi-account configs inherit top-level `channels.slack.groupPolicy` instead of silently overriding with per-account `allowlist`.

(PR #17579)

---

## Provider Fixes

### Anthropic: Skip context-1m Beta for OAuth Tokens
Skip `context-1m-*` beta injection for OAuth/subscription tokens (`sk-ant-oat-*`).

**Impact:** Avoids Anthropic 401 auth failures when `params.context1m` is enabled with OAuth/subscription tokens. (PR #10647, #20354)

### OpenRouter: Remove Conflicting `reasoning_effort`
Remove conflicting top-level `reasoning_effort` when injecting nested `reasoning.effort`.

**Impact:** Prevents OpenRouter 400 payload-validation failures for reasoning models. (PR #24120)

---

## Gateway Fix

### WebSocket: Unauthorized Request Flood Protection
Close repeated post-handshake `unauthorized role:*` request floods per connection. Sample duplicate rejection logs.

**Impact:** Prevents a single misbehaving client from degrading gateway responsiveness. (PR #20168)

---

## Config Fix

### Config Write Immutability + Path Traversal Hardening
Apply `unsetPaths` with immutable path-copy updates so config writes never mutate caller-provided objects.

**Security:** Harden `openclaw config get/set/unset` path traversal by rejecting prototype-key segments and inherited-property traversal. (PR #24134)

---

## Security Fixes

### Exec: Obfuscated Command Detection
Detect obfuscated commands before exec allowlist decisions and require explicit approval for obfuscation patterns.

**Impact:** Closes a class of allowlist bypass via command obfuscation. (PR #8592)

### Skills/openai-image-gen: Stored XSS Prevention
Escape user-controlled prompt, filename, and output-path values in `openai-image-gen` HTML gallery generation.

**Impact:** Prevents stored XSS in generated `index.html` output when images are served locally. (PR #12538)

### Skills/skill-creator: Symlink Escape Prevention
Harden `skill-creator` packaging by skipping symlink entries and rejecting files whose resolved paths escape the selected skill root.

(PR #24260, #16959)

### OTEL: Credential Redaction
Redact sensitive values (API keys, tokens, credential fields) from diagnostics-otel log bodies, log attributes, and error/reason span fields before OTLP export.

(PR #12542)

### CI: Pre-Commit Security Coverage
Add pre-commit security hook coverage for private-key detection and production dependency auditing. Enforce in CI alongside baseline secret scanning.

---

## Python Skills Hardening

### Packaging Validation
- Skip self-including `.skill` outputs
- Handle CRLF frontmatter parsing
- Strict `--days` validation
- Safer image file loading

### CI + Pre-Commit Linting
- Add `ruff` linting for Python scripts/tests under `skills/`
- Add pytest discovery coverage including package test execution from repo root

---

## Config Changes

| Key | Change |
|-----|--------|
| `agents.defaults.model` | Type formalized as `string \| {primary, fallbacks}` |
| `agents.defaults.imageModel` | Type formalized as `string \| {primary, fallbacks}` |

---

## Upgrade Notes for v2026.2.23

Now that `v2026.2.23` is released, apply these checks after upgrade:

1. **Session keys:** If you have case-variant session keys in legacy configs, they will be automatically migrated to lowercase.
2. **Reasoning with thinking:** Models with `thinking=low` will no longer enable auto-reasoning by default — verify your reasoning-dependent workflows.
3. **Vercel AI Gateway:** Shorthand Claude refs now normalize automatically — no config change needed.
4. **OTEL users:** Review OTLP exports — credential fields are now redacted.

---

# Synthesis Review: v2026.2.22 → v2026.2.23 (Released)

> **Window analyzed:** `v2026.2.22..v2026.2.23` (release window)
> **Status:** RELEASED (2026-02-24)

---

## Executive Summary

This update cycle continues the security-hardening focus with expanded exec approval protections, channel security unification, and automated reasoning/thinking leak prevention. Key themes include:

1. **Exec approval system hardened** — Multi-layer fixes for wrapper bypasses, busybox/toybox applet handling, shell env sanitization, and two-phase approval registration.
2. **Channel security unified** — `dangerouslyAllowNameMatching` policy applied consistently across all channels with shared audit/doctor detection.
3. **Reasoning block leaks eliminated** — Automatic suppression of thinking/reasoning blocks in channel delivery paths, with provider-specific handling.
4. **Webhooks and plugins stabilized** — Synology Chat webhook deregistration, plugin config schema fallback, and Feishu media handling fixes.

---

## Critical Security Fixes

### Exec Approval System

| Fix | Impact |
|-----|--------|
| Bind `host=node` approvals to explicit `nodeId` | Prevents cross-node replay attacks |
| Two-phase approval registration + wait-decision | Eliminates orphaned `/approve` flows and immediate-return races |
| Canonical wrapper execution plans | Prevents `env -S/--split-string` interpretation-mismatch bypasses |
| Busybox/toybox applet recognition | Inner executables persisted instead of wrapper binaries |
| `autoAllowSkills` pathless invocation requirement | `./<skill-bin>`/absolute-path basename collisions blocked |
| Safe-bin long-option validation | Unknown/ambiguous GNU long-option abbreviations rejected |
| Sort filesystem-dependent flags denied | `--random-source`, `--temporary-directory`, `-T` blocked |
| Shell env fallback hardening | Only trust login shells registered in `/etc/shells` |

### Channel & Messaging Security

| Fix | Impact |
|-----|--------|
| WhatsApp JID allowlist on all sends | Tool-initiated sends now validate JID |
| Reasoning/thinking payload suppression | Non-Telegram channels no longer emit internal reasoning blocks |
| Telegram media SSRF policy | RFC2544 benchmark range blocked by default with explicit opt-in |
| `commands.allowFrom` sender-only matching | Blocks conversation-shaped `From` identities |
| Config prototype pollution prevention | Blocks `__proto__`, `constructor`, `prototype` traversal |

---

## Major Changes

### Providers

- **Kilo Gateway** — First-class `kilocode` provider support with auth, onboarding, implicit detection, and default model `kilocode/anthropic/claude-opus-4.6` (#20212)
- **Vercel AI Gateway** — Claude shorthand normalization (`vercel-ai-gateway/claude-*`) (#23985)
- **Moonshot/Kimi** — Native video provider, grounded web search support, `supportsDeveloperRole=false` enforced

### Tools & Capabilities

- **`web_search` provider: "kimi"** — Moonshot search with citation extraction (#16616, #18822)
- **Media understanding/Video** — Native Moonshot video provider with auto key detection (#12063)
- **`agents.defaults.subagents.runTimeoutSeconds`** — Configurable default timeout for `sessions_spawn`

### Sessions & Cron

- **Session maintenance hardening** — `openclaw sessions cleanup`, per-agent store targeting, disk-budget controls (#24753)
- **Isolated cron sessions** — Full prompt mode for skill/extension availability (#24944)
- **Cron webhook deregistration** — Synology Chat stale route cleanup (#24971)

### Gateway & Channels

- **HTTP security headers** — Optional `gateway.http.securityHeaders.strictTransportSecurity` (#24753)
- **Telegram reactions** — Soft-fail errors, snake_case `message_id`, context fallback (#20236, #21001)
- **Telegram polling** — Scoped offsets, single runner-stop path (#10850, #11347)
- **Slack group policy** — Inheritance fix for multi-account configs (#17579)
- **Discord threading** — Parent ID recovery, thread-bound subagents (#24897)
- **Web UI i18n** — Locale hydration on startup (#24795)

---

## Breaking Changes

1. **Control UI origin requirements** — Non-loopback Control UI requires explicit `gateway.controlUi.allowedOrigins`; fails closed without `dangerouslyAllowHostHeaderOriginFallback=true`
2. **Channel `allowFrom` ID-only default** — Mutable name/tag/email matching disabled by default; use `dangerouslyAllowNameMatching=true` for compatibility mode (#24907)

---

## Fixes (Release Selection)

### Agents & Runtime
- Bootstrap file snapshots cached per session key, cleared on reset (#22220)
- Context pruning `cache-ttl` extended to Moonshot/Kimi and ZAI/GLM (#24497)
- Kimi `token limit` failures classified as context overflows (#9562)
- Consecutive-user turn merging for non-OpenAI providers (#7693)
- OpenRouter `cache_control` injection for Anthropic models (#17473)
- Anthropic OAuth tokens skip `context-1m` beta (#10647, #20354)
- Bedrock cache retention disabled for non-Anthropic models (#20866, #22303)
- Groq TPM errors no longer classified as context overflow (#16176)

### Telegram
- Reaction soft-fail for policy/token/emoji/API errors (#20236)
- Polling offset scoped to bot identity (#10850, #11347)
- Reasoning suppression when `/reasoning off` active (#24626, #24518)

### WhatsApp
- Group policy `groupAllowFrom` fix when no explicit `groups` (#24670)
- DM routing only updates main-session when bound (#24949)
- `selfChatMode` honored in access control (#24738)
- Outbound recipient identifier redaction (#24980)

### Webchat/UI
- Locale hydration during startup (#24795)
- Assistant `final` payloads applied directly (#14928, #11139)
- External session routing metadata preserved (#23258)
- Session `label` preserved across resets (#23755)

### Gateway & Infra
- Browser control loads `server.js` reliably (#23974)
- Chrome relay debugger detach handling (#19766)
- Relay port derivation clarified for custom gateway ports (#22252)
- Pairing recovery shows explicit approval hints (#24771)
- OAuth scope failures classified as auth failures (#24761)

---

## Recommended Operational Checks (Post-Release)

1. **Review `allowFrom` configurations** — If relying on mutable name matching, migrate to stable IDs or set `dangerouslyAllowNameMatching=true`
2. **Control UI origin settings** — Non-loopback deployments need explicit `gateway.controlUi.allowedOrigins`
3. **Test reasoning suppression** — Verify thinking-capable models don't leak reasoning blocks in channel replies
4. **Review exec allowlists** — Busybox/toybox and wrapper command patterns may need re-approval
5. **Kimi provider testing** — Verify `kimi-coding/k2p5` works without 403 errors

---

*Generated: 2026-02-24*
