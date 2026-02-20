# OpenClaw Core Architecture — Part 1: Module Analysis

**Updated:** 2026-02-20 | **Version:** v2026.2.19  
**Codebase:** ~/src/openclaw  
**Total lines (6 modules):** ~94,080

---

## 1. `src/gateway/` — HTTP/WebSocket Server & API Endpoints

**Lines:** ~83,178 | **Files:** ~120+ .ts files  
**Purpose:** The heart of OpenClaw — runs the gateway server that accepts WebSocket connections from CLI/plugins, exposes HTTP endpoints (OpenAI-compatible API, control UI), manages agent sessions, chat routing, cron, browser control, node subscriptions, and plugin lifecycle.

#### v2026.2.19 Changes
- **Gateway auth defaults** — Unresolved auth defaults to token mode with auto-generated token; explicit `mode: "none"` required for open loopback. See DEVELOPER-REFERENCE.md §6 for config reference
- **hooks.token ≠ gateway.auth.token** — Startup validation rejects matching tokens
- **Rate-limited control-plane RPCs** — `config.apply`, `config.patch`, `update.run` limited to 3/min per device+IP with 30s restart coalesce
- **Security headers** — `X-Content-Type-Options: nosniff` and `Referrer-Policy: no-referrer` on all HTTP responses
- **Plaintext ws:// blocked** — WebSocket connections to non-loopback hosts must use `wss://`
- **Config change audit logging** — Actor, device, IP, and changed paths now logged on config mutations
- **Drain-before-restart** — Gateway restart coalesced with cooldown to allow in-flight requests to complete

### Key Files & Roles

| File | Role |
|------|------|
| `server.ts` | Barrel — re-exports `startGatewayServer`, `GatewayServer` |
| `server.impl.ts` | Main server setup — wires all subsystems, starts HTTP/WS |
| `boot.ts` | `runBootOnce()` — one-shot gateway boot for CLI commands |
| `server-http.ts` | HTTP route registration |
| `server/http-listen.ts` | Bind HTTP server to port |
| `server/tls.ts` | TLS certificate handling |
| `server/ws-connection.ts` | WebSocket connection lifecycle |
| `server/ws-connection/auth-messages.ts` | WS auth handshake |
| `server/ws-connection/message-handler.ts` | WS message dispatch |
| `server/ws-types.ts` | WS type definitions |
| `server/close-reason.ts` | `truncateCloseReason()` |
| `server/health-state.ts` | Health check state |
| `server/hooks.ts` | Server lifecycle hooks |
| `server/plugins-http.ts` | Plugin HTTP route mounting |
| **Chat & Sessions** | |
| `server-chat.ts` | Chat message processing pipeline |
| `server-session-key.ts` | Session key resolution for incoming messages |
| `session-utils.ts` / `session-utils.fs.ts` | Session file I/O, history |
| `sessions-resolve.ts` | Session resolution logic |
| `sessions-patch.ts` | Session metadata patching |
| `chat-abort.ts` | Abort in-flight chat requests |
| `chat-attachments.ts` | Attachment handling |
| `chat-sanitize.ts` | Input sanitization |
| **Server Methods (WS RPC)** | |
| `server-methods.ts` | Method registry |
| `server-methods-list.ts` | Method listing |
| `server-methods/*.ts` | 30+ individual RPC handlers: `agent.ts`, `chat.ts`, `config.ts`, `connect.ts`, `cron.ts`, `devices.ts`, `exec-approval.ts`, `health.ts`, `logs.ts`, `models.ts`, `nodes.ts`, `send.ts`, `sessions.ts`, `skills.ts`, `system.ts`, `talk.ts`, `tts.ts`, `update.ts`, `usage.ts`, `voicewake.ts`, `web.ts`, `wizard.ts`, `browser.ts`, `channels.ts` |
| **Protocol Schema** | |
| `protocol/index.ts` | Protocol barrel |
| `protocol/schema.ts` | Combined schema |
| `protocol/schema/*.ts` | Zod schemas for frames, agents, channels, config, cron, devices, exec-approvals, logs, nodes, sessions, snapshot, wizard |
| `protocol/client-info.ts` | Client identification |
| **Subsystems** | |
| `server-browser.ts` | Browser control server |
| `server-channels.ts` | Channel plugin management |
| `server-cron.ts` | Cron job scheduling |
| `server-lanes.ts` | Request lane management (concurrency) |
| `server-maintenance.ts` | Maintenance mode |
| `server-model-catalog.ts` | Model discovery/catalog |
| `server-plugins.ts` | Plugin initialization |
| `server-broadcast.ts` | WS broadcast to connected clients |
| `server-discovery.ts` / `server-discovery-runtime.ts` | Bonjour/mDNS discovery |
| `server-mobile-nodes.ts` | Mobile node management |
| `server-node-events.ts` / `server-node-events-types.ts` | Node event bus |
| `server-node-subscriptions.ts` | Node event subscriptions |
| `server-tailscale.ts` | Tailscale integration |
| `server-wizard-sessions.ts` | Setup wizard |
| `server-ws-runtime.ts` | WS runtime state |
| **Auth & Security** | |
| `auth.ts` | Token auth |
| `auth-rate-limit.ts` | Rate limiting |
| `device-auth.ts` | Device authentication |
| `http-auth-helpers.ts` | HTTP auth utilities |
| `origin-check.ts` | Origin validation |
| `probe-auth.ts` / `probe.ts` | Health probes |
| **OpenAI Compatibility** | |
| `openai-http.ts` | OpenAI-compatible HTTP API (`/v1/chat/completions`) |
| `openresponses-http.ts` | Open Responses API |
| `open-responses.schema.ts` | Schema for Open Responses |
| **Misc** | |
| `agent-prompt.ts` | Agent system prompt assembly |
| `assistant-identity.ts` | Assistant identity injection |
| `call.ts` | Outbound LLM call orchestration |
| `client.ts` | Gateway client (for connecting to remote gateways) |
| `config-reload.ts` | Hot config reload |
| `control-ui.ts` / `control-ui-shared.ts` | Web control panel |
| `exec-approval-manager.ts` | Exec approval workflow |
| `hooks.ts` / `hooks-mapping.ts` | Hook execution |
| `live-image-probe.ts` | Image URL probing |
| `net.ts` | Network utilities |
| `node-command-policy.ts` | Node command security |
| `node-invoke-sanitize.ts` | Node invocation sanitization |
| `node-registry.ts` | Connected node registry |
| `server-close.ts` | Graceful shutdown |
| `server-reload-handlers.ts` | Reload event handlers |
| `server-restart-sentinel.ts` | Restart detection |
| `server-runtime-config.ts` | Runtime config state |
| `server-runtime-state.ts` | Runtime state container |
| `server-startup.ts` / `server-startup-log.ts` / `server-startup-memory.ts` | Startup sequence |
| `server-shared.ts` / `server-utils.ts` / `server-constants.ts` | Shared utilities |
| `tools-invoke-http.ts` | Tool invocation over HTTP |
| `ws-log.ts` / `ws-logging.ts` | WS logging |

### Key Exports
- `startGatewayServer(opts): GatewayServer` — main entry point
- `runBootOnce(params)` — one-shot boot for CLI
- `GatewayServer` type — server handle with `close()`, state accessors
- `truncateCloseReason()` — WS close reason helper
- Protocol schemas (Zod) for all WS frame types

### Dependencies (imports from)
- `config/` (81 imports) — config types, loading, sessions, validation
- `infra/` (101 imports) — utilities, restart, heartbeat, exec approvals, outbound messaging
- `agents/` — agent scope, subagent registry, skills
- `channels/` — channel plugins
- `plugins/` — plugin registry, hook runner
- `routing/` (18 imports) — session key resolution, bindings
- `auto-reply/` — reply dispatcher
- `logging/` — subsystem logger
- `cli/` — CLI deps
- `process/` — command queue

### Dependents (who imports gateway/)
- `commands/` (24) — CLI commands that start/manage gateway
- `cli/` (19) — CLI entry points
- `agents/` (15) — agent runtime needs gateway context
- `acp/` (7), `security/` (6), `browser/` (6), `tui/` (6)

### Data Flow
```
Incoming: Channel plugin → WS frame → server/ws-connection → message-handler → server-methods/* → server-chat → call.ts → LLM provider
                                                                                                     ↓
Outgoing: LLM response → chat pipeline → outbound/deliver → channel plugin → user
                                                                                                     
HTTP:     Client → openai-http.ts or server/plugins-http → server-chat → same pipeline
```

---

## 2. `src/config/` — Configuration Loading, Schema & Validation

**Lines:** ~48,565 | **Files:** ~85 .ts files  
**Purpose:** Loads, validates, merges, and provides access to `openclaw.json` configuration. Defines all config types, Zod schemas, session management, legacy migration, and path resolution.

### Key Files & Roles

| File | Role |
|------|------|
| `config.ts` | Barrel — `loadConfig()`, `clearConfigCache()`, `createConfigIO()`, `parseConfigJson5()`, re-exports types, paths, runtime-overrides |
| `io.ts` | Low-level config file I/O (read/write JSON5) |
| `types.ts` | Barrel re-exporting all `types.*.ts` files |
| `types.base.ts` | `OpenClawConfig` root type |
| `types.agents.ts` | Agent config types |
| `types.channels.ts` | Channel config types |
| `types.gateway.ts` | Gateway config types |
| `types.telegram.ts` / `.discord.ts` / `.slack.ts` / `.signal.ts` / `.whatsapp.ts` / `.imessage.ts` / `.irc.ts` / `.googlechat.ts` / `.msteams.ts` | Per-channel config types |
| `types.models.ts` | Model/provider config types |
| `types.tools.ts` / `.skills.ts` / `.hooks.ts` / `.cron.ts` / `.memory.ts` / `.queue.ts` / `.sandbox.ts` / `.browser.ts` / `.tts.ts` / `.auth.ts` / `.approvals.ts` / `.plugins.ts` / `.openclaw.ts` / `.node-host.ts` / `.messages.ts` / `.agent-defaults.ts` | Feature-specific config types |
| **Zod Schemas** | |
| `zod-schema.ts` | `OpenClawSchema` — main Zod schema barrel |
| `zod-schema.core.ts` | Core schema definitions |
| `zod-schema.agents.ts` | Agent schema |
| `zod-schema.channels.ts` | Channel schemas |
| `zod-schema.providers.ts` / `.providers-core.ts` / `.providers-whatsapp.ts` | Provider schemas |
| `zod-schema.session.ts` | Session config schema |
| `zod-schema.hooks.ts` | Hook schemas |
| `zod-schema.approvals.ts` | Approval schemas |
| `zod-schema.allowdeny.ts` | Allow/deny list schemas |
| `zod-schema.sensitive.ts` | Sensitive field schemas |
| `zod-schema.agent-defaults.ts` / `.agent-runtime.ts` | Agent default/runtime schemas |
| **Validation** | |
| `validation.ts` | `validateConfigObject()`, `validateConfigObjectWithPlugins()` |
| `schema.ts` | `buildConfigSchema()` — JSON Schema generation |
| `schema.help.ts` / `schema.hints.ts` / `schema.labels.ts` / `schema.irc.ts` | Schema UI hints and help text |
| **Sessions** | |
| `sessions.ts` | Barrel re-exporting `sessions/*.ts` |
| `sessions/store.ts` | Session store (CRUD) |
| `sessions/main-session.ts` | `resolveMainSessionKey()`, `resolveAgentMainSessionKey()` |
| `sessions/session-key.ts` | Session key normalization |
| `sessions/paths.ts` | Session file path resolution |
| `sessions/reset.ts` | Session reset policies |
| `sessions/group.ts` | Group session handling |
| `sessions/metadata.ts` | Session metadata derivation |
| `sessions/transcript.ts` | Transcript file handling |
| `sessions/delivery-info.ts` | Delivery info extraction |
| `sessions/types.ts` | Session type definitions |
| **Merging & Migration** | |
| `merge-config.ts` | Config merging logic |
| `merge-patch.ts` | JSON Merge Patch |
| `includes.ts` / `includes-scan.ts` | Config file includes |
| `legacy.ts` / `legacy-migrate.ts` / `legacy.migrations.ts` / `legacy.migrations.part-1/2/3.ts` / `legacy.rules.ts` / `legacy.shared.ts` | Legacy config migration |
| **Misc** | |
| `defaults.ts` | Default config values |
| `paths.ts` / `config-paths.ts` / `normalize-paths.ts` | Path utilities |
| `commands.ts` | Custom command definitions |
| `telegram-custom-commands.ts` | Telegram bot commands |
| `env-substitution.ts` / `env-vars.ts` / `env-preserve.ts` | Environment variable handling |
| `cache-utils.ts` | Config caching |
| `channel-capabilities.ts` | Channel capability detection |
| `agent-dirs.ts` | Agent directory resolution |
| `agent-limits.ts` | Agent resource limits |
| `backup-rotation.ts` | Config backup rotation |
| `group-policy.ts` | Group message policies |
| `logging.ts` | Config-related logging |
| `markdown-tables.ts` | Markdown table formatting |
| `plugin-auto-enable.ts` | Auto-enable plugins |
| `port-defaults.ts` | Default port assignments |
| `redact-snapshot.ts` | Redact sensitive config for snapshots |
| `runtime-overrides.ts` | Runtime config overrides |
| `talk.ts` | Talk mode config |
| `version.ts` | Config version tracking |

### Key Exports
- `loadConfig(opts?): OpenClawConfig` — load and validate config
- `createConfigIO()` — config file read/write handle
- `parseConfigJson5(text)` — parse JSON5 config text
- `clearConfigCache()` — invalidate cached config
- `validateConfigObject(raw)` / `validateConfigObjectWithPlugins(raw)` — validate config
- `OpenClawConfig` (type) — the main config type
- `OpenClawSchema` — Zod schema for full validation
- `buildConfigSchema()` — generate JSON Schema
- `migrateLegacyConfig()` — migrate old configs
- All session utilities (`resolveMainSessionKey`, `resolveSessionTranscriptPath`, etc.)
- All `types.*` re-exports (agent, channel, model, tool config types)

### Dependencies (imports from)
- `channels/` (24) — channel type definitions
- `infra/` (14) — home dir, errors, utilities
- `agents/` (12) — agent scope
- `routing/` (8) — session key functions
- `plugins/` (8) — plugin types
- `cli/` (6) — CLI format helpers
- `logging/` (4)

### Dependents (who imports config/)
- `commands/` (151), `agents/` (138), `auto-reply/` (98), `gateway/` (81), `infra/` (60), `telegram/` (46), `channels/` (36), `cli/` (31), `web/` (29), `plugins/` (24) — **most-imported module in the codebase**

### Data Flow
```
openclaw.json → io.ts (read) → parseConfigJson5 → merge-config (includes) → validation.ts (Zod) → OpenClawConfig object
                                                                                ↓
                                                              legacy-migrate (if old format)
                                                                                ↓
                                                              Cached in memory → consumed by gateway, agents, channels, CLI
```

---

## 3. `src/routing/` — Session Key Resolution & Message Routing

**Lines:** ~1,606 (3 files) | **Files:** 3 .ts files  
**Purpose:** Resolves which agent handles a message based on channel, chat type, account, and configured bindings. Builds session keys that uniquely identify conversations.

### Files

| File | Role |
|------|------|
| `resolve-route.ts` | `resolveAgentRoute()` — given channel/peer/group info, determines which agent and session key to use; `buildAgentSessionKey()` |
| `session-key.ts` | Session key primitives: `normalizeMainKey()`, `toAgentStoreSessionKey()`, `resolveAgentIdFromSessionKey()`, `classifySessionKeyShape()`, `buildAgentPeerSessionKey()`, `buildGroupHistoryKey()`, `resolveThreadSessionKeys()`, `normalizeAgentId()`, `sanitizeAgentId()`, `normalizeAccountId()`, `buildAgentMainSessionKey()` + constants `DEFAULT_AGENT_ID`, `DEFAULT_MAIN_KEY`, `DEFAULT_ACCOUNT_ID` |
| `bindings.ts` | `listBindings()`, `listBoundAccountIds()`, `resolveDefaultAgentBoundAccountId()`, `buildChannelAccountBindings()`, `resolvePreferredAccountId()` — maps agents to channel accounts |

### Key Types
- `RoutePeer` — `{ id, name, chatType }`
- `ResolveAgentRouteInput` — channel, account, peer, group info
- `ResolvedAgentRoute` — resolved agent ID, session key, account
- `AgentBinding` — agent-to-channel binding
- `SessionKeyShape` — `"missing" | "agent" | "legacy_or_alias" | "malformed_agent"`

### Dependencies
- `config/` (8) — `OpenClawConfig` type
- `channels/` (10) — `ChatType`, channel normalization
- `agents/` (4) — `resolveDefaultAgentId`
- `sessions/` (2) — session utilities

### Dependents
- `agents/` (21), `gateway/` (18), `commands/` (16), `web/` (12), `telegram/` (12), `channels/` (11), `discord/` (9), `infra/` (8), `auto-reply/` (8), `slack/` (7)

### Data Flow
```
Incoming message (channel, sender, group) → resolveAgentRoute() → { agentId, sessionKey, accountId }
                                                ↓
                                         Uses bindings.ts to check agent↔channel mappings
                                         Uses session-key.ts to build canonical key
                                                ↓
                                         Gateway uses result to route to correct agent + session
```

---

## 4. `src/infra/` — Infrastructure Utilities

**Lines:** ~71,198 | **Files:** ~130+ .ts files  
**Purpose:** The utility layer — everything from retry logic, restart management, error handling, home directory resolution, outbound message delivery, exec approvals, heartbeat, update checking, device pairing, network utilities, provider usage tracking, and more.

### Major Subsystems

#### Outbound Messaging (`outbound/`)
| File | Role |
|------|------|
| `deliver.ts` | `deliverOutboundPayloads()` — main outbound delivery |
| `message.ts` | `sendMessage()`, `sendPoll()` — high-level message sending |
| `envelope.ts` | `buildOutboundResultEnvelope()` |
| `payloads.ts` | Payload normalization |
| `channel-adapters.ts` | Channel-specific formatting |
| `channel-selection.ts` | Channel selection logic |
| `channel-target.ts` | Target resolution |
| `delivery-queue.ts` | Queued delivery |
| `directory-cache.ts` | User directory cache |
| `format.ts` | Message formatting |
| `identity.ts` | Sender identity |
| `target-resolver.ts` / `target-normalization.ts` / `targets.ts` / `target-errors.ts` | Target resolution pipeline |
| `outbound-policy.ts` | Send policy enforcement |
| `outbound-send-service.ts` | Send service |
| `outbound-session.ts` | Session-aware sending |
| `tool-payload.ts` | Tool result payloads |
| `message-action-params.ts` / `message-action-runner.ts` / `message-action-spec.ts` | Message action execution |
| `abort.ts` | Abort outbound requests |
| `agent-delivery.ts` | Agent-to-agent delivery |

#### Core Utilities
| File | Role |
|------|------|
| `errors.ts` | `extractErrorCode()`, `formatErrorMessage()`, `formatUncaughtError()` |
| `retry.ts` | `retryAsync()` — generic retry with backoff |
| `retry-policy.ts` | Retry policy configuration |
| `backoff.ts` | Backoff calculation |
| `restart.ts` | `emitGatewayRestart()`, `triggerOpenClawRestart()`, `scheduleGatewaySigusr1Restart()`, `deferGatewayRestartUntilIdle()` |
| `restart-sentinel.ts` | Restart state tracking |
| `home-dir.ts` | `resolveEffectiveHomeDir()`, `expandHomePrefix()` |
| `env.ts` | Environment variable utilities |
| `dotenv.ts` | .env file loading |
| `env-file.ts` | Env file parsing |
| `fetch.ts` | HTTP fetch wrapper |
| `fs-safe.ts` | Safe filesystem operations |
| `json-file.ts` | JSON file read/write |
| `file-lock.ts` | File locking |
| `gateway-lock.ts` | Gateway process lock |
| `is-main.ts` | ESM main module detection |

#### Exec Approvals
| File | Role |
|------|------|
| `exec-approvals.ts` | Approval store and logic |
| `exec-approvals-allowlist.ts` | Allowlist management |
| `exec-approvals-analysis.ts` | Command risk analysis |
| `exec-approval-forwarder.ts` | Forward approvals to UI |
| `exec-host.ts` | Exec host resolution |
| `exec-safety.ts` | Command safety checks |

#### Heartbeat & Events
| File | Role |
|------|------|
| `heartbeat-runner.ts` | `startHeartbeatRunner()` — periodic heartbeat |
| `heartbeat-events.ts` | Heartbeat event bus |
| `heartbeat-events-filter.ts` | Event filtering |
| `heartbeat-active-hours.ts` | Active hours detection |
| `heartbeat-visibility.ts` | Visibility state |
| `heartbeat-wake.ts` | Wake-on-heartbeat |
| `agent-events.ts` | Agent lifecycle events |
| `system-events.ts` | System event bus |
| `diagnostic-events.ts` / `diagnostic-flags.ts` | Diagnostic event tracking |

#### Update System
| File | Role |
|------|------|
| `update-check.ts` | Check for updates |
| `update-startup.ts` | `scheduleGatewayUpdateCheck()` |
| `update-runner.ts` | Run update |
| `update-global.ts` | Global update state |
| `update-channels.ts` | Update channel management |

#### Provider Usage Tracking
| File | Role |
|------|------|
| `provider-usage.ts` | Main usage tracking |
| `provider-usage.load.ts` | Load usage data |
| `provider-usage.format.ts` | Format usage for display |
| `provider-usage.fetch.ts` | Fetch usage from providers |
| `provider-usage.fetch.claude.ts` / `.gemini.ts` / `.codex.ts` / `.copilot.ts` / `.minimax.ts` / `.zai.ts` / `.antigravity.ts` / `.shared.ts` | Provider-specific fetchers |
| `provider-usage.auth.ts` | Usage auth |
| `provider-usage.types.ts` / `.shared.ts` | Types |

#### Networking & Discovery
| File | Role |
|------|------|
| `bonjour.ts` / `bonjour-ciao.ts` / `bonjour-discovery.ts` / `bonjour-errors.ts` | mDNS/Bonjour discovery |
| `tailscale.ts` / `tailnet.ts` | Tailscale integration |
| `widearea-dns.ts` | DNS resolution |
| `ports.ts` / `ports-inspect.ts` / `ports-lsof.ts` / `ports-format.ts` / `ports-types.ts` | Port management |
| `ssh-config.ts` / `ssh-tunnel.ts` | SSH tunnel support |
| `net/fetch-guard.ts` / `net/ssrf.ts` | Fetch safety (SSRF protection) |
| `ws.ts` | WebSocket utilities |
| `jsonl-socket.ts` | JSONL-over-socket IPC |
| `transport-ready.ts` | Transport readiness |

#### Device & Node Pairing
| File | Role |
|------|------|
| `device-identity.ts` | Device ID generation |
| `device-auth-store.ts` | Device auth persistence |
| `device-pairing.ts` | Device pairing flow |
| `node-pairing.ts` | Node pairing |
| `pairing-files.ts` | Pairing file management |
| `pairing-token.ts` | Token generation |
| `node-shell.ts` | Node shell execution |

#### Misc
| File | Role |
|------|------|
| `os-summary.ts` | OS info string |
| `machine-name.ts` | Machine display name |
| `clipboard.ts` | Clipboard access |
| `archive.ts` | Archive/compression |
| `path-env.ts` / `path-prepend.ts` | PATH management |
| `install-package-dir.ts` / `install-safe-path.ts` | Package installation |
| `npm-registry-spec.ts` | NPM registry |
| `detect-package-manager.ts` | Package manager detection |
| `brew.ts` | Homebrew detection |
| `wsl.ts` | WSL detection |
| `dedupe.ts` | Deduplication utilities |
| `channel-activity.ts` / `channel-summary.ts` / `channels-status-issues.ts` | Channel status |
| `canvas-host-url.ts` | Canvas host URL resolution |
| `control-ui-assets.ts` | Control UI static assets |
| `openclaw-root.ts` | OpenClaw root directory |
| `tmp-openclaw-dir.ts` | Temp directory |
| `process-respawn.ts` | Process respawn logic |
| `runtime-guard.ts` | Runtime guards |
| `shell-env.ts` | Shell environment |
| `skills-remote.ts` | Remote skills |
| `state-migrations.ts` / `state-migrations.fs.ts` | State file migrations |
| `session-cost-usage.ts` / `session-cost-usage.types.ts` | Session cost tracking |
| `session-maintenance-warning.ts` | Maintenance warnings |
| `system-presence.ts` | System presence |
| `system-run-command.ts` | System command execution |
| `tls/fingerprint.ts` / `tls/gateway.ts` | TLS utilities |
| `format-time/format-datetime.ts` / `format-duration.ts` / `format-relative.ts` | Time formatting |
| `unhandled-rejections.ts` | Unhandled rejection handler |
| `voicewake.ts` | Voice wake detection |
| `warning-filter.ts` | Warning suppression |
| `binaries.ts` | Binary detection |

### Dependencies
- `config/` (100) — the biggest consumer of config types
- `agents/` (32), `process/` (26), `auto-reply/` (26), `utils` (22), `plugins/` (20), `channels/` (14), `cli/` (12), `routing/` (10), `logging/` (10), `gateway/` (6)

### Dependents
- `gateway/` (101), `commands/` (71), `cli/` (49), `agents/` (44), `auto-reply/` (28), `telegram/` (20), `discord/` (17), `plugin-sdk/` (13), `web/` (11), `cron/` (10) — **second most-imported module**

### Data Flow
```
Outbound: Agent response → outbound/message.ts → target-resolver → channel-adapters → deliver.ts → channel plugin
Restart:  Signal/CLI → restart.ts → SIGUSR1/process exit → daemon restarts
Retry:    Any async op → retryAsync() → backoff → success/fail
Heartbeat: Timer → heartbeat-runner → heartbeat-events → agent proactive actions
```

---

## 5. `src/daemon/` — Process Management & Service Lifecycle

**Lines:** ~5,098 | **Files:** ~25 .ts files  
**Purpose:** Manages OpenClaw as a system service — installing/uninstalling launchd (macOS), systemd (Linux), and schtasks (Windows) services. Handles service runtime, diagnostics, and log paths.

### Files

| File | Role |
|------|------|
| `service.ts` | `resolveGatewayService()` — detects platform, returns service handle (`GatewayService` type with `install/uninstall/start/stop/restart/status`) |
| `launchd.ts` | macOS launchd: `buildLaunchAgentPlist()`, `isLaunchAgentLoaded()`, `installLaunchAgent()`, `uninstallLaunchAgent()`, `readLaunchAgentRuntime()` |
| `launchd-plist.ts` | Plist XML generation |
| `systemd.ts` | Linux systemd: `installSystemdService()`, `uninstallSystemdService()`, `restartSystemdService()`, `readSystemdServiceRuntime()` |
| `systemd-unit.ts` | Systemd unit file generation |
| `systemd-hints.ts` | Systemd troubleshooting hints |
| `systemd-linger.ts` | `enableSystemdUserLinger()` — keep user services running |
| `schtasks.ts` / `schtasks-exec.ts` | Windows scheduled tasks |
| `paths.ts` | `resolveHomeDir()`, `resolveGatewayStateDir()`, `resolveUserPathWithHome()` |
| `constants.ts` | Service name constants |
| `diagnostics.ts` | Service diagnostics |
| `inspect.ts` | Service inspection |
| `output.ts` | Formatted output |
| `program-args.ts` | Program argument construction |
| `exec-file.ts` | Child process execution |
| `runtime-format.ts` / `runtime-parse.ts` / `runtime-paths.ts` | Runtime info formatting/parsing |
| `service-audit.ts` | Service audit |
| `service-env.ts` | Service environment |
| `service-runtime.ts` | Runtime state |
| `node-service.ts` | Node host service management |
| `arg-split.ts` | Argument splitting |

### Key Exports
- `resolveGatewayService(): GatewayService` — platform-aware service manager
- `GatewayService` type — `{ install, uninstall, start, stop, restart, status, logs }`
- Launchd/systemd/schtasks individual functions for direct access

### Dependencies
- `infra/` (2) — home dir, utilities
- `terminal/` (2) — formatted output
- `process/` (2) — process management
- `cli/` (2) — CLI helpers

### Dependents
- `commands/` (37) — `gateway start/stop/install/uninstall` commands
- `cli/` (17) — CLI service management
- `wizard/` (2), `infra/` (1)

### Data Flow
```
CLI command (gateway install) → commands/ → daemon/service.ts → resolveGatewayService() → launchd.ts/systemd.ts/schtasks.ts
                                                                        ↓
                                                    Generate plist/unit file → write to disk → load service
                                                                        ↓
Service runs: openclaw gateway start → server.impl.ts (gateway module)
```

---

## 6. `src/types/` — Shared Type Definitions

**Lines:** ~9 files, minimal | **Files:** 9 `.d.ts` files  
**Purpose:** Ambient TypeScript declarations for third-party modules that lack types.

### Files

| File | Declares types for |
|------|-------------------|
| `cli-highlight.d.ts` | `cli-highlight` — syntax highlighting |
| `lydell-node-pty.d.ts` | `@lydell/node-pty` — PTY (pseudo-terminal) |
| `napi-rs-canvas.d.ts` | `@napi-rs/canvas` — canvas rendering |
| `node-edge-tts.d.ts` | `node-edge-tts` — Edge TTS |
| `node-llama-cpp.d.ts` | `node-llama-cpp` — local LLM inference |
| `osc-progress.d.ts` | `osc-progress` — terminal progress |
| `pdfjs-dist-legacy.d.ts` | `pdfjs-dist` — PDF parsing |
| `proper-lockfile.d.ts` | `proper-lockfile` — file locking |
| `qrcode-terminal.d.ts` | `qrcode-terminal` — QR code display |

### Dependencies
None — these are ambient declarations.

### Dependents
None directly — consumed by TypeScript compiler globally via `tsconfig.json` includes.

### Data Flow
No runtime data flow — compile-time only.

---

## Cross-Module Dependency Graph

```
                    ┌─────────┐
                    │  types/  │  (ambient, no runtime deps)
                    └─────────┘

                    ┌─────────┐
              ┌────▶│ config/ │◀──── Most imported module (700+ imports)
              │     └────┬────┘
              │          │
              │          ▼
         ┌────┴───┐  ┌──────────┐
         │routing/ │  │  infra/  │◀─── Second most imported (400+ imports)
         └────┬───┘  └────┬─────┘
              │           │
              ▼           ▼
         ┌─────────────────┐
         │    gateway/     │  ← The server, wires everything together
         └─────────────────┘
                    ▲
                    │
              ┌─────┴────┐
              │  daemon/  │  ← Manages gateway as OS service
              └──────────┘
```

**Key insight:** `config/` is the foundation — nearly every module depends on it. `infra/` is the utility belt. `gateway/` is the orchestrator that consumes both. `routing/` is small but critical for message dispatch. `daemon/` is isolated, only consumed by CLI commands. `types/` is compile-time only.
