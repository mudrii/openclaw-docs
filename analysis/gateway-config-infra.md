# OpenClaw Core Architecture — Part 1: Module Analysis
<!-- markdownlint-disable MD024 -->

**Updated:** 2026-05-04 | **Version:** v2026.5.2
**Codebase:** OpenClaw release tag `v2026.5.2`
**Total lines (6 modules):** release-tag snapshot across gateway/config/infra/daemon/routing/types

---

## 1. `src/gateway/` — HTTP/WebSocket Server & API Endpoints

**Lines:** 84,429 | **Files:** 358 `.ts` / `.tsx` files
**Purpose:** The heart of OpenClaw — runs the gateway server that accepts WebSocket connections from CLI/plugins, exposes HTTP endpoints (OpenAI-compatible API, control UI), manages agent sessions, chat routing, cron, browser control, node subscriptions, and plugin lifecycle.

### v2026.2.19 Changes
- **Gateway auth defaults** — Unresolved auth defaults to token mode with auto-generated token; explicit `mode: "none"` required for open loopback. See DEVELOPER-REFERENCE.md §6 for config reference
- **hooks.token ≠ gateway.auth.token** — Startup validation rejects matching tokens
- **Rate-limited control-plane RPCs** — `config.apply`, `config.patch`, `update.run` limited to 3/min per device+IP with 30s restart coalesce
- **Security headers** — `X-Content-Type-Options: nosniff` and `Referrer-Policy: no-referrer` on all HTTP responses
- **Plaintext ws:// blocked** — WebSocket connections to non-loopback hosts must use `wss://`
- **Config change audit logging** — Actor, device, IP, and changed paths now logged on config mutations
- **Drain-before-restart** — Gateway restart coalesced with cooldown to allow in-flight requests to complete

### v2026.2.21 Changes <!-- v2026.2.21 -->

- **Tailscale tokenless auth scoped to WebSocket** (`fix(gateway)`) — Tailscale-based tokenless authentication is now only permitted for WebSocket connections. HTTP API calls always require explicit auth; the tokenless path is no longer reachable from HTTP entrypoints.

- **Gateway credential resolution unified** (`refactor(gateway)`) — Credential-source precedence for call/probe/status/auth entrypoints now uses shared resolver helpers with table-driven parity. Auth context is typed; `auth.deviceToken` support added in connect frames. Source: `server.auth.e2e.test.ts` covers parity across entrypoints.

- **Insecure auth toggle messaging aligned** (`fix(gateway)`) — The log/startup message shown when `gateway.auth.mode: "none"` is configured is now consistent across all entrypoints (was inconsistent between HTTP and WS paths).

- **WebSocket message handler hardened** — `src/gateway/server/ws-connection/message-handler.ts` now strips inbound metadata blocks from messages before processing, preventing metadata leakage into the agent context via WebSocket frames.

- **BlueBubbles webhook auth required** (`fix(security)`) — BlueBubbles (iMessage) channel webhooks now require authentication. Previously, webhook delivery from the BlueBubbles server was unauthenticated.

- **Gateway nodes API improvements** — `src/gateway/server-methods/nodes.ts` updated for gateway multi-node reliability improvements.

- **`customBindHost` config key** — Gateway binding now supports an explicit `customBindHost` config key to override the bind address independent of the `host` setting.

### v2026.2.22 Changes <!-- v2026.2.22 -->

**Gateway Auth:**
- **Unified credential-source precedence** — Call/probe/status/auth entrypoints use shared resolver helpers with table-driven parity. WebSocket auth handshake uses shared typed auth contexts. Explicit `auth.deviceToken` support in connect frames.
- **Device-auth signature v1 REMOVED** — v2 payloads required with `connect.challenge` nonce.
- **Security:** Gateway emits startup warning when dangerous config flags enabled. Insecure non-loopback `ws://` targets rejected in onboarding validation.

**Config:**
- **`channels.modelByChannel` allowlisted** — Previously caused "unknown channel id" errors in config validation.
- **`bindings[].comment` optional** — Field now optional in strict validation.
- **Array-valued config paths** — Compared structurally during diffing (fixes false restart-required reloads for QMD paths).

### v2026.2.23 Changes <!-- v2026.2.23 -->

**Gateway Auth:**
- **WS flood protection** — Repeated unauthorized request floods closed per-connection with sampled rejection logging.

**Config Write:**
- **`unsetPaths` immutable updates** — Config write operations apply `unsetPaths` with immutable path-copy updates.
- **Prototype-key traversal hardening** — `config get/set/unset` rejects prototype-key path segments.

### v2026.2.24 Changes <!-- v2026.2.24 -->

**Security / Audit:**
- **Multi-user heuristic** — `security.trust_model.multi_user_heuristic` config key added; flags likely shared-user ingress patterns and documents hardening guidance (`sandbox.mode="all"`, workspace-scoped FS, reduced tool surface, no personal/private identities on shared runtimes). <!-- v2026.2.24 -->

**Gateway / Auth:**
- **Trusted-proxy WS sessions** (#25428): trusted-proxy authenticated Control UI WebSocket sessions can now skip device pairing when device identity is absent, preventing false "pairing required" failures behind trusted reverse proxies. Contributor: @SidQin-cyber. <!-- v2026.2.24 -->

**Control UI:**
- **Chat image URL safety** (#25444): image click URL opening now uses a centralized allowlist (`http/https/blob` + opt-in `data:image/*`) with opener isolation (`noopener,noreferrer` + `window.opener = null`) to prevent tabnabbing. Contributor: @shakkernerd. <!-- v2026.2.24 -->

### v2026.2.25 Changes <!-- v2026.2.25 -->

**Security / Gateway WebSocket auth** (@luz-oasis): origin checks are now enforced for all direct browser WebSocket clients beyond Control UI/Webchat. Password-auth failure throttling applies to browser-origin loopback attempts (including `localhost`). Silent auto-pairing is blocked for non-Control-UI browser clients, preventing cross-origin brute-force and session takeover chains.

**Security / Gateway trusted proxy operator role** (@tdjackey): the Control UI trusted-proxy pairing bypass now requires `operator` role; unpaired `node` sessions can no longer connect via `client.id=control-ui` and invoke node event methods.

**Security / Gateway operator pairing** (@tdjackey): pairing is required for operator device-identity sessions authenticated with shared token auth; unpaired devices can no longer self-assign operator scopes.

**Security / macOS beta OAuth path removed** (@zdi-disclosures): the Anthropic OAuth sign-in path and legacy `oauth.json` onboarding that exposed the PKCE verifier via OAuth `state` have been removed from the macOS beta onboarding path. Subscription auth is now setup-token-only.

**Gateway / `/api/channels` auth enforcement** (#25753, @bmendonca3): gateway auth is enforced for the exact `/api/channels` plugin root path (plus `/api/channels/` descendants), with regression coverage for query/trailing-slash variants and near-miss paths that must remain plugin-owned.

### v2026.3.1 Changes <!-- v2026.3.1 -->

**Gateway HTTP / Health Probes:**
- **Built-in liveness/readiness endpoints** (#31272): gateway now registers `/health`, `/healthz` (liveness) and `/ready`, `/readyz` (readiness) via a path→status map in `server-http.ts`. These are unauthenticated and suitable for Docker `HEALTHCHECK`, Kubernetes `livenessProbe`/`readinessProbe`, and external uptime monitors. Probe handlers run only when no plugin route claims the same path, so plugin routes keep precedence. The Dockerfile documents these endpoints; container health checks should probe `/healthz` or `/readyz`.
- **Control UI method guard for non-UI routes**: plugin-owned HTTP routes under `/plugins` and `/api` are excluded from the Control UI SPA fallback, preventing untrusted plugins from claiming arbitrary UI paths. POST is allowed for non-UI routes via extracted `server/http-auth.ts`.
- **Control UI CSP: Google Fonts origins**: `style-src` includes `https://fonts.googleapis.com`, `font-src` includes `https://fonts.gstatic.com` for deployments loading external Google Fonts.
- **Control UI origins wildcard handling**: `origin-check.ts` now accepts `"*"` in `gateway.controlUi.allowedOrigins` (values are trimmed and lowercased via a `Set` for O(1) lookup).
- **Control UI debug log**: full-width payload rendering in the debug event log via `debug-event-log__payload` CSS class.

**Gateway WS / Security:**
- **Plaintext ws:// loopback-only**: `isSecureWebSocketUrl()` in `net.ts` allows `ws://` only for loopback addresses (`localhost`, `127.x.x.x`, `::1`). All non-loopback `ws://` connections are rejected (CWE-319). This was established in v2026.2.19 and remains enforced; v2026.3.1 quiets noisy loopback WS close logs.
- **WS flood protection: post-handshake role requests**: repeated unauthorized role escalation requests on an already-authenticated connection are closed. This extends the v2026.2.23 per-connection flood protection.

**Gateway Auth / Pairing:**
- **Subagent TLS pairing skip**: authenticated local gateway-client connections (Docker/LAN) can skip device pairing for subagent sessions. Auth logic refactored into new `server/http-auth.ts` module.
- **Device-auth v2 migration diagnostics** (#28305, @vincentkoc): `connect-error-details.ts` now emits specific detail codes for device-auth failures (`DEVICE_AUTH_INVALID`, `DEVICE_AUTH_DEVICE_ID_MISMATCH`, `DEVICE_AUTH_SIGNATURE_EXPIRED`, etc.). `normalizeDeviceMetadataForAuth()` extracted to `device-metadata-normalization.ts` for cross-module reuse.

**Gateway Infra:**
- **Health-monitor restart cap raised**: `DEFAULT_MAX_RESTARTS_PER_HOUR` increased from 3 to 10 in `channel-health-monitor.ts`, with a new `staleEventThresholdMs` parameter.
- **macOS supervised restart**: current `v2026.3.8` launchd repair/restart runs `bootout` -> `enable` -> `bootstrap` -> `kickstart`, while supervised respawn exits and relies on launchd `KeepAlive` rather than in-process self-kickstart.
- **Node browser proxy routing**: browser bridge server hardened with `Cache-Control: no-store` on noVNC token endpoint and token-based observer URL construction.
- **Control UI origin seeding**: new `startup-control-ui-origins.ts` auto-seeds `gateway.controlUi.allowedOrigins` for non-loopback bind modes at startup, preventing crash loops on upgrade (issue #29385).

**Docker / Container:**
- **OCI labels/annotations**: Dockerfile includes `org.opencontainers.image.source`, `.url`, `.documentation`, `.licenses`, `.title`, `.description`, and base image digest annotations.
- **Image permissions normalization**: copied extension/agent directories get `chmod 755` (dirs) and `chmod 644` (files) to prevent plugin safety check rejections from world-writable source modes.
- **Browser sandbox**: `OPENCLAW_BROWSER_NO_SANDBOX=1` env var passed into sandbox browser containers since Chromium's setuid/namespace sandbox cannot work inside Docker without elevated privileges.
- **Docker bridge networking docs**: Dockerfile comments document loopback bind unreachability with `docker -p` and recommend `--network host` or `--bind lan`.
- **GHCR image source**: OCI label `org.opencontainers.image.source` points to `https://github.com/openclaw/openclaw`.
- **Sandbox bootstrap hardening**: opt-in parsing, custom socket paths, rollback support in `agents/bootstrap-files.ts` and `agents/sandbox/` modules.

**Cron UI:**
- **i18n: localized cron page**: all cron labels, filters, error messages, and status options use `t()` translation function. Locales updated: `en`, `zh-CN`, `de` (new), `pt-BR`, `zh-TW`.
- **German (de) locale**: new `ui/src/i18n/locales/de.ts` with full Control UI translation.
- **Model suggestions**: cron model field includes configured agent model defaults via `agents.list[].heartbeat` and `agents.defaults.heartbeat` merge.
- **Delivery mode none**: `"None (internal)"` option explicitly available in cron editor delivery mode dropdown.
- **Cron editor viewport**: `.cron-workspace-form` is sticky with `max-height: calc(100vh - 74px - 32px)` and `overflow-y: auto` for scrollable form panels.
- **New filters**: schedule kind filter (`all|at|every|cron`) and last status filter (`all|ok|error|skipped`) added to cron jobs list.
- **Failure alerts**: cron jobs support `failureAlertAfter` and `failureAlertCooldownSeconds` fields with `sendCronFailureAlert()` delivery.

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
| `server-discovery.ts` / `server-discovery-runtime.ts` | Bonjour/mDNS discovery |
| `server-mobile-nodes.ts` | Mobile node management |
| `server-node-events.ts` / `server-node-events-types.ts` | Node event bus |
| `server-node-subscriptions.ts` | Node event subscriptions |
| `server-tailscale.ts` | Tailscale integration |
| `server-wizard-sessions.ts` | Setup wizard |
| `server-ws-runtime.ts` | WS runtime state |
| `channel-health-monitor.ts` | Channel health monitoring |
| `events.ts` | Gateway events system |
| `control-plane-audit.ts` | Control plane audit trail |
| `control-plane-rate-limit.ts` | Control plane rate limiting |
| **Auth & Security** | |
| `auth.ts` | Token auth |
| `auth-rate-limit.ts` | Rate limiting |
| `device-auth.ts` | Device authentication |
| `http-auth-helpers.ts` | HTTP auth utilities |
| `origin-check.ts` | Origin validation |
| `probe-auth.ts` / `probe.ts` | Health probes |
| `server/http-auth.ts` | HTTP auth enforcement (canvas, plugin routes) — extracted in v2026.3.1 |
| `security-path.ts` | Protected plugin route path matching |
| `startup-control-ui-origins.ts` | Auto-seed Control UI allowedOrigins for non-loopback bind |
| `device-metadata-normalization.ts` | Device metadata normalization for auth payloads |
| **OpenAI Compatibility** | |
| `openai-http.ts` | OpenAI-compatible HTTP API (`/v1/chat/completions`) |
| `models-http.ts` | GET `/v1/models` and `/v1/models/:id` endpoint |
| `embeddings-http.ts` | POST `/v1/embeddings` proxy endpoint |
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

**Lines:** ~36,614 | **Files:** 198 .ts files  
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
| `gateway-control-ui-origins.ts` | Control UI allowedOrigins seeding/validation for non-loopback bind |
| `byte-size.ts` | Byte-size parsing utilities |

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

### Recent Changes

- **v2026.3.13:** Config/validation: four previously rejected config keys are now accepted in strict validation: `agents.list[].params` (per-agent `cacheRetention`, `temperature`, `maxTokens` overrides) (#41171); `tools.web.fetch.readability` and `tools.web.fetch.firecrawl` (#42583); `channels.signal.groups` (#27199); `discovery.wideArea.domain` (#35615). Source: `src/config/zod-schema.agent-runtime.ts`, `src/config/zod-schema.ts`.
- **v2026.2.22:** `channels.modelByChannel` allowlisted in config validation (previously caused "unknown channel id" errors). `bindings[].comment` field now optional in strict validation. Array-valued config paths compared structurally during diffing (fixes false restart-required reloads for QMD paths).
- **v2026.2.23:** Config write operations apply `unsetPaths` with immutable path-copy updates. Path traversal hardening for `config get/set/unset` rejects prototype-key segments.
- **v2026.3.1:** Gateway bind host alias normalization migration (`gateway.bind.host-alias->bind-mode`) maps raw bind values like `0.0.0.0`, `::`, `[::]`, `*` to `"lan"` and `127.0.0.1`, `localhost`, `::1` to `"loopback"`. New `gateway-control-ui-origins.ts` module handles `allowedOrigins` seeding for non-loopback bind and startup guard. Legacy migration in `part-3` auto-seeds `gateway.controlUi.allowedOrigins` for existing non-loopback installs to prevent crash loops on upgrade (#29385).

---

## 3. `src/routing/` — Session Key Resolution & Message Routing

**Lines:** ~1,804 | **Files:** 10 .ts files  
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

**Lines:** ~58,260 | **Files:** 325 .ts files  
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
| `provider-usage.fetch.claude.ts` / `.gemini.ts` / `.codex.ts` / `.copilot.ts` / `.minimax.ts` / `.zai.ts` / `.shared.ts` | Provider-specific fetchers |
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

**Lines:** ~6,093 | **Files:** 42 .ts files  
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

**Lines:** ~139 | **Files:** 8 `.d.ts` files  
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

## 7. Auto-Updater (`update.auto.*`)

### Auto-Updater (`update.auto.*`)
v2026.2.22 — Optional built-in auto-updater for package installs, default-off.
- `update.auto.enabled: true` to enable
- Stable rollout: delay + jitter before applying updates
- Beta: hourly cadence
- `openclaw update --dry-run` previews without applying

---

## 8. Control UI

### Recent Changes

- **v2026.3.13:** Control UI/insecure auth: explicit shared token and password auth preserved on plain-HTTP Control UI connects so LAN and reverse-proxy sessions no longer drop shared auth before the first WebSocket handshake (#45088). Gateway/Control UI: operator-only device-auth bypass restored; browser connect failures now classified so origin and device-identity problems no longer surface as generic auth errors in Control UI and web chat (#45512).
- **v2026.3.12:** Control UI/dashboard-v2: gateway dashboard refreshed with modular overview, chat, config, agent, and session views, plus command palette, mobile bottom tabs, and richer chat tools.
- **v2026.2.22:** Full cron edit parity (clone, validation, run history with pagination/search/sort). Tools panel data-driven from `tools.catalog`. Version status pill in web header. WS: stop/clear browser gateway client on teardown; stable per-tab `instanceId` in connect frames.

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

## v2026.3.7 Delta Notes

- Gateway channel-backed readiness probes (PR #18446, @vibecodooor, @mahsumaktas, @vincentkoc): `/ready` and `/readyz` readiness endpoints now verify that channel listeners are active before reporting ready, preventing premature traffic routing after restart.
- Webchat route safety: cross-channel leakage in webchat route resolution fixed; webchat sessions are strictly scoped to their originating channel.
- Outbound delivery replay safety: two-phase ACK pattern added to outbound delivery, preventing duplicate message delivery on transient failures.
- Session duplicate suppression synthesis: duplicate session creation race conditions resolved with synthesized suppression logic at the gateway routing layer.
- Legacy session route inheritance: legacy session keys now correctly inherit route bindings from their parent session entries.
- Route binding scalability improvements: route binding lookup performance improved for configurations with large numbers of bindings.
- TLS multi-arch Docker base digest pinning: TLS base image digests are pinned per-architecture (arm64/amd64) in the Dockerfile for reproducible multi-arch builds.

**Key insight:** `config/` is the foundation — nearly every module depends on it. `infra/` is the utility belt. `gateway/` is the orchestrator that consumes both. `routing/` is small but critical for message dispatch. `daemon/` is isolated, only consumed by CLI commands. `types/` is compile-time only.

## v2026.3.8 Delta Notes

- Control UI bundled assets can now be served from symlinked global wrappers and auto-detected package-proven roots while configured/custom roots stay on the strict hardlink boundary.
- Daemon start/restart now pre-validates config before handoff, and invalid restart-triggered shutdown drains exit non-zero so launchd/systemd retry instead of treating the stop as clean.
- launchd supervision detection now includes `XPC_SERVICE_NAME`, and LaunchAgent repair/restart paths `enable` services before `bootstrap`.
- Gateway config RPC replies now surface the live config path from the active IO layer, matching runtime-resolved config state.

## v2026.3.11 Changes

### Gateway / WebSocket Origin (Security — GHSA-5wcw-8jjv-m286)
- **Browser origin validation enforced for all browser-originated connections:** `src/gateway/origin-check.ts` previously skipped `enforceOriginCheckForAnyClient` when proxy headers were present, allowing browser-originated WebSocket connections from untrusted origins to bypass validation via a trusted reverse proxy. An attacker serving a page from an untrusted origin could inherit proxy-injected identity and reach `sharedAuthOk` / `roleCanSkipDeviceIdentity` paths without any origin restriction. The `hasProxyHeaders` exemption is removed — origin validation now runs for all browser-originated connections regardless of proxy headers. Source: `fix(gateway): enforce browser origin check regardless of proxy headers`, fixes GHSA-5wcw-8jjv-m286.

### Gateway / Config Errors (#42664)
- **Config validation issues surfaced in RPC error messages:** `src/gateway/server-methods/config.ts` now includes up to three validation issue details in the top-level error message for `config.set`, `config.patch`, and `config.apply` RPCs, while preserving the full structured issue list in the response payload. Previously, the top-level error was opaque. Source: `Gateway/Dashboard: surface config validation issues (#42664)`.

### Gateway / Control UI (#42664)
- **Control UI surfaces config validation issues:** The Control UI dashboard now displays the validation issue details returned by the `config.*` RPCs, providing immediate feedback on config errors. Part of the same commit as the RPC change above.

### Gateway / before_tool_call Hook (#43476)
- **`before_tool_call` hook runs for HTTP tools:** `fix(gateway): run before_tool_call for HTTP tools` (`8cc0c9baf`): the `before_tool_call` gateway hook was previously skipped for tool invocations arriving via HTTP (e.g. OpenAI-compatible API). It now fires for HTTP-path tool calls consistently with WebSocket-path tool calls.

### Gateway / Auth Fail-Closed (#42672)
- **Local SecretRef auth fails closed when unavailable:** `Gateway: fail closed unresolved local auth SecretRefs` (`0125ce1f4`): when local `SecretRefs` are configured for gateway auth and cannot be resolved, the gateway now fails closed (rejects the connection) instead of silently falling back to remote credentials. Source: `src/gateway/` auth path, `fix: fail closed for unresolved local gateway auth refs`.

### Gateway / Auth Recovery (#42507)
- **One trusted device-token retry on shared-token mismatch:** `fix(gateway): harden token fallback/reconnect behavior and docs` (`a76e81019`): gateway clients now allow one trusted device-token retry attempt on a shared-token mismatch, with recovery hints surfaced to the caller. Reconnect gating is tightened across client types. Source: `src/cli/daemon-cli/gateway-token-drift.ts`.

### Gateway / Session Reset Auth (Security)
- **`/new` and `/reset` split from admin-only `sessions.reset` RPC:** `fix(gateway): split conversation reset from admin reset` (`c91d1622d`): the gateway conversation reset path (`/new` command and user-facing session `/reset`) is extracted from the admin-only `sessions.reset` RPC into `src/gateway/session-reset-service.ts`. This prevents non-admin clients from reaching admin reset paths by routing through the wrong RPC handler.

### macOS / launchd Restarts (Fixes #43311, #43406, #43035, #43049)
- **`bootout` replaced with `kickstart -k` for launchd restarts:** `fix(daemon): replace bootout with kickstart -k for launchd restarts on macOS` (`3c0fd3dff`): `launchctl bootout` permanently unloaded the LaunchAgent plist, causing the gateway to become unresponsive to KeepAlive respawn after any restart trigger (agent-session restart, SIGTERM on config reload, gateway self-restart, hot reload). `restartLaunchAgent()` in `src/daemon/launchd.ts` now uses `launchctl kickstart -k`, which force-kills and restarts the service without unloading the plist. A new detached handoff helper `src/daemon/launchd-restart-handoff.ts` handles restarts originating from inside the launchd-managed process tree to avoid the caller being killed mid-command. Self-restart paths in `process-respawn.ts` schedule the detached start-after-exit handoff before exiting.
- **Kickstart restart hardened:** Follow-up `fix(daemon): address clanker review findings for kickstart restart` (`841ee2434`): fixed sleep replaced with caller-PID polling in both kickstart and start-after-exit handoff modes; the enable+bootstrap fallback is gated on `isLaunchctlNotLoaded()` so re-registration is only attempted when kickstart fails due to an absent job, not for other kickstart failures.
- **LaunchAgent remains registered during explicit restarts:** The LaunchAgent plist stays registered throughout the restart sequence; launchd KeepAlive remains effective across all restart trigger paths listed above.

### macOS / LaunchAgent Install Permissions
- **LaunchAgent directory and plist permissions tightened:** `fix(launchd): harden macOS launchagent install permissions` (`ce9e91fdf`): `src/daemon/launchd.ts` now applies stricter permissions on the LaunchAgent directory and plist file during install, reducing the risk of local privilege escalation through LaunchAgent plist modification.

### macOS / Daemon Kickstart
- **`bootout` replaced with `kickstart -k` for daemon restarts:** Same as the launchd restart change above — the `daemon` module restart path now uses `kickstart -k` instead of `bootout`. See macOS/launchd Restarts above for full details.

### Daemon / Scheduled Restarts
- **Scheduled gateway restarts handled consistently in CLI:** `fix(cli): handle scheduled gateway restarts consistently` (`b31836317`): `src/cli/daemon-cli/lifecycle-core.ts` and `lifecycle.ts` now handle scheduled gateway restart signals consistently, preventing state divergence between the CLI restart poller and the daemon restart path.

### Runtime Version
- **Runtime version exposed in gateway status:** `feat: expose runtime version in gateway status` (`5ca780fa7`): `src/commands/status.types.ts` and `status.summary.ts` now include the runtime version in the gateway status output.

### Git / Runtime State
- **`.dev-state` ignored (#41848):** `.dev-state` added to `.gitignore` so local runtime state does not appear as untracked repo noise. This is also noted in the core-engine delta for completeness. Source: `chore: add .dev-state to .gitignore`.

---

## v2026.3.12 Changes

### Gateway / Session Discovery (#44176)
- **Disk-only and retired ACP session stores discovered under custom `session.store` roots:** `src/gateway/sessions-resolve.ts` (and related session resolution helpers) now walk custom templated `session.store` roots to find stores that are present on disk but no longer registered as active. This keeps ACP reconciliation, session-id/session-label targeting, and run-id fallback working correctly after gateway restart when non-default store path templates are configured.

### Gateway / Main-Session Routing (#43918)
- **TUI and UI main-session sends kept on internal surface when `deliver` is enabled:** When a session has a persisted Telegram, WhatsApp, or other channel route and `deliver` is enabled, sends originating from TUI (`mode:TUI`) and Control UI (`mode:UI`) main-session connections are now kept on the internal delivery surface. Previously they could inherit the persisted external channel route, causing unexpected external delivery.

### Gateway / Hooks (Auth bucketing + Idempotency)
- **Hook auth failures bucketed by client IP behind trusted proxies:** When the gateway is behind a trusted reverse proxy, hook authentication failures are now attributed to the forwarded client IP (from `X-Forwarded-For` or equivalent), not the proxy IP. A startup warning is emitted when `hooks.allowedAgentIds` is unset or set to `['*']`, since this leaves hook routing unrestricted to any configured agent ID. Source: `src/gateway/hooks.ts`.
- **Hook request deduplication by idempotency key (#44438):** Repeated hook requests carrying a matching idempotency key (via `idempotency-key` or `x-openclaw-idempotency-key` HTTP header, or `payload.idempotencyKey`) reuse the first completed run instead of launching duplicate agent executions. Source: `src/gateway/hooks.ts`.

### Gateway / Session Stores (#44266)
- **Swift push-test protocol models regenerated:** Swift bindings in `Sources/OpenClawKit/` regenerated after session-store schema updates. **Windows native session-store realpath:** realpath handling aligned on Windows so session-store path comparisons use resolved paths consistently.

### Security / Bootstrap Tokens (v2026.3.12 phase)
- **Setup codes switched to short-lived bootstrap tokens:** `/pair` and `openclaw qr` setup codes no longer embed shared gateway credentials. They now carry short-lived bootstrap tokens generated at pairing-request time. The token is consumed on first use (`src/infra/device-bootstrap.ts`). Full single-use enforcement lands in v2026.3.13.

### Config / Gateway
- No new config keys in the `src/config/` or `src/gateway/` modules specific to this release window beyond those noted above.

---

## v2026.3.13 Changes

### Gateway / RPC Timeout (#45689)
- **Unanswered RPC calls rejected after bounded timeout:** `GatewayClient.request()` calls that receive no response within a configured timeout are now rejected and their pending state cleared. Previously a stalled or disconnected WebSocket could leave hanging promises indefinitely. Source: `src/gateway/client.ts`.

### Gateway / Session Reset (#44773)
- **`lastAccountId` and `lastThreadId` preserved across session resets:** `src/config/sessions/store.ts` now carries `lastAccountId` and `lastThreadId` through the reset path. Previously a `/reset` could clear these fields, causing replies to route to the wrong account or thread.

### Gateway / Status (`--require-rpc`)
- **`openclaw gateway status --require-rpc` flag added:** `src/cli/daemon-cli/status.ts` accepts a new `requireRpc` flag. When set and the RPC probe is not successful, the command exits non-zero. This allows automation to fail hard on RPC probe misses instead of treating a printed error as a green result. Clearer failure messages also added for Linux non-interactive daemon-install scenarios.

### Gateway / Control UI (#45512)
- **Operator-only device-auth bypass restored:** The Control UI WebSocket path's operator-only device-auth bypass was restored. Browser connect failures are now classified (origin error vs. device-identity error) so the Control UI and web chat surface actionable error types rather than generic auth failures.

### Gateway / WebSocket Pairing Bypass (#42931)
- **Device-pairing enforcement skipped when `gateway.auth.mode=none`:** Control UI connections behind reverse proxies no longer get stuck on `pairing required` (close code 1008) when gateway auth is explicitly disabled.

### Config / Validation (#41171, #42583, #27199, #35615)
- **Four undocumented-but-valid config paths now accepted in strict validation:**
  - `agents.list[].params` — per-agent stream params (`cacheRetention`, `temperature`, `maxTokens`). Source: `src/config/types.agents.ts` line 85; schema in `src/config/zod-schema.agents.ts`.
  - `tools.web.fetch.readability` — boolean, enables readability extraction. `tools.web.fetch.firecrawl` — object, enables Firecrawl fallback. Source: `src/config/zod-schema.agent-runtime.ts`.
  - `channels.signal.groups` — per-group `requireMention`, `tools`, and `toolsBySender` overrides for Signal.
  - `discovery.wideArea.domain` — unicast DNS-SD domain for wide-area gateway discovery. Source: `src/config/zod-schema.ts`.

### Security / Bootstrap Tokens (single-use enforcement)
- **Setup codes made single-use:** The bootstrap token record is consumed before the token is returned in `src/infra/device-bootstrap.ts`. This prevents a pending device pairing request from being replayed by a second device before the first device completes approval.

### Daemon / Linux Non-Interactive Install
- **Clearer failure reporting:** Linux daemon install failures in non-interactive environments now produce more actionable error messages, distinguishing between systemd unavailability and permission issues.

---

## v2026.3.24 Changes

### Gateway / OpenAI-Compatible Endpoints
- **`/v1/models` and `/v1/embeddings`**: new OpenAI-compatible endpoints. `models-http.ts` serves GET `/v1/models` and `/v1/models/:id`; `embeddings-http.ts` proxies POST `/v1/embeddings`.

### Gateway / Restart Sentinel
- **Heartbeat-wake replacing best-effort note**: gateway restart sentinel now uses heartbeat-wake instead of best-effort note, with thread/topic routing preserved across restarts.

### Gateway / Channel Startup
- **Sequential startup with per-channel failure isolation**: channels start sequentially; a failure in one channel no longer prevents other channels from starting.

### Gateway / Task Registry (`src/tasks/`)
- **Durable task lifecycle records for ACP/subagent work**: the gateway task registry abstraction (`src/tasks/task-registry.ts`, `task-registry.types.ts`, `task-registry.store.ts`, `task-registry.maintenance.ts`) keeps durable lifecycle records for ACP/subagent spawned tasks. Each `TaskRecord` tracks `taskId`, `runtime` (`subagent` | `acp` | `cli`), `requesterSessionKey`, `status`, `deliveryStatus`, and `notifyPolicy`. ACP completion and failure updates are delivered through the real requester chat path instead of session-only stream events. Maintenance is started via `startTaskRegistryMaintenance()` called from `server.impl.ts` on gateway boot.

### Docker
- **Avoid pre-start namespace loop**: Docker setup fix to prevent a namespace loop during pre-start initialization.

### Runtime
- **Node 22.14 floor**: minimum Node.js version raised to 22.14.

---

## 9. `src/mcp/` — Gateway-Backed Channel MCP Bridge

**Files:** 4 `.ts` files
**Purpose:** Exposes OpenClaw gateway conversations, messages, and approval workflows to Codex/Claude and other MCP-capable clients over the Model Context Protocol. The bridge connects to the local gateway as a WebSocket client and surfaces channel conversation tools to the MCP consumer.

### Files

| File | Role |
|------|------|
| `channel-server.ts` | `createOpenClawChannelMcpServer()` / `serveOpenClawChannelMcp()` — creates an MCP server backed by `OpenClawChannelBridge`, registers tools and notification handlers, manages `StdioServerTransport` lifecycle with clean shutdown on `SIGINT`/`SIGTERM`/stdin close |
| `channel-bridge.ts` | `OpenClawChannelBridge` class — connects to the gateway via `GatewayClient`, subscribes to `sessions.subscribe`, enqueues `QueueEvent` items (messages, exec/plugin approvals, Claude permission requests), exposes `listConversations`, `readMessages`, `sendMessage`, `waitForEvent`, `pollEvents`, `respondToApproval` |
| `channel-tools.ts` | `registerChannelMcpTools()` — registers 8 MCP tools on the server: `conversations_list`, `conversation_get`, `messages_read`, `attachments_fetch`, `events_poll`, `events_wait`, `messages_send`, `permissions_list_open`, `permissions_respond`. `getChannelMcpCapabilities()` — declares `experimental: { "claude/channel": {}, "claude/channel/permission": {} }` capabilities when Claude channel mode is not `"off"` |
| `channel-shared.ts` | Shared types (`ConversationDescriptor`, `QueueEvent`, `WaitFilter`, `PendingApproval`, `ClaudePermissionRequest`, `ClaudeChannelMode`), `ClaudePermissionRequestSchema` (Zod), helper functions (`toConversation`, `toText`, `matchEventFilter`, `resolveConversationChannel`, `extractAttachmentsFromMessage`, `summarizeResult`, `resolveMessageId`, `normalizeApprovalId`) |

### Key Exports
- `createOpenClawChannelMcpServer(opts)` — returns `{ server, bridge, start, close }`
- `serveOpenClawChannelMcp(opts)` — full stdio MCP server lifecycle (used as the MCP entry point)
- `OpenClawChannelBridge` class
- MCP tools: `conversations_list`, `conversation_get`, `messages_read`, `attachments_fetch`, `events_poll`, `events_wait`, `messages_send`, `permissions_list_open`, `permissions_respond`

### MCP Capabilities
- `experimental.claude/channel` — Claude channel notification mode for broadcasting inbound user messages to the MCP client
- `experimental.claude/channel/permission` — Claude permission-request/response flow for tool approval via chat reply

### ClaudeChannelMode
- `"auto"` (default) — enables capabilities; Claude channel notifications are active
- `"on"` — same as auto
- `"off"` — capabilities block omitted; no Claude channel notifications

### Event Queue
`OpenClawChannelBridge` maintains an in-memory `QueueEvent[]` queue (capped at 1,000 entries) with a monotonic cursor. Event types: `message`, `claude_permission_request`, `exec_approval_requested`, `exec_approval_resolved`, `plugin_approval_requested`, `plugin_approval_resolved`.

### Dependencies
- `gateway/client.ts` — `GatewayClient` for WebSocket gateway connection
- `gateway/call.ts` — `buildGatewayConnectionDetails()`
- `gateway/connection-auth.ts` — `resolveGatewayConnectionAuth()`
- `gateway/method-scopes.ts` — `READ_SCOPE`, `WRITE_SCOPE`, `APPROVALS_SCOPE`
- `config/` — `loadConfig()`, `OpenClawConfig`
- `@modelcontextprotocol/sdk` — `McpServer`, `StdioServerTransport`

---

## v2026.3.28 Changes

### Gateway / Health — Webhook-vs-Polling Account Mode in Snapshots
- **Passive channels skip false stale-socket health failures (PR #47488)**: `src/gateway/channel-health-policy.ts` `evaluateChannelHealth()` now inspects `snapshot.mode` and skips the stale-socket check for any channel explicitly operating in webhook mode (`snapshot.mode === "webhook"`), in addition to the existing Telegram long-polling special-case (`policy.channelId === "telegram"`). This means passive channels such as LINE and BlueBubbles, which use webhook delivery and have no persistent outgoing socket, no longer fail the 30-minute stale-event threshold check and trigger false restart cycles. The `mode` field on `ChannelHealthSnapshot` is populated from channel account descriptors at runtime.

### MCP / Gateway-Backed Channel MCP Bridge (New Feature)
- **`src/mcp/` module added**: a new `src/mcp/` directory introduces `channel-bridge.ts`, `channel-server.ts`, `channel-tools.ts`, and `channel-shared.ts`. The bridge gives Codex/Claude-facing MCP clients a gateway-connected view of OpenClaw channel conversations. See section 9 above for full details.
- **Conversation tools**: `conversations_list`, `conversation_get`, `messages_read`, `attachments_fetch`, `events_poll`, `events_wait`, `messages_send` provide read/write access to channel-routed sessions.
- **Approval tools**: `permissions_list_open` and `permissions_respond` surface exec and plugin approval requests to MCP clients.
- **Claude channel notifications**: the `notifications/claude/channel` MCP notification forwards inbound user messages to Claude/Codex in real time. `notifications/claude/channel/permission` handles tool permission request/response via chat reply (pattern: `yes <5-char-id>` or `no <5-char-id>`).
- **Safer stdio bridge lifecycle**: `serveOpenClawChannelMcp()` registers a single `shutdown()` handler across `SIGINT`, `SIGTERM`, stdin `end`, and stdin `close` with deduplication, and handles transport `onclose` for clean reconnect without leaking gateway connections.

### Plugins / Startup Auto-Load from Config Refs
- **Bundled provider and CLI-backend plugins auto-load from explicit config refs**: `src/plugins/bundled-compat.ts` `withBundledPluginAllowlistCompat()` injects bundled plugin IDs into `plugins.allow` when a non-empty allowlist is present and the bundled plugin is not yet listed. This means bundled Claude CLI, Codex CLI, and Gemini CLI message-provider setups referenced in config no longer require a manual `plugins.allow` entry — the loader adds them automatically at startup. `withBundledPluginEnablementCompat()` similarly injects `{ enabled: true }` entries under `plugins.entries` for bundled plugin IDs that have no explicit entry yet.

### Config / Doctor — Legacy Migration Cutoff (Breaking)
- **Automatic config migrations older than two months are dropped**: very old legacy config keys no longer trigger rewrite on load or by `openclaw doctor`. They now fail validation instead. Operators with configs older than ~two months must manually update or remove the stale keys.

### Config / Validation Fixes
- **`tools.web.fetch.maxResponseBytes` accepted (#53401)**: `src/config/zod-schema.agent-runtime.ts` now includes `maxResponseBytes: z.number().int().positive().optional()` under the `tools.web.fetch` schema, so this previously rejected key passes strict validation. Source: `src/config/types.tools.ts` also declares `maxResponseBytes?: number` on the fetch tool config type.
- **`buttons` schema kept optional in merged tool definitions (#54418)**: the shared `buttons` schema field in merged tool definitions is now optional, so plain `action=send` calls no longer fail validation when no buttons are provided.

## v2026.3.31 Delta Notes

### Gateway Auth / Node Trust

- **`trusted-proxy` is stricter on the stable line:** mixed shared-token configs are rejected, local-direct fallback requires the configured token, and browser-origin `Origin` checks are part of the released trust boundary.
- **Node commands are approval-gated:** pairing alone no longer exposes declared node commands, and node-originated runs stay on a reduced trusted surface even after approval.

### Gateway / MCP / Runtime Isolation

- **HTTP tool-invoke and plugin-auth routing are tighter:** stable `v2026.3.31` narrows plugin route runtime scopes and keeps HTTP invoke authorization closer to the effective runtime boundary.
- **Remote MCP transports are stable behavior:** gateway/infrastructure assumptions now need to account for remote HTTP/SSE servers plus `streamable-http`, not only local stdio workers.

## v2026.4.9 Changes

### Gateway / Startup and Restart Behavior

- **`gateway.mode` defaults to `local` when unset:** the current release line assumes local mode by default and hardens lock/restart handling on macOS and Windows.
- **Restart reliability is a headline fix area:** launchd/scheduled-task recovery, stale lock detection, and shutdown bounding all received release-line fixes, so restart semantics are materially different from earlier snapshots.

### Gateway / Reply Delivery

- **Commentary buffering reaches the gateway layer:** `/v1/responses`, replay, embedded replies, previews, and channel partials now expect commentary to stay buffered until `final_answer`.
- **Structured plan/progress surfaces:** the gateway has to pass through structured plan/update events for compatible UIs alongside classic commentary/final text.

### Approval / Channel Runtime

- **APNs and Matrix approvals are part of the stable gateway path:** approval state and delivery now span mobile reconnect, modal fetch-after-auth, and Matrix room/thread-native resolution in addition to older approval channels.

### Channel / Runtime Registry

- **Initial channel registry is pinned more defensively:** the stable line explicitly guards against later plugin-registry churn emptying `channels.status` or losing configured channel visibility.

---

## Changes in v2026.4.15–v2026.4.21

### v2026.4.15

#### Gateway / Startup — Spurious SIGUSR1 Restart Loop Fixed (#67436, #67557)
- **Plugin auto-enable no longer triggers restart loop on Linux/systemd:** when plugin auto-enable was the only config write performed at startup, the gateway was emitting an authorized SIGUSR1 restart unnecessarily, producing a loop under systemd's KeepAlive. The fix gates the SIGUSR1 authorization so a config write that only reflects plugin auto-enable state does not qualify as a restart-triggering change. Source verified: `src/infra/restart.ts` (`sigusr1AuthorizedCount`, `scheduleGatewaySigusr1Restart`).

#### Gateway / Auth — Per-Request Bearer Resolution (#66651)
- **Active bearer token resolved per-request via `getResolvedAuth()`:** gateway auth credential resolution for bearer tokens is now performed fresh on each request rather than being captured at connection time. This ensures token rotation and revocation are reflected immediately without requiring a reconnect. Source verified: `src/gateway/call.ts`, `src/gateway/server-runtime-config.ts`, `src/gateway/server.impl.ts`.

---

### v2026.4.20

#### Config / Write-Through Single-File Top-Level Includes (#41050, #66048)
- **Config writes use write-through for single-file top-level `$include` targets:** when the active config is a top-level `$include` pointing to a single file, mutations (set/patch/apply) are written through to the included file rather than clobbering the outer config. This fixes plugins install overwriting modular `$include`-based config layouts. Source verified: `src/config/includes.ts`, `src/config/mutate.test.ts`.

#### Config / Reload Plans from Source-Authored Config (#68732)
- **Reload diff computed against source-authored config, not derived state:** config reload plans are now built by diffing the incoming config against the source-authored (user-written) config rather than the previously derived state that included provider and plugin config paths. This prevents false restart signals when provider or plugin config paths change at runtime without any user-authored config change. Source verified: `src/config/heartbeat-config-honor.inventory.test.ts`.

#### Config / Gateway Config Recovery — Last-Known-Good Restore
- **Critical clobber signatures trigger last-known-good restoration:** the config IO observer (`src/config/io.observe-recovery.ts`) detects three critical clobber signatures and restores the last-known-good config: missing metadata, missing `gateway.mode`, and sharp size drops (tracked as `size-drop-vs-last-good`). The clobbered file is saved with a timestamped `.clobbered.<timestamp>` suffix before restoration. Fields verified in source: `lastKnownGood`, `clobberedPath`, `size-drop-vs-last-good` reason prefix.

#### Config / Non-JSON Prefix Recovery
- **Config files accidentally prefixed with non-JSON output are recovered:** if the config file begins with non-JSON content (e.g., from a startup script or `openclaw doctor --fix` run that emitted text before JSON), the IO layer strips the prefix, recovers the JSON, and preserves the corrupted file as a `.clobbered.<timestamp>` backup.

#### Gateway / Usage Cache Bounded with FIFO Eviction (#68842)
- **Gateway usage cache is now size-bounded with FIFO eviction:** the in-memory usage cache grows to a configured maximum and evicts the oldest entries first, preventing unbounded memory growth in long-running gateway instances. Source verified: `src/gateway/session-utils.fs.ts` area.

#### Gateway / Pairing — Loopback Shared-Secret Clients Treated as Local (#69431)
- **Loopback shared-secret node-host, TUI, and gateway clients bypass pairing requirement:** clients connecting from loopback using a shared-secret credential (node-host, TUI, gateway-to-gateway) are now classified as local for pairing purposes, consistent with how other trusted local clients are treated. Source verified: `src/gateway/server.auth.modes.suite.ts`, `src/gateway/server/ws-connection/handshake-auth-helpers.test.ts`.

#### Gateway / Session Keys — `allowRequestSessionKey` Gate on Template Mappings (#69381)
- **`allowRequestSessionKey` enforced on template-rendered mapping `sessionKey`s:** the `allowRequestSessionKey` config gate (`src/config/zod-schema.ts` line 610) is now enforced when a session key is produced via a template-rendered mapping, closing a path where a caller-supplied session key could bypass the gate by going through a mapping that rendered to the caller's value. Source verified: `src/config/zod-schema.ts`.

---

### v2026.4.21

#### Onboard / Wizard — Long-List Pickers Replaced with Searchable Autocompletes
- **Wizard and onboarding long-list pickers switched to searchable autocomplete UI:** picker components in the setup wizard and onboarding flows that previously displayed flat scrollable lists now use a searchable autocomplete (`src/tui/components/searchable-select-list.ts`), improving usability for large option sets (model lists, channel lists, etc.). Source verified: `src/tui/components/searchable-select-list.test.ts`, `src/tui/components/selectors.ts`.

---

## v2026.4.22–v2026.4.26 Delta

**v2026.4.26:**
- **Gateway/effective tool inventory (#72365):** the gateway now exposes the *effective* tool inventory for a session (after plugin manifest takeover, allow/deny lists, and node-trust filtering), so clients no longer have to re-derive which tools are actually callable.
- **Gateway/device tokens (#66773):** device-token issuance and rotation are first-class on the stable line — pairing flows mint per-device bearer tokens and the gateway scopes them to the originating node.
- **Gateway-to-gateway hop hardening (#72720):** federated/multi-node gateway calls reject loops, enforce the same approval gate as direct calls, and propagate node-trust context across the hop.
- **Gateway/proxy support (#43821):** outbound HTTP from the gateway honors `HTTPS_PROXY`/`NO_PROXY` for provider, MCP-over-HTTP, and Bonjour discovery traffic; previously some paths bypassed system proxy settings.
- **Gateway/install path (#69400):** `--wrapper`-style installs are validated end-to-end so launchd/systemd units point at the wrapper rather than a now-removed binary after upgrades.
- **`gateway.health` endpoint:** lightweight liveness/readiness JSON used by macOS launchd, systemd, and Docker compose health-checks; reports plugin-registry, channel-registry, and config-IO observer state without exposing session contents.
- **Gateway/models:** model registry rebuilds are debounced and snapshot-shared across plugin-manifest reloads, so a plugin reload no longer briefly empties `/v1/models`.
- **Gateway/startup:** plugin-startup degraded mode (see security-plugins.md v2026.4.26 delta) is reported through `gateway.health` and surfaced in the TUI status line — broken plugins no longer silently disappear from the session.
- **Logging:** verbosity controls (#51075), structured-log envelope (#40353), and per-channel filters (#42982) are stable. The shared logger redacts known secret-shaped strings (provider keys, OAuth tokens) before the OTEL bridge receives them.
- **Diagnostics/OTEL (#33832):** the OTEL bridge from v2026.4.25 picks up bounded counters for restart authorization (`sigusr1AuthorizedCount`), config recovery events, and plugin-startup degraded entries; cardinality is capped to keep cost predictable.

**v2026.4.25:**
- Gateway/usage cache continues to be size-bounded; eviction now logs once per N evictions instead of per event.
- Config IO recovery: clobber detection extended to `gateway.health`-shaped writes from older daemons.

**v2026.4.24:**
- Gateway/auth: per-request bearer resolution (#66651) extended to MCP-over-HTTP transports.

**v2026.4.23:**
- Restart loop guard (#67436, #67557) gained an additional gate for plugin-stage hydration writes — same shape as the original auto-enable fix.

**v2026.4.22:**
- Config write-through for single-file `$include` targets (#41050, #66048) extended to nested top-level `$include` arrays where exactly one entry resolves to a writable file.
