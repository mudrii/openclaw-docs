# OpenClaw Codebase Analysis — PART 5: Security, Plugins & Extensions
<!-- markdownlint-disable MD024 -->

> Updated: 2026-03-12 | Version: v2026.3.11

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

#### v2026.2.21 Changes <!-- v2026.2.21 -->
- **system.run command resolution** — `fix(security)`: centralized and hardened in `src/node-host/invoke-system-run.ts` (new ~420-line module). Validates command path, sanitizes env overrides via `sanitizeSystemRunEnvOverrides()`, and resolves command through `resolveSystemRunCommand()` before any allowlist evaluation
- **Startup-file env injection blocked** — `fix(security)`: environment variable injection via startup files (`.bashrc`, `.zshrc`, etc.) blocked across host execution paths via `sanitizeSystemRunEnvOverrides()` / `sanitizeHostExecEnv()`
- **Heredoc allowlist parsing hardened** — `fix(security)`: more heredoc patterns recognized and blocked in allowlist parsing
- **Command substitution in heredocs blocked** — `fix(security)`: `$(cmd)` and backtick substitution in unquoted heredoc bodies now blocked
- **macOS exec shell-chain checks** — `fix(macos)`: chained shell commands evaluated against exec allowlist more strictly on macOS
- **Runtime command override gating** — `fix(security)`: dynamic injection of command overrides at runtime is now prevented
- **grep safe-bin file-read bypass blocked** — `fix(security)`: `grep` as a safe-bin entry could be abused to read arbitrary files; this path is now blocked
- **Compaction counter reset blocked** — `fix(security)`: OC-65 agents can no longer reset their own compaction counters to circumvent context exhaustion limits
- **Prototype-chain traversal in webhook templates** — `security(hooks)`: prototype-chain traversal blocked in webhook template `getByPath` (#22213); prevents prototype pollution via crafted template paths
- **Per-wrapper IDs on untrusted content** — `Security`: each untrusted content wrapper now receives a unique per-wrapper ID for attribution (#19009)

#### v2026.2.22–v2026.2.23 Exec Approval Hardening

**Safe-bin PATH trust removed (v2026.2.22):**
- `tools.exec.safeBins` no longer trusts PATH-derived directories
- Add `tools.exec.safeBinTrustedDirs` for explicit trusted directories
- Pin safe-bin shell execution to resolved absolute executable paths
- Add `tools.exec.safeBinProfiles` for custom binaries requiring explicit profiles
- Unprofiled interpreter-style entries cannot be treated as stdin-safe

**Shell wrapper transparency (v2026.2.22):**
- `env`, `nice`, `nohup`, `stdbuf`, `timeout` treated as transparent wrappers during allowlist analysis
- Policy checks match effective executable/inline shell payload
- `allow-always` for shell-wrapper commands: inner executable path persisted (not wrapper binary)

**Shell line continuations (v2026.2.22):**
- `\\\n`/`\\\r\n` line continuations trigger fail-closed behavior
- Shell-wrapper execution requires approval in allowlist mode

**Exec env sanitization (v2026.2.22):**
- Block request-scoped `HOME`, `ZDOTDIR` overrides in host exec env sanitizers
- Block `SHELLOPTS`/`PS4` — prevent xtrace/PS4 prompt-substitution allowlist bypasses
- Shell-wrapper env restricted to explicit allowlist: `TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`
- Login-shell executable paths validated against `/etc/shells` + trusted prefixes
- Block `SHELL`/`HOME`/`ZDOTDIR` in config env ingestion before fallback execution

**Command obfuscation detection (v2026.2.23):**
- Detect obfuscated commands before exec allowlist decisions
- Require explicit approval for obfuscation patterns (base64-encoded, hex-encoded, etc.)

**macOS app allowlist (v2026.2.22):**
- Enforce path-only matching (drop basename matches like `echo`)
- Migrate legacy basename entries to resolved paths
- Fail closed on unsafe shell-chain parse/control syntax (quoted command substitution, backticks)

**Other exec fixes (v2026.2.22):**
- `sort --compress-program` blocked in allowlist mode
- `tools.exec.host=sandbox` fails closed when sandbox runtime unavailable
- Node/macOS exec host: strict Windows allowlist for `cmd.exe /c` shell-wrapper runs

#### v2026.2.23 Skills/Skill-Creator Hardening

- **Skill-creator packaging:** Skip symlink entries during packaging; reject files whose resolved paths escape selected skill root
- **openai-image-gen XSS fix:** Escape user-controlled prompt, filename, and output-path values in HTML gallery generation — prevents stored XSS in generated `index.html`
- **Python skill hardening:** Skip self-including `.skill` outputs; handle CRLF frontmatter; strict `--days` validation; safer image file loading. Add `ruff` linting and pytest discovery coverage for `skills/` directory

#### v2026.2.22 Plugin Security

- **Plugin install manifest sanitization:** Strip `workspace:*` devDependency entries from copied plugin manifests before `npm install --omit=dev`. Ignore backup/disabled directory patterns (`.backup-*`, `.bak`, `.disabled*`). Move updater backup dirs under `.openclaw-install-backups`
- **Plugin allowlist:** `openclaw plugins enable` updates allowlists via shared plugin-enable policy. Auto-enable writes `channels.<id>.enabled=true` (not `plugins.entries.<id>`). Built-in channels also allowlisted when `plugins.allow` is active

#### v2026.3.1 Plugin & Install Changes

- **NPM spec install .tgz detection** — `packNpmSpecToArchive()` in `install-source-utils.ts` now detects `.tgz` archives via `parsePackedArchiveFromStdout()` when `npm pack` JSON output is empty, and falls back to `findPackedArchiveInDir()` scanning the working directory for the most recently modified `.tgz` file. E404 errors now return a user-friendly "Package not found on npm" message with a doc link.
- **Windows plugin install EINVAL** — `exec.ts` resolves `node` and `npm` to their CLI script paths on Windows to avoid `EINVAL` spawn errors from shim executables. Contributor: @codertony.
- **ACP/ACPX streaming** — ACPX extension pinned to 0.1.15, with configurable command/version probing. Health-check tolerates missing `--version` output via `ensure.ts` fallback. ACPX Windows cmd wrapper spawning hardened with strict wrapper policy.
- **Plugin discovery order** — Global auto-discovered extensions (`~/.openclaw/extensions/`) now discovered after bundled plugins, giving bundled versions precedence.

#### v2026.2.22 Security Audit New Findings

- `gateway.nodes.allow_commands_dangerous` — audit finding for risky `gateway.nodes.allowCommands` overrides
- `security.exposure.open_groups_with_runtime_or_fs` — finding for open group policies exposing runtime/filesystem tools without sandbox guards
- `gateway.real_ip_fallback_enabled` — conditional severity: warn for loopback proxies, critical for non-loopback proxies

#### v2026.2.23 OTEL/Telemetry Redaction

- Redact sensitive values (API keys, tokens, credential fields) from diagnostics-otel log bodies, log attributes, and error/reason span fields before OTLP export

#### v2026.2.23 CI/Pre-commit Security Hooks

- Pre-commit security hooks for private-key detection and production dependency auditing
- Enforced in CI alongside baseline secret scanning
- `ruff` linting added for Python scripts in `skills/`

#### v2026.2.24 Changes

- **Exec approvals / argv binding** (security, @tdjackey): `system.run` command display/approval text is now bound to the full argv when shell-wrapper inline payloads carry positional argv values; payload-only `rawCommand` mismatches for wrapper-carrier forms are now rejected, preventing hidden command execution under misleading approval text. Released in `v2026.2.24`.
- **Exec approvals / wildcard allowlist** (#25250): bare `*` in exec allowlist is now treated as a true wildcard for parsed executables including unresolved PATH lookups, so global opt-in allowlists work as configured. Contributor: @widingmarcus-cyber.
- **Doctor / Plugins auto-enable** (#25275): auto-enable now resolves third-party channel plugins by manifest plugin id (not channel id), preventing invalid `plugins.entries.<channelId>` writes when the ids differ. Contributor: @zerone0x.
- **Security / Audit — multi-user heuristic**: `security.trust_model.multi_user_heuristic` config key added to flag likely shared-user ingress patterns and clarify the personal-assistant trust model; hardening guidance provided for intentional multi-user setups (`sandbox.mode="all"`, workspace-scoped FS, reduced tool surface).

#### v2026.2.25 Changes

- **Security / Signal reaction authorization** (@tdjackey): `dmPolicy`/`groupPolicy` authorization is now enforced before Signal reaction-only notification enqueue. Previously, unauthorized senders could inject reaction system events; reactions now require the same channel access checks as normal messages.
- **Security / Discord reaction authorization** (@tdjackey): DM policy/allowlist authorization is enforced before Discord reaction-event system enqueue in direct messages and guilds. `groupPolicy` channel gating also applies to reaction ingress, aligning it with normal message preflight.
- **Security / Slack reactions + pins authorization** (@tdjackey): `reaction_*` and `pin_*` system-event enqueue is gated through shared sender authorization: DM `dmPolicy`/`allowFrom` and channel `users` allowlists are enforced for non-message ingress.
- **Security / Telegram reaction authorization** (@tdjackey): `dmPolicy`/`allowFrom` and group allowlist authorization is enforced on `message_reaction` events before enqueueing reaction system events, preventing unauthorized reaction-triggered input in DMs and groups.
- **Security / Slack interactions** (@tdjackey): channel/DM authorization and modal actor binding (`private_metadata.userId`) are enforced before enqueueing `block_action`/`view_submission`/`view_closed` system events.
- **Security / MS Teams file consent** (@tdjackey): `fileConsent/invoke` upload acceptance/decline is bound to the originating conversation before consuming pending uploads, preventing cross-conversation pending-file upload or cancellation via leaked `uploadId` values.
- **Security / Gateway pairing for operator sessions** (@tdjackey): pairing is now required for operator device-identity sessions authenticated with shared token auth; unpaired devices can no longer self-assign operator scopes.
- **Security / Exec approvals — argv binding and spawn hardening** (@tdjackey): approval matching is bound to exact argv identity and whitespace; symlink `cwd` paths and non-canonical executable argv are rejected at spawn time, blocking mutable-cwd symlink retarget chains between approval and execution.
- **Security / Telegram + MS Teams group fail-closed** (#25988, #26111, @bmendonca3): DM pairing-store fallback removed from group allowlist evaluation for both Telegram and MS Teams; group sender access now requires explicit `groupAllowFrom` or per-group `allowFrom`.
- **Security / Nextcloud Talk replay dedupe + unsigned webhook rejection** (@aristorechina, @bmendonca3): replayed signed webhook events are dropped with persistent per-account dedupe; unsigned traffic rejected before full body reads; unexpected webhook backend origins rejected when account base URL is configured.

#### v2026.2.26 Changes

- **External secrets workflow** — introduces `openclaw secrets` (`audit`, `configure`, `apply`, `reload`) with stricter `secrets apply` target-path validation, safer migration scrubbing, and ref-only auth-profile support (security hardening for secret handling).
- **DM allowlist inheritance enforcement** — `dmPolicy: "allowlist"` now applies effective account-plus-parent config across account-capable channels, with `openclaw doctor` validation aligned so DM traffic does not silently drop after upgrades.
- **Allowlist safety checks** — `dmPolicy: "allowlist"` with empty `allowFrom` is rejected; `openclaw doctor --fix` can restore missing `allowFrom` entries from pairing-store data.
- **Gemini CLI OAuth warning gate** — explicit account-risk warning and confirmation before starting Gemini CLI OAuth flow; docs updated accordingly.
- **Temp-dir permission hardening** — Linux temp dirs forced to `0700` with self-healing before trust checks to avoid insecure writable temp paths.

#### v2026.3.1 Changes

**Prompt spoofing hardening:**
- Queued runtime events (subagent/cron completion announces) are no longer injected as user-role prompt text. A new `src/agents/internal-events.ts` module provides structured `AgentInternalEvent` types routed through trusted system-prompt context via `formatAgentInternalEventsForPrompt()`. Subagent announce delivery in `subagent-announce.ts` now builds typed `AgentTaskCompletionInternalEvent` payloads and forwards them through `internalEvents` on the announce queue, rather than concatenating freeform `[System Message]` blocks into user-facing messages.
- System prompt in `system-prompt.ts` removes `[System Message]` spoof markers from messaging guidelines; completion events are now described generically as "runtime-generated completion events."
- Unicode bracket folding in `external-content.ts` expanded: 14 additional Unicode angle bracket/ornament codepoints (U+00AB, U+00BB, U+300A, U+300B, U+27EA-U+27EF, U+276C-U+276F) are now folded to ASCII `<`/`>` to neutralize visually-similar spoof markers.

**Exec approval payloads (BREAKING):**
- `SystemRunApprovalBindingV1` and `SystemRunApprovalPlanV2` renamed to `SystemRunApprovalBinding` and `SystemRunApprovalPlan` respectively; the `version` field is removed from both types. Node exec approval payloads now require `systemRunPlan` (was `systemRunPlanV2`) and `systemRunBinding` (was `systemRunBindingV1`) in `ExecApprovalRequestPayload`. Gateway-side matching and normalization functions renamed accordingly (`normalizeSystemRunApprovalPlan`, `buildSystemRunApprovalBinding`, `matchSystemRunApprovalBinding`).
- `buildSystemRunApprovalPlanV2()` renamed to `buildSystemRunApprovalPlan()` in `invoke-system-run-plan.ts`. The `shellCommand` parameter removed from `hardenApprovedExecutionPaths()`: all argv entries are now resolved through canonical realpath via the new `resolveCommandResolutionFromArgv()` in `src/infra/exec-command-resolution.ts`.

**Shell env markers:**
- `OPENCLAW_SHELL` environment variable injected into child shell environments during exec, ACP client, ACPX, and TUI local shell runs. Values include `exec`, `acp-client`, `acpx`, `tui-local` to identify the runtime context. Contributor: @vincentkoc.

**Feishu webhook ingress hardening:**
- Feishu webhook rate-limit state bounded with `FEISHU_WEBHOOK_RATE_LIMIT_MAX_TRACKED_KEYS` (4,096 keys) hard cap. Stale-window pruning runs periodically via `maybePruneWebhookRateLimitState()`, evicting entries older than the 60s rate-limit window. When the key cap is exceeded, oldest entries are evicted via FIFO trimming (`trimWebhookRateLimitState()`). Contributor: @bmendonca3.

**Compaction audit injection removed:**
- Post-compaction audit injection message (Layer 3) removed from `agent-runner.ts`. This audit injected `enqueueSystemEvent` user-role messages referencing `WORKFLOW_AUTO.md` after context compaction, creating a persistent prompt injection vector. Layer 1 (compaction summary) and Layer 2 (workspace context refresh from AGENTS.md) remain intact.

**Compaction identifier preservation:**
- New `buildCompactionSummarizationInstructions()` in `compaction.ts` adds configurable identifier preservation policy (`AgentCompactionIdentifierPolicy`: `strict`, `custom`, `off`) to compaction summarization. Default `strict` policy instructs the summarizer to preserve all opaque identifiers (UUIDs, hashes, IDs, tokens, URLs, file names) exactly as written.

**Workspace boundary errors:**
- `SafeOpenErrorCode` gains `outside-workspace` code. `openFileWithinRoot()` in `fs-safe.ts` now throws `SafeOpenError("outside-workspace", "file is outside workspace root")` instead of the misleading generic `"path escapes root"` when resolved paths fall outside the workspace root. Home-prefix expansion (`~/`) added before path resolution.

**Sandbox mkdirp boundary checks:**
- `fs-safe.ts` adds pre-open `lstat` directory rejection to prevent `EISDIR` errors from leaking to messaging channels. `SafeOpenSyncAllowedType` parameter added to `openVerifiedFileSync` for directory-aware boundary checks. Contributor: @glitch418x.

**Docker sandbox browser hardening:**
- Sandbox browser `--no-sandbox` flag now correctly applied in Docker container environments. Contributor: @Lukavyi.

**Signal sync message null-handling:**
- `sentTranscript` sync messages filtered from bypass of loop protection. Sync message presence filtering hardened. Contributors: @Sid-Qin.

**Security audit new findings:**
- `gateway.control_ui.allowed_origins_wildcard` — audit finding when `gateway.controlUi.allowedOrigins` includes `"*"`, which disables origin allowlisting for Control UI/WebChat requests. Severity is critical for non-loopback bind, warn for loopback.
- `channels.feishu.doc_owner_open_id` — audit finding when Feishu doc tool is enabled, warning that `feishu_doc` action `create` can grant document access to the trusted requesting Feishu user.

**Model failover expansion:**
- `resolveFailoverReasonFromError()` in `failover-error.ts` now triggers model failover on `ECONNREFUSED`, `ENETUNREACH`, `EHOSTUNREACH`, `ENETRESET`, and `EAI_AGAIN` error codes (previously only `ETIMEDOUT`, `ESOCKETTIMEDOUT`, `ECONNRESET`, `ECONNABORTED`).

**Model directive @ parsing fix:**
- `splitTrailingAuthProfile()` in `model-ref-profile.ts` now splits at the first `@` after the last `/` (was: last `@` in the string). This fixes auth-profile extraction for model refs containing `@` characters in the organization/path portion (e.g., `org@team/model-name@profile`). Contributor: @haosenwang1018.

**Secrets/Auth profile normalization:**
- Inline `SecretRef` `token`/`key` fields normalized to canonical `tokenRef`/`keyRef` in runtime config snapshots.

**Copilot token refresh:**
- Copilot token refresh now runs before expiry and retries on auth errors.

**Plugin discovery order:**
- Bundled extensions now discovered before global auto-discovered extensions. Global extensions from `~/.openclaw/extensions/` are scanned after bundled plugins, so bundled versions take precedence unless users explicitly override via `plugins.load.paths`.

**Plugin loader refactor:**
- `activatePluginRegistry()` extracted as shared helper in `loader.ts`, ensuring both fresh-load and cached-registry paths call `setActivePluginRegistry()` and `initializeGlobalHookRunner()` consistently.

**Plugin tool context:**
- `OpenClawPluginToolContext` in `plugins/types.ts` gains `requesterSenderId` (trusted sender ID from inbound context) and `senderIsOwner` (whether the trusted sender is an owner).

---

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
| `exec-command-resolution.ts` | <!-- v2026.3.1 --> Canonical command resolution from argv — `resolveCommandResolutionFromArgv()` resolves raw executable to realpath for approval binding |
| **New (src/agents/)** | |
| `internal-events.ts` | <!-- v2026.3.1 --> Structured internal event types (`AgentInternalEvent`, `AgentTaskCompletionInternalEvent`) and `formatAgentInternalEventsForPrompt()` — routes runtime events through trusted system-prompt context instead of user-role text |

### Exported API
- `runSecurityAudit()` → `SecurityAuditReport` (findings with severity: critical/warn/info)
- `fixSecurityFootguns()` → `SecurityFixResult` (auto-remediation actions)
- `wrapExternalContent()`, `detectSuspiciousPatterns()` — injection defense
- `safeEqualSecret()` — timing-safe comparison
- `isPathInside()` — path traversal prevention
- `buildUntrustedChannelMetadata()` — safe metadata formatting
- `DEFAULT_GATEWAY_HTTP_TOOL_DENY`, `DANGEROUS_ACP_TOOLS` — risk constants
- <!-- v2026.2.21 --> Per-wrapper untrusted content IDs — `wrapExternalContent()` now attaches a unique per-wrapper ID to each untrusted content boundary for attribution (#19009)
- <!-- v2026.3.1 --> `formatAgentInternalEventsForPrompt()` — structured internal event rendering for system-prompt context
- <!-- v2026.3.1 --> `buildCompactionSummarizationInstructions()` — configurable identifier preservation policy for compaction summarization

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
- `installPluginFromArchive/Dir/File/NpmSpec/Path()`, `uninstallPlugin()`, `updateNpmInstalledPlugins()`
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

#### v2026.3.1 Changes
- **OPENCLAW_SHELL env marker** — ACP client tags spawned bridge environments with `OPENCLAW_SHELL=acp-client` to identify runtime context in child shell processes.
- **ACP stream char limits** — Stream character limits renamed to `output`/`sessionUpdate` for clarity.

### Key Files

| File | Role |
|------|------|
| `index.ts` | Barrel: exports `serveAcpGateway`, `createInMemorySessionStore` |
| `server.ts` | ACP server — connects to gateway WebSocket as a client, bridges ACP ↔ gateway |
| `client.ts` | ACP client — permission resolver that auto-approves safe tools, prompts for dangerous ones; <!-- v2026.3.1 --> now injects `OPENCLAW_SHELL=acp-client` into spawned bridge env |
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

#### v2026.3.1 Changes
- **Exec approval payloads (BREAKING)** — `buildSystemRunApprovalPlanV2()` renamed to `buildSystemRunApprovalPlan()`; `invoke.ts` now calls `buildSystemRunApprovalPlan()`. The `shellCommand` parameter removed from `hardenApprovedExecutionPaths()` — all argv entries resolved through canonical realpath via `resolveCommandResolutionFromArgv()`. The `version` field removed from approval plan and binding types.
- **system.run canonical realpath** — `invoke-system-run-plan.ts` replaces inline `isPathLikeExecutableToken()` with shared `resolveCommandResolutionFromArgv()` from `src/infra/exec-command-resolution.ts`, pinning executable paths to their resolved realpath.

### Key Files

| File | Role |
|------|------|
| `config.ts` | Node host config (nodeId, token, gateway connection) — read/write `node.json` |
| `runner.ts` | Main node host runner — connects to gateway as node client, handles invoke requests |
| `invoke.ts` | Command execution handler — routes `system.run` to `invoke-system-run.ts`, validates against exec allowlists, spawns processes, caps output; <!-- v2026.3.1 --> calls `buildSystemRunApprovalPlan()` (was `buildSystemRunApprovalPlanV2()`) |
| `invoke-system-run.ts` | <!-- v2026.2.21 --> Centralized `system.run` hardening (~478 lines at v2026.3.1) — `handleSystemRunInvoke()`: resolves command via `resolveSystemRunCommand()`, sanitizes env overrides, evaluates shell/argv allowlists, gates macOS exec host, handles approval decisions |
| `invoke-system-run-plan.ts` | <!-- v2026.3.1 --> Approval plan building and execution path hardening — `buildSystemRunApprovalPlan()`, `hardenApprovedExecutionPaths()` now uses `resolveCommandResolutionFromArgv()` for canonical realpath resolution |
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
| `exec.ts` | Core exec functions — `runExec`, `runCommandWithTimeout`, Windows command resolution; <!-- v2026.3.1 --> resolves `node`/`npm` to CLI script paths on Windows to avoid EINVAL spawn errors |
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

## 8. `apps/macos/` — macOS-Specific

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
Used by: CLI `openclaw setup` / `openclaw onboard`

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
| `synology-chat/` | Synology Chat channel |
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

## v2026.3.7 Delta Notes

- Config validation fail-closed on errors: config validation errors now cause the load to fail closed rather than proceeding with a partially valid config.
- ZIP archive path traversal hardening: archive extraction now enforces same-directory containment, blocking `../` path traversal in ZIP entries (PR context: `src/infra/install-safe-path.ts`).
- SecretRef models.json persistence hardening (PR #38955): `models.json` writes for SecretRef entries are now atomic, preventing corruption on interrupted writes.
- Password-file input hardening (PR #39067): password-file reading validates file permissions and rejects world-readable files before consuming credentials.
- Control UI device auth token signing alignment: device auth token signing now uses a consistent algorithm across Control UI and gateway paths.
- Auth token/key snippet removal from status output: API keys and auth tokens are fully redacted from `openclaw status` and gateway status snapshots.
- Plugin hook policy validation: hook policies are validated against the plugin manifest at load time, rejecting hooks that claim permissions not declared in the manifest.
- Nodes system.run approval enforcement: `system.run` commands invoked through the nodes subsystem now go through the same approval enforcement as host exec, closing a bypass path.
- Plugin SDK subpath scoping: plugin SDK imports are restricted to declared subpath exports, preventing plugins from importing internal engine modules directly.
```

## v2026.3.8 Delta Notes

- ACP provenance now carries `originSessionId`, optional receipt text, and reserved gateway fields for ACP-only provenance transport.
- Plugin onboarding clears discovery cache after installs, and bundled channel plugins now outrank duplicate npm-installed copies during onboarding/update sync unless the operator intentionally overrides via explicit config paths.
- Release-check now validates bundled-extension manifest/root-dependency mirrors so bundled runtime dependencies do not silently drift from the root package.
- `system.run` approvals for `bun` / `deno run` now bind script operands to on-disk file snapshots, blocking post-approval rewrites.
- Skill download installs pin the validated per-skill tools root before writing archives, preventing lexical-path rebinding from redirecting writes.
- Feishu's bundled runtime dependency mirror is now enforced at the root-package level so bundled installs do not miss required SDK runtime bits.

## v2026.3.11 Delta Notes

### Plugin Hook Context Parity (#42362)

- **`trigger` and `channelId` in embedded hook contexts** — `llm_input`, `agent_end`, and `llm_output` hooks now receive `trigger` and `channelId` fields through the embedded agent-run context (`PluginHookAgentRunContext` in `types.ts`). Previously these fields were present in `session_start`/`message_received` hook payloads but absent from the LLM-phase hooks, meaning plugins could not correlate hook phases to the originating channel or run trigger. `trigger` values include `"user"`, `"heartbeat"`, `"cron"`, and `"memory"`; `channelId` is the channel identifier string (e.g., `"telegram"`, `"discord"`).

### Plugin Global Hook Runner State Hardening (#40184)

- **Singleton state leak prevention** — `hook-runner-global.ts` singleton handling is hardened so that reuse of the shared global hook runner across registry reloads (e.g., after plugin install/uninstall) does not carry over stale runner state or corrupt hook registrations. The fix ensures `initializeGlobalHookRunner()` always initializes from the current registry rather than reusing a partially-torn-down runner instance.

### Plugin Discovery Environment Isolation

- **Plugin discovery env isolated from global state** — The plugin discovery process (`discovery.ts`) now runs with an isolated environment snapshot. Previously, mutations to process-level global state during gateway startup could bleed into the discovery env, causing non-deterministic candidate filtering. The isolation change ensures the discovery result depends only on the filesystem state and config, not on runtime global mutations.

### Archive Extraction Hardening (Plugin Installs)

- **TAR and `tar.bz2` symlink escape prevention** — Plugin installs from TAR and external `tar.bz2` archives now extract into a staging directory first. The staging pass detects destination symlinks and pre-existing child-symlink entries that could redirect extracted paths outside the plugin install root before any files are written to the final location.

### SecretRef Exec Traversal Rejection (#42370)

- **Traversal IDs rejected across schema, runtime, and gateway** — `exec` SecretRef IDs that contain traversal segments are now rejected at all three validation layers (schema validation, runtime SecretRef resolution, gateway auth). Previously, a crafted traversal ID could escape the expected SecretRef namespace.

### Security/Plugin Runtime: Unauthenticated HTTP Routes

- **Plugin HTTP routes do not inherit admin gateway scopes** — Unauthenticated plugin HTTP routes no longer inherit synthetic `operator.admin` gateway scopes when invoking `runtime.subagent.*`. A plugin's HTTP route that lacks a gateway auth token now executes subagent calls with the scope level corresponding to the actual request auth, not an elevated synthetic scope that was being injected for gateway-initiated plugin calls.

### ACP/Context Engine: Plugin SDK Model Auth (#41090)

- **`runtime.modelAuth` for plugins** — `runtime.modelAuth` and matching plugin-sdk auth helpers are now exposed so plugins can resolve provider/model API keys through the normal OpenClaw auth pipeline rather than accessing credential files directly. This allows plugins to call provider APIs with properly scoped auth without requiring raw secret file access.
