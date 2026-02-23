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

### Google Vertex AI for Claude Models
Claude models can now route through Google Vertex AI (PR #23985).

**Usage:** Use `vertex-ai/claude-*` model references.

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
