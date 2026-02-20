# OpenClaw Codebase Analysis — PART 5: Security, Plugins & Extensions

> Updated: 2026-02-20 | Version: v2026.2.19

## 1. `src/security/` — Security Guards, Audit, SSRF, Auth

### Purpose
Comprehensive security audit framework, content sanitization, skill/plugin code scanning, and filesystem permission hardening. The central defense-in-depth layer.

#### v2026.2.15 Changes
- **Skill scanner updates** — updated detection rules in `skill-scanner.ts` for broader pattern coverage
- **Redact sensitive status for non-admin** — `status.ts` now redacts sensitive fields when queried by non-admin users
- **Restrict skill download target paths** — new `src/infra/install-safe-path.ts` sanitizes install target directory names with `unscopedPackageName()`, `safeDirName()`, `safePathSegmentHashed()`; new `src/infra/path-safety.ts` provides `resolveSafeBaseDir()` and `isWithinDir()` for cross-platform path containment

#### v2026.2.19 Changes
- **SSRF hardening** — NAT64/6to4/Teredo IPv6 transition addresses and octal/hex/short/packed IPv4 representations now blocked in SSRF guard (see DEVELOPER-REFERENCE.md §9 for gotchas 33–45)
- **Gateway auth defaults** — Unresolved auth defaults to token mode with auto-generated token; security audit emits `gateway.http.no_auth` findings when `mode="none"` leaves APIs reachable
- **hooks.token ≠ gateway.auth.token** — Startup validation rejects matching tokens, preventing credential reuse
- **safeBins trusted dirs** — `tools.exec.safeBins` binaries must resolve from trusted bin directories; PATH-hijacked binaries rejected
- **Plugin discovery hardening** — Blocks unsafe plugin candidates (root escapes, world-writable directories, suspicious ownership)
- **Plugin/hook path containment** — Plugin and hook paths validated with `realpath` checks to prevent symlink escapes
- **Security headers** — `X-Content-Type-Options: nosniff` and `Referrer-Policy: no-referrer` added to gateway HTTP responses
- **ACP hardening** — Session rate limiting, idle reaping, and prompt size bounds (2 MiB) for Agent Client Protocol (see DEVELOPER-REFERENCE.md §6 for config reference)
- **Windows daemon cmd injection** — Hardened Windows daemon service commands against command injection

### Key Files

| File | Role |
|------|------|
| `audit.ts` | Main `SecurityAuditReport` orchestrator — runs all checks, probes gateway, aggregates findings |
| `audit-channel.ts` | Channel-specific security findings (open DMs, group policies, allowFrom validation) |
| `audit-extra.ts` | Re-export barrel splitting sync/async collectors |
| `audit-extra.sync.ts` | Config-only checks: secrets in config, sandbox docker noop, small model risk, gateway HTTP key overrides, hooks hardening, exposure matrix |
| `audit-extra.async.ts` | I/O checks: file permissions, installed skill code safety, plugin trust/code scanning |
| `audit-fs.ts` | Filesystem permission inspection (`inspectPathPermissions`, `safeStat`) — POSIX mode bits + Windows ACL |
| `audit-tool-policy.ts` | Sandbox tool policy allow/deny merging (`pickSandboxToolPolicy`) |
| `channel-metadata.ts` | Wraps untrusted channel metadata (usernames, bios) in safety boundaries |
| `dangerous-tools.ts` | Centralized constants: `DEFAULT_GATEWAY_HTTP_TOOL_DENY`, `DANGEROUS_ACP_TOOLS` — tools requiring approval |
| `external-content.ts` | Prompt injection defense: `detectSuspiciousPatterns`, `wrapExternalContent` with boundary markers |
| `fix.ts` | Auto-remediation: `chmod` state dirs, config files, OAuth dirs to safe permissions |
| `scan-paths.ts` | Path traversal guard: `isPathInside`, extension scanner path checks |
| `secret-equal.ts` | Timing-safe string comparison via `crypto.timingSafeEqual` |
| `skill-scanner.ts` | Static analysis scanner for skill/plugin code — detects dangerous patterns (eval, exec, fetch) |
| `windows-acl.ts` | Windows-specific ACL inspection via `icacls` |
| **New (src/infra/)** | |
| `install-safe-path.ts` | Sanitize skill/plugin install target paths — `unscopedPackageName()`, `safeDirName()`, `safePathSegmentHashed()` |
| `path-safety.ts` | Cross-platform path containment — `resolveSafeBaseDir()`, `isWithinDir()` |

### Exported API
- `runSecurityAudit()` → `SecurityAuditReport` (findings with severity: critical/warn/info)
- `runSecurityFix()` → `SecurityFixResult` (auto-remediation actions)
- `wrapExternalContent()`, `detectSuspiciousPatterns()` — injection defense
- `safeEqualSecret()` — timing-safe comparison
- `isPathInside()` — path traversal prevention
- `buildUntrustedChannelMetadata()` — safe metadata formatting
- `DEFAULT_GATEWAY_HTTP_TOOL_DENY`, `DANGEROUS_ACP_TOOLS` — risk constants

### Dependencies
Imports from: `agents/`, `browser/config`, `channels/`, `cli/`, `config/`, `gateway/`, `infra/`, `pairing/`, `plugins/`, `process/`, `routing/`

### Dependents
Used by: `cli/` (security command), `gateway/` (HTTP tool deny), `acp/` (dangerous tools check), `plugins/install` (scan paths), `web/` (media SSRF)

### Data Flow
```
Config + Filesystem → audit collectors → SecurityAuditFinding[] → SecurityAuditReport
                                                                   ↓
External content → detectSuspiciousPatterns → wrapExternalContent → safe LLM input
```

---

## 2. `src/plugins/` — Plugin System, Discovery, Loader, Hook Runner

### Purpose
Full plugin lifecycle: discovery, loading, validation, registration, hook execution, CLI commands, HTTP routes, config schema validation, install/uninstall/update.

#### v2026.2.15 Changes
- **LLM input/output hook payloads (#16724)** — `PluginHookLlmInputEvent` now includes `runId`, `sessionId`, `provider`, `model`, `systemPrompt`, `prompt`, `historyMessages`, `imagesCount`; `PluginHookLlmOutputEvent` includes `runId`, `sessionId`, `provider`, `model`, `assistantTexts`, `lastAssistant`, `usage` (with input/output/cacheRead/cacheWrite/total); both fired via `runVoidHook` in `hooks.ts`; new test file `wired-hooks-llm.test.ts`
- **Lazy-create jiti loader** — `loader.ts` now lazily initializes the `jiti` TypeScript/ESM loader on first use rather than eagerly at module load
- **Reuse `createEmptyPluginRegistry()`** — `registry.ts` now exports `createEmptyPluginRegistry()` for reuse (e.g., in tests and loader initialization)
- **Share plugin-sdk helpers** — see plugin-sdk section below for new shared helpers (auth, status, webhook paths)
- **Clear plugin manifest cache after install** — manifest registry now invalidates cached manifests after plugin install/uninstall to pick up changes immediately

#### v2026.2.19 Changes
- **Plugin discovery hardening** — Blocks unsafe plugin candidates: root escapes, world-writable directories, suspicious ownership
- **Plugin/hook path containment** — Plugin and hook paths validated with `realpath` checks to prevent symlink escapes
- **Plugin install records** — Install records now include `name`, `version`, `spec`, integrity, and shasum (`--pin` flag for npm plugins). See DEVELOPER-REFERENCE.md §5 for pre-PR checklist on plugin changes
- **Plugin uninstall** — Full plugin uninstall support added to CLI lifecycle

### Key Files

| File | Role |
|------|------|
| `bundled-dir.ts` | Resolves `extensions/` directory (npm dev vs bun compile) |
| `cli.ts` | Registers plugin-provided CLI commands into Commander |
| `commands.ts` | Plugin command registry — commands that bypass LLM agent |
| `config-schema.ts` | Empty plugin config schema factory |
| `config-state.ts` | Normalizes plugin config from `openclaw.json` (allow/deny lists, slots, entries) |
| `discovery.ts` | Discovers plugin candidates from bundled, global, workspace, config paths |
| `enable.ts` | Enable/disable plugin in config |
| `hook-runner-global.ts` | Singleton global hook runner, initialized at gateway startup |
| `hooks.ts` | Hook execution engine — priority ordering, error handling, async support for all lifecycle hooks |
| `http-path.ts` | Normalize plugin HTTP webhook paths |
| `http-registry.ts` | Register/deregister plugin HTTP route handlers |
| `install.ts` | Install plugins from npm, tarballs, or local paths |
| `installs.ts` | Record plugin install metadata in config |
| `loader.ts` | Main plugin loader — discovers, validates manifests, loads modules via jiti, builds registry |
| `manifest-registry.ts` | Pre-load manifest metadata without executing plugin code |
| `manifest.ts` | Read/parse `openclaw.plugin.json` manifests |
| `providers.ts` | Resolve provider plugins from registry |
| `registry.ts` | `PluginRegistry` creation — stores tools, hooks, channels, providers, HTTP routes, services |
| `runtime.ts` | Global singleton plugin registry state (get/set active registry) |
| `runtime/index.ts` | Plugin runtime context factory — provides API surface to plugins |
| `runtime/native-deps.ts` | Native dependency resolution for plugins |
| `runtime/types.ts` | Plugin runtime type definitions |
| `schema-validator.ts` | JSON Schema validation via Ajv for plugin configs |
| `services.ts` | Start/stop long-running plugin services |
| `slots.ts` | Plugin slot system (e.g., memory slot assignment) |
| `source-display.ts` | Format plugin source for display |
| `status.ts` | Plugin status reporting |
| `tools.ts` | Plugin tool registration helpers |
| `types.ts` | All plugin type definitions (hooks, commands, config schema, etc.) |
| `uninstall.ts` | Uninstall plugins |
| `update.ts` | Update plugins |
| `wired-hooks-llm.test.ts` | Tests for LLM input/output hook wiring (new in v2026.2.15) |

### Exported API
- `loadOpenClawPlugins()` → `PluginRegistry`
- `createHookRunner()` / `initializeGlobalHookRunner()` — lifecycle hook execution
- `installPlugin()`, `uninstallPlugin()`, `updatePlugin()`
- `registerPluginCommand()`, `registerPluginHttpRoute()`
- `discoverOpenClawPlugins()` — plugin discovery
- `setActivePluginRegistry()` / `getActivePluginRegistry()`

### Dependencies
Imports from: `agents/`, `channels/`, `cli/`, `compat/`, `config/`, `gateway/`, `hooks/`, `infra/`, `logging/`, `process/`, `security/`, `utils`

### Dependents
Used by: `gateway/` (startup loads plugins), `channels/` (channel plugins), `cli/` (plugin commands), `agents/` (tool registration), `web/` (HTTP routes)

### Data Flow
```
extensions/ dirs + config → discovery → manifest loading → jiti module loading
  → PluginRegistry { tools, hooks, channels, providers, httpRoutes, services }
    → hook-runner (lifecycle events) → agents/gateway consume registered items
```

---

## 3. `src/plugin-sdk/` — Plugin Development SDK

### Purpose
Utility library re-exported for plugin authors. Provides helpers without coupling plugins to internal modules.

#### v2026.2.15 Changes
- **New shared helper utilities:**
  - `status-helpers.ts` — `createDefaultChannelRuntimeState()`, `buildBaseChannelStatusSummary()` for standardized channel status reporting
  - `webhook-path.ts` — `normalizeWebhookPath()`, `resolveWebhookPath()` for consistent webhook URL/path handling
  - `agent-media-payload.ts` — `AgentMediaPayload` type and `buildAgentMediaPayload()` for structured media payloads
  - `provider-auth-result.ts` — `buildOauthProviderAuthResult()` for standardized OAuth provider auth results

### Key Files

| File | Role |
|------|------|
| `index.ts` | Main barrel — re-exports channel types, plugin types, tool types, and SDK utilities |
| `account-id.ts` | Re-exports `normalizeAccountId` from routing |
| `allow-from.ts` | `formatAllowFromLowercase()` — normalize allowFrom lists |
| `config-paths.ts` | `resolveChannelAccountConfigBasePath()` — config path resolution for channel accounts |
| `file-lock.ts` | File-based mutex lock (PID-aware, stale detection, reentrant) |
| `onboarding.ts` | `promptAccountId()` — interactive account selection for setup wizards |
| `text-chunking.ts` | `chunkTextForOutbound()` — split long messages at word/line boundaries |
| `status-helpers.ts` | `createDefaultChannelRuntimeState()`, `buildBaseChannelStatusSummary()` — standardized channel status (new) |
| `webhook-path.ts` | `normalizeWebhookPath()`, `resolveWebhookPath()` — webhook URL/path handling (new) |
| `agent-media-payload.ts` | `AgentMediaPayload` type, `buildAgentMediaPayload()` — structured media payloads (new) |
| `provider-auth-result.ts` | `buildOauthProviderAuthResult()` — OAuth provider auth result builder (new) |

### Dependencies
Imports from: `channels/plugins/`, `config/`, `routing/`, `wizard/`

### Dependents
Used by: All `extensions/` plugins import from `@openclaw/plugin-sdk` (mapped to this)

---

## 4. `src/acp/` — Agent Client Protocol

### Purpose
Implements the ACP (Agent Client Protocol) server and client for IDE/editor integration. Bridges ACP sessions to gateway sessions.

#### v2026.2.19 Changes
- **Session rate limiting** — ACP sessions now rate-limited to prevent abuse
- **Idle reaping** — Idle ACP sessions automatically reaped to free resources
- **Prompt size bounds** — Prompt input capped at 2 MiB to prevent memory exhaustion
- See DEVELOPER-REFERENCE.md §9 (gotchas 33–45) for related security hardening details

### Key Files

| File | Role |
|------|------|
| `index.ts` | Barrel: exports `serveAcpGateway`, `createInMemorySessionStore` |
| `server.ts` | ACP server — connects to gateway WebSocket as a client, bridges ACP ↔ gateway |
| `client.ts` | ACP client — permission resolver that auto-approves safe tools, prompts for dangerous ones |
| `translator.ts` | `AcpGatewayAgent` — translates between ACP protocol events and gateway methods |
| `session.ts` | In-memory session store with abort controller tracking |
| `session-mapper.ts` | Maps ACP session metadata (labels, keys) to gateway session keys |
| `event-mapper.ts` | Converts ACP `ContentBlock[]` to gateway text + attachments |
| `commands.ts` | Available ACP commands list (help, status, model, think, etc.) |
| `meta.ts` | Type-safe metadata readers (`readString`, `readBool`, `readNumber`) |
| `types.ts` | ACP type definitions |

### Dependencies
Imports from: `@agentclientprotocol/sdk`, `config/`, `gateway/`, `infra/`, `security/dangerous-tools`, `utils/`

### Dependents
Used by: CLI `openclaw acp` command

### Data Flow
```
IDE/Editor → ACP SDK (stdin/stdout ndjson) → AcpGatewayAgent → GatewayClient (WebSocket)
  → Gateway processes request → response flows back through translator → ACP client
```

---

## 5. `src/pairing/` — Device Pairing

### Purpose
Manages device pairing flow — generates codes, stores pending requests, validates approvals for channel access control.

### Key Files

| File | Role |
|------|------|
| `pairing-store.ts` | File-based pairing store with file locking — generates codes, stores/approves/rejects requests |
| `pairing-messages.ts` | Formats pairing reply messages with codes and CLI approve commands |
| `pairing-labels.ts` | Resolves channel-specific ID labels for pairing prompts |

### Dependencies
Imports from: `channels/plugins/`, `cli/`, `config/paths`, `infra/file-lock`, `infra/home-dir`

### Dependents
Used by: `security/audit-channel` (reads allowFrom store), `security/fix` (reads channel allow store), `channels/` (pairing flow)

---

## 6. `src/node-host/` — Node Hosting & Remote Exec

### Purpose
Runs OpenClaw as a paired "node" device — connects to gateway, executes commands remotely, provides browser proxy. As of v2026.2.19, includes APNs wake support and Apple Watch companion integration.

#### v2026.2.19 Changes
- **APNs wake support** — Node host supports APNs wake-before-invoke for iOS/macOS nodes
- **Apple Watch companion** — New `watch-companion/` module for glanceable status, quick actions, haptic notifications
- **Plaintext ws:// blocked** — WebSocket connections to non-loopback hosts must use `wss://`

### Key Files

| File | Role |
|------|------|
| `config.ts` | Node host config (nodeId, token, gateway connection) — read/write `node.json` |
| `runner.ts` | Main node host runner — connects to gateway as node client, handles invoke requests |
| `invoke.ts` | Command execution handler — validates against exec allowlists, spawns processes, caps output |
| `invoke-browser.ts` | Browser proxy for remote nodes — routes browser commands through local browser control |
| `with-timeout.ts` | Generic timeout wrapper with AbortController |

### Dependencies
Imports from: `agents/`, `browser/`, `config/`, `gateway/`, `infra/`, `utils/`

### Dependents
Used by: CLI `openclaw node` command

---

## 7. `src/process/` — Process Management

### Purpose
Process spawning, command queuing with lanes, child process lifecycle management.

### Key Files

| File | Role |
|------|------|
| `exec.ts` | Core exec functions — `runExec`, `runCommandWithTimeout`, Windows command resolution |
| `command-queue.ts` | Lane-based command queue — serializes execution per lane (main/cron/subagent/nested) with concurrency control |
| `lanes.ts` | `CommandLane` enum: Main, Cron, Subagent, Nested |
| `child-process-bridge.ts` | Signal forwarding bridge (SIGTERM/SIGINT/SIGHUP) from parent to child |
| `spawn-utils.ts` | Spawn helpers: stdio resolution, error formatting, fallback spawn with retry |
| `restart-recovery.ts` | Restart iteration hook for in-process restart loops |

### Dependencies
Imports from: `globals`, `logger`, `logging/diagnostic`

### Dependents
Used by: `node-host/invoke` (exec), `security/fix` (runExec), `plugins/install` (runCommandWithTimeout), `gateway/` (command queue)

---

## 8. `src/macos/` — macOS-Specific

### Purpose
macOS app integration — gateway daemon entry point, Swift relay bridge, deep links.

### Key Files

| File | Role |
|------|------|
| `gateway-daemon.ts` | Entry point for macOS bundled gateway daemon — version handling, Bun Long polyfill, starts gateway |
| `relay.ts` | Entry point for macOS relay process — version, smoke tests, starts relay |
| `relay-smoke.ts` | QR smoke test for relay validation |

### Dependencies
Imports from: `infra/gateway-lock`, `infra/process-respawn`, `web/qr-image`

### Dependents
Used by: macOS app (compiled entry points)

---

## 9. `src/wizard/` — Setup Wizard & Onboarding

### Purpose
Interactive CLI setup wizard using `@clack/prompts` for initial configuration.

### Key Files

| File | Role |
|------|------|
| `onboarding.ts` | Main onboarding flow orchestration |
| `onboarding.gateway-config.ts` | Gateway configuration step |
| `onboarding.finalize.ts` | Finalization step — writes config, starts services |
| `onboarding.completion.ts` | Completion messages and next steps |
| `onboarding.types.ts` | Onboarding type definitions |
| `clack-prompter.ts` | Adapter: `@clack/prompts` → `WizardPrompter` interface |
| `prompts.ts` | `WizardPrompter` interface definition (select, text, confirm, etc.) |
| `session.ts` | Wizard session state management |

### Dependencies
Imports from: `@clack/prompts`, `cli/`, `config/`, `channels/`, `plugins/`

### Dependents
Used by: CLI `openclaw init` / `openclaw setup`

---

## 10. `src/logging/` — Logging Subsystem

### Purpose
Structured logging with tslog, subsystem filtering, console patching, log rotation, diagnostic session tracking, and sensitive data redaction.

### Key Files

| File | Role |
|------|------|
| `logger.ts` | Core logger setup — tslog wrapper, file rotation (24h), log dir resolution |
| `subsystem.ts` | `createSubsystemLogger()` — per-module logger with console/file filtering |
| `console.ts` | Console settings resolution, console patching (redirect to logger) |
| `config.ts` | Read logging config from `openclaw.json` |
| `levels.ts` | Log level definitions and normalization (silent→trace) |
| `state.ts` | Global mutable logging state (cached logger, console patches) |
| `diagnostic.ts` | Diagnostic logger — webhook stats, session state tracking, lane metrics |
| `diagnostic-session-state.ts` | Per-session processing state tracking with TTL pruning |
| `parse-log-line.ts` | Parse structured JSON log lines back into components |
| `redact.ts` | Sensitive data redaction — regex patterns for API keys, tokens, PEM blocks |
| `redact-identifier.ts` | SHA256-based identifier redaction |

### Dependencies
Imports from: `config/`, `globals`, `infra/`, `runtime`, `terminal/`

### Dependents
Used by: Nearly every module (via `createSubsystemLogger`)

---

## 11. `src/utils/` — Shared Utilities

### Purpose
Grab-bag of small, focused utility functions used across the codebase.

### Key Files

| File | Role |
|------|------|
| `account-id.ts` | `normalizeAccountId()` |
| `boolean.ts` | `parseBooleanValue()` — flexible boolean parsing |
| `delivery-context.ts` | `normalizeDeliveryContext()` — channel/target/thread routing |
| `directive-tags.ts` | Parse inline directives (`[[audio_as_voice]]`, `[[reply_to:...]]`) |
| `fetch-timeout.ts` | Fetch wrapper with AbortController timeout |
| `message-channel.ts` | Channel normalization, markdown capability detection, gateway client constants |
| `normalize-secret-input.ts` | Strip line breaks from pasted credentials |
| `provider-utils.ts` | `isReasoningTagProvider()` — provider-specific reasoning mode detection |
| `queue-helpers.ts` | Queue state types and text elision |
| `reaction-level.ts` | Reaction level parsing (off/ack/minimal/extensive) |
| `safe-json.ts` | `safeJsonStringify()` — handles BigInt, Error, Uint8Array |
| `shell-argv.ts` | Shell argument splitting (respects quotes, escapes) |
| `transcript-tools.ts` | Extract tool call names from transcript messages |
| `usage-format.ts` | Token count and cost formatting |

### Dependencies
Imports from: `agents/`, `channels/`, `config/`, `gateway/`, `plugins/`

### Dependents
Used by: Broadly across the codebase

---

## 12. `src/shared/` — Shared Types & Constants

### Purpose
Pure types, constants, and small pure functions shared across modules with minimal dependencies.

### Key Files

| File | Role |
|------|------|
| `chat-envelope.ts` | Parse channel envelope headers from messages |
| `config-eval.ts` | `isTruthy()`, `resolveConfigPath()` — config value utilities |
| `device-auth.ts` | Device auth types and normalizers |
| `entry-metadata.ts` | Resolve emoji/homepage from metadata + frontmatter |
| `frontmatter.ts` | Frontmatter parsing, string list normalization |
| `model-param-b.ts` | `inferParamBFromIdOrName()` — extract parameter count from model names (e.g., "70b") |
| `net/ipv4.ts` | IPv4 address validation |
| `node-match.ts` | Node matching by ID, display name, or IP |
| `requirements.ts` | Binary/env/config requirement checking |
| `subagents-format.ts` | Duration and token formatting for subagent displays |
| `text/reasoning-tags.ts` | Parse/strip `<think>`/`<final>` reasoning tags |
| `usage-aggregates.ts` | Usage aggregation types |

### Dependencies
Minimal — imports from `compat/legacy-names`, `utils/boolean`

### Dependents
Used by: `security/`, `logging/`, `agents/`, `config/`, `plugins/`

---

## 13. `src/compat/` — Compatibility Layers

### Purpose
Single file managing project name constants and legacy name mappings.

### Key Files

| File | Role |
|------|------|
| `legacy-names.ts` | `PROJECT_NAME`, `MANIFEST_KEY`, legacy project/manifest/plugin names, macOS app paths |

### Dependencies
None

### Dependents
Used by: `plugins/manifest`, `shared/frontmatter`, `plugins/install`, `security/audit-extra`

---

## 14. `src/scripts/` — Build/Maintenance Scripts

### Purpose
Build-time utility scripts (test-only files found).

### Key Files

| File | Role |
|------|------|
| `canvas-a2ui-copy.test.ts` | Test for canvas A2UI copy script |

*Note: No production `.ts` files found — scripts may be in other formats or executed differently.*

---

## 15. `src/web/` — Web/WhatsApp Layer

### Purpose
WhatsApp Web integration via Baileys — session management, message monitoring, media handling, auto-reply orchestration, QR login, outbound messaging.

### Key Files

| File | Role |
|------|------|
| `session.ts` | WhatsApp Web socket creation via Baileys, auth state management |
| `login.ts` | WhatsApp QR login flow |
| `login-qr.ts` | QR code generation for login |
| `outbound.ts` | Send messages via WhatsApp (markdown conversion, media, polls) |
| `media.ts` | Media download, HEIC conversion, image optimization, SSRF policy |
| `accounts.ts` | WhatsApp account resolution |
| `auth-store.ts` | Credentials backup/restore for WhatsApp |
| `active-listener.ts` | Active WhatsApp listener registry |
| `reconnect.ts` | Reconnection logic |
| `vcard.ts` | vCard formatting |
| `qr-image.ts` | QR PNG rendering |
| `auto-reply.ts` / `auto-reply.impl.ts` | Auto-reply barrel exports |
| `auto-reply/monitor.ts` | Main WhatsApp channel monitor |
| `auto-reply/monitor/on-message.ts` | Message handler — routing, activation, gating |
| `auto-reply/monitor/process-message.ts` | Message processing pipeline |
| `auto-reply/monitor/broadcast.ts` | Broadcast group handling |
| `auto-reply/monitor/group-activation.ts` | Group mention/activation detection |
| `auto-reply/monitor/group-gating.ts` | Group access control |
| `auto-reply/monitor/commands.ts` | Slash command processing |
| `auto-reply/heartbeat-runner.ts` | Heartbeat/cron execution |
| `auto-reply/deliver-reply.ts` | Reply delivery with chunking |
| `auto-reply/mentions.ts` | @mention detection |
| `auto-reply/session-snapshot.ts` | Session state snapshots |
| `inbound.ts` / `inbound/*.ts` | Inbound message extraction, dedup, access control, media handling |

### Dependencies
Imports from: `@whiskeysockets/baileys`, `channels/`, `cli/`, `config/`, `gateway/`, `infra/`, `logging/`, `markdown/`, `media/`, `security/`

### Dependents
Used by: `channels/plugins/whatsapp`, `macos/relay`

---

## 16. `src/terminal/` — Terminal Utilities

### Purpose
ANSI styling, tables, progress lines, and terminal UI helpers for CLI output.

### Key Files

| File | Role |
|------|------|
| `ansi.ts` | `stripAnsi()` — remove ANSI SGR codes and OSC-8 hyperlinks |
| `health-style.ts` | Health status styling |
| `links.ts` | Terminal hyperlink formatting (OSC-8) |
| `note.ts` | Boxed note formatting |
| `palette.ts` | Color palette |
| `progress-line.ts` | Single-line progress indicator |
| `prompt-style.ts` | Prompt styling |
| `restore.ts` | Terminal state restoration |
| `stream-writer.ts` | Stream writing utilities |
| `table.ts` | Table formatting |
| `theme.ts` | Theme configuration |

### Dependencies
Imports from: `chalk`

### Dependents
Used by: `logging/`, `cli/`

---

## 17. `src/docs/` — Documentation Generation

### Purpose
Documentation generation (only test files found).

### Key Files

| File | Role |
|------|------|
| `slash-commands-doc.test.ts` | Tests for slash command documentation generation |

---

## 18. `extensions/` — All Extension Directories

### Channel Plugins (messaging platforms)

| Extension | Description |
|-----------|-------------|
| `bluebubbles/` | BlueBubbles (iMessage bridge) channel |
| `discord/` | Discord channel |
| `feishu/` | Feishu/Lark channel (community: @m1heng) |
| `googlechat/` | Google Chat channel |
| `imessage/` | iMessage channel (native macOS) |
| `irc/` | IRC channel |
| `line/` | LINE channel |
| `matrix/` | Matrix channel |
| `mattermost/` | Mattermost channel |
| `msteams/` | Microsoft Teams channel |
| `nextcloud-talk/` | Nextcloud Talk channel |
| `nostr/` | Nostr (NIP-04 encrypted DMs) channel |
| `signal/` | Signal channel |
| `slack/` | Slack channel |
| `telegram/` | Telegram channel |
| `tlon/` | Tlon/Urbit channel |
| `twitch/` | Twitch channel |
| `voice-call/` | Voice call plugin |
| `whatsapp/` | WhatsApp channel |
| `zalo/` | Zalo (Official Account) channel |
| `zalouser/` | Zalo (Personal Account via zca-cli) channel |

### Provider Plugins (auth/model providers)

| Extension | Description |
|-----------|-------------|
| `copilot-proxy/` | Copilot Proxy provider |
| `google-antigravity-auth/` | Google Antigravity OAuth provider |
| `google-gemini-cli-auth/` | Gemini CLI OAuth provider |
| `minimax-portal-auth/` | MiniMax Portal OAuth provider |
| `qwen-portal-auth/` | Qwen Portal OAuth provider |

### Tool/Feature Plugins

| Extension | Description |
|-----------|-------------|
| `device-pair/` | Device pairing flow (bundled, enabled by default) |
| `diagnostics-otel/` | OpenTelemetry diagnostics exporter |
| `llm-task/` | JSON-only LLM task tool |
| `lobster/` | Lobster workflow tool (typed pipelines + resumable approvals) |
| `memory-core/` | Core memory search |
| `memory-lancedb/` | LanceDB-backed long-term memory with auto-recall |
| `open-prose/` | OpenProse VM skill pack |
| `phone-control/` | Phone control (bundled, enabled by default) |
| `talk-voice/` | Talk voice (bundled, enabled by default) |
| `thread-ownership/` | Thread ownership management |

### Extension Structure (typical)
```
extensions/<name>/
├── index.ts              # Plugin entry point — exports definePlugin()
├── package.json          # @openclaw/<name>, description
├── openclaw.plugin.json  # Manifest: id, configSchema, kind, channels
├── src/
│   ├── channel.ts        # Channel adapter implementation
│   ├── runtime.ts        # Runtime/connection management
│   ├── accounts.ts       # Multi-account support
│   ├── config-schema.ts  # Zod/JSON schema for plugin config
│   └── ...
└── tsconfig.json
```

### Bundled-by-Default Extensions
Per `config-state.ts`: `device-pair`, `phone-control`, `talk-voice` are enabled by default without explicit config.

---

## Cross-Module Dependency Summary

```
                    ┌─────────────┐
                    │  extensions/ │ (uses plugin-sdk)
                    └──────┬──────┘
                           │
              ┌────────────▼────────────┐
              │      plugins/           │ (loads, registers, runs hooks)
              │  plugin-sdk/ (API)      │
              └────────────┬────────────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
    ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
    │  security/ │   │   acp/    │   │ node-host/ │
    │  (audit)   │   │  (bridge) │   │  (remote)  │
    └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
              ┌────────────▼────────────┐
              │    process/ + logging/  │
              │  shared/ + utils/ +     │
              │  compat/ + terminal/    │
              └─────────────────────────┘
```
