# OpenClaw Codebase Analysis: Security, Web & Browser Cluster
<!-- markdownlint-disable MD024 -->

> Updated: 2026-03-15 | Version: v2026.3.13-1 | Codebase: OpenClaw release tag `v2026.3.13-1` | Modules: security, web, browser, canvas-host, plugins, plugin-sdk, acp

---

## Table of Contents

1. [src/security](#1-srcsecurity)
2. [src/web](#2-srcweb)
3. [src/browser](#3-srcbrowser)
4. [src/canvas-host](#4-srccanvas-host)
5. [src/plugins](#5-srcplugins)
6. [src/plugin-sdk](#6-srcplugin-sdk)
7. [src/acp](#7-srcacp)

---

## 1. src/security

### Module Overview
Central security audit, remediation, and content-safety module. Provides comprehensive security auditing of OpenClaw configurations, filesystem permissions, external content sanitization (prompt injection defense), skill/plugin code scanning, and auto-fix capabilities. Entry points: `runSecurityAudit()` (audit.ts), `fixSecurityFootguns()` (fix.ts).

#### v2026.2.15 Changes
- **Sandbox docker config hardening** — new validation tests (`config.sandbox-docker.test.ts`) for `resolveSandboxBrowserConfig` / `validateConfigObject`
- **Prompt path sanitization** — new `src/infra/path-safety.ts` with `resolveSafeBaseDir()` and `isWithinDir()` for cross-platform path containment
- **Restrict skill download paths** — new `src/infra/install-safe-path.ts` provides `unscopedPackageName()`, `safeDirName()`, `safePathSegmentHashed()` to sanitize install target paths
- **Scope session tools/webhook** — session-scoped tool/process isolation via `resolveSandboxScopeKey()` (`src/agents/sandbox/shared.ts`)
- **Preserve control-UI scopes in bypass mode** — device pairing preserves existing token scopes when rotating without scopes
- **Harden chat.send input sanitization** — tighter validation on outbound message parameters
- **Control UI XSS fix** — JSON endpoint + CSP lockdown in `src/infra/control-ui-assets.ts`
- **Skill scanner updates** — updated detection rules in `skill-scanner.ts`

#### v2026.2.19 Changes
- **SSRF hardening** — NAT64/6to4/Teredo IPv6 transition addresses and octal/hex/short/packed IPv4 blocked in SSRF guard
- **Browser SSRF** — Browser navigation routed through SSRF guard (configurable via `browser.ssrfPolicy`). See DEVELOPER-REFERENCE.md §6 for config reference
- **Security headers** — `X-Content-Type-Options: nosniff` and `Referrer-Policy: no-referrer` on gateway HTTP responses
- **Plugin/hook path containment** — `realpath` checks prevent symlink escapes
- **safeBins trusted dirs** — Binaries must resolve from trusted bin directories; untrusted PATH entries rejected
- **Cron webhook SSRF guard** — Webhook delivery URLs validated through SSRF guard before dispatch

#### v2026.2.25 Security Hardening

- **Exec approval argv binding and spawn hardening** (@tdjackey): `system.run` approval matching is now bound to exact argv identity and preserves argv whitespace in rendered command text, preventing trailing-space executable path swaps from reusing mismatched approvals. Approval-bound execution on node hosts additionally rejects symlink `cwd` paths and canonicalizes path-like executable argv before spawn, blocking mutable-cwd symlink retarget chains.
- **Browser uploads revalidation** (security): upload paths are revalidated at use-time in Playwright file-chooser and direct-input flows so missing or rebound paths are rejected before `setFiles`; strict missing-path handling is enforced with regression coverage.
- **Browser temp path symlink hardening** (@tdjackey): trace/download output-path handling is hardened against symlink-root and symlink-parent escapes via realpath-based write-path checks plus secure fallback tmp-dir validation that fails closed on unsafe fallback symlinks.
- **Workspace FS hardlink rejection** (@tdjackey): `tools.fs.workspaceOnly` and `tools.exec.applyPatch.workspaceOnly` boundary checks (including sandbox mount-root guards) now reject in-workspace hardlinked file aliases pointing outside the workspace, closing a hardlink-based boundary bypass that realpath alone could not detect.
- **`agents.files` path hardening** (@tdjackey): `agents.files.get`/`agents.files.set` now blocks out-of-workspace symlink targets while keeping in-workspace symlink targets supported; gateway regression coverage added for both blocked escapes and allowed in-workspace symlinks.
- **SSRF / IPv6 multicast** (@zpbrent): IPv6 multicast literals (`ff00::/8`) are now classified as blocked/private-internal targets in shared SSRF IP checks, preventing multicast literals from bypassing URL-host preflight and DNS answer validation.

#### v2026.3.1 Changes

- **Feishu webhook ingress bounded rate-limit state** (@bmendonca3): unauthenticated webhook rate-limit state is now bounded with stale-window pruning and a hard key cap to prevent unbounded pre-auth memory growth from rotating source keys (#26050).
- **Feishu doc create grants bound to trusted requester context** (@Takhoffman): `feishu.docx.create` tool grants are now scoped to verified inbound requester identity via plugin tool context, with audit coverage for grant enforcement (#31184).
- **Workspace safe writes TOCTOU hardening** (@tdjackey): `writeFileWithinRoot` opens existing files without truncation, creates missing files with exclusive create (`O_EXCL`), defers truncation until post-open identity+boundary validation, and removes out-of-root create artifacts on blocked races.
- **fs-safe EISDIR sanitization**: directory-read failures are caught pre-open via `lstat` and sanitized defensively so raw `EISDIR` text never leaks to messaging surfaces (openclaw#31186).
- **Sandbox media TOCTOU hardening**: media read paths in sandbox contexts are revalidated at use-time via `openFileWithinRoot` with hardlink rejection to prevent TOCTOU escape chains between workspace boundary check and file consumption.
- **Web search citation redirect SSRF**: Gemini web search citation redirect resolution now enforces strict SSRF defaults so redirects to localhost/private/internal targets are blocked.
- **Root-scoped writes symlink race hardening**: `writeFileWithinRoot` race window between path resolution and file open is closed with atomic create/open patterns.
- **Security audit: wildcard origin and Feishu owner grant warnings**: `runSecurityAudit()` now emits warnings when `gateway.controlUi.allowedOrigins` contains wildcards and when Feishu owner grants are overly broad.
- **Feishu system preview prompt leakage** (@stakeswky): inbound Feishu message previews are no longer enqueued as system events, preventing user preview text from being injected into later turns as trusted `System:` context (#31209).
- **Subagent runtime events structured context**: ad-hoc subagent completion system-message handoff replaced with typed internal completion events (`task_completion`) rendered consistently across direct and queued announce paths, eliminating a prompt injection vector in system-message content.
- **RFC2544 fake-IP compatibility for trusted fetch** (@sunkinux): `WEB_TOOLS_TRUSTED_NETWORK_SSRF_POLICY` now sets `allowRfc2544BenchmarkRange: true` so proxy fake-IP networking modes using `198.18.0.0/15` do not trigger false SSRF blocks on trusted web-tool fetch endpoints. The general SSRF guard still blocks the range (#31176).
- **Edit workspace boundary error preservation** (@haosenwang1018): editing files outside workspace roots now surfaces the real "Path escapes workspace root" failure instead of a misleading access/file-not-found error (#31015).
- **Sandbox mkdirp boundary checks** (@glitch418x): `SandboxFsBridge.mkdirp()` now passes `allowedType: "directory"` to boundary validation, allowing existing in-boundary subdirectories to pass validation without false `cannot create directories` failures (#30610).
- **fs-safe outside-workspace error code** (@YuzuruS): new `"outside-workspace"` code in `SafeOpenErrorCode` union distinguishes root-escape checks from other `"invalid-path"` errors in `openFileWithinRoot`/`writeFileWithinRoot`, enabling consumers to surface accurate error messages.
- **ACPX Windows spawn hardening** (@tdjackey): `.cmd/.bat` wrappers resolved via PATH/PATHEXT with strict fail-closed handling (`strictWindowsCmdWrapper`) by default for unresolvable wrappers on Windows.

#### v2026.2.22 SSRF Hardening

**RFC range expansion:**
- Block RFC2544 benchmarking range (`198.18.0.0/15`)
- Block TEST-NET ranges (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`)
- Block multicast and reserved/broadcast blocks
- Centralize range checks into single CIDR policy table
- Reuse one shared host/IP classifier across literal + DNS checks

**IPv6 normalization:**
- Normalize IPv6 dotted-quad transition literals (`::127.0.0.1`, `64:ff9b::8.8.8.8`)
- Block RFC2544 benchmarking range across direct and embedded-IP paths

**IPv4/IPv6 fallback:**
- Enable `autoSelectFamily` on pinned undici dispatchers with attempt timeout
- IPv6-unreachable environments quickly fall back to IPv4 for guarded fetch paths

**MSTeams media (v2026.2.22):**
- Enforce allowlist checks for SharePoint reference attachment URLs and redirect targets
- Route attachment auth-retry and Graph SharePoint download redirects through shared `safeFetch`
- Validate each hop with allowlist + DNS/IP checks across full redirect chain

**Cron webhook SSRF (previously in v2026.2.19):**
- All cron delivery webhooks validated through SSRF guard

#### v2026.2.22 Symlink Escape Prevention

- **Archive extraction:** Block zip symlink escapes during archive extraction
- **Media sandbox:** Enforce symlink-escape checks before sandbox-validated reads; prevent relative `../` escapes when sandbox lives under tmp
- **Control UI static files:** Block symlink-based out-of-root reads via realpath containment + file-identity checks
- **Gateway avatars:** Block symlink traversal during local avatar `data:` URL resolution; realpath containment + file-identity checks
- **Avatar serving (`/avatar/:agentId`):** Reject symlink paths; fd-level file identity + size checks; 2 MB max size
- **Hooks transforms:** Enforce symlink-safe containment for `hooks.transformsDir` and `hooks.mappings[].transform.module` paths; in-root symlinks still supported
- **Browser uploads:** Accept in-root upload paths when uploads directory is a symlink alias (e.g., `/tmp` → `/private/tmp` on macOS)

#### v2026.2.22 Auth/Identity Hardening

- **Elevated scope bypass fixed:** Match `tools.elevated.allowFrom` against sender identities only (not recipient `ctx.To`)
- **Feishu display-name collision:** Enforce ID-only allowlist matching; ignore mutable display names
- **Group policy `toolsBySender`:** Require explicit sender-key types (`id:`, `e164:`, `username:`, `name:`); legacy untyped keys on deprecated ID-only path
- **Discord allowlist:** Canonicalize resolved names to IDs at runtime. Security audit warnings for name/tag-based entries
- **Hook auth rate limits:** Normalize IPv4/IPv4-mapped IPv6 to one throttle bucket
- **Owner display secret:** Auto-generate `commands.ownerDisplaySecret`; remove gateway token fallback
- **Prototype pollution:** Block `__proto__`, `constructor`, `prototype` traversal in config merge helpers

#### v2026.2.22 Browser Extension Hardening

- **MV3 worker hardening:** Preserve debugger attachments across relay drops. Auto-reconnect with bounded backoff+jitter. Persist/rehydrate attached tab state via `chrome.storage.session`. Recover from `target_closed` navigation detaches. Enforce per-tab operation locks and per-request timeouts
- **Relay state guards:** Treat extension websocket as connected only when `OPEN`. Guard stale socket message/close handlers. Regression coverage for `409` duplicate rejection and reconnect-after-close races
- **Remote CDP:** Extend stale-target recovery to reuse sole available tab for remote CDP profiles

#### v2026.2.21 Changes <!-- v2026.2.21 -->
- **SHA-256 for synthetic IDs** — `feat(security)`: migrate sha1 hashes to sha256 for synthetic IDs (#22528/#7343). All synthetically-generated IDs now use SHA-256; SHA-1 collision resistance was insufficient for untrusted content
- **Sandbox --no-sandbox disabled by default** — `Security`: Chrome/Chromium sandbox now enabled by default in sandbox container runs (#22451); `--no-sandbox` flag is no longer set by default
- **Sandbox browser network defaults hardened** — `fix(security)`: outbound network restrictions tightened in sandboxed browser runs
- **Sandbox browser hash migration** — `fix(security)`: force sandbox browser hash migration and audit stale labels — SHA-1 → SHA-256 for browser session labels; stale labels flagged by `sandbox.browser_container.hash_epoch_stale` audit check (see `audit.test.ts` line 574)
- **Non-network navigation schemes blocked** — `fix(browser)`: non-network navigation schemes blocked in browser module (e.g., `file://`, `javascript:`, `data:` URIs)
- **noVNC observer tokens** — `fix(sandbox)`: noVNC observer sessions now require one-time token auth plus mandatory password auth

### File Inventory (29 TypeScript files in `v2026.3.1`: 19 source + 10 tests)

| File | Description |
|------|-------------|
| `audit.ts` | Main security audit orchestrator — `runSecurityAudit()` collects findings from all sub-collectors |
| `audit-channel.ts` | Channel-specific security findings (allowFrom, groupPolicy, DM policy per channel) |
| `audit-extra.ts` | Re-export barrel for sync and async audit collectors |
| `audit-extra.sync.ts` | Synchronous config-based audit checks (attack surface, secrets, hooks, sandbox, models, exposure matrix) |
| `audit-extra.async.ts` | Async I/O-based audit checks (file permissions, plugin trust, skill code safety) |
| `audit-fs.ts` | Filesystem permission inspection (POSIX mode bits, symlink detection, Windows ACL) |
| `audit-tool-policy.ts` | Resolves sandbox tool allow/deny policies for audit |
| `channel-metadata.ts` | Safely wraps untrusted channel metadata (truncation, dedup, external content wrapping) |
| `dangerous-config-flags.ts` | Detects dangerous configuration flags and surfaces audit findings |
| `dangerous-tools.ts` | Shared constants for high-risk tool names (gateway HTTP deny list, ACP dangerous tools) |
| `dm-policy-shared.ts` | Shared DM policy enforcement logic |
| `external-content.ts` | Prompt injection defense — wraps external content with security boundaries, detects suspicious patterns |
| `fix.ts` | Auto-remediation — `fixSecurityFootguns()` tightens groupPolicy, file permissions, allowFrom normalization |
| `mutable-allowlist-detectors.ts` | Detect mutable allowlist entries (display names, tags) and flag for audit |
| `safe-regex.ts` | Safe regex construction and validation utilities |
| `scan-paths.ts` | Path containment utilities (`isPathInside`, skip node_modules/dotfiles) |
| `secret-equal.ts` | Timing-safe secret comparison using `crypto.timingSafeEqual` |
| `skill-scanner.ts` | Static analysis scanner for skill/plugin code — detects dangerous exec, eval, network, fs patterns |
| `windows-acl.ts` | Windows-specific ACL inspection/remediation via icacls |
| **New in v2026.2.15 (src/infra/)** | |
| `install-safe-path.ts` | Sanitize skill/plugin install target paths — `unscopedPackageName()`, `safeDirName()`, `safePathSegmentHashed()` |
| `install-safe-path.test.ts` | Tests for install path sanitization |
| `path-safety.ts` | Cross-platform path containment — `resolveSafeBaseDir()`, `isWithinDir()` |
| `path-safety.test.ts` | Tests for path safety checks |
| `audit.test.ts` | Tests for main audit orchestrator |
| `audit-extra.sync.test.ts` | Tests for sync audit collectors |
| `dm-policy-channel-smoke.test.ts` | Channel-level DM policy smoke tests |
| `dm-policy-shared.test.ts` | Shared DM policy logic tests |
| `external-content.test.ts` | Tests for prompt injection detection and content wrapping |
| `fix.test.ts` | Tests for security auto-fix |
| `safe-regex.test.ts` | Tests for safe regex utilities |
| `skill-scanner.test.ts` | Tests for code scanning rules |
| `temp-path-guard.test.ts` | Tests for temp path guard validation |
| `windows-acl.test.ts` | Tests for Windows ACL parsing |

### Key Types & Interfaces

| Type | File | Description |
|------|------|-------------|
| `SecurityAuditSeverity` | audit.ts | `"info" \| "warn" \| "critical"` |
| `SecurityAuditFinding` | audit.ts | Finding with checkId, severity, title, detail, remediation |
| `SecurityAuditSummary` | audit.ts | Count of critical/warn/info findings |
| `SecurityAuditReport` | audit.ts | Full audit report (ts, summary, findings, deep gateway probe) |
| `SecurityAuditOptions` | audit.ts | Options for `runSecurityAudit` (env, exec, verbose flags) |
| `PermissionCheck` | audit-fs.ts | File permission result (mode, bits, world/group writable/readable, symlink) |
| `PermissionCheckOptions` | audit-fs.ts | Options for platform, env, exec |
| `SecurityFixResult` | fix.ts | Result of auto-fix (changes, actions, errors) |
| `SecurityFixAction` | fix.ts | Union of chmod and icacls actions |
| `SkillScanSeverity` | skill-scanner.ts | `"info" \| "warn" \| "critical"` |
| `SkillScanFinding` | skill-scanner.ts | Code scan finding (ruleId, severity, file, line, evidence) |
| `SkillScanSummary` | skill-scanner.ts | Scan summary with counts and findings |
| `ExternalContentSource` | external-content.ts | Content source type for wrapping |
| `WrapExternalContentOptions` | external-content.ts | Options for content wrapping (source, sender, includeWarning) |
| `WindowsAclEntry` | windows-acl.ts | Parsed Windows ACL entry (principal, rights, canRead, canWrite) |
| `WindowsAclSummary` | windows-acl.ts | ACL summary (ok, entries, untrusted, trusted) |
| `ExecFn` | windows-acl.ts | Type alias for `typeof runExec` |

### Key Functions

| Function | File | Description |
|----------|------|-------------|
| `runSecurityAudit(opts)` | audit.ts | Main entry — runs all audit checks, returns `SecurityAuditReport` |
| `fixSecurityFootguns(opts?)` | fix.ts | Auto-remediates open groupPolicies, loose file perms, allowFrom normalization |
| `detectSuspiciousPatterns(content)` | external-content.ts | Regex-based prompt injection detection |
| `wrapExternalContent(content, opts)` | external-content.ts | Wraps untrusted content with XML-style security boundaries |
| `wrapWebContent(content, opts)` | external-content.ts | Wraps web-fetched content |
| `buildSafeExternalPrompt(params)` | external-content.ts | Builds complete safe prompt for external content |
| `isExternalHookSession(sessionKey)` | external-content.ts | Checks if session originates from external hook |
| `scanSource(source, filePath)` | skill-scanner.ts | Scans source code for dangerous patterns |
| `scanDirectory(dir, opts?)` | skill-scanner.ts | Scans all files in directory |
| `scanDirectoryWithSummary(dir, opts?)` | skill-scanner.ts | Scans directory, returns summary with counts |
| `safeEqualSecret(provided, expected)` | secret-equal.ts | Timing-safe string comparison |
| `inspectPathPermissions(path, opts?)` | audit-fs.ts | Returns `PermissionCheck` for a path |
| `collectChannelSecurityFindings(params)` | audit-channel.ts | Audits channel configs (allowFrom, groupPolicy) |
| `collectAttackSurfaceSummaryFindings(cfg)` | audit-extra.sync.ts | Summarizes attack surface (hooks, gateway, browser) |
| `collectExposureMatrixFindings(cfg)` | audit-extra.sync.ts | Matrix of enabled exposure points |
| `collectSecretsInConfigFindings(cfg)` | audit-extra.sync.ts | Detects secrets/tokens in config |
| `collectSmallModelRiskFindings(params)` | audit-extra.sync.ts | Warns about small models with weak safety |
| `collectPluginsCodeSafetyFindings(params)` | audit-extra.async.ts | Scans installed plugin code |
| `collectInstalledSkillsCodeSafetyFindings(params)` | audit-extra.async.ts | Scans installed skill code |
| `buildUntrustedChannelMetadata(params)` | channel-metadata.ts | Safely formats channel metadata |
| `pickSandboxToolPolicy(config?)` | audit-tool-policy.ts | Resolves tool allow/deny policy |
| `isPathInside(basePath, candidatePath)` | scan-paths.ts | Checks path containment |

### Internal Dependencies
- `../agents/*` — agent-scope, sandbox, skills, tool-policy, workspace-dirs, pi-tools.policy
- `../browser/config.js`, `../browser/control-auth.js` — browser config for audit
- `../channels/plugins/*` — channel plugin listing and helpers
- `../cli/command-format.js` — CLI command formatting
- `../config/*` — config loading, paths, commands, includes-scan
- `../gateway/*` — auth, call, node-command-policy, probe, probe-auth
- `../infra/errors.js` — error utilities
- `../pairing/pairing-store.js` — allowFrom store reading
- `../plugins/config-state.js` — plugin normalization
- `../process/exec.js` — shell execution for Windows ACL
- `../routing/session-key.js` — session key normalization
- `../shared/model-param-b.js` — model size inference

### External Dependencies
- `node:fs/promises`, `node:path`, `node:os`, `node:crypto`

### Data Flow
1. `runSecurityAudit()` orchestrates ~20 audit check functions
2. Each collector returns `SecurityAuditFinding[]` with severity levels
3. Findings are aggregated, counted, and returned as `SecurityAuditReport`
4. `fixSecurityFootguns()` reads config, applies auto-fixes, writes back
5. External content wrapping adds XML boundary tags + security notice before LLM processing
6. Skill scanner applies regex-based rules line-by-line against source files

### Security Model
- **Prompt injection defense**: `wrapExternalContent()` wraps all untrusted input with `<<<EXTERNAL_UNTRUSTED_CONTENT>>>` boundaries and security notices
- **Suspicious pattern detection**: Regex patterns for "ignore previous instructions", system prompt overrides, exec injection, etc.
- **File permission auditing**: Checks for world-writable/group-writable config/state files
- **Code scanning**: Static analysis of plugins/skills for `child_process`, `eval`, network access, filesystem access
- **Tool deny lists**: `DEFAULT_GATEWAY_HTTP_TOOL_DENY` prevents RCE via gateway HTTP; `DANGEROUS_ACP_TOOLS` requires approval
- **Timing-safe comparison**: `safeEqualSecret()` prevents timing attacks on token comparison
- **Auto-fix**: Tightens open groupPolicies to allowlist, fixes file permissions to 0o700/0o600

### Configuration
- Reads: `channels.*` (groupPolicy, dmPolicy, allowFrom), `hooks.*`, `gateway.*`, `sandbox.*`, `logging.redactSensitive`, `browser.*`
- Writes (fix): `channels.*.groupPolicy`, file permissions on state/config dirs

### Test Coverage (10 test files)
- `audit.test.ts` — Full audit flow with stubbed channel plugins
- `audit-extra.sync.test.ts` — Attack surface, hooks findings
- `dm-policy-channel-smoke.test.ts` — DM policy channel smoke tests
- `dm-policy-shared.test.ts` — Shared DM policy logic tests
- `external-content.test.ts` — Injection detection, content wrapping
- `fix.test.ts` — Auto-fix tightens policies and permissions
- `safe-regex.test.ts` — Safe regex validation
- `skill-scanner.test.ts` — Code scanning rules
- `temp-path-guard.test.ts` — Temp path guard validation
- `windows-acl.test.ts` — Windows ACL parsing

---

## 2. src/web

### Module Overview
WhatsApp Web integration module using `@whiskeysockets/baileys`. Handles WhatsApp authentication, session management, inbound message monitoring, outbound message sending, auto-reply orchestration, heartbeat scheduling, media handling, and group policy enforcement. Entry points: `loginWeb()`, `monitorWebInbox()`, `monitorWebChannel()`, `sendMessageWhatsApp()`.

#### v2026.2.15 Changes
- **Disallow workspace-\* roots without explicit localRoots** — image tool now requires explicit `localRoots` config to access workspace-prefixed directories (`src/agents/tools/image-tool.ts`)
- **Omit direct conversation labels from inbound metadata** — direct chats no longer inject conversation labels into inbound metadata to prevent metadata leakage

#### v2026.2.21 Changes <!-- v2026.2.21 -->
- **WhatsApp JID allowlist** — `fix(security)`: OC-91 all outbound WhatsApp sends now validate the destination JID against the configured allowlist in every send path. Centralized outbound auth returns 403 on tool auth errors

### File Inventory (non-test, 35+ source files)

| File | Description |
|------|-------------|
| `accounts.ts` | WhatsApp multi-account management (list, resolve, auth dirs) |
| `active-listener.ts` | Global registry of active WhatsApp listeners per account |
| `auth-store.ts` | WhatsApp credential storage (read/write/backup/restore creds) |
| `auto-reply.ts` | Re-export barrel for auto-reply implementation |
| `auto-reply.impl.ts` | Re-exports from auto-reply submodules |
| `inbound.ts` | Re-export barrel for inbound message handling |
| `login.ts` | WhatsApp login flow orchestration |
| `login-qr.ts` | QR code-based WhatsApp login (display + scan flow) |
| `media.ts` | Media loading/optimization (JPEG/PNG compression, local/remote fetch) |
| `outbound.ts` | Send WhatsApp messages, reactions, polls |
| `qr-image.ts` | QR code PNG rendering via manual QR library |
| `reconnect.ts` | Reconnection policy (exponential backoff, heartbeat config) |
| `session.ts` | WhatsApp socket creation and connection management |
| `vcard.ts` | vCard parsing for contact messages |
| **inbound/** | |
| `inbound/access-control.ts` | Inbound message allowFrom/denyFrom enforcement |
| `inbound/dedupe.ts` | Message deduplication (recent message cache) |
| `inbound/extract.ts` | Extract text, media placeholders, locations, mentions from raw messages |
| `inbound/media.ts` | Download inbound media attachments |
| `inbound/monitor.ts` | Main inbound message monitoring loop (socket event handler) |
| `inbound/send-api.ts` | Create send API for inbound context |
| `inbound/types.ts` | `WebInboundMessage`, `WebListenerCloseReason` types |
| **auto-reply/** | |
| `auto-reply/constants.ts` | Default media byte limits |
| `auto-reply/deliver-reply.ts` | Deliver AI reply to WhatsApp (chunking, media, reactions) |
| `auto-reply/heartbeat-runner.ts` | Heartbeat scheduling (periodic prompts to maintain sessions) |
| `auto-reply/loggers.ts` | Subsystem loggers for WhatsApp |
| `auto-reply/mentions.ts` | Bot mention detection config and resolution |
| `auto-reply/monitor.ts` | Main auto-reply monitor (connects to WhatsApp, routes messages to AI) |
| `auto-reply/session-snapshot.ts` | Capture session state for debugging |
| `auto-reply/types.ts` | `WebChannelStatus`, `WebMonitorTuning` types |
| `auto-reply/util.ts` | Text elision, crypto error detection |
| **auto-reply/monitor/** | |
| `monitor/ack-reaction.ts` | Send acknowledgment reactions (👀) on message receipt |
| `monitor/broadcast.ts` | Broadcast messages to multiple groups |
| `monitor/commands.ts` | Status command detection, mention stripping |
| `monitor/echo.ts` | Echo tracker (detect bot's own messages) |
| `monitor/group-activation.ts` | Group policy resolution (open/allowlist/mention-required) |
| `monitor/group-gating.ts` | Group conversation gating (cooldown, history tracking) |
| `monitor/group-members.ts` | Track and format group member lists |
| `monitor/last-route.ts` | Background task tracking, last route updates |
| `monitor/message-line.ts` | Format inbound messages for AI context |
| `monitor/on-message.ts` | Main message handler factory |
| `monitor/peer.ts` | Resolve peer ID from message |
| `monitor/process-message.ts` | Core message processing pipeline (access control → group gating → AI routing) |

### Key Types & Interfaces

| Type | File | Description |
|------|------|-------------|
| `ResolvedWhatsAppAccount` | accounts.ts | Account config with authDir, accountId, enabled status |
| `ActiveWebListener` | active-listener.ts | Active socket reference with send capability |
| `ActiveWebSendOptions` | active-listener.ts | Options for sending via active listener |
| `WebInboundMessage` | inbound/types.ts | Parsed inbound message (text, media, sender, group) |
| `WebListenerCloseReason` | inbound/types.ts | Why a listener closed |
| `InboundAccessControlResult` | inbound/access-control.ts | Access control decision (allowed/denied + reason) |
| `WebChannelStatus` | auto-reply/types.ts | Channel connection status |
| `WebMonitorTuning` | auto-reply/types.ts | Tuning params for monitor behavior |
| `WebMediaResult` | media.ts | Media load result (buffer, mime, path) |
| `MentionConfig` | auto-reply/mentions.ts | Bot mention detection config |
| `MentionTargets` | auto-reply/mentions.ts | Resolved mention targets |
| `GroupHistoryEntry` | monitor/group-gating.ts | Group conversation history tracking |
| `ReconnectPolicy` | reconnect.ts | Backoff + heartbeat config |
| `EchoTracker` | monitor/echo.ts | Tracks bot's own messages to avoid loops |

### Key Functions

| Function | File | Description |
|----------|------|-------------|
| `loginWeb(params)` | login.ts | Orchestrates WhatsApp login |
| `startWebLoginWithQr(params)` | login-qr.ts | QR-based login with terminal display |
| `waitForWebLogin(params)` | login-qr.ts | Wait for QR scan completion |
| `createWaSocket(params)` | session.ts | Create Baileys WhatsApp socket |
| `monitorWebInbox(options)` | inbound/monitor.ts | Start monitoring inbound messages |
| `monitorWebChannel(params)` | auto-reply/monitor.ts | Full auto-reply monitor (connect + monitor + reply) |
| `sendMessageWhatsApp(params)` | outbound.ts | Send text/media messages |
| `sendReactionWhatsApp(params)` | outbound.ts | Send emoji reactions |
| `sendPollWhatsApp(params)` | outbound.ts | Send polls |
| `checkInboundAccessControl(params)` | inbound/access-control.ts | Enforce allowFrom/denyFrom |
| `processMessage(params)` | monitor/process-message.ts | Core message processing pipeline |
| `deliverWebReply(params)` | auto-reply/deliver-reply.ts | Deliver AI response to WhatsApp |
| `runWebHeartbeatOnce(opts)` | auto-reply/heartbeat-runner.ts | Execute one heartbeat cycle |
| `loadWebMedia(params)` | media.ts | Load media from URL/path with optimization |
| `optimizeImageToJpeg(params)` | media.ts | Compress images for WhatsApp |
| `extractText(rawMessage)` | inbound/extract.ts | Extract text from Baileys message |
| `extractMediaPlaceholder(msg)` | inbound/extract.ts | Create placeholder for media messages |
| `applyGroupGating(params)` | monitor/group-gating.ts | Apply group cooldown/gating logic |
| `parseVcard(vcard)` | vcard.ts | Parse vCard contact strings |
| `renderQrPngBase64(text)` | qr-image.ts | Render QR as PNG base64 |

### Internal Dependencies
- `../auto-reply/*` — heartbeat, tokens, reply, chunk, command-detection, templating
- `../agents/identity.js` — agent identity
- `../channels/*` — ack-reactions, location, mention-gating, reply-prefix, plugins
- `../config/*` — config loading, paths, sessions, group-policy
- `../globals.js` — global state
- `../infra/*` — channel-activity, dedupe, heartbeat-events, system-events, backoff, net/ssrf
- `../logging/*` — subsystem loggers
- `../markdown/*` — WhatsApp markdown formatting
- `../media/*` — store, image-ops, fetch, mime, constants
- `../outbound.js` — generic outbound
- `../pairing/*` — pairing store/messages
- `../routing/*` — session-key, resolve-route
- `../runtime.js` — runtime state
- `../session.js` — session management

### External Dependencies
- `@whiskeysockets/baileys` — WhatsApp Web protocol implementation
- `sharp` — Image processing/optimization
- `qrcode-terminal` — QR code terminal display
- `node:crypto`, `node:events`, `node:fs`, `node:fs/promises`, `node:os`, `node:path`, `node:url`

### Data Flow
1. `monitorWebChannel()` creates a Baileys socket via `createWaSocket()`
2. Socket connects to WhatsApp servers with stored credentials from `auth-store`
3. `monitorWebInbox()` hooks into socket events for incoming messages
4. Each message goes through: dedup → access control → group gating → mention check → AI routing
5. AI response is delivered via `deliverWebReply()` which chunks text, sends media
6. Heartbeat runner periodically sends prompts to maintain active conversations
7. Reconnection uses exponential backoff on disconnects

### Security Model
- **Access control**: `checkInboundAccessControl()` enforces allowFrom/denyFrom per channel config
- **Group policy**: `resolveGroupActivationFor()` enforces open/allowlist/mention-required policies
- **Deduplication**: Prevents processing duplicate messages
- **Echo detection**: Prevents bot from responding to its own messages
- **Media limits**: Default 5MB cap on media
- **SSRF protection**: Uses `../infra/net/ssrf.js` for URL validation
- **WhatsApp JID allowlist** <!-- v2026.2.21 -->: OC-91 — all outbound sends validate destination JID against configured allowlist; centralized outbound auth returns 403 on tool auth errors

### Configuration
- `channels.whatsapp.*` — account config, groupPolicy, dmPolicy, allowFrom, mentionMode
- `heartbeat.*` — heartbeat intervals, recipients
- Credential storage in state dir (`whatsapp-auth/`)

### Test Coverage (35 test files)
- Login flow tests, QR rendering, session management
- Inbound monitoring, access control, media handling
- Auto-reply routing, broadcast, group gating, mentions
- Heartbeat runner, deliver-reply, message formatting
- E2E tests for reconnection, compression, monitor logging

---

## 3. src/browser

### Module Overview
Browser automation module providing a local HTTP control server for Playwright and Chrome DevTools Protocol (CDP) based browser control. Supports multiple browser profiles, Chrome extension relay, screenshots, DOM/ARIA snapshots, page interactions, storage management, and AI-assisted browser actions. Entry points: `startBrowserControlServerFromConfig()`, `browserStatus()`, `browserAct()`.

#### v2026.2.15 Changes
- **Stop LLM retry loop when browser control unavailable** — `client-fetch.ts` now returns a clear "browser is currently unavailable" message instead of letting the LLM retry indefinitely
- **Share CDP fetch helpers** — extracted `cdp.helpers.ts` as shared module with `getHeadersWithAuth()`, `CdpSendFn` type, WebSocket helpers, and loopback detection (previously inlined)
- **Share common server middleware** — new `server-middleware.ts` extracts `installBrowserCommonMiddleware()` (abort signal, JSON parsing, CSRF guard) and `installBrowserAuthMiddleware()` from server.ts
- **Isolate profile hot-reload config refresh** — new `resolved-config-refresh.ts` extracts `refreshResolvedBrowserConfigFromDisk()` and `applyResolvedConfig()` for cleaner hot-reload lifecycle

#### v2026.2.19 Changes
- **Browser SSRF policy** — Browser URL navigation now routed through SSRF guard; configurable via `browser.ssrfPolicy` config key (see DEVELOPER-REFERENCE.md §6)
- **Chrome extension relay auth** — Both `/extension` and `/cdp` endpoints now require `gateway.auth.token` authentication
- **Canvas node-scoped sessions** — Canvas session capabilities are node-scoped, replacing shared-IP fallback auth

#### v2026.2.23 Changes
- **BREAKING: Browser SSRF default flipped to trusted-network** — `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` now defaults to `true` (trusted-network mode) when unset, allowing browser navigation to private/LAN addresses. Canonical config key is `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` (was `browser.ssrfPolicy.allowPrivateNetwork`). `openclaw doctor --fix` migrates the legacy key automatically. To enforce strict SSRF blocking, explicitly set `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: false`.

#### v2026.2.24 Changes

- **Sandbox / Symlink-parent bypass** (security, @tdjackey): bind-mount source paths are now canonicalized via existing-ancestor `realpath`, so symlink-parent + non-existent-leaf paths can no longer bypass allowed-source-roots or blocked-path checks. Released in `v2026.2.24`.
- **Native images / workspaceOnly enforcement** (security, @tdjackey): `tools.fs.workspaceOnly` is now enforced for native prompt image auto-load (including history refs), preventing out-of-workspace sandbox mounts from being implicitly ingested as vision input. Released in `v2026.2.24`. Contributor: @tdjackey (reported).
- **Control UI / Chat image URL safety** (#25444): image click URL opening now uses a centralized allowlist (`http/https/blob` + opt-in `data:image/*`) with opener isolation (`noopener,noreferrer` + `window.opener = null`) to prevent tabnabbing and unsafe schemes. Contributor: @shakkernerd.

#### v2026.3.1 Changes

- **Browser auth fail-closed bootstrap** (@ijxpwastaken): if browser-control auth auto-setup fails and no explicit `token`/`password` is configured, the browser control HTTP server now aborts startup instead of running unauthenticated. `server.ts` tracks `browserAuthBootstrapFailed` and checks for fallback auth before proceeding. New file `server.auth-fail-closed.test.ts` covers regression.
- **Sandbox browser Docker `OPENCLAW_BROWSER_NO_SANDBOX=1`** (@Lukavyi): sandbox browser containers now receive `OPENCLAW_BROWSER_NO_SANDBOX=1` env var, and the security hash epoch is bumped to `2026-02-28-no-sandbox-env` so existing containers are recreated on upgrade. Comment in `browser.ts` clarifies that Chromium's setuid/namespace sandbox cannot work inside Docker containers without additional privileges (#29879).
- **noVNC observer hardening**: observer token TTL reduced from 5 minutes to 60 seconds (`NOVNC_TOKEN_TTL_MS`), password entropy increased (8-char alphanumeric via `crypto.randomInt` over `NOVNC_PASSWORD_ALPHABET` instead of 4-byte hex). Token redemption no longer issues a `302` redirect (which placed credentials in `Location` query strings); `bridge-server.ts` now returns a no-cache/no-referrer bootstrap HTML page that client-side navigates to the noVNC URL with credentials in a fragment hash. The `NoVncObserverTokenEntry` type now stores `noVncPort`+`password` instead of a pre-built URL.
- **Writable output path hardening**: new `output-atomic.ts` module provides `writeViaSiblingTempPath()` — browser download and trace outputs are finalized via sibling `.openclaw-output-{uuid}-{name}.part` temp files with atomic rename, and existing hardlinked writable targets are rejected (`nlink > 1` check in `validateCanonicalPathWithinRoot`), blocking hardlink-alias overwrite paths under browser temp roots.
- **Browser open & navigate `url` alias** (@vincentkoc): `open` and `navigate` browser tool actions accept `url` as an alias parameter alongside existing parameter names via `browser-tool.schema.ts` and `browser-tool.ts` (#29260).
- **Browser paths outside-workspace error** (@YuzuruS): `resolveCheckedPathsWithinRoot` now maps `SafeOpenError` with code `"outside-workspace"` to a specific `"File is outside {scopeLabel}"` message instead of a generic invalid-path error. New `outside-workspace` safe-open error code propagated across edit/browser/media consumers (#29715).
- **Navigate targetId resolution** (#25326): after renderer-swap navigations, the response now returns the correct `targetId` from the new target instead of the stale pre-swap ID.
- **Snapshot route test coverage**: new `routes/agent.snapshot.test.ts` adds contract tests for snapshot routes.

### File Inventory (non-test, ~78 source files, ~47 test files)

| File | Description |
|------|-------------|
| **Core Infrastructure** | |
| `config.ts` | Browser config resolution (profiles, CDP URLs, enabled state) |
| `constants.ts` | Default browser settings (enabled, evaluate, colors, profile names, snapshot limits) |
| `paths.ts` | Browser temp dirs, download/upload dirs, path resolution |
| `target-id.ts` | Resolve targetId from tab list |
| `trash.ts` | Move files to trash safely |
| **Chrome Management** | |
| `chrome.ts` | Chrome launch/stop, CDP connectivity check, WebSocket URL discovery |
| `chrome.executables.ts` | Find Chrome executables across Mac/Linux/Windows |
| `chrome.profile-decoration.ts` | Decorate Chrome profiles with OpenClaw branding |
| **CDP Layer** | |
| `cdp.ts` | CDP operations — screenshot, JS evaluation, ARIA/DOM snapshot, target creation |
| `cdp.helpers.ts` | CDP helpers — WebSocket, JSON fetch, auth headers, loopback detection |
| **Playwright Layer** | |
| `pw-session.ts` | Playwright session management — page/context state, ref tracking, connection management |
| `pw-ai.ts` | Re-exports from `@anthropic-ai/sdk` Playwright tools |
| `pw-ai-module.ts` | Lazy-load Playwright AI module |
| `pw-ai-state.ts` | Track if PW AI module is loaded |
| `pw-role-snapshot.ts` | Role-based accessibility snapshot (ref mapping, stats) |
| `pw-tools-core.ts` | Barrel re-export of all Playwright tool operations |
| `pw-tools-core.activity.ts` | Page errors, network requests, console messages via Playwright |
| `pw-tools-core.downloads.ts` | File upload, dialog, download handling via Playwright |
| `pw-tools-core.interactions.ts` | Click, hover, drag, type, fill, evaluate, screenshot, scroll, wait via Playwright |
| `pw-tools-core.responses.ts` | HTTP response body capture |
| `pw-tools-core.shared.ts` | Shared helpers (ref validation, timeout normalization) |
| `pw-tools-core.snapshot.ts` | ARIA/AI/Role snapshots, navigation, PDF via Playwright |
| `pw-tools-core.state.ts` | Browser state setters (offline, headers, geolocation, media, timezone, device) |
| `pw-tools-core.storage.ts` | Cookie and localStorage get/set/clear via Playwright |
| `pw-tools-core.trace.ts` | Trace start/stop via Playwright |
| **Server** | |
| `server.ts` | Start/stop browser control HTTP server |
| `server-context.ts` | Server state management (profiles, tabs, hot-reload, context creation) |
| `server-context.types.ts` | Types for server state, profile runtime, route context |
| `server-middleware.ts` | Express middleware (CORS, auth) |
| `control-service.ts` | Browser control service lifecycle |
| `control-auth.ts` | Browser control auth token management (generate, persist, resolve) |
| `http-auth.ts` | HTTP request authorization check |
| `csrf.ts` | CSRF protection middleware for browser mutations |
| **Routes** | |
| `routes/index.ts` | Register all browser routes |
| `routes/basic.ts` | Basic routes (status, profiles, start, stop, reset) |
| `routes/tabs.ts` | Tab management routes (list, open, close, focus) |
| `routes/agent.ts` | Register all agent routes |
| `routes/agent.act.ts` | Agent act routes (click, type, press, hover, drag, fill, etc.) |
| `routes/agent.act.shared.ts` | Act kind definitions and parsing |
| `routes/agent.snapshot.ts` | Snapshot routes (ARIA, AI, role snapshots, navigate, screenshot) |
| `routes/agent.storage.ts` | Storage routes (cookies, localStorage) |
| `routes/agent.debug.ts` | Debug routes (console, network, errors, trace) |
| `routes/agent.shared.ts` | Shared route helpers (body parsing, error handling, profile resolution) |
| `routes/dispatcher.ts` | Route dispatcher for programmatic access |
| `routes/types.ts` | Route handler types |
| `routes/utils.ts` | Route utilities (profile context, JSON error, type coercion) |
| `routes/path-output.ts` | Re-export paths |
| **Client** | |
| `client.ts` | Browser client API (status, profiles, start, stop, tabs, snapshots) |
| `client-fetch.ts` | HTTP fetch for browser control server (auth, retry) |
| `client-actions.ts` | Barrel re-export of all client actions |
| `client-actions-core.ts` | Core client actions (navigate, act, screenshot, dialog, download) |
| `client-actions-observe.ts` | Observe actions (console, PDF, errors, requests, trace) |
| `client-actions-state.ts` | State actions (cookies, storage, offline, headers, geolocation, media) |
| `client-actions-types.ts` | Client action result types |
| **Extension & Bridge** | |
| `extension-relay.ts` | Chrome extension relay server (WebSocket bridge for Chrome extension) |
| `bridge-server.ts` | Browser bridge WebSocket server |
| `bridge-auth-registry.ts` | Auth registry for bridge connections |
| **Other** | |
| `profiles.ts` | Profile management (CDP port allocation, name validation, colors) |
| `profiles-service.ts` | Profile CRUD service (create, delete, list) |
| `proxy-files.ts` | Proxy file persistence for browser results |
| `screenshot.ts` | Screenshot normalization (resize, byte limit) |
| `output-atomic.ts` | Atomic write-via-rename for browser download/trace output files |
| `resolved-config-refresh.ts` | Hot-reload browser config from disk |

### Key Types & Interfaces

| Type | File | Description |
|------|------|-------------|
| `ResolvedBrowserConfig` | config.ts | Resolved browser config (enabled, base URL, profiles, evaluate) |
| `ResolvedBrowserProfile` | config.ts | Single profile config (name, cdpUrl, color, managed) |
| `BrowserStatus` | client.ts | Status response (running, profiles) |
| `ProfileStatus` | client.ts | Profile status (name, running, cdpUrl, pages) |
| `BrowserTab` | client.ts | Tab info (targetId, url, title) |
| `SnapshotAriaNode` | client.ts | ARIA snapshot node |
| `SnapshotResult` | client.ts | Snapshot response (snapshot text, refs, stats) |
| `BrowserActRequest` | client-actions-core.ts | Union type for all act request kinds |
| `BrowserActResponse` | client-actions-core.ts | Act response (output, screenshot, refs) |
| `BrowserFormField` | client-actions-core.ts | Form field for fill action |
| `RunningChrome` | chrome.ts | Running Chrome instance (process, CDP URL, user data dir) |
| `BrowserExecutable` | chrome.executables.ts | Found browser executable (path, channel, version) |
| `BrowserServerState` | server-context.types.ts | Server state (config, profiles, routes) |
| `BrowserRouteContext` | server-context.types.ts | Route handler context (state, config, profile ops) |
| `ProfileRuntimeState` | server-context.types.ts | Per-profile runtime state (running, cdpUrl, chrome ref) |
| `ProfileContext` | server-context.types.ts | Resolved profile for route handler |
| `BrowserControlAuth` | control-auth.ts | Auth token for browser control |
| `RoleRef` / `RoleRefMap` | pw-role-snapshot.ts | Role-based element reference system |
| `RoleSnapshotStats` | pw-role-snapshot.ts | Snapshot statistics (elements, refs, chars) |
| `ActKind` | routes/agent.act.shared.ts | Union of all action kinds (click, type, press, hover, drag, select, fill, etc.) |
| `BrowserRequest` / `BrowserResponse` | routes/types.ts | Route handler request/response types |
| `BrowserRouteRegistrar` | routes/types.ts | Route registration interface |
| `ChromeExtensionRelayServer` | extension-relay.ts | Relay server instance type |
| `AriaSnapshotNode` / `RawAXNode` | cdp.ts | CDP accessibility tree node types |
| `DomSnapshotNode` | cdp.ts | DOM snapshot node type |
| `PageState` / `ContextState` | pw-session.ts | Playwright page/context state tracking |
| `BrowserConsoleMessage` | pw-session.ts | Captured console message |
| `BrowserPageError` | pw-session.ts | Captured page error |
| `BrowserNetworkRequest` | pw-session.ts | Captured network request |

### Key Functions

| Function | File | Description |
|----------|------|-------------|
| `startBrowserControlServerFromConfig()` | server.ts | Start browser control HTTP server |
| `stopBrowserControlServer()` | server.ts | Stop browser control server |
| `createBrowserRouteContext(opts)` | server-context.ts | Create route context with profile management |
| `launchOpenClawChrome(params)` | chrome.ts | Launch Chrome with CDP debugging port |
| `stopOpenClawChrome(running)` | chrome.ts | Stop Chrome instance |
| `resolveBrowserConfig(cfg)` | config.ts | Resolve browser config from OpenClaw config |
| `resolveBrowserControlAuth(cfg)` | control-auth.ts | Resolve/generate auth token |
| `browserAct(params)` | client-actions-core.ts | Execute browser action (click, type, etc.) |
| `browserNavigate(params)` | client-actions-core.ts | Navigate to URL |
| `browserScreenshotAction(params)` | client-actions-core.ts | Take screenshot |
| `browserStatus(baseUrl?)` | client.ts | Get browser status |
| `browserStart(baseUrl?, opts?)` | client.ts | Start browser profile |
| `browserStop(baseUrl?, opts?)` | client.ts | Stop browser profile |
| `captureScreenshot(opts)` | cdp.ts | CDP screenshot capture |
| `snapshotAria(opts)` | cdp.ts | CDP ARIA tree snapshot |
| `evaluateJavaScript(opts)` | cdp.ts | CDP JS evaluation |
| `buildRoleSnapshotFromAriaSnapshot(params)` | pw-role-snapshot.ts | Build role-ref snapshot from ARIA tree |
| `snapshotRoleViaPlaywright(opts)` | pw-tools-core.snapshot.ts | Full role snapshot via Playwright |
| `clickViaPlaywright(opts)` | pw-tools-core.interactions.ts | Click element by ref |
| `typeViaPlaywright(opts)` | pw-tools-core.interactions.ts | Type text into element |
| `evaluateViaPlaywright(opts)` | pw-tools-core.interactions.ts | Evaluate JS in page context |
| `ensureChromeExtensionRelayServer(opts)` | extension-relay.ts | Start/ensure Chrome extension relay |
| `registerBrowserRoutes(app, ctx)` | routes/index.ts | Register all HTTP routes |
| `createBrowserRouteDispatcher(ctx)` | routes/dispatcher.ts | Programmatic route dispatcher |
| `shouldRejectBrowserMutation(params)` | csrf.ts | CSRF check for mutations |
| `isAuthorizedBrowserRequest(params)` | http-auth.ts | Auth check for requests |

### Internal Dependencies
- `../config/*` — config, paths, port-defaults, types
- `../cli/command-format.js` — CLI formatting
- `../gateway/*` — auth, net
- `../infra/*` — env, errors, ports, tmp-openclaw-dir, ws
- `../logging/subsystem.js` — subsystem loggers
- `../media/*` — image-ops, store
- `../process/exec.js` — shell execution
- `../security/secret-equal.js` — timing-safe auth comparison

### External Dependencies
- `playwright-core` — Browser automation framework
- `express` — HTTP server for control API
- `ws` — WebSocket for CDP and extension relay
- `sharp` — Image processing for screenshots
- `undici` — HTTP client
- `node:child_process`, `node:crypto`, `node:fs`, `node:http`, `node:net`, `node:os`, `node:path`, `node:stream`

### Data Flow
1. `startBrowserControlServerFromConfig()` launches Express server with auth middleware
2. Routes register under `/agent/*` (act, snapshot, storage, debug) and basic endpoints
3. Client sends requests via `fetchBrowserJson()` with auth token
4. Server resolves profile → gets CDP URL → uses Playwright or raw CDP to execute
5. Chrome managed profiles: launched via `launchOpenClawChrome()` with allocated CDP port
6. Chrome extension relay: WebSocket bridge lets Chrome extension attach existing tabs
7. Snapshots: ARIA tree fetched via CDP, converted to role-ref format with numbered refs (e1, e2...)
8. Actions: ref-based element targeting → Playwright locator → interaction

### Security Model
- **Auth tokens**: Browser control server requires bearer token (auto-generated, stored in state)
- **CSRF protection**: Mutation requests require matching token
- **Loopback detection**: CDP connections validated as loopback
- **Bridge auth registry**: Per-port auth for bridge connections
- **Evaluate gating**: JS evaluation can be disabled via config (`browser.evaluateEnabled: false`)
- **Path containment**: File operations restricted to workspace root
- **Non-network navigation schemes blocked** <!-- v2026.2.21 -->: `file://`, `javascript:`, and `data:` URI navigation blocked in browser module
- **noVNC observer auth** <!-- v2026.2.21 -->: noVNC observer sessions require one-time tokens plus mandatory password auth
- **Sandbox --no-sandbox disabled** <!-- v2026.2.21 -->: Chrome/Chromium sandbox enabled by default in container runs (#22451)
- **Sandbox browser network hardened** <!-- v2026.2.21 -->: outbound network restrictions tightened for sandboxed browser runs
- **Sandbox browser hash migration** <!-- v2026.2.21 -->: browser session labels migrated from SHA-1 to SHA-256; stale labels flagged by audit check `sandbox.browser_container.hash_epoch_stale`
- **SHA-256 synthetic IDs** <!-- v2026.2.21 -->: all synthetically-generated IDs migrated from SHA-1 to SHA-256 (#22528/#7343)
- **Auth fail-closed bootstrap** <!-- v2026.3.1 -->: browser control server startup aborts if auth bootstrap fails and no explicit fallback auth exists
- **noVNC observer hardening** <!-- v2026.3.1 -->: shorter token TTL (60s), higher password entropy, credentials kept out of `Location` headers via fragment-hash bootstrap page
- **Writable output atomic finalization** <!-- v2026.3.1 -->: download/trace output files written via sibling temp + atomic rename; hardlinked targets rejected
- **Outside-workspace error propagation** <!-- v2026.3.1 -->: dedicated `outside-workspace` error code prevents misleading not-found/invalid-path fallbacks

### Configuration
- `browser.enabled` — Enable/disable browser control (default: true)
- `browser.evaluateEnabled` — Enable/disable JS evaluation (default: true)
- `browser.profiles.*` — Profile definitions (`cdpUrl`, color, managed)
- `browser.defaultProfile` — Default profile name (default: "chrome")
- `browser.relayBindHost` — Optional non-loopback bind host for the extension relay in WSL2/cross-namespace setups
- Browser control auth token stored in state dir

### Test Coverage (47 test files)
- Chrome launch/detection, CDP helpers, profile management
- Server auth, CSRF, route contracts (snapshots, act, tabs, storage)
- Playwright session management, role snapshots, AI integration
- Extension relay auth, client fetch auth
- Screenshots, downloads, interactions, evaluate abort
- Config resolution, target ID resolution

---

## 4. src/canvas-host

### Module Overview
Lightweight HTTP server for hosting Canvas UI content (HTML pages, A2UI push updates). Provides file serving with live reload WebSocket support for development. Entry points: `startCanvasHost()`, `createCanvasHostHandler()`.

### File Inventory (5 files)

| File | Description |
|------|-------------|
| `a2ui.ts` | A2UI (Agent-to-UI) HTTP handler and live reload injection |
| `file-resolver.ts` | Secure file path resolution within root directory |
| `server.ts` | Canvas host HTTP server (Express-based, file serving, WebSocket) |
| `server.test.ts` | Tests for canvas host server |
| `server.state-dir.test.ts` | Tests for state directory handling |

### Key Types & Interfaces

| Type | File | Description |
|------|------|-------------|
| `CanvasHostOpts` | server.ts | Canvas host options (stateDir, port, host) |
| `CanvasHostServerOpts` | server.ts | Server options extending CanvasHostOpts |
| `CanvasHostServer` | server.ts | Running server handle (port, close) |
| `CanvasHostHandler` | server.ts | Handler without own server (middleware, websocket handler) |
| `CanvasHostHandlerOpts` | server.ts | Handler creation options |

### Key Functions

| Function | File | Description |
|----------|------|-------------|
| `startCanvasHost(opts)` | server.ts | Start standalone canvas host server |
| `createCanvasHostHandler(opts)` | server.ts | Create handler for embedding in existing server |
| `handleA2uiHttpRequest(req, res)` | a2ui.ts | Handle A2UI push requests |
| `injectCanvasLiveReload(html)` | a2ui.ts | Inject WebSocket live reload script into HTML |
| `normalizeUrlPath(rawPath)` | file-resolver.ts | Normalize URL path |
| `resolveFileWithinRoot(root, urlPath)` | file-resolver.ts | Securely resolve file within root (prevents traversal) |

### Key Constants

| Constant | File | Description |
|----------|------|-------------|
| `A2UI_PATH` | a2ui.ts | `/__openclaw__/a2ui` — A2UI endpoint |
| `CANVAS_HOST_PATH` | a2ui.ts | `/__openclaw__/canvas` — Canvas endpoint |
| `CANVAS_WS_PATH` | a2ui.ts | `/__openclaw__/ws` — WebSocket endpoint |

### Internal Dependencies
- Minimal — primarily uses standard Node.js APIs

### External Dependencies
- `node:fs`, `node:path`, `node:http`

### Data Flow
1. Canvas host serves static HTML/CSS/JS files from a state directory
2. A2UI endpoint receives push updates from agents
3. WebSocket connection enables live reload of canvas content
4. `injectCanvasLiveReload()` adds reload script to served HTML

### Security Model
- **Path traversal prevention**: `resolveFileWithinRoot()` prevents directory traversal attacks
- Files only served from within configured root directory

### Configuration
- Canvas host port, state directory

### Test Coverage (2 test files)
- `server.test.ts` — Server lifecycle, file serving
- `server.state-dir.test.ts` — State directory resolution

---

## 5. src/plugins

### Module Overview
Plugin system for OpenClaw — handles discovery, installation, loading, lifecycle management, hooks, tools, commands, HTTP routes, providers, and configuration validation. Supports bundled plugins, npm-installed plugins, and local directory plugins. Entry points: `loadOpenClawPlugins()`, `installPluginFromPath()`, `createHookRunner()`.

### File Inventory (non-test, ~30 source files)

| File | Description |
|------|-------------|
| `bundled-dir.ts` | Resolve bundled plugins directory |
| `cli.ts` | Register plugin CLI commands (install, uninstall, list, update) |
| `commands.ts` | Plugin command system (register, match, execute custom commands) |
| `config-schema.ts` | Empty plugin config schema factory |
| `config-state.ts` | Plugin enable/disable state normalization, slot decisions |
| `discovery.ts` | Plugin discovery (scan bundled, npm, workspace dirs for plugins) |
| `enable.ts` | Enable plugin in config |
| `hook-runner-global.ts` | Global hook runner singleton management |
| `hooks.ts` | Hook runner implementation (lifecycle hooks: session, message, gateway, compaction, tool-call) |
| `http-path.ts` | Normalize plugin HTTP paths |
| `http-registry.ts` | Register plugin HTTP routes |
| `install.ts` | Plugin installation (from archive, directory, npm, file) |
| `installs.ts` | Plugin install record tracking |
| `loader.ts` | Plugin loading (discover → load manifests → require modules → register) |
| `manifest.ts` | Plugin manifest parsing (`openclaw.plugin.json`) |
| `manifest-registry.ts` | Plugin manifest registry (load and index all manifests) |
| `providers.ts` | Resolve plugin-provided LLM providers |
| `registry.ts` | Plugin registry (stores all registrations: tools, hooks, channels, providers, commands) |
| `runtime.ts` | Active plugin registry singleton |
| `schema-validator.ts` | JSON Schema validation for plugin config |
| `services.ts` | Plugin service lifecycle (start/stop) |
| `slots.ts` | Exclusive plugin slot system (e.g., memory slot) |
| `source-display.ts` | Format plugin source location for display |
| `status.ts` | Build plugin status report |
| `tools.ts` | Resolve plugin-provided tools |
| `types.ts` | Core plugin type definitions |
| `uninstall.ts` | Plugin uninstallation |
| `update.ts` | Plugin update (npm update, channel sync) |
| `runtime/index.ts` | Plugin runtime implementation (logger, config, tools registration API) |
| `runtime/native-deps.ts` | Native dependency hints for plugins |
| `runtime/types.ts` | Plugin runtime type definitions |

### Key Types & Interfaces

| Type | File | Description |
|------|------|-------------|
| `PluginManifest` | manifest.ts | Plugin manifest schema (id, version, main, channels, hooks) |
| `PluginRegistry` | registry.ts | Central registry of all loaded plugins and their registrations |
| `PluginRecord` | registry.ts | Single plugin's registrations |
| `PluginToolRegistration` | registry.ts | Registered tool from a plugin |
| `PluginHookRegistration` | registry.ts | Registered hook from a plugin |
| `PluginChannelRegistration` | registry.ts | Registered channel from a plugin |
| `PluginProviderRegistration` | registry.ts | Registered provider from a plugin |
| `PluginCommandRegistration` | registry.ts | Registered command from a plugin |
| `PluginHttpRouteRegistration` | registry.ts | Registered HTTP route from a plugin |
| `PluginServiceRegistration` | registry.ts | Registered service from a plugin |
| `PluginRuntime` | runtime/types.ts | Runtime API passed to plugins (logger, config, tools, hooks) |
| `RuntimeLogger` | runtime/types.ts | Plugin logger interface |
| `HookRunner` | hooks.ts | Hook execution engine |
| `HookRunnerOptions` | hooks.ts | Hook runner config |
| `PluginCandidate` | discovery.ts | Discovered plugin candidate |
| `PluginDiscoveryResult` | discovery.ts | Discovery result (candidates, errors) |
| `PluginLoadResult` | loader.ts | Alias for `PluginRegistry` |
| `PluginLoadOptions` | loader.ts | Loader options (cfg, workspaceDir, bundledDir) |
| `InstallPluginResult` | install.ts | Installation result |
| `UninstallPluginResult` | uninstall.ts | Uninstall result |
| `PluginUpdateOutcome` | update.ts | Per-plugin update result |
| `PluginUpdateSummary` | update.ts | Update batch summary |
| `NormalizedPluginsConfig` | config-state.ts | Normalized plugin enable/disable config |
| `PluginStatusReport` | status.ts | Full plugin status for display |
| `PluginManifestRecord` | manifest-registry.ts | Manifest with resolved paths |
| `PluginManifestRegistry` | manifest-registry.ts | Indexed manifest registry |
| `OpenClawPluginConfigSchema` | types.ts | Plugin config schema definition |
| `OpenClawPluginToolContext` | types.ts | Context passed to tool factories |
| `OpenClawPluginToolFactory` | types.ts | Factory function for creating tools |
| `PluginKind` | types.ts | `"memory"` — plugin kind |
| `ProviderAuthMethod` | types.ts | Auth method for provider plugins |
| `SlotSelectionResult` | slots.ts | Exclusive slot selection result |
| `CommandRegistrationResult` | commands.ts | Command registration result |
| `PluginServicesHandle` | services.ts | Handle for started plugin services |

### Key Functions

| Function | File | Description |
|----------|------|-------------|
| `loadOpenClawPlugins(options)` | loader.ts | Main loader: discover → load → register all plugins |
| `discoverOpenClawPlugins(params)` | discovery.ts | Scan for plugin candidates in bundled/npm/workspace dirs |
| `createPluginRegistry(params)` | registry.ts | Create empty plugin registry |
| `createHookRunner(registry, options)` | hooks.ts | Create hook execution engine |
| `initializeGlobalHookRunner(registry)` | hook-runner-global.ts | Initialize global singleton |
| `installPluginFromPath(params)` | install.ts | Install plugin from local path |
| `installPluginFromNpmSpec(params)` | install.ts | Install plugin from npm |
| `uninstallPlugin(params)` | uninstall.ts | Uninstall plugin |
| `updateNpmInstalledPlugins(params)` | update.ts | Batch update npm plugins |
| `enablePluginInConfig(cfg, pluginId)` | enable.ts | Enable plugin in config |
| `registerPluginCommand(params)` | commands.ts | Register custom command |
| `matchPluginCommand(input)` | commands.ts | Match input against registered commands |
| `executePluginCommand(params)` | commands.ts | Execute matched command |
| `registerPluginHttpRoute(params)` | http-registry.ts | Register HTTP route |
| `resolvePluginTools(params)` | tools.ts | Resolve tools from registry |
| `resolvePluginProviders(params)` | providers.ts | Resolve LLM providers from registry |
| `normalizePluginsConfig(cfg)` | config-state.ts | Normalize plugin enable/disable state |
| `applyExclusiveSlotSelection(params)` | slots.ts | Apply exclusive slot logic |
| `createPluginRuntime()` | runtime/index.ts | Create runtime API for plugin |
| `loadPluginManifestRegistry(params)` | manifest-registry.ts | Load all plugin manifests |
| `buildPluginStatusReport(params)` | status.ts | Build display-ready status report |
| `registerPluginCliCommands(program, cfg)` | cli.ts | Register CLI subcommands |
| `validateJsonSchemaValue(params)` | schema-validator.ts | Validate config against JSON schema |

### Internal Dependencies
- `../agents/tools/common.js` — tool types
- `../channels/plugins/*` — channel plugin types
- `../compat/legacy-names.js` — manifest key
- `../config/*` — config loading
- `../routing/session-key.js` — session key normalization

### External Dependencies
- `commander` — CLI command registration
- `jiti` — TypeScript/ESM module loading
- `jszip` — Archive extraction
- `tar` — Tarball extraction
- `ajv` — JSON Schema validation
- `node:crypto`, `node:fs`, `node:fs/promises`, `node:http`, `node:module`, `node:os`, `node:path`, `node:url`

### Data Flow
1. `loadOpenClawPlugins()` calls `discoverOpenClawPlugins()` to find candidates
2. Each candidate's manifest is loaded and validated
3. Plugin modules are `require()`'d via `jiti` for TypeScript support
4. Plugins call runtime API to register tools, hooks, commands, channels, providers, HTTP routes
5. `createHookRunner()` wraps registry for hook execution (session start/end, message, gateway, compaction, tool-call)
6. Hook runner fires hooks in registration order with error isolation

### Security Model
- **Plugin code scanning**: Security module scans plugin code for dangerous patterns
- **Plugin trust**: Audit checks for unsigned/untrusted plugins
- **Exclusive slots**: Prevents conflicting plugins (e.g., two memory plugins)
- **JSON Schema validation**: Plugin config validated against declared schemas

### Configuration
- `plugins.*` — Plugin enable/disable state
- `plugins.entries.<id>.enabled` — Per-plugin toggle
- Plugin manifests in `openclaw.plugin.json`

### Test Coverage (16 test files)
- Discovery, loader, manifest registry, config state, CLI
- Hook wiring (session, message, gateway, compaction, tool-call)
- Commands, slots, source display
- Install/uninstall, tools optional behavior
- Voice call plugin test

---

## 6. src/plugin-sdk

### Module Overview
Public SDK for plugin authors. Re-exports essential types and utilities from internal modules so plugins don't need to import from deep internal paths. Entry point: `index.ts`.

### File Inventory (44 TypeScript files in `v2026.3.1`: 29 source + 15 tests)

| File | Description |
|------|-------------|
| `index.ts` | Main barrel export — re-exports types and utilities for plugin authors |
| `account-id.ts` | Re-exports `DEFAULT_ACCOUNT_ID`, `normalizeAccountId` |
| `allow-from.ts` | Format allowFrom entries for lowercase comparison |
| `config-paths.ts` | Resolve channel account config base path |
| `file-lock.ts` | File-based locking (advisory locks for plugin state files) |
| `onboarding.ts` | Interactive account ID prompting for plugin setup |
| `text-chunking.ts` | Chunk text for outbound messages (respects char limits) |
| `index.test.ts` | Tests for SDK exports |

### Key Types & Interfaces

| Type | File | Description |
|------|------|-------------|
| `FileLockOptions` | file-lock.ts | Lock options (timeout, retries, stale threshold) |
| `FileLockHandle` | file-lock.ts | Lock handle with release method |
| `PromptAccountIdParams` | onboarding.ts | Params for interactive account ID prompt |

### Key Functions

| Function | File | Description |
|----------|------|-------------|
| `acquireFileLock(path, opts?)` | file-lock.ts | Acquire advisory file lock |
| `withFileLock(path, fn, opts?)` | file-lock.ts | Execute function with file lock |
| `chunkTextForOutbound(text, limit)` | text-chunking.ts | Split text into chunks respecting limit |
| `formatAllowFromLowercase(params)` | allow-from.ts | Normalize allowFrom for comparison |
| `resolveChannelAccountConfigBasePath(params)` | config-paths.ts | Get config base path for channel account |
| `promptAccountId(params)` | onboarding.ts | Interactive account ID selection |

### Key Re-exports from index.ts
- `ChannelPlugin`, `PluginRuntime`, `RuntimeLogger` types
- `normalizePluginHttpPath`, `registerPluginHttpRoute`, `emptyPluginConfigSchema`
- `OpenClawConfig` type
- `CHANNEL_MESSAGE_ACTION_NAMES`
- Channel types: `ChannelId`, `ChannelDock`, etc.

### Internal Dependencies
- `../routing/session-key.js` — account ID normalization
- `../channels/plugins/*` — channel types and action names
- `../plugins/*` — runtime types, http-path, http-registry, config-schema
- `../config/config.js` — OpenClawConfig type

### External Dependencies
- `node:fs`, `node:path`, `node:crypto`

### Test Coverage (11 test files)
- `index.test.ts`, `allow-from.test.ts`, `command-auth.test.ts`, `fetch-auth.test.ts`, `group-access.test.ts`, `persistent-dedupe.test.ts`, `ssrf-policy.test.ts`, `status-helpers.test.ts`, `temp-path.test.ts`, `text-chunking.test.ts`, `webhook-targets.test.ts`

---

## 7. src/acp

### Module Overview
Agent Client Protocol (ACP) implementation — provides an MCP-compatible server that translates ACP requests into OpenClaw gateway sessions. Enables external ACP clients (IDEs, tools) to interact with OpenClaw agents via standardized protocol. Entry points: `serveAcpGateway()`, `createAcpClient()`.

#### v2026.2.19 Changes
- **Session rate limiting** — ACP sessions now rate-limited to prevent abuse
- **Idle reaping** — Idle ACP sessions automatically cleaned up
- **Prompt size bounds** — Prompt input capped at 2 MiB to prevent memory exhaustion
- See DEVELOPER-REFERENCE.md §9 (gotchas 33–45) for related hardening details

### File Inventory (43 TypeScript files in `v2026.3.1`: 30 source + 13 tests)

| File | Description |
|------|-------------|
| `index.ts` | Barrel exports for ACP module |
| `server.ts` | ACP gateway server (stdio-based, wraps gateway API) |
| `client.ts` | ACP client (connect to ACP server, interactive REPL, permission resolution) |
| `translator.ts` | `AcpGatewayAgent` — translates ACP protocol to OpenClaw gateway calls |
| `session.ts` | In-memory session store for ACP sessions |
| `session-mapper.ts` | Map ACP sessions to OpenClaw session keys |
| `event-mapper.ts` | Map OpenClaw events to ACP events (text extraction, attachments, tool formatting) |
| `commands.ts` | Available ACP commands |
| `meta.ts` | Metadata helpers (readString, readBool, readNumber) |
| `types.ts` | ACP types (session, server options, agent info) |
| `client.test.ts` | Client tests |
| `session.test.ts` | Session store tests |
| `session-mapper.test.ts` | Session mapper tests |
| `event-mapper.test.ts` | Event mapper tests |

### Key Types & Interfaces

| Type | File | Description |
|------|------|-------------|
| `AcpSession` | types.ts | ACP session (id, sessionKey, createdAt, messages) |
| `AcpServerOptions` | types.ts | Server options (port, gatewayUrl, auth) |
| `ACP_AGENT_INFO` | types.ts | Agent info constant for ACP protocol |
| `AcpSessionStore` | session.ts | Session store interface (get, set, delete, list) |
| `AcpSessionMeta` | session-mapper.ts | Parsed session metadata |
| `AcpClientOptions` | client.ts | Client options (serverUrl, auth) |
| `AcpClientHandle` | client.ts | Client handle (send, close) |
| `AcpGatewayAgent` | translator.ts | Agent class implementing ACP → gateway translation |
| `GatewayAttachment` | event-mapper.ts | Attachment extracted from prompt |

### Key Functions

| Function | File | Description |
|----------|------|-------------|
| `serveAcpGateway(opts)` | server.ts | Start ACP gateway server |
| `createAcpClient(opts)` | client.ts | Create ACP client connection |
| `runAcpClientInteractive(opts)` | client.ts | Run interactive ACP REPL |
| `resolvePermissionRequest(params)` | client.ts | Resolve tool permission approval |
| `createInMemorySessionStore()` | session.ts | Create in-memory session store |
| `parseSessionMeta(meta)` | session-mapper.ts | Parse session metadata from ACP |
| `resolveSessionKey(params)` | session-mapper.ts | Map ACP session to OpenClaw session key |
| `resetSessionIfNeeded(params)` | session-mapper.ts | Reset session on config change |
| `extractTextFromPrompt(prompt)` | event-mapper.ts | Extract text from content blocks |
| `extractAttachmentsFromPrompt(prompt)` | event-mapper.ts | Extract attachments from content blocks |
| `formatToolTitle(name, params)` | event-mapper.ts | Format tool call for display |
| `inferToolKind(name?)` | event-mapper.ts | Infer tool risk kind from name |
| `getAvailableCommands()` | commands.ts | List available ACP commands |

### Internal Dependencies
- Gateway API (via HTTP calls to gateway URL)
- Uses `@agentclientprotocol/sdk` for protocol implementation

### External Dependencies
- `@agentclientprotocol/sdk` — ACP protocol SDK
- `node:child_process`, `node:crypto`, `node:fs`, `node:path`, `node:readline`, `node:stream`, `node:url`

### Data Flow
1. `serveAcpGateway()` starts stdio-based ACP server
2. `AcpGatewayAgent` receives ACP requests and translates to gateway HTTP calls
3. Session mapper resolves ACP session IDs to OpenClaw session keys
4. Event mapper converts OpenClaw streaming events to ACP events
5. Tool calls flow through permission system — `DANGEROUS_ACP_TOOLS` require explicit approval
6. Client can run interactive REPL or programmatic API

### Security Model
- **Dangerous tool gating**: `DANGEROUS_ACP_TOOLS` (from security module) require explicit user approval
- **Tool kind inference**: `inferToolKind()` classifies tools for permission UI
- **Session isolation**: Each ACP session maps to isolated OpenClaw session
- **Auth tokens**: Client/server auth via tokens

### Configuration
- ACP server options: port, gatewayUrl, auth token
- Maps to gateway configuration for session routing

### Test Coverage (4 test files)
- `client.test.ts` — Client connection and messaging
- `session.test.ts` — Session store CRUD
- `session-mapper.test.ts` — Session key mapping
- `event-mapper.test.ts` — Event/text/attachment extraction

---

## Cross-Module Summary

### Security Enforcement Flow
```
External Input → external-content.ts (wrap with boundaries)
                     ↓
Channel Config → audit-channel.ts (enforce allowFrom/groupPolicy)
                     ↓
Tool Invocation → dangerous-tools.ts (deny lists)
                     ↓
Plugin Loading → skill-scanner.ts (code scanning)
                     ↓
File Access → audit-fs.ts (permission checks)
                     ↓
Browser Control → control-auth.ts + csrf.ts (auth + CSRF)
                     ↓
ACP Tools → DANGEROUS_ACP_TOOLS (approval required)
```

### Module Dependency Graph
```
plugin-sdk → plugins → (channels, config, agents, routing)
acp → gateway API (HTTP)
security → (agents, browser, channels, config, gateway, plugins, pairing)
browser → (config, gateway, infra, logging, media, security)
web → (auto-reply, agents, channels, config, globals, infra, logging, media, pairing, routing)
canvas-host → (minimal, mostly standalone)
```

### Total Counts
| Module | Source Files | Test Files | Exported Types | Exported Functions |
|--------|-------------|------------|----------------|-------------------|
| security | 19 | 10 | 16 | ~30 |
| web | ~35 | ~35 | ~14 | ~40 |
| browser | ~78 | ~47 | ~30 | ~80 |
| canvas-host | 3 | 2 | 5 | 6 |
| plugins | ~30 | ~16 | ~25 | ~40 |
| plugin-sdk | 29 | 15 | 3 | 6 |
| acp | 10 | 4 | 8 | 13 |

## v2026.3.7 Delta Notes

- ZIP archive path traversal hardening: archive extraction enforces same-directory containment, blocking `../` traversal in ZIP entries. Relevant to `src/security/` and plugin install paths.
- Auth token/key snippet removal from status output: API keys and auth tokens fully redacted from all status surfaces; no browser CDP session tokens are included in diagnostic output.
- Plugin hook policy validation: hook policies validated against plugin manifest at load time; malformed or over-privileged hooks are rejected before any CDP or browser session is established.
- Plugin SDK subpath scoping: plugin SDK imports restricted to declared subpath exports, reducing the attack surface for malicious plugins attempting to reach browser or CDP internals.
- No browser/CDP-specific security changes were introduced in v2026.3.7 beyond the above cross-cutting hardening. Existing CDP hardening (MV3 worker, relay state guards, remote CDP stale-target recovery) from prior releases remains in effect.

## v2026.3.8 Delta Notes

- Direct `ws://` / `wss://` CDP endpoints are now supported as first-class browser profiles, with loopback WS endpoints normalized back to HTTP(S) for `/json/*` tab operations.
- Remote `/json/version` debugger URLs like `ws://0.0.0.0` and `ws://[::]` are rewritten back to the externally reachable host/port, fixing Browserless-style container endpoints.
- `browser.relayBindHost` allows the extension relay to bind non-loopback addresses for WSL2 and other cross-namespace setups while keeping loopback-only defaults.
- Strict browser navigation now validates redirect hops and fails closed when remote tab-open flows cannot inspect the chain.
- Control UI bundled/package-proven static roots may use hardlinked assets in auto-detected installs, while configured/custom roots remain on the strict hardlink boundary.

## v2026.3.11 Delta Notes

### Gateway/WebSocket Origin Enforcement (GHSA-5wcw-8jjv-m286)

- **Browser origin check unconditional on proxy headers** — `message-handler.ts` now derives `enforceOriginCheckForAnyClient` from the presence of an HTTP `Origin` header alone. Previously, requests reaching the gateway through a `trusted-proxy` auth path could bypass the `checkBrowserOrigin()` call if `isControlUi`/`isWebchat` flags were not set. The fix makes the origin gate fire for *any* browser-originated connection (detected via `hasBrowserOriginHeader`), regardless of the auth mode. This closes a cross-site WebSocket hijacking path where an untrusted origin could reach `operator.admin` scopes through `trusted-proxy` mode. Verified in `server.auth.browser-hardening.test.ts` which asserts that a `trusted-proxy`-authenticated connect request with `origin: "https://evil.example"` returns `{ ok: false, reason: "origin not allowed" }` even when all proxy headers are valid.
- **`checkBrowserOrigin()` contract** (`origin-check.ts`) — The function enforces three checks in order: (1) explicit `allowedOrigins` allowlist (including `"*"` wildcard), (2) opt-in `dangerouslyAllowHostHeaderOriginFallback` Host-header match, (3) loopback-only `isLocalClient` dev fallback. A missing or `"null"` Origin always returns `{ ok: false }`. No changes to the function signature itself; the fix is at the call site in `message-handler.ts`.
- **Control UI dashboard token scoping (#40892)** — Dashboard auth tokens are now kept in session-scoped browser storage and are scoped to the selected gateway URL; the bootstrap flow uses fragment-only token delivery to prevent credential leakage via HTTP Referer or server logs.

### Sandbox/FS Bridge Write Hardening

- **Staged writes pinned to verified parent directory** — `fs-bridge-mutation-helper.ts` implements `write_atomic()` in the injected Python helper using `dir_fd`-anchored file operations throughout: the parent directory is opened with `O_RDONLY | O_DIRECTORY | O_NOFOLLOW`, temp files are created inside that fd with `O_WRONLY | O_CREAT | O_EXCL | O_NOFOLLOW`, and `os.replace()` is issued with matching `src_dir_fd`/`dst_dir_fd` to atomically rename within the same directory. This prevents a temporary write file from materializing outside the allowed sandbox mount before the atomic replace, even if a symlink race replaces a directory component between path resolution and file creation.

---

## v2026.3.12 Delta Notes

### Security — Exec Approval Hardening

- **GHSA-pcqg-f7rg-xfvv: Invisible Unicode in exec approval prompts** — Invisible Unicode format characters are escaped to `\u{...}` visible notation in exec approval prompts before display, preventing approval UI spoofing via hidden directional or formatting codepoints (#43687).
- **GHSA-9r3v-37xh-2cf6: Unicode normalization before obfuscation checks** — Unicode compatibility normalization and invisible formatting character stripping are applied before exec obfuscation detection, closing a bypass via Unicode confusable characters (#44091).
- **GHSA-f8r2-vg7x-gh8m: POSIX glob matching hardening in exec allowlist** — Exec allowlist glob matching now preserves POSIX case sensitivity and constrains `?` to a single path segment, preventing pattern matches that could cross directory boundaries (#43798).
- **GHSA-57jw-9722-6rf2, GHSA-jvqh-rfmh-jh27, GHSA-x7pp-23xv-mmr4, GHSA-jc5j-vg4r-j5jx: Exec approval hardening — inline loader, POSIX shell flags, pnpm/npm exec/npx runners** — Ambiguous inline loader/shell-payload detection hardened; POSIX shell flag bypass blocked; pnpm and npm exec/npx script runner forms unwrapped before approval matching (#44247).
- **Exec approval: fail closed for Ruby `-r`/`--require`/`-I` flags** — Ruby exec approval fails closed on load-path injection flags.
- **Exec approval: Mongolian selectors stripped in obfuscation detector** — Mongolian free variation selectors are stripped before obfuscation checks.
- **GHSA-jf5v-pqgw-gm5m: `GIT_EXEC_PATH` blocked in host exec environments** — `GIT_EXEC_PATH` inheritance is blocked in sanitized host exec environments to prevent git exec-path injection (#43685).

### Security — Authentication and Scoping

- **GHSA-r7vr-gr74-94p8: Sender ownership required for `/config` and `/debug`** — Gateway `/config` and `/debug` endpoints now require sender ownership; non-owner requests are rejected (#44305).
- **GHSA-rqpp-rjj8-7wv8: Unbound client-declared scopes cleared on shared-token WebSocket connects** — Prevents scope escalation via client-declared scope claims on shared-token connections (#44306).
- **GHSA-2pwv-x786-56f8: Device-token scopes capped to approved baseline** — Device token scopes are now capped to the paired device's approved scope baseline (#43686).
- **GHSA-wcxr-59v9-rxr8: Sandbox session-tree visibility enforced in `session_status`** — Cross-sandbox session reads and mutations are blocked by enforcing sandbox session-tree visibility checks (#43754).
- **GHSA-2rqg-gjgv-84jm: Public spawned-run lineage fields rejected** — Public requests with spawned-run lineage fields are rejected; workspace inheritance only on internal paths (#43801).

### Security — WebSocket and Handshake

- **GHSA-jv4g-m82p-2j93 + GHSA-xwx2-ppv2-wx98: Handshake retention shortened; oversized pre-auth frames rejected** — Unauthenticated handshake retention window is shortened and oversized pre-auth frames are rejected before processing (#44089).

### Security — Media Store

- **GHSA-6rph-mmhp-h7h9: Shared media-store size cap restored for browser proxy files** — The size cap for the shared media store is restored for browser proxy files, preventing unbounded accumulation (#43684).

### Security — Webhooks

- **GHSA-g353-mgv3-8pcj: Feishu webhook requires `encryptKey`** — Feishu webhook ingress now requires `encryptKey`; unencrypted webhooks are rejected (#44087).
- **GHSA-mhxh-9pjm-w7q5: LINE webhook requires signatures** — LINE webhook requests are rejected unless they carry a valid signature (#44090).
- **GHSA-5m9r-p9g7-679c: Zalo webhook rate-limits invalid secrets** — Zalo webhook ingress rate-limits requests with invalid secrets (#44173).

### Security — Plugins (v2026.3.12)

- **GHSA-99qw-6mr3-36qr: Workspace plugin auto-load disabled without explicit trust** — Plugins discovered in cloned workspaces can no longer execute without an explicit trust decision (`plugins.allow`). Non-bundled workspace-discovered plugins emit a warning when `plugins.allow` is not configured (#44174).
- **Plugin discovery/load cache and provenance tracking** — Discovery and load caches corrected for env-scoped roots; provenance tracking updated for workspace-relative roots (#44046).

### Security — Pairing (v2026.3.12)

- **Bootstrap tokens for `/pair` and `openclaw qr`** — Setup codes are now short-lived bootstrap tokens, reducing exposure window for intercepted pairing codes.

### Browser — Existing Session (v2026.3.12)

- **Stop reporting fake CDP ports/URLs for live attached Chrome** — When attaching to an existing Chrome session via Chrome DevTools MCP, no fake CDP port or URL is reported. The profile renders `transport: chrome-mcp` to accurately reflect the transport type.
- **Transport-aware timeout diagnostics** — Browser operation timeout messages now reflect the active transport type (`chrome-mcp` vs. standard CDP), producing accurate diagnostic guidance.

### Security Model Updates (v2026.3.12)

The following entries are added to the Security Model section for `src/browser`:

- **GHSA-vmhq-cqm9-6p7q: Persistent browser profile create/delete blocked via write-scoped `browser.request`** — Creating or deleting persistent browser profiles via write-scoped `browser.request` calls is now blocked (#43800).
- **`nodes` tool marked owner-only** — The `nodes` system tool requires owner authorization.

---

## v2026.3.13 Delta Notes

### Security — Exec Approval Hardening

- **Single-use bootstrap token setup codes** — Setup codes are now invalidated after first redemption; captured codes cannot be replayed.
- **Security/exec: fail closed for Perl `-M` and `-I` flags** — Exec approval for Perl fails closed on load-path and module injection flags.
- **Security/exec: PowerShell `-File`/`-f` recognized as script-runner wrapper** — PowerShell script invocations via `-File`/`-f` now receive consistent approval treatment as script-runner forms.
- **Security/exec: additional pnpm runner forms unwrapped** — `pnpm --reporter exec` and `pnpm node <file>` forms are unwrapped before approval matching.
- **Security/exec: `env` dispatch wrappers unwrapped in shell-segment allowlist on macOS** — `env` wrappers inside shell-segment allowlist resolution are unwrapped on macOS so approval decisions reflect the effective executable.
- **Security/exec: backslash-newline treated as shell line continuation on macOS** — Backslash-newline sequences are recognized as shell line continuations during macOS shell-chain parsing, closing a bypass via escaped newlines.
- **Security/exec: macOS skill auto-allow trust bound to executable name and resolved path** — macOS skill auto-allow trust is bound to both the executable name and its realpath, preventing trust from incorrectly applying to a different binary at the same resolved path.
- **macOS/exec approvals: per-agent exec approval settings respected in gateway prompter** — Gateway prompter respects per-agent exec approval settings for interactive approval requests (#13707).

### Security — External Content

- **Strip zero-width and soft-hyphen chars during boundary sanitization** — Zero-width characters and soft-hyphen (U+00AD) are stripped during external content boundary sanitization, closing a boundary-marker-splitting bypass.

### Security — Webhooks and Channels

- **iMessage: reject unsafe remote attachment paths before SCP** — iMessage remote attachment paths are validated before spawning SCP, rejecting paths that could escape the expected attachment directory.
- **Telegram webhook validates secret before reading/parsing request body** — The webhook secret is validated before the request body is read or parsed, preventing parser-level exposure of unauthenticated input.

### Browser — Chrome DevTools MCP Attach Mode (v2026.3.13)

- **Official Chrome DevTools MCP attach mode** — The `existing-session` driver is the formalized Chrome DevTools MCP attach mode for live signed-in Chrome sessions. The driver communicates via the `chrome-devtools-mcp` package (spawned via `npx -y chrome-devtools-mcp@latest --autoConnect --experimentalStructuredContent --experimental-page-id-routing`). Tab IDs are numeric page IDs from the MCP structured content, mapped to `BrowserTab.targetId` strings.
- **Built-in `profile="user"` for logged-in host browser** — The `user` profile is auto-injected into `browser.profiles` (driver `existing-session`, `attachOnly: true`, color `#00AA00`) when not explicitly configured. It provides attach-only access to the host's signed-in Chrome via Chrome DevTools MCP.
- **Built-in `profile="chrome-relay"` for extension relay** — The `chrome-relay` profile is auto-injected (driver `extension`, `cdpUrl: http://127.0.0.1:<controlPort+1>`, color `#00AA00`) when not explicitly configured, providing extension-relay access.
- **Profile capabilities: `usesChromeMcp` flag** — `BrowserProfileCapabilities` in `profile-capabilities.ts` gains `usesChromeMcp: boolean`. Profiles with driver `existing-session` have `usesChromeMcp: true`, `requiresRelay: false`, `usesPersistentPlaywright: false`, `supportsReset: false`.
- **Browser act: batched actions, selector targeting, delayed clicks** — Browser act automation supports batched action sequences, explicit selector-based element targeting, and delayed click timing (contributed by @vincentkoc).
- **Browser/existing-session: driver validation and session lifecycle hardened** — The Chrome DevTools MCP session lifecycle is validated more strictly; shared ARIA role sets are extracted to a shared module for consistent snapshot output (#45682).
- **Browser/existing-session: text-only `list_pages`/`new_page` responses accepted** — The Chrome DevTools MCP integration accepts text-only (non-structured) responses from `list_pages` and `new_page` MCP tool calls, improving compatibility with different versions of the `chrome-devtools-mcp` package.
