# OpenClaw Codebase Analysis — PART 5: Security, Plugins & Extensions
<!-- markdownlint-disable MD024 -->

> Updated: 2026-04-09 | Version: v2026.4.9 | Codebase: OpenClaw release tag `v2026.4.9`

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
| `audit.runtime.ts` | Runtime-phase audit checks |
| `audit.deep.runtime.ts` | Deep runtime audit checks |
| `audit.nondeep.runtime.ts` | Non-deep runtime audit checks |
| `audit-channel.ts` | Channel-specific security findings (open DMs, group policies, allowFrom validation) |
| `audit-channel.allow-from.runtime.ts` | Channel allowFrom runtime audit |
| `audit-channel.collect.runtime.ts` | Channel collection runtime audit |
| `audit-channel.discord.runtime.ts` | Discord-specific channel audit |
| `audit-channel.telegram.runtime.ts` | Telegram-specific channel audit |
| `audit-channel.zalouser.runtime.ts` | ZaloUser-specific channel audit |
| `audit-extra.ts` | Re-export barrel splitting sync/async collectors |
| `audit-extra.sync.ts` | Config-only checks: secrets in config, sandbox docker noop, small model risk, gateway HTTP key overrides, hooks hardening, exposure matrix |
| `audit-extra.async.ts` | I/O checks: file permissions, installed skill code safety, plugin trust/code scanning |
| `audit-fs.ts` | Filesystem permission inspection (`inspectPathPermissions`, `safeStat`) — POSIX mode bits + Windows ACL |
| `audit-tool-policy.ts` | One-line re-export barrel for sandbox tool policy (implementation is elsewhere) |
| `channel-metadata.ts` | Wraps untrusted channel metadata (usernames, bios) in safety boundaries |
| `dangerous-tools.ts` | Centralized constants: `DEFAULT_GATEWAY_HTTP_TOOL_DENY`, `DANGEROUS_ACP_TOOLS` — tools requiring approval |
| `external-content.ts` | Prompt injection defense: `detectSuspiciousPatterns`, `wrapExternalContent` with boundary markers |
| `fix.ts` | Auto-remediation: `chmod` state dirs, config files, OAuth dirs to safe permissions |
| `scan-paths.ts` | Path traversal guard: `isPathInside`, extension scanner path checks |
| `secret-equal.ts` | Timing-safe string comparison via `crypto.timingSafeEqual` |
| `skill-scanner.ts` | Static analysis scanner for skill/plugin code — detects dangerous patterns (eval, exec, fetch) |
| `config-regex.ts` | Configuration regex patterns for security validation |
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
| `onboarding.ts` | Re-export shim — `promptAccountId()` implementation is in `src/channels/plugins/setup-wizard-helpers.ts` |
| `text-chunking.ts` | `chunkTextForOutbound()` — split long messages at word/line boundaries |
| `status-helpers.ts` | `createDefaultChannelRuntimeState()`, `buildBaseChannelStatusSummary()` — standardized channel status (new) |
| `webhook-path.ts` | `normalizeWebhookPath()`, `resolveWebhookPath()` — webhook URL/path handling (new) |
| `agent-media-payload.ts` | `AgentMediaPayload` type, `buildAgentMediaPayload()` — structured media payloads (new) |
| `provider-auth-result.ts` | `buildOauthProviderAuthResult()` — OAuth provider auth result builder (new) |

### Dependencies
Imports from: `channels/plugins/`, `config/`, `routing/`, `wizard/`

### Dependents
Used by: released `extensions/` plugins import from `openclaw/plugin-sdk/<subpath>`; root-level compatibility aliases still exist, but new imports should target focused subpaths.

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
| `server.ts` | ACP server — connects to gateway WebSocket as a client, bridges ACP ↔ gateway; exports `serveAcpGateway` |
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

> **Note (v2026.3.24):** `src/acp/` has expanded to 55+ files including a `control-plane/` subdirectory. The functions previously attributed to `index.ts` are distributed across `server.ts` and `session.ts`.

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

## 15. `extensions/whatsapp/` — WhatsApp Layer

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

> **Note:** The table below is representative, not exhaustive. OpenClaw ships ~40 bundled provider plugins. Only notable or recently-added entries are listed here; see `extensions/` in the source tree for the full set.

| Extension | Description |
|-----------|-------------|
| `amazon-bedrock/` | Amazon Bedrock provider |
| `amazon-bedrock-mantle/` | Amazon Bedrock Mantle — OpenAI-compatible Bedrock surface (added v2026.4.7) |
| `anthropic/` | Anthropic (Claude) provider |
| `anthropic-vertex/` | Anthropic on Vertex AI provider |
| `arcee/` | Arcee AI provider — direct API + OpenRouter routing (added v2026.4.7, #62068) |
| `copilot-proxy/` | Copilot Proxy provider |
| `deepseek/` | DeepSeek provider |
| `google/` | Google Gemini provider |
| `google-gemini-cli-auth/` | Gemini CLI OAuth provider |
| `groq/` | Groq provider |
| `microsoft/` | Microsoft Azure OpenAI provider |
| `microsoft-foundry/` | Microsoft Foundry provider with Entra ID and API key auth (added v2026.4.7) |
| `minimax-portal-auth/` | MiniMax Portal OAuth provider |
| `ollama/` | Ollama local-model provider (with vision auto-detection and thinking support) |
| `openai/` | OpenAI provider |
| `openrouter/` | OpenRouter provider |
| `qwen/` | Alibaba Qwen provider |
| `xai/` | xAI (Grok) provider |
| `zai/` | Z.AI provider |

### Tool/Feature Plugins

| Extension | Description |
|-----------|-------------|
| `device-pair/` | Device pairing flow (bundled, enabled by default) |
| `diagnostics-otel/` | OpenTelemetry diagnostics exporter |
| `diffs/` | Diff viewer and diffs skill pack |
| `llm-task/` | JSON-only LLM task tool |
| `lobster/` | Lobster workflow tool (typed pipelines + resumable approvals) |
| `memory-core/` | Core memory search, QMD, and sync orchestration |
| `memory-lancedb/` | LanceDB-backed long-term memory with auto-recall |
| `memory-wiki/` | Persistent wiki compiler — durable knowledge vault with claim/evidence metadata and Obsidian integration (added v2026.4.7) |
| `open-prose/` | OpenProse VM skill pack |
| `openshell/` | OpenShell sandbox backend — alternative to the default node-host exec sandbox (added v2026.4.7) |
| `phone-control/` | Phone control (bundled, enabled by default) |
| `searxng/` | SearXNG self-hosted web search provider (added v2026.4.1) |
| `talk-voice/` | Talk voice (bundled, enabled by default) |
| `tavily/` | Tavily search + extract plugin — registers `web_search` provider and `tavily_search`/`tavily_extract` agent tools (added v2026.4.7) |
| `thread-ownership/` | Thread ownership management |
| `webhooks/` | Inbound/outbound webhook support |

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

---

## v2026.3.12 Delta Notes

### Security — Exec Approval Hardening

- **GHSA-pcqg-f7rg-xfvv: Invisible Unicode in exec prompts** — Invisible Unicode format characters (zero-width joiners, directional marks, etc.) are now escaped to `\u{...}` visible notation in exec approval prompts, preventing spoofed command display (#43687).
- **GHSA-9r3v-37xh-2cf6: Unicode normalization before obfuscation checks** — Unicode compatibility normalization and invisible formatting character stripping are applied before obfuscation detection checks, closing an obfuscation bypass via Unicode confusables (#44091).
- **GHSA-f8r2-vg7x-gh8m: POSIX glob matching hardening** — Exec allowlist glob matching now preserves POSIX case sensitivity and constrains `?` to remain within a single path segment, preventing glob patterns from matching across directory boundaries (#43798).
- **GHSA-57jw-9722-6rf2, GHSA-jvqh-rfmh-jh27, GHSA-x7pp-23xv-mmr4, GHSA-jc5j-vg4r-j5jx: Exec approval — ambiguous inline loader/shell-payload, POSIX shell flags, pnpm/npm exec/npx script runners** — Four additional exec approval hardening fixes: ambiguous inline loader/shell-payload approval detection hardened; POSIX shell flags blocked; pnpm and npm exec/npx script runner forms are now properly unwrapped before approval matching (#44247).
- **Security/exec: fail closed for Ruby `-r`/`--require`/`-I` flags** — Exec approval flow for Ruby now fails closed when `-r`, `--require`, or `-I` flags are encountered, preventing load-path injection through approved Ruby invocations.
- **Security/Mongolian selectors: strip in exec obfuscation detector** — Mongolian free variation selectors (Unicode Mongolian selector codepoints) are stripped before exec obfuscation checks to prevent selector-based command hiding.
- **GIT_EXEC_PATH blocked in sanitized host exec environments (GHSA-jf5v-pqgw-gm5m)** — `GIT_EXEC_PATH` is now blocked from being inherited in sanitized host exec environments, preventing git exec-path injection into approved commands (#43685).

### Security — Authentication and Scoping

- **GHSA-r7vr-gr74-94p8: `/config` and `/debug` require sender ownership** — The `/config` and `/debug` gateway endpoints now require that the requesting sender is the owner of the resource; non-owner requests are rejected (#44305).
- **GHSA-rqpp-rjj8-7wv8: Clear unbound client-declared scopes on shared-token WebSocket connects** — Unbound client-declared scopes are cleared when a shared-token WebSocket client connects, preventing scope escalation via client-declared scope claims (#44306).
- **GHSA-2pwv-x786-56f8: Device-token scope cap** — Device token scopes are now capped to the paired device's approved scope baseline, preventing a device from claiming scopes that were not granted during pairing (#43686).
- **GHSA-wcxr-59v9-rxr8: Sandbox session-tree visibility in `session_status`** — The `session_status` tool now enforces sandbox session-tree visibility checks before allowing reads or mutations, preventing cross-sandbox session access (#43754).
- **GHSA-2rqg-gjgv-84jm: Reject public spawned-run lineage fields** — Public-facing spawned-run requests with lineage fields are now rejected; workspace inheritance continues on the internal path only (#43801).

### Security — WebSocket and Handshake

- **GHSA-jv4g-m82p-2j93 + GHSA-xwx2-ppv2-wx98: Handshake retention and pre-auth frame size** — Unauthenticated handshake retention window is shortened; oversized pre-auth frames are now rejected before processing (#44089).

### Security — Webhooks

- **GHSA-g353-mgv3-8pcj: Feishu webhook requires `encryptKey`** — Feishu webhook ingress now requires `encryptKey` to be configured; unencrypted Feishu webhooks are rejected (#44087).
- **GHSA-mhxh-9pjm-w7q5: LINE webhook requires signatures** — LINE webhook ingress now requires signature validation before processing requests (#44090).
- **GHSA-5m9r-p9g7-679c: Zalo webhook rate-limits invalid secrets** — Zalo webhook ingress now rate-limits requests with invalid secrets (#44173).

### Security — Media Store

- **GHSA-6rph-mmhp-h7h9: Shared media-store size cap restored for browser proxy files** — The shared media-store size cap is restored for browser proxy files, preventing unbounded proxy file accumulation in the shared store (#43684).

### Plugins — Workspace Trust Requirement

- **GHSA-99qw-6mr3-36qr: Explicit trust required for workspace-discovered plugins** — Cloned repositories can no longer execute workspace plugin code without an explicit trust decision. `plugins.allow` must now include the plugin ID for any workspace-discovered plugin to load. Non-bundled plugins discovered in workspaces emit a warning if `plugins.allow` is unset, and loading proceeds only when trust has been explicitly granted (#44174).
- **Plugin discovery/load cache and provenance tracking** — Plugin discovery and load caches are fixed for env-scoped roots; provenance tracking is updated to correctly attribute plugins discovered from workspace-relative roots (#44046).

### Plugins — Provider Architecture

- **Provider-plugin architecture for Ollama, vLLM, SGLang** — Ollama, vLLM, and SGLang providers are moved onto a provider-plugin architecture. Each provider now owns its onboarding flow, model discovery, and model-picker setup, rather than these being handled by the core engine.

### Pairing — Bootstrap Tokens

- **Bootstrap tokens for `/pair` and `openclaw qr`** — Setup codes generated by `/pair` and `openclaw qr` are now short-lived bootstrap tokens rather than long-lived pairing codes, limiting the window of exposure for intercepted setup codes.

### Security — Nodes Tool

- **`nodes` tool marked as explicitly owner-only** — The `nodes` system tool is now marked as requiring owner-level authorization, ensuring it cannot be invoked by non-owner senders.

### Browser — Existing Session (v2026.3.12)

- **Stop reporting fake CDP ports for live attached Chrome** — When attaching to an existing Chrome session via Chrome DevTools MCP, OpenClaw no longer reports a fake CDP port or URL. The profile now renders `transport: chrome-mcp` to accurately reflect the attach mechanism.
- **Transport-aware timeout diagnostics** — Timeout diagnostic messages for browser operations are now aware of the active transport type, producing accurate guidance for `chrome-mcp` sessions versus standard CDP sessions.

---

## v2026.3.22-v2026.3.23 Delta Notes

### Security — Exec Approval Hardening

- **Single-use setup codes** — Bootstrap token setup codes are now single-use; a setup code is invalidated after first use, preventing replay of captured pairing codes.
- **Security/exec: fail closed for Perl `-M` and `-I` flags** — Exec approval flow for Perl now fails closed when `-M` (module load) or `-I` (include path) flags are encountered, preventing load-path injection through approved Perl invocations.
- **Security/exec: PowerShell `-File`/`-f` wrapper forms recognized** — Exec approval now recognizes PowerShell `-File` and `-f` as script-runner wrappers, applying consistent approval handling for PowerShell script invocations.
- **Security/exec: pnpm forms unwrapped** — Additional pnpm runner forms (`pnpm --reporter exec`, `pnpm node <file>`) are now unwrapped before exec approval matching, ensuring the underlying script receives the same approval scrutiny as direct invocations.
- **Security/exec: `env` dispatch wrappers unwrapped in shell-segment allowlist on macOS** — `env` dispatch wrappers inside shell-segment allowlist resolution are now unwrapped on macOS, so approval decisions reflect the effective executable rather than the `env` wrapper binary.
- **Security/exec: backslash-newline as shell line continuation on macOS** — Backslash-newline sequences are now treated as shell line continuations during macOS shell-chain parsing, closing a chain-break bypass via backslash-escaped newlines.
- **Security/exec: bind macOS skill auto-allow trust to both executable name and resolved path** — macOS skill auto-allow trust is now bound to both the executable name and its resolved (realpath) path, preventing trust from applying to a different binary that happens to occupy the same path.
- **macOS/exec approvals: per-agent exec approval settings respected in gateway prompter** — The gateway prompter now respects per-agent exec approval settings when processing approval requests, aligning interactive gateway behavior with the per-agent configuration (#13707).

### Security — External Content

- **Strip zero-width and soft-hyphen marker-splitting chars during boundary sanitization** — Zero-width characters and soft-hyphen (U+00AD) used for marker splitting are stripped during external content boundary sanitization, closing a bypass that could split boundary markers with invisible characters.

### Security — Webhooks

- **GHSA (iMessage): reject unsafe remote attachment paths before spawning SCP** — iMessage remote attachment handling now validates attachment paths before spawning SCP, rejecting paths that could escape the expected attachment directory.
- **Telegram webhook validates secret before reading/parsing request body** — Telegram webhook ingress now validates the webhook secret token before reading or parsing the request body, preventing parser-level exposure of untrusted input prior to auth.

### Plugins — Provider Architecture

- No additional provider-plugin changes in v2026.3.13 beyond the v2026.3.12 architecture migration.

### Browser — Chrome DevTools MCP Existing-Session (released line)

- **Official Chrome DevTools MCP attach mode for signed-in live Chrome sessions** — The `existing-session` driver is formalized as Chrome DevTools MCP attach mode. The `user` built-in profile (driver `existing-session`, `attachOnly: true`) provides attach-only access to the host's signed-in Chrome via the `chrome-devtools-mcp` package, without launching a separate browser process.
- **Built-in `profile="user"` for logged-in host browser** — The `user` browser profile is now a built-in profile (auto-injected if not explicitly configured) that routes through Chrome DevTools MCP for existing signed-in sessions.
- **Legacy extension relay removed** — `v2026.3.22` removed the built-in/current `chrome-relay` path and the `browser.relayBindHost` setting. The current released attach path is `existing-session` with the built-in `user` profile and optional `userDataDir`.
- **ClawHub released behavior changed materially** — ClawHub is now the default first-party plugin/skill marketplace path, macOS auth-path handling was fixed in `v2026.3.23`, and the shipped `v2026.3.23-2` correction build validates compatibility against the active runtime version instead of a stale fixed constant.
- **Browser act automation: batched actions, selector targeting, delayed clicks** — The browser act tool supports batched action sequences, explicit selector-based element targeting, and delayed click timing for more robust automation workflows (contributed by @vincentkoc).
- **Browser/existing-session: hardened driver validation and session lifecycle** — The `existing-session` driver validates its MCP session lifecycle more strictly and extracts shared ARIA role sets for consistent snapshot output across sessions (#45682).
- **Browser/existing-session: accept text-only `list_pages` and `new_page` responses** — The Chrome DevTools MCP integration now accepts text-only (non-structured) responses from `list_pages` and `new_page` tool calls, improving compatibility with variants of the `chrome-devtools-mcp` package.

---

## v2026.3.24 Delta Notes

### Security — Sandbox Fixes

- **Sandbox `alsoAllow`/re-allows fix** — Fixed sandbox `alsoAllow` and re-allow logic to correctly apply accumulated permission grants.
- **Sandbox explain hints sanitization** — Session keys in sandbox explain hints are now redacted with `shellEscapeSingleArg()` to prevent information leakage.

### Security — Media/File URL Bypass

- **`mediaUrl`/`fileUrl` alias bypass closed** — A bypass where `mediaUrl` and `fileUrl` aliases could circumvent URL validation has been closed.

### Security — Plugin SDK

- **Plugin-sdk `moduleUrl` threading fix for external plugins** — Fixed `moduleUrl` threading for external plugins so the plugin SDK correctly resolves module paths across external plugin boundaries.

### Security — Context Engine

- **Context engine retry for legacy `assemble()`** — Added retry logic for the legacy `assemble()` path in the context engine to improve resilience.

### Hooks — `before_dispatch`

- **New `before_dispatch` hook added to plugin hooks** — `src/plugins/types.ts` (lines 1705-1731) defines the `before_dispatch` plugin hook. First handler returning `handled: true` wins; the reply routes through the final-delivery path. This enables plugins to intercept messages before they reach the agent runtime.

---

## v2026.3.28 Delta Notes

### Plugin Hooks — `requireApproval` in `before_tool_call` (PR #55339)

- **`requireApproval` field added to `PluginHookBeforeToolCallResult`** — `src/plugins/types.ts` defines a new optional `requireApproval` object on the `before_tool_call` hook result. When a plugin returns `requireApproval`, the hook runner pauses tool execution and surfaces an approval prompt before the tool call proceeds.
- **`requireApproval` shape** — The object carries `title` (string), `description` (string), optional `severity` (`"info" | "warning" | "critical"`), optional `timeoutMs` (default 120 000 ms), optional `timeoutBehavior` (`"allow" | "deny"`), and an optional async `onResolution` callback that receives the final `PluginApprovalResolution` outcome (`"approved"`, `"denied"`, `"timed_out"`, or `"cancelled"`). The `pluginId` field is set automatically by the hook runner; plugins must not set it.
- **First-plugin-wins merge semantics** — The hook runner (`src/plugins/hooks.ts`) accumulates `requireApproval` from the first plugin that sets it; subsequent plugins cannot override it. Params are frozen after `requireApproval` is set so lower-priority plugins cannot modify the tool parameters. A lower-priority plugin can still set `block: true` even after `requireApproval` is already set.
- **Approval surfaces** — The approval request is dispatched via the gateway `plugin.approval.request` / `plugin.approval.waitDecision` RPC pair (`src/agents/pi-tools.before-tool-call.ts`). Approval can be resolved through the exec-approval overlay in connected clients, Telegram buttons (requires `isTelegramExplicitApprover`), Discord interactions (requires `isDiscordExecApprovalApprover`), or the `/approve` text command.
- **`/approve` command unified handling** — `src/auto-reply/reply/commands-approve.ts` detects `plugin:`-prefixed approval IDs and routes them directly to `plugin.approval.resolve`, bypassing exec-approval client checks. Unprefixed IDs try `exec.approval.resolve` first and automatically fall back to `plugin.approval.resolve` when the exec approval is not found, so the same `/approve <id> allow-once|allow-always|deny` syntax handles both exec and plugin approvals.

### Plugin SDK — `moduleUrl` Threading Fix for External Plugins (PR #54283)

- **`moduleUrl` threaded through plugin-sdk alias resolution** — `src/plugins/sdk-alias.ts` now accepts and propagates an optional `moduleUrl` hint through all alias-resolution helpers (`resolvePluginSdkAliasMap`, `resolvePluginSdkScopedAliasMap`, etc.). When `loader.ts` loads a plugin that lives outside the OpenClaw install directory (e.g., `~/.openclaw/extensions/`), it passes its own `import.meta.url` as the `moduleUrl` hint so the alias resolver can locate the correct `openclaw/plugin-sdk/*` subpath CJS shims even when argv-based root detection cannot find the OpenClaw root.
- **`plugin-sdk:check-exports` gated in `release:check`** — The `pnpm plugin-sdk:check-exports` script (which validates exported Plugin SDK subpaths) is now included in the `release:check` script (`package.json`) so drift between declared subpath exports and the actual SDK surface is caught before releases.

### Plugin SDK — Context Engine `assemble()` Retry for Pre-Prompt Engines (PR #50848)

- **Strict legacy `assemble()` retry without `prompt` field** — The context engine wrapper (`src/context-engine/`) now retries `assemble()` calls without the new `prompt` field when an older engine throws an "Unrecognized key(s)" error on receiving `prompt`. This preserves prompt-aware retrieval compatibility for plugins that registered a context engine before the `prompt` parameter was introduced, while still passing `prompt` to engines that support it. The first-retry result memoizes legacy mode for subsequent calls on the same engine instance.

### Plugin Startup — Auto-Load Bundled Provider and CLI-Backend Plugins

- **Bundled provider and CLI-backend plugins auto-loaded from explicit config refs** — The plugin loader now auto-loads bundled provider plugins and CLI-backend plugins when they appear in explicit config references, even when `plugins.allow` is not set. This ensures configured provider integrations (e.g., `acp.dispatch.backend = "acpx"`) activate the correct bundled plugin at startup without requiring a separate explicit `plugins.allow` entry.

### Security — Web Search Key Audit Extended (PR #56540)

- **New credential providers recognized in web search key audit** — `src/security/audit-extra.sync.ts` delegates web search credential detection to `hasBundledWebSearchCredential()` in `src/plugins/bundled-web-search-registry.ts`, which iterates all registered bundled web search providers. As of v2026.3.28, this includes Gemini (`GEMINI_API_KEY` / `GOOGLE_API_KEY` via the `google` extension), Grok/xAI (`XAI_API_KEY` via the `xai` extension), Kimi/Moonshot (`KIMI_API_KEY` / `KIMICODE_API_KEY` / `MOONSHOT_API_KEY` via the `moonshot` extension), and OpenRouter (`OPENROUTER_API_KEY` via the `openrouter` extension). The registry shim resolves credentials through the bundled web search provider registry, so the audit recognizes all keys that would auto-activate web search, not just the previously hard-coded set.

### Security — LINE Timing-Safe HMAC (PR #55663)

- **Timing-safe HMAC validation even when signature length mismatches** — `extensions/line/src/signature.ts` pads both the computed HMAC and the supplied signature to the same length before calling `crypto.timingSafeEqual()`, then gates acceptance on `hashBuffer.length === signatureBuffer.length`. This ensures `timingSafeEqual` is always called (no early return on length mismatch) so the comparison is constant-time regardless of whether the supplied signature has the wrong length, preventing timing side-channels in LINE webhook validation.

### ACP/ACPX — Agent Mirror Alignment and Unknown Agent ID Hardening (PR #28321)

- **Built-in agent mirror aligned with `openclaw/acpx` defaults** — `extensions/acpx/src/runtime-internals/mcp-agent-command.ts` maintains a `ACPX_BUILTIN_AGENT_COMMANDS` map that mirrors the built-in agent registry of the `acpx` CLI. The map uses pinned, exact-version `npx` invocations (e.g., `npx -y pi-acp@0.0.22`, `npx -y @zed-industries/codex-acp@0.9.5`) to ensure the bundled ACPX plugin and the external `acpx` binary agree on which agent commands are used for each well-known agent ID.
- **Unknown ACP agent IDs no longer fall through to raw `--agent` execution on MCP-proxy path** — `resolveAcpxAgentCommand()` returns `null` for agent IDs not found in either the user config overrides or the built-in map. The MCP-proxy dispatch path in `extensions/acpx/src/runtime.ts` treats a `null` result as an unresolvable agent and surfaces an error rather than passing the raw unknown agent ID as a `--agent` command argument, closing a bypass where an unrecognized agent ID could be forwarded as an arbitrary CLI argument.

### Breaking Changes (Security-Relevant)

- **Providers/Qwen: `qwen-portal-auth` OAuth integration removed** — The `qwen-portal-auth` bundled extension has been removed from the extensions directory. Users relying on Qwen Portal OAuth authentication should migrate to direct API key auth. The extension no longer appears in `openclaw plugins list`.
- **Config/Doctor: old migration keys fail validation** — Legacy config migration paths have been dropped. Config keys that previously triggered migrations now fail validation. Run `openclaw doctor` to identify affected config and follow the output guidance to update to current key paths.

## v2026.3.31 Delta Notes

### Install / Plugin Trust Boundaries

- **Plugin and skill installs now fail closed:** built-in dangerous-code `critical` findings and install-time scan failures block installs by default unless the operator explicitly opts into `--dangerously-force-unsafe-install`.
- **ACPX plugin-tools bridge is a released explicit trust boundary:** the bundled ACPX bridge is default-off and should be documented as an intentional operator choice, not an implicit plugin capability.

### Security Hardening In The Stable Line

- **Host exec env override blocking expands:** released hardening now blocks more request-scoped env overrides that could redirect Docker endpoints, trust roots, compilers, or language-specific runtime environments.
- **Nostr inbound DMs verify signatures before side effects:** forged events no longer create pairing requests or reply attempts on the stable line.

---

## v2026.4.1 Delta Notes

### Exec Approval Hardening

- **`allow-always` durability fix:** `allow-always` now persists as durable user-approved trust instead of behaving like `allow-once`. Exact-command trust is reused on shell-wrapper paths that cannot safely persist an executable allowlist entry.
- **Static allowlist bypass closed:** static allowlist entries no longer silently bypass `ask:"always"`, so configured always-ask policies are enforced even when a matching static entry exists.
- **Windows approval plan fallback:** Windows environments now require explicit approval when the system cannot build an allowlist execution plan, instead of hard-dead-ending remote exec.
- **Slack and Discord alignment:** Slack and Discord native approval handling aligned with inferred approvers and real channel enablement so remote exec stops falling into false approval timeouts and disabled states.
- **Telegram topic-aware approvals:** forum-topic exec approval followups route through Telegram-owned threading and approval-target parsing, staying in the originating topic.

### Exec / Cron Alignment

- **Isolated cron no-route dead-ends resolved:** approval dead-ends for isolated cron runs resolved from the effective host fallback policy when trusted automation is allowed.
- **`openclaw doctor` exec policy warning:** `openclaw doctor` now warns when `tools.exec` is broader than `~/.openclaw/exec-approvals.json` so stricter host-policy conflicts are explicit.
- **Per-job tool allowlists:** `openclaw cron --tools` enables per-job tool allowlists, with gateway-level tool-invoke gating for cron-scoped tool restrictions.

### Plugin Security

- **SearXNG plugin:** bundled SearXNG web search provider plugin (`extensions/searxng/`) with configurable host support and SSRF-safe fetch through the shared guard path.
- **Bedrock Guardrails:** Amazon Bedrock provider plugin gains Guardrails support, injecting `guardrailConfig` (identifier, version, optional stream processing mode and trace) into streaming request payloads via `before_llm_call` hooks.

### ACP Security

- **Dangerous-tool semantic approval classes:** ACP's dangerous-tool name override replaced with semantic approval classes so only narrow readonly reads/searches can auto-approve while indirect exec-capable and control-plane tools always require explicit prompt approval.

---

## v2026.4.7 Delta Notes

### New Provider Plugins

- **Arcee AI provider (`extensions/arcee/`, #62068):** first-party Arcee AI provider plugin. Supports direct Arcee platform API key auth (`arceeaiApiKey` / `ARCEE_AI_API_KEY`) and OpenRouter-routed Arcee models. Uses `openai-compatible` replay family hooks. Config path: `providers.arcee` or `providers.arcee-openrouter`.
- **Amazon Bedrock Mantle (`extensions/amazon-bedrock-mantle/`):** OpenAI-compatible Bedrock surface via the `@aws/bedrock-token-generator` runtime dependency. Distinct from the existing `amazon-bedrock` provider; targets the Bedrock Converse / OpenAI-compat endpoint.
- **Microsoft Foundry (`extensions/microsoft-foundry/`):** Microsoft Foundry provider with both Entra ID (service-principal) and API key authentication paths. Separate from the existing `microsoft` (Azure OpenAI) provider.

### New Tool/Feature Plugins

- **OpenShell sandbox (`extensions/openshell/`):** alternative sandbox backend implementing the `registerSandboxBackend("openshell", ...)` plugin-sdk seam. Registers a factory and manager for OpenShell-backed exec/file-tool sandboxes. Config path: `plugins.entries.openshell.config`. Useful for operators who want an OpenShell-native isolation layer instead of the default node-host sandbox.
- **Tavily (`extensions/tavily/`):** registers a Tavily web-search provider (`createTavilyWebSearchProvider`) plus two agent tools — `tavily_search` and `tavily_extract`. Config path: `plugins.entries.tavily.config`; requires a Tavily API key.
- **Memory Wiki (`extensions/memory-wiki/`):** see v2026.4.9 Delta Notes in `memory-cron-media.md` for full coverage.

### Pluggable Compaction Provider Registry (#56224)

- **`feat: add pluggable compaction provider registry`:** the compaction subsystem gains a provider registry API so plugins can register custom compaction backends. Previously compaction was hard-wired to built-in strategies; the registry seam allows extension-level compaction policies and checkpointing strategies.

### Security Hardening (v2026.4.7 window)

- **`openclaw infer` exec approval path hardened:** the new `openclaw infer` CLI respects the same exec approval gates as other OpenClaw CLI commands; no separate approval bypass.
- **SSRF guard no longer rejects operator-configured proxy hostnames (#62312):** `fix(gateway): stop SSRF guard rejecting operator-configured proxy hostnames` — operators who configure a forward proxy via `gateway.proxy` or `HTTPS_PROXY` no longer have those hostnames blocked by the SSRF guard.
- **Heartbeat targets main session only (#61803):** `fix(agents): heartbeat always targets main session — prevent routing to active subagent sessions` — prevents heartbeat messages from incorrectly being delivered to a running subagent session instead of the main session.

---

## v2026.4.9 Delta Notes

### Breaking / Public Config Surface

- **Legacy public aliases are retired:** the stable docs surface no longer treats old public aliases such as `talk.voiceId`, `talk.apiKey`, `agents.*.sandbox.perSession`, `browser.ssrfPolicy.allowPrivateNetwork`, `hooks.internal.handlers`, and channel/group/room `allow` toggles as first-class configuration.

### Plugin Runtime

- **Embedded ACPX runtime:** bundled ACPX now owns its runtime in process instead of relying on an extra external CLI hop, which changes how plugin/runtime isolation and reply interception should be documented.
- **Generic `reply_dispatch` hook:** bundled plugins can now intercept reply delivery through a generic hook rather than core ACP-specific paths.
- **Plugin install/update UX expanded:** plugin-config prompts in onboarding and `openclaw plugins install --force` are now part of the release-line plugin management story.

### Security / Allowlists / Hook Failure

- **Plugin-only allowlists stay restrictive:** release-line security fixes explicitly preserve restrictive plugin-only tool allowlists and require owner access for `/allowlist add` and `/allowlist remove`.
- **`before_tool_call` failure is fail-closed:** crashing tool hooks are now part of the release-line fail-closed security behavior rather than an edge case left to plugin authors.
