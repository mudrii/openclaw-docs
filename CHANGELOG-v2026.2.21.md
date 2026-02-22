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
- `openclaw security --audit` warns when browser sandbox runs on bridge network without source-range limits.

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
- `CHANGELOG-v2026.2.21.md` — this synthesized review.

---

*Generated: 2026-02-23*
