# OpenClaw Core Engine — Comprehensive Analysis
<!-- markdownlint-disable MD024 -->

> Updated: 2026-03-08 | Version: v2026.3.7 | Codebase: /path/to/openclaw
> Modules: agents (690 files), gateway (298 files), sessions (9 files), routing (10 files), providers (11 files), hooks (38 files), context-engine (5 files)

---

## Table of Contents

1. [Module: sessions](#1-module-sessions)
2. [Module: routing](#2-module-routing)
3. [Module: providers](#3-module-providers)
4. [Module: hooks](#4-module-hooks)
5. [Module: agents](#5-module-agents)
6. [Module: gateway](#6-module-gateway)
7. [Cross-Module Data Flow](#7-cross-module-data-flow)

---

## 1. Module: sessions

### Overview
Lightweight utility module for session metadata, key parsing, transcript events, and policy enforcement. No classes — pure functions. Provides the foundational vocabulary (types + helpers) that the routing and gateway modules build on.

### File Inventory (7 source, 2 tests)

| File | Description |
|------|-------------|
| `input-provenance.ts` | Tracks origin of user messages (external_user, inter_session, internal_system) |
| `level-overrides.ts` | Parse/apply verbose level overrides on session entries |
| `model-overrides.ts` | Apply model/provider/auth-profile overrides to SessionEntry; clears stale runtime model identity on override change |
| `send-policy.ts` | Rule-based policy engine deciding allow/deny for session sends |
| `session-key-utils.ts` | Parse agent session keys, detect cron/subagent/ACP/thread keys |
| `session-label.ts` | Validate session labels (max 64 chars) |
| `transcript-events.ts` | Pub/sub for session transcript file updates (observer pattern) |

### Key Types & Interfaces

| Type | File | Description |
|------|------|-------------|
| `InputProvenanceKind` | input-provenance.ts | `"external_user" \| "inter_session" \| "internal_system"` |
| `InputProvenance` | input-provenance.ts | `{ kind, sourceSessionKey?, sourceChannel?, sourceTool? }` |
| `ModelOverrideSelection` | model-overrides.ts | `{ provider, model, isDefault? }` |
| `SessionSendPolicyDecision` | send-policy.ts | `"allow" \| "deny"` |
| `ParsedAgentSessionKey` | session-key-utils.ts | `{ agentId, rest }` |
| `ParsedSessionLabel` | session-label.ts | Result type with ok/error discriminant |

### Key Functions

| Function | File | Signature | Purpose |
|----------|------|-----------|---------|
| `normalizeInputProvenance` | input-provenance.ts | `(value: unknown) → InputProvenance \| undefined` | Safely parse provenance from unknown |
| `applyInputProvenanceToUserMessage` | input-provenance.ts | `(msg, prov) → AgentMessage` | Attach provenance to user messages |
| `hasInterSessionUserProvenance` | input-provenance.ts | `(msg) → boolean` | Check if message came from another session |
| `applyModelOverrideToSessionEntry` | model-overrides.ts | `(params) → { updated }` | Mutate session entry with model override |
| `resolveSendPolicy` | send-policy.ts | `(params) → "allow" \| "deny"` | Evaluate send policy rules against session context |
| `parseAgentSessionKey` | session-key-utils.ts | `(key) → ParsedAgentSessionKey \| null` | Parse `agent:<id>:<rest>` format |
| `isSubagentSessionKey` | session-key-utils.ts | `(key) → boolean` | Detect subagent session keys |
| `getSubagentDepth` | session-key-utils.ts | `(key) → number` | Count `:subagent:` nesting depth |
| `isCronSessionKey` | session-key-utils.ts | `(key) → boolean` | Detect cron-triggered sessions |
| `resolveThreadParentSessionKey` | session-key-utils.ts | `(key) → string \| null` | Extract parent key from thread/topic keys |
| `onSessionTranscriptUpdate` | transcript-events.ts | `(listener) → unsubscribe` | Subscribe to transcript file changes |
| `emitSessionTranscriptUpdate` | transcript-events.ts | `(sessionFile) → void` | Notify listeners of transcript changes |

### Internal Dependencies
- `config/sessions.ts` — `SessionEntry` type
- `config/config.ts` — `OpenClawConfig` type
- `auto-reply/thinking.ts` — `VerboseLevel`, `normalizeVerboseLevel`
- `channels/chat-type.ts` — `normalizeChatType`
- `@mariozechner/pi-agent-core` — `AgentMessage` type

### External Dependencies
None (pure TypeScript)

### Data Flow
Sessions module is a **leaf dependency** — it provides utilities consumed by routing and gateway. `transcript-events.ts` implements a simple in-process pub/sub for real-time UI updates when session files change.

### Configuration
- Reads: `session.sendPolicy.rules`, `session.sendPolicy.default` (via `resolveSendPolicy`)
- Writes: `SessionEntry.verboseLevel`, `SessionEntry.providerOverride`, `SessionEntry.modelOverride`, `SessionEntry.authProfileOverride`

### Test Coverage
- `send-policy.test.ts` — Rule matching, channel/chatType filtering, key prefix matching
- `model-overrides.test.ts` — Override application, stale runtime field clearing, fallback notice cleanup (added v2026.3.1)
- Session-key parsing is validated through routing/session-key tests in `src/routing/` (no dedicated `session-key-utils.test.ts` in this release).

### Known Patterns
- **Observer** — `transcript-events.ts` (listener set with add/remove)
- **Result type** — `ParsedSessionLabel` uses discriminated union `{ ok: true } | { ok: false }`
- **Normalization** — Every function defensively trims/lowercases inputs

### Recent Changes

- **v2026.3.1:** `model-overrides.ts` now clears stale runtime `model`/`modelProvider` fields when the user switches model overrides, so status surfaces immediately reflect the selected model. Clears `fallbackNoticeSelectedModel`/`fallbackNoticeActiveModel`/`fallbackNoticeReason` on any override update. New `model-overrides.test.ts` added.
- **v2026.2.23:** Session keys canonicalized to lowercase; legacy case-variant entries migrated automatically. (`sessions/store.ts`)
- **v2026.2.22:** `session.dmScope` defaults to `per-channel-peer` on new CLI installs. Symlinked state-dir aliases resolved during transcript-path validation.

---

## 2. Module: routing

### Overview
Maps incoming channel messages to agent sessions. Given a channel, account, peer (group/DM), guild, and roles, resolves which agent handles it and builds the canonical session key. Central to multi-agent, multi-channel routing.

### File Inventory (5 source, 5 tests)

| File | Description |
|------|-------------|
| `bindings.ts` | Config-driven channel→agent account bindings |
| `resolve-route.ts` | Main routing engine — resolves agent + session key from input |
| `session-key.ts` | Session key construction, normalization, identity linking |

### Key Types & Interfaces

| Type | File | Description |
|------|------|-------------|
| `RoutePeer` | resolve-route.ts | `{ kind: ChatType, id: string }` — identifies a group/channel/DM target |
| `ResolveAgentRouteInput` | resolve-route.ts | Full routing context (cfg, channel, account, peer, guild, roles) |
| `ResolvedAgentRoute` | resolve-route.ts | Result: `{ agentId, channel, accountId, sessionKey, mainSessionKey, matchedBy }` |
| `SessionKeyShape` | session-key.ts | `"missing" \| "agent" \| "legacy_or_alias" \| "malformed_agent"` |
| `AgentBinding` | (from config) | Match rules for channel→agent assignment |

### Key Functions

| Function | File | Purpose |
|----------|------|---------|
| `resolveAgentRoute` | resolve-route.ts | **Primary entry point** — resolves agent + session key with tiered binding matching |
| `buildAgentSessionKey` | resolve-route.ts | Build session key from agent/channel/peer with DM scope support |
| `buildAgentPeerSessionKey` | session-key.ts | Full session key builder with identity link resolution |
| `buildAgentMainSessionKey` | session-key.ts | Build `agent:<id>:main` key |
| `resolveThreadSessionKeys` | session-key.ts | Append `:thread:<id>` suffix for threaded conversations |
| `buildGroupHistoryKey` | session-key.ts | Build history key for group chats |
| `normalizeAgentId` | session-key.ts | Lowercase, sanitize to `[a-z0-9_-]`, max 64 chars |
| `listBindings` | bindings.ts | Extract binding array from config |
| `listBoundAccountIds` | bindings.ts | List account IDs bound to a channel |
| `buildChannelAccountBindings` | bindings.ts | Build channel→agent→accounts map |
| `resolvePreferredAccountId` | bindings.ts | Pick preferred account from bound list |

### Routing Algorithm (resolve-route.ts)
The route resolver uses a **tiered matching** strategy against config bindings:

1. **binding.peer** — Exact peer (group/channel ID) match (group/channel kinds treated as equivalent via `peerKindMatches`)
2. **binding.peer.parent** — Parent peer match (thread inheritance)
3. **binding.guild+roles** — Discord guild + role-based routing
4. **binding.guild** — Discord guild match
5. **binding.team** — Slack team match
6. **binding.account** — Specific account binding
7. **binding.channel** — Wildcard channel binding
8. **default** — Falls back to default agent

Uses a `WeakMap` cache on the config object for evaluated bindings (max 2000 keys).

### Multi-Account Default Routing
`channels.<channel>.defaultAccount` allows selecting a preferred account ID when multiple accounts are configured for a channel. The value is normalized before matching against configured account IDs. Falls back to the first configured account if the default is unset or does not match.

### DM Session Scoping
Controlled by `session.dmScope`:
- `"main"` — All DMs share one session per agent
- `"per-peer"` — Separate session per peer (cross-channel)
- `"per-channel-peer"` — Separate per channel+peer
- `"per-account-channel-peer"` — Fully isolated

### Identity Linking
`session.identityLinks` config maps multiple channel identities to a canonical name, enabling cross-channel session continuity.

### Internal Dependencies
- `sessions/session-key-utils.ts` — `parseAgentSessionKey`, re-exports
- `agents/agent-scope.ts` — `resolveDefaultAgentId`
- `channels/chat-type.ts` — `normalizeChatType`, `ChatType`
- `config/config.ts` — `OpenClawConfig`
- `config/types.agents.ts` — `AgentBinding`
- `channels/registry.ts` — `normalizeChatChannelId`
- `globals.ts` — `shouldLogVerbose`
- `logger.ts` — `logDebug`

### External Dependencies
None

### Configuration
- Reads: `bindings[]`, `agents.list[]`, `session.dmScope`, `session.identityLinks`, `channels.<channel>.defaultAccount`

### Test Coverage
- `resolve-route.test.ts` — Tier matching, guild/role routing, DM scoping, identity links, group/channel peer kind equivalence
- `session-key.test.ts` — Key building, normalization, thread suffixes

### Known Patterns
- **Strategy/tiered matching** — Binding resolution uses ordered predicate tiers
- **WeakMap caching** — Config-scoped binding evaluation cache
- **Normalization** — Aggressive input sanitization throughout

### Recent Changes

- **v2026.3.1:** `peerKindMatches()` treats `group` and `channel` peer kinds as equivalent for binding scope matching, fixing bindings that targeted group peers but received channel-typed inbound messages (or vice versa). `channels.<channel>.defaultAccount` config key added for multi-account default routing. Inbound metadata now includes `account_id` in trusted inbound context (`fix(inbound-meta): #30984`).

---

## 3. Module: providers

### Overview
Provider-specific authentication and model discovery for GitHub Copilot and Qwen Portal. These are specialized OAuth/token-exchange flows that don't fit the general auth-profiles system.

### File Inventory (6 source, 5 tests)

| File | Description |
|------|-------------|
| `github-copilot-auth.ts` | Interactive CLI device-code OAuth flow for GitHub Copilot |
| `github-copilot-models.ts` | Default Copilot model definitions (gpt-4o, gpt-4.1, o1, etc.) |
| `github-copilot-token.ts` | Token exchange: GitHub OAuth → Copilot API token with caching |
| `google-shared.test-helpers.ts` | Test helpers for Google provider tests |
| `qwen-portal-oauth.ts` | OAuth refresh flow for Qwen Portal (chat.qwen.ai) |

### Key Types

| Type | File | Description |
|------|------|-------------|
| `CachedCopilotToken` | github-copilot-token.ts | `{ token, expiresAt, updatedAt }` — cached API token |
| `DeviceCodeResponse` | github-copilot-auth.ts | GitHub device flow response (device_code, user_code, verification_uri) |

### Key Functions

| Function | File | Purpose |
|----------|------|---------|
| `githubCopilotLoginCommand` | github-copilot-auth.ts | Full interactive login flow: device code → poll → save profile |
| `getDefaultCopilotModelIds` | github-copilot-models.ts | Returns list of default model IDs |
| `buildCopilotModelDefinition` | github-copilot-models.ts | Build ModelDefinitionConfig for a Copilot model |
| `resolveCopilotApiToken` | github-copilot-token.ts | Exchange GitHub token → Copilot token (with disk cache) |
| `deriveCopilotApiBaseUrlFromToken` | github-copilot-token.ts | Extract API base URL from semicolon-delimited token |
| `refreshQwenPortalCredentials` | qwen-portal-oauth.ts | Refresh Qwen OAuth access token using refresh token |

### Internal Dependencies
- `agents/auth-profiles.ts` — `ensureAuthProfileStore`, `upsertAuthProfile`
- `commands/models/shared.ts` — `updateConfig`
- `commands/onboard-auth.ts` — `applyAuthProfileConfig`
- `config/config.ts` — `OpenClawConfig`, `ModelDefinitionConfig`
- `config/paths.ts` — `resolveStateDir`
- `infra/json-file.ts` — `loadJsonFile`, `saveJsonFile`
- `cli/command-format.ts` — `formatCliCommand`

### External Dependencies
- `@clack/prompts` — Interactive CLI prompts (intro, note, outro, spinner)
- Node `fetch` — HTTP requests

### Configuration
- Token cache stored at: `<stateDir>/credentials/github-copilot.token.json`
- Auth profiles saved via `upsertAuthProfile`

### Test Coverage
- `github-copilot-token.test.ts` — Token exchange, caching, expiry, base URL derivation
- `google-shared.*.test.ts` (2 files) — Google function call ordering, parameter preservation
- `qwen-portal-oauth.test.ts` — Refresh flow, error handling

### Known Patterns
- **Device Code Flow** — GitHub OAuth device authorization grant
- **Token caching** — Disk-persisted token with expiry check (5-min safety margin)
- **Dependency injection** — `resolveCopilotApiToken` accepts `fetchImpl`, `loadJsonFileImpl`, etc. for testing

### Recent Changes

- **v2026.3.1:** Copilot token refresh: proactively refresh GitHub Copilot API tokens before expiry and retry on 401 auth errors during long-running embedded turns, preventing mid-session auth failures for Copilot-provider subagents.
- **v2026.2.23:** Vercel AI Gateway normalizes `vercel-ai-gateway/claude-*` shorthand refs to canonical Anthropic-routed IDs. Anthropic OAuth tokens (`sk-ant-oat-*`) skip `context-1m-*` beta injection. OpenRouter: conflicting top-level `reasoning_effort` removed when injecting `reasoning.effort`. Groq: TPM limit errors no longer classified as context overflow.
- **v2026.2.22:** Mistral provider added (embeddings + voice). Grounded Gemini web search via `tools.webSearch.provider: "gemini"`. Google Vertex AI available for Claude models. OpenRouter: inject `cache_control` on system prompts for Anthropic models.

---

## 4. Module: hooks

### Overview
Event-driven extensibility system. Hooks are triggered on lifecycle events (command, session, agent, gateway, message) and loaded from bundled, managed, workspace, or plugin directories. Each hook has a `HOOK.md` manifest with frontmatter metadata. Includes Gmail webhook integration as a major built-in hook.

### Architecture Pattern
- **Plugin architecture** with directory-based discovery
- **Event bus** (register/trigger pattern in `internal-hooks.ts`)
- **Frontmatter-driven metadata** (HOOK.md files)

### File Inventory (23 source, 15 tests)

| File | Description |
|------|-------------|
| `bundled-dir.ts` | Resolves path to bundled hooks directory (npm/dev/compiled) |
| `bundled/boot-md/handler.ts` | Runs boot checklist on gateway startup |
| `bundled/bootstrap-extra-files/handler.ts` | Injects extra files into agent bootstrap from config patterns |
| `bundled/command-logger/handler.ts` | Example: logs all commands to JSONL file |
| `bundled/session-memory/handler.ts` | Saves session context to memory files on `/new` command |
| `config.ts` | Hook eligibility checking (OS, bins, env, config requirements) |
| `frontmatter.ts` | Parse HOOK.md frontmatter → OpenClawHookMetadata |
| `gmail-ops.ts` | Gmail webhook setup CLI (gcloud, Pub/Sub, Tailscale) |
| `gmail-setup-utils.ts` | Gmail setup utilities (gcloud auth, topic/subscription management) |
| `gmail-watcher.ts` | Gmail watcher service (auto-starts `gog gmail watch serve`) |
| `gmail.ts` | Gmail hook config types, defaults, argument builders |
| `hooks-status.ts` | Build hook status report for UI/CLI display |
| `hooks.ts` | Re-exports from internal-hooks.ts (public API) |
| `install.ts` | Hook installation from archives, npm, or local paths |
| `installs.ts` | Record hook installations in config |
| `internal-hooks.ts` | Core event bus: register, unregister, trigger handlers |
| `llm-slug-generator.ts` | Generate filename slugs via LLM for session memory |
| `loader.ts` | Load and register hooks from directories + legacy config |
| `src/plugins/hooks.ts` | Plugin hook runner (lifecycle/tool/message/subagent hooks) |
| `types.ts` | All hook type definitions |
| `workspace.ts` | Discover hooks from bundled/managed/workspace/extra directories |

### Key Types

| Type | File | Description |
|------|------|-------------|
| `InternalHookEventType` | internal-hooks.ts | `"command" \| "session" \| "agent" \| "gateway" \| "message"` |
| `InternalHookEvent` | internal-hooks.ts | Event payload with type, action, sessionKey, context, messages |
| `InternalHookHandler` | internal-hooks.ts | `(event) → Promise<void> \| void` |
| `AgentBootstrapHookEvent` | internal-hooks.ts | Specialized event for agent bootstrap with workspaceDir, bootstrapFiles |
| `Hook` | types.ts | `{ name, description, source, filePath, baseDir, handlerPath }` |
| `HookEntry` | types.ts | `{ hook, frontmatter, metadata?, invocation? }` |
| `HookSource` | types.ts | `"openclaw-bundled" \| "openclaw-managed" \| "openclaw-workspace" \| "openclaw-plugin"` |
| `OpenClawHookMetadata` | types.ts | Requirements, events, OS, install specs, emoji, homepage |
| `HookInstallSpec` | types.ts | `{ kind: "bundled"\|"npm"\|"git", package?, repository?, bins? }` |
| `HookEligibilityContext` | types.ts | Remote platform/binary checking context |
| `GmailHookRuntimeConfig` | gmail.ts | Full resolved Gmail config (account, topic, serve, tailscale) |

### Key Functions

| Function | File | Purpose |
|----------|------|---------|
| `registerInternalHook` | internal-hooks.ts | Register handler for event key (e.g. `"command:new"`) |
| `triggerInternalHook` | internal-hooks.ts | Fire event to all matching handlers |
| `clearInternalHooks` | internal-hooks.ts | Reset all handlers (testing) |
| `loadInternalHooks` | loader.ts | Load all hooks from directories + legacy config |
| `shouldIncludeHook` | config.ts | Check eligibility (OS, bins, env, config requirements) |
| `resolveHookConfig` | config.ts | Get per-hook config from `hooks.internal.entries.<key>` |
| `loadWorkspaceHookEntries` | workspace.ts | Discover all hooks from all directories |
| `buildWorkspaceHookSnapshot` | workspace.ts | Build snapshot of eligible hooks |
| `installHooksFromNpmSpec` | install.ts | Install hook pack from npm registry |
| `installHooksFromArchive` | install.ts | Install from .tar.gz/.zip archive |
| `installHooksFromPath` | install.ts | Install from local directory or archive |
| `startGmailWatcher` | gmail-watcher.ts | Start Gmail watch service (spawns gog process) |
| `stopGmailWatcher` | gmail-watcher.ts | Stop Gmail watch service |
| `runGmailSetup` | gmail-ops.ts | Full Gmail setup CLI (gcloud, Pub/Sub, Tailscale, config) |
| `generateSlugViaLLM` | llm-slug-generator.ts | Generate filename slug from session content via embedded agent |
| `createHookRunner` | src/plugins/hooks.ts | Build typed plugin hook runner for lifecycle/tool/message events |

### Hook Discovery Order (workspace.ts)
Hooks are loaded from multiple sources with later sources overriding by name:
1. **Extra dirs** (`hooks.internal.load.extraDirs`) — lowest precedence
2. **Bundled** — shipped with OpenClaw
3. **Managed** — installed via `openclaw hooks install` (`~/.openclaw/hooks/`)
4. **Workspace** — in agent workspace `hooks/` dir — highest precedence

### Internal Dependencies
- `agents/agent-scope.ts` — `resolveDefaultAgentId`, `resolveAgentWorkspaceDir`
- `agents/pi-embedded.ts` — `runEmbeddedPiAgent` (for LLM slug generation)
- `agents/workspace.ts` — `WorkspaceBootstrapFile`, `loadExtraBootstrapFiles`
- `agents/skills.ts` — `hasBinary`
- `config/config.ts` — `OpenClawConfig`, `loadConfig`, `writeConfigFile`
- `config/paths.ts` — `resolveStateDir`
- `compat/legacy-names.ts` — `MANIFEST_KEY`
- `logging/subsystem.ts` — `createSubsystemLogger`
- `markdown/frontmatter.ts` — `parseFrontmatterBlock`
- `shared/config-eval.ts` — `hasBinary`, `isConfigPathTruthyWithDefaults`
- `shared/frontmatter.ts` — Frontmatter parsing helpers
- `process/exec.ts` — `runCommandWithTimeout`
- `plugins/types.ts` — `OpenClawPluginApi`
- `gateway/boot.ts` — `runBootOnce` (bundled boot hook)

### External Dependencies
- `node:fs`, `node:fs/promises`, `node:path`, `node:child_process`, `node:crypto`
- `@clack/prompts` (indirect, via gmail-ops)

### Configuration
- Reads: `hooks.enabled`, `hooks.internal.enabled`, `hooks.internal.handlers[]`, `hooks.internal.entries.<key>`, `hooks.internal.load.extraDirs`, `hooks.internal.installs`, `hooks.path`, `hooks.token`, `hooks.presets[]`, `hooks.gmail.*`
- Writes: `hooks.gmail.*`, `hooks.token`, `hooks.presets`, `hooks.internal.installs`

### Test Coverage (representative tests)
- `internal-hooks.test.ts` — Register, trigger, unregister, error handling
- `loader.test.ts` — Hook loading from directories, legacy config
- `workspace.test.ts` — Hook discovery, precedence, snapshot building
- `frontmatter.test.ts` — Metadata parsing from HOOK.md
- `install.test.ts` — Archive/npm/path installation
- `hooks-install.test.ts` — Hook installation validation flows
- `gmail.test.ts` — Config resolution, argument building
- `gmail-watcher-lifecycle.test.ts` — Watcher lifecycle, address-in-use detection
- `gmail-setup-utils.test.ts` — gcloud/dependency utilities
- `bundled/bootstrap-extra-files/handler.test.ts` — Extra file injection
- `bundled/session-memory/handler.test.ts` — Session memory saving

### Known Patterns
- **Event bus / Observer** — `internal-hooks.ts` register/trigger
- **Plugin architecture** — Directory-based discovery with HOOK.md manifests
- **Strategy** — Eligibility checking with requirement predicates
- **Factory** — `installHooksFrom*` family of functions
- **Process management** — Gmail watcher spawns/monitors child processes

---

## 5. Module: agents

### Overview
The largest module (348 source files, 335 tests). This is the **AI agent runtime** — it manages model selection, tool execution, system prompt construction, session management, sandbox environments, authentication, skills, subagents, and the embedded pi-agent integration. The core loop: receive message → build system prompt → select model → call LLM → handle tool calls → stream response.

### Architecture Pattern
- **Embedded agent runtime** — Wraps `@mariozechner/pi-ai` / `pi-agent-core` / `pi-coding-agent`
- **Tool registry** — Modular tool definitions in `tools/` subdirectory
- **Layered configuration** — Config cascade: defaults → agent config → session overrides → runtime
- **Streaming pipeline** — `pi-embedded-subscribe` processes SSE chunks into deliverable messages

### Entry Points
- `pi-embedded.ts` — Public API re-exports (`runEmbeddedPiAgent`, `abortEmbeddedPiRun`, etc.)
- `pi-embedded-runner.ts` — Runner barrel (re-exports from runner subdirectory)
- `pi-embedded-runner/run.ts` — Core `runEmbeddedPiAgent()` function
- `system-prompt.ts` — System prompt construction
- `pi-tools.ts` — Tool registry and creation
- `bash-tools.ts` — Shell execution tools (exec, process)

### Sub-module Structure

#### Core Runtime (`pi-embedded-*`)
| File/Dir | Purpose |
|----------|---------|
| `pi-embedded.ts` | Public API barrel |
| `pi-embedded-runner.ts` | Runner barrel |
| `pi-embedded-runner/run.ts` | Main agent run loop |
| `pi-embedded-runner/runs.ts` | Active run registry (abort, queue, status) |
| `pi-embedded-runner/compact.ts` | Session compaction (context window management) |
| `pi-embedded-runner/history.ts` | History turn limiting |
| `pi-embedded-runner/model.ts` | Model resolution for a run |
| `pi-embedded-runner/system-prompt.ts` | System prompt override logic |
| `pi-embedded-runner/extensions.ts` | pi-agent extensions (context pruning, compaction safeguard) |
| `pi-embedded-runner/extra-params.ts` | Extra model parameters |
| `pi-embedded-runner/google.ts` | Google-specific turn ordering fixes |
| `pi-embedded-runner/lanes.ts` | Lane (concurrency) resolution |
| `pi-embedded-runner/sandbox-info.ts` | Sandbox info for system prompt |
| `pi-embedded-runner/tool-split.ts` | Split SDK vs custom tools |
| `pi-embedded-runner/tool-result-truncation.ts` | Truncate large tool results |
| `pi-embedded-runner/types.ts` | Runtime type definitions |
| `pi-embedded-subscribe.ts` | SSE stream subscription and chunk processing |
| `pi-embedded-subscribe.handlers.ts` | Stream event handlers (messages, tools, compaction, lifecycle) |
| `pi-embedded-subscribe.types.ts` | Subscription type definitions |
| `pi-embedded-helpers.ts` | Helper barrel (errors, images, thinking, messaging) |
| `pi-embedded-messaging.ts` | Outbound messaging (send responses to channels) |
| `pi-embedded-block-chunker.ts` | Split long text blocks for streaming |
| `pi-embedded-utils.ts` | Embedded runtime utilities |

#### System Prompt
| File | Purpose |
|------|---------|
| `system-prompt.ts` | Main system prompt builder (sections: skills, memory, identity, time, workspace, tooling, runtime) |
| `system-prompt-params.ts` | Resolve system prompt parameters from config/session |
| `system-prompt-report.ts` | System prompt debugging/reporting |

#### Model Management
| File | Purpose |
|------|---------|
| `defaults.ts` | Default provider/model constants (anthropic, claude-opus-4-6) |
| `model-selection.ts` | Select model based on config, overrides, session state |
| `model-catalog.ts` | Load and cache model catalog |
| `model-compat.ts` | Model compatibility checking |
| `model-fallback.ts` | Fallback model resolution |
| `model-forward-compat.ts` | Forward compatibility for new model IDs |
| `model-scan.ts` | Scan for available models |
| `model-auth.ts` | Model authentication resolution |
| `model-alias-lines.ts` | Model alias display |
| `models-config.ts` | Write models.json config for pi-agent |
| `models-config.providers.ts` | Provider-specific model config generation |
| `live-auth-keys.ts` | Live auth key resolution |
| `live-model-filter.ts` | Runtime model filtering |
| `pi-model-discovery.ts` | Discover models via pi-ai registry |
| `synthetic-models.ts` | Synthetic/virtual model definitions |
| `together-models.ts` | Together.ai model catalog |
| `venice-models.ts` | Venice.ai model catalog |
| `huggingface-models.ts` | HuggingFace model catalog |
| `opencode-zen-models.ts` | OpenCode Zen model catalog |
| `minimax-vlm.ts` | MiniMax VLM model handling |
| `bedrock-discovery.ts` | AWS Bedrock model discovery |

#### Authentication
| File | Purpose |
|------|---------|
| `auth-profiles.ts` | Auth profile management barrel |
| `auth-profiles/*.ts` | Sub-modules: store, profiles, order, oauth, display, doctor, repair, session-override, usage, external-cli-sync, types, constants, paths |
| `auth-health.ts` | Auth health checking |
| `pi-auth-json.ts` | Write auth.json for pi-agent |
| `cloudflare-ai-gateway.ts` | Cloudflare AI Gateway proxy config |
| `chutes-oauth.ts` | Chutes provider OAuth |

#### Tools
| File | Purpose |
|------|---------|
| `pi-tools.ts` | Tool creation and registration (creates OpenClaw + coding tools) |
| `pi-tools.policy.ts` | Tool policy enforcement |
| `pi-tools.read.ts` | File read tool implementation |
| `pi-tools.schema.ts` | Tool schema definitions |
| `pi-tools.types.ts` | Tool type definitions |
| `pi-tools.abort.ts` | Tool abort handling |
| `pi-tools.before-tool-call.ts` | Pre-tool-call hooks |
| `pi-tool-definition-adapter.ts` | Adapt tool definitions between formats |
| `tool-policy.ts` | Tool access policy (allow/deny/ask) |
| `tool-policy-pipeline.ts` | Policy pipeline for tool calls |
| `tool-policy.conformance.ts` | Policy conformance checking |
| `tool-mutation.ts` | Track tool-caused mutations |
| `tool-display.ts` | Tool result display formatting |
| `tool-display-common.ts` | Common display helpers |
| `tool-summaries.ts` | Summarize tool results |
| `tool-call-id.ts` | Tool call ID generation |
| `tool-images.ts` | Image handling in tool results |
| `channel-tools.ts` | Channel-specific tool registration |
| `openclaw-tools.ts` | OpenClaw-specific tools (sessions, subagents, camera) |

#### Tools Subdirectory (`tools/`)
47 files implementing individual tools:

| File | Tool |
|------|------|
| `web-search.ts` | Web search (Brave, Perplexity, etc.) |
| `web-fetch.ts` | URL content fetching |
| `web-tools.ts` | Web tool registration |
| `web-shared.ts` | Shared web utilities |
| `web-fetch-utils.ts` | Fetch utilities (SSRF protection, etc.) |
| `browser-tool.ts` | Browser automation |
| `browser-tool.schema.ts` | Browser tool schema |
| `canvas-tool.ts` | Canvas control |
| `image-tool.ts` | Image analysis/generation |
| `image-tool.helpers.ts` | Image tool helpers |
| `memory-tool.ts` | Memory search/get |
| `message-tool.ts` | Send messages via channels |
| `tts-tool.ts` | Text-to-speech |
| `cron-tool.ts` | Cron job management |
| `nodes-tool.ts` | Mobile node control |
| `nodes-utils.ts` | Node utility helpers |
| `sessions-send-tool.ts` | Send to other sessions |
| `sessions-send-tool.a2a.ts` | Agent-to-agent messaging |
| `sessions-send-helpers.ts` | Send helper utilities |
| `sessions-list-tool.ts` | List sessions |
| `sessions-history-tool.ts` | Session history retrieval |
| `sessions-spawn-tool.ts` | Spawn new sessions |
| `sessions-helpers.ts` | Session tool helpers |
| `sessions-announce-target.ts` | Announce to target sessions |
| `session-status-tool.ts` | Session status reporting |
| `subagents-tool.ts` | Subagent management |
| `agents-list-tool.ts` | List configured agents |
| `agent-step.ts` | Agent step execution |
| `gateway-tool.ts` | Gateway control |
| `gateway.ts` | Gateway tool helpers |
| `telegram-actions.ts` | Telegram-specific actions |
| `discord-actions.ts` | Discord actions barrel |
| `discord-actions-guild.ts` | Discord guild management |
| `discord-actions-messaging.ts` | Discord messaging |
| `discord-actions-moderation.ts` | Discord moderation |
| `discord-actions-presence.ts` | Discord presence/status |
| `slack-actions.ts` | Slack actions |
| `whatsapp-actions.ts` | WhatsApp actions |
| `common.ts` | Shared tool utilities |

#### Bash/Shell Execution
| File | Purpose |
|------|---------|
| `bash-tools.ts` | Shell tool barrel |
| `bash-tools.exec.ts` | Command execution (exec tool) |
| `bash-tools.exec-runtime.ts` | Exec runtime helpers |
| `bash-tools.process.ts` | Process management (send-keys, kill, log) |
| `bash-tools.shared.ts` | Shared bash utilities |
| `bash-process-registry.ts` | Track running processes |
| `pty-dsr.ts` | PTY device status report handling |
| `pty-keys.ts` | PTY key mapping |
| `shell-utils.ts` | Shell utility functions |
| `apply-patch.ts` | Patch application |
| `apply-patch-update.ts` | Patch update tracking |

#### Sessions & Workspace
| File | Purpose |
|------|---------|
| `workspace.ts` | Workspace bootstrap files, extra files, workspace validation |
| `workspace-dir.ts` | Workspace directory resolution |
| `workspace-dirs.ts` | Multi-workspace directory handling |
| `workspace-run.ts` | Workspace run context |
| `workspace-templates.ts` | Workspace file templates |
| `session-write-lock.ts` | File-based session write locking |
| `session-file-repair.ts` | Repair corrupted session files |
| `session-transcript-repair.ts` | Repair session transcripts |
| `session-slug.ts` | Generate session slugs |
| `session-tool-result-guard.ts` | Guard against oversized tool results |
| `session-tool-result-guard-wrapper.ts` | Wrapper for tool result guard |

#### Skills
| File | Purpose |
|------|---------|
| `skills.ts` | Skill discovery, binary checking, prompt building |
| `skills-status.ts` | Skill status reporting |
| `skills-install.ts` | Skill installation |
| `skills/*.ts` | Sub-modules: types, frontmatter, config, workspace, bundled-dir/context, plugin-skills, refresh, serialize, env-overrides |

#### Agent Scope & Identity
| File | Purpose |
|------|---------|
| `agent-scope.ts` | Agent ID resolution, workspace/dir resolution, multi-agent support |
| `agent-paths.ts` | Agent directory path resolution |
| `identity.ts` | Agent identity (name, personality, prompt prefix) |
| `identity-file.ts` | Identity file loading |
| `identity-avatar.ts` | Avatar resolution |

#### Subagents
| File | Purpose |
|------|---------|
| `subagent-registry.ts` | Subagent lifecycle management |
| `subagent-registry.store.ts` | Persistent subagent store |
| `subagent-announce.ts` | Announce subagent results to parent |
| `subagent-announce-queue.ts` | Queue subagent announcements |
| `subagent-depth.ts` | Subagent nesting depth tracking |
| `subagent-spawn.ts` | Subagent spawning with sandbox mode and delivery params |
| `internal-events.ts` | Typed `AgentInternalEvent` system for structured task completion handoff (new v2026.3.1) |

#### Other
| File | Purpose |
|------|---------|
| `compaction.ts` | Context window compaction strategies |
| `context-window-guard.ts` | Guard against context overflow |
| `context.ts` | Model context window lookup |
| `current-time.ts` | Current time formatting |
| `date-time.ts` | Date/time formatting and timezone handling |
| `timeout.ts` | Agent timeout management |
| `usage.ts` | Token usage tracking |
| `lanes.ts` | Concurrency lane management |
| `announce-idempotency.ts` | Dedup announcement delivery |
| `cache-trace.ts` | Cache hit tracing |
| `docs-path.ts` | Documentation path resolution |
| `failover-error.ts` | Failover error handling |
| `glob-pattern.ts` | Glob pattern matching |
| `transcript-policy.ts` | Transcript retention policy |
| `pi-settings.ts` | pi-agent settings resolution |
| `pi-extensions/*.ts` | Extensions: compaction-safeguard, context-pruning |
| `sandbox.ts` | Sandbox barrel |
| `sandbox/*.ts` | Docker sandbox: config, docker, manage, prune, registry, fs-bridge, browser, workspace, etc. |
| `schema/*.ts` | JSON schema helpers (TypeBox, Gemini cleaning) |
| `ollama-stream.ts` | Ollama streaming adapter |
| `anthropic-payload-log.ts` | Anthropic request logging |
| `bootstrap-files.ts` | Bootstrap file loading |
| `bootstrap-hooks.ts` | Bootstrap hook integration |
| `cli-session.ts` | CLI session management |
| `cli-runner.ts` | CLI agent runner |
| `cli-backends.ts` | CLI backend selection |
| `cli-credentials.ts` | CLI credential management |
| `claude-cli-runner.ts` | Claude CLI integration |

### Key Types (selected)

| Type | File | Description |
|------|------|-------------|
| `EmbeddedPiRunResult` | pi-embedded-runner/types.ts | Result of an agent run (payloads, usage, error) |
| `EmbeddedPiAgentMeta` | pi-embedded-runner/types.ts | Agent metadata for a run |
| `PromptMode` | system-prompt.ts | `"full" \| "minimal" \| "none"` — system prompt detail level |
| `ToolPolicyAction` | tool-policy.ts | `"allow" \| "deny" \| "ask"` |
| `AgentInternalEvent` | internal-events.ts | Typed internal event union (`task_completion`) for structured subagent/cron handoff |
| `AgentTaskCompletionInternalEvent` | internal-events.ts | `{ type: "task_completion", source, childSessionKey, status, result, replyInstruction }` |

### Internal Dependencies (major)
- `config/*` — Configuration types and loading
- `routing/*` — Session key building
- `sessions/*` — Session utilities
- `hooks/*` — Hook triggering
- `auto-reply/*` — Reply dispatching
- `channels/*` — Channel plugins
- `infra/*` — Infrastructure utilities
- `plugins/*` — Plugin system

### External Dependencies
- `@mariozechner/pi-ai` — Core AI agent framework
- `@mariozechner/pi-agent-core` — Agent core types
- `@mariozechner/pi-coding-agent` — Coding agent tools
- `@sinclair/typebox` — JSON schema builder
- Various Node built-ins

### Configuration (major keys)
- `agents.list[]` — Multi-agent definitions
- `agents.list[].model` — Default model per agent
- `agents.list[].skills` — Skill configuration
- `agents.list[].tools` — Tool configuration
- `agents.list[].sandbox` — Sandbox configuration
- `agents.list[].identity` — Agent identity
- `agents.list[].workspace` — Workspace path
- `session.dmScope` — DM session scoping
- `models.*` — Model provider configuration
- `authProfiles` — Authentication profiles

### Test Coverage
267 test files covering virtually every aspect. Mix of unit tests and e2e tests. Notable patterns:
- Long descriptive test filenames (e.g., `pi-tools.create-openclaw-coding-tools.adds-claude-style-aliases-schemas-without-dropping.e2e.test.ts`)
- Test harness files for complex scenarios
- Mock files for gateway integration tests

### Known Patterns
- **Barrel pattern** — `pi-embedded.ts` → `pi-embedded-runner.ts` → individual modules
- **Middleware pipeline** — Tool policy pipeline, before-tool-call hooks
- **Registry** — Bash process registry, subagent registry, active run registry
- **Factory** — Tool creation (`createOpenClawCodingTools`, `createOpenClawTools`)
- **Strategy** — Model selection, auth profile ordering
- **Observer** — Stream subscription handlers
- **Builder** — System prompt construction
- **Guard/Circuit breaker** — Context window guard, tool result guard, compaction safeguard

### Recent Changes

- **v2026.3.1:** Subagent runtime events: typed `task_completion` internal events (`AgentInternalEvent` in new `internal-events.ts`) replace ad-hoc system-message handoff for subagent/cron completion announces. `AnnounceQueueItem` carries `internalEvents` array through queue drain and direct delivery paths. Cron completions skip the subagent status header. Announce steer messages use `formatAgentInternalEventsForPrompt()` instead of raw trigger strings. `sessions_spawn` rejects unsupported channel-delivery params (`target`, `transport`, `channel`, `to`, `threadId`, `replyTo`) with a `ToolInputError`. Thinking defaults: Claude 4.6 defaults to `adaptive` thinking; `thinkingDefault` config now accepts `"adaptive"` level. `thinkingDefault` priority: per-model defaults take precedence over agent-level defaults, and session entry `thinkingLevel`/`verboseLevel`/`reasoningLevel` are checked before agent defaults. Model failover: `ECONNREFUSED`, `ENETUNREACH`, `EHOSTUNREACH`, `ENETRESET`, `EAI_AGAIN` classified as failover-worthy network errors alongside existing `ETIMEDOUT`/`ESOCKETTIMEDOUT`/`ECONNRESET`/`ECONNABORTED`. Failover reason classification: `hasRateLimitTpmHint()` uses `\btpm\b` word-boundary regex instead of substring `includes("tpm")`, preventing false rate-limit classification from unrelated error messages containing `tpm` substrings. Copilot token refresh: proactively refresh GitHub Copilot tokens before expiry during long-running embedded turns. Sessions list: `transcriptPath` resolution now resolves per-agent store paths (including `{agentId}` expansion and `~` home-dir expansion) instead of requiring a pre-resolved store path. Subagent Slack thread delivery: `threadId` no longer blindly set to `conversationId`; only explicit requester thread hints are preserved, fixing invalid `thread_ts` on Slack DM/top-level delivery.
- **v2026.2.23:** Reasoning: when `thinking=low` (model-default thinking), auto-reasoning stays disabled. Reasoning-required errors no longer classified as context overflow. Context overflow: detect additional error shapes + Chinese patterns. HTTP 502/503/504 treated as failover-eligible transient timeouts.
- **v2026.2.22:** Moonshot: `supportsDeveloperRole=false` forced. Kimi token limit errors classified as context overflow. Google: non-base64 `thought_signature` sanitized from replay transcripts. Mistral: tool-call IDs sanitized. Ollama: large integer args preserved as exact strings. Transcripts: tool-call names validated before persistence.

---

## 6. Module: gateway

### Overview
The gateway is OpenClaw's **server process** — a WebSocket + HTTP server that bridges channels (Telegram, Discord, Slack, WhatsApp, CLI) to the agent runtime. Handles authentication, message routing, session management, cron jobs, node management, browser control, plugin HTTP, and real-time streaming to connected clients.

### Architecture Pattern
- **WebSocket RPC** — JSON-RPC-style request/response over WebSocket
- **Event-driven** — Server broadcasts events to connected clients
- **Method handlers** — Each RPC method in `server-methods/` subdirectory
- **Protocol versioning** — `PROTOCOL_VERSION` for client compatibility

### File Inventory (187 source, 107 tests)

#### Server Core
| File | Purpose |
|------|---------|
| `server.ts` | Public API barrel |
| `server.impl.ts` | Main `startGatewayServer()` — wires everything together |
| `server-startup.ts` | Sidecar startup (hooks, channels, browser, Gmail, memory) |
| `server-startup-log.ts` | Startup logging |
| `server-startup-memory.ts` | Memory backend initialization |
| `server-http.ts` | HTTP server setup |
| `server-close.ts` | Graceful shutdown handler |
| `server-shared.ts` | Shared server state/types |
| `server-constants.ts` | Server constants |
| `server-utils.ts` | Server utility functions |
| `server-runtime-config.ts` | Runtime config resolution |
| `server-runtime-state.ts` | Mutable server state |
| `server-restart-sentinel.ts` | Restart sentinel for zero-downtime restarts |

#### WebSocket
| File | Purpose |
|------|---------|
| `server/ws-connection.ts` | WebSocket connection handling |
| `server/ws-connection/auth-messages.ts` | Auth message handling |
| `server/ws-connection/message-handler.ts` | Message routing to method handlers |
| `server/ws-types.ts` | WebSocket type definitions |
| `server-ws-runtime.ts` | WebSocket runtime state |
| `server-broadcast.ts` | Broadcast events to connected clients |
| `ws-log.ts` | WebSocket logging |
| `ws-logging.ts` | Extended WS logging |

#### HTTP Endpoints
| File | Purpose |
|------|---------|
| `server/http-listen.ts` | HTTP listener setup |
| `server/http-auth.ts` | Canvas auth, plugin route auth, and `isCanvasPath` helpers (extracted from server-http.ts in v2026.3.1) |
| `server/plugins-http.ts` | Plugin HTTP routes |
| `server/tls.ts` | TLS configuration |
| `server-http.ts` | HTTP request router; includes health probe handler (`/health`, `/healthz`, `/ready`, `/readyz`) for container checks (added v2026.3.1) |
| `openai-http.ts` | OpenAI-compatible HTTP API |
| `openresponses-http.ts` | OpenAI Responses API compatibility |
| `open-responses.schema.ts` | Responses API schema |
| `http-auth-helpers.ts` | HTTP auth middleware |
| `http-common.ts` | Common HTTP utilities |
| `http-endpoint-helpers.ts` | Endpoint helper functions |
| `http-utils.ts` | HTTP utility functions |
| `hooks.ts` | Webhook HTTP endpoint |
| `hooks-mapping.ts` | Webhook event mapping |
| `tools-invoke-http.ts` | HTTP tool invocation |

#### Protocol
| File | Purpose |
|------|---------|
| `protocol/index.ts` | Protocol barrel |
| `protocol/schema.ts` | Schema barrel |
| `protocol/client-info.ts` | Client info types |
| `protocol/schema/*.ts` | Schema definitions: frames, sessions, agents, channels, config, cron, devices, exec-approvals, logs-chat, nodes, primitives, snapshot, types, wizard, error-codes |

#### RPC Method Handlers (`server-methods/`)
| File | Method |
|------|--------|
| `agent.ts` | Agent management |
| `agents.ts` | List/describe agents |
| `agent-job.ts` | Agent job execution |
| `agent-timestamp.ts` | Agent timestamp tracking |
| `attachment-normalize.ts` | Normalize message attachments |
| `browser.ts` | Browser control |
| `channels.ts` | Channel management |
| `chat.ts` | Chat message handling |
| `config.ts` | Config read/write |
| `connect.ts` | Client connection |
| `cron.ts` | Cron job management |
| `devices.ts` | Device management |
| `exec-approval.ts` | Single exec approval |
| `exec-approvals.ts` | Exec approval management |
| `health.ts` | Health check |
| `logs.ts` | Log retrieval |
| `models.ts` | Model listing/management |
| `nodes.ts` | Node management |
| `nodes.helpers.ts` | Node helpers |
| `nodes.handlers.invoke-result.ts` | Node invoke result handling |
| `send.ts` | Send messages |
| `sessions.ts` | Session management |
| `skills.ts` | Skills listing |
| `system.ts` | System info |
| `talk.ts` | Talk (voice) |
| `tts.ts` | Text-to-speech |
| `types.ts` | Method handler types |
| `update.ts` | Update checking |
| `usage.ts` | Usage reporting |
| `voicewake.ts` | Voice wake |
| `web.ts` | Web search/fetch |
| `wizard.ts` | Onboarding wizard |

#### Server Features
| File | Purpose |
|------|---------|
| `server-chat.ts` | Chat run registry, agent event handling |
| `server-channels.ts` | Channel manager (start/stop channel plugins) |
| `server-cron.ts` | Cron service (scheduled agent runs) |
| `server-discovery.ts` | mDNS/Bonjour discovery |
| `server-discovery-runtime.ts` | Discovery runtime |
| `server-lanes.ts` | Concurrency lane management |
| `server-maintenance.ts` | Maintenance timers (cleanup, health) |
| `server-model-catalog.ts` | Model catalog serving |
| `server-plugins.ts` | Plugin lifecycle management |
| `server-tailscale.ts` | Tailscale integration |
| `server-session-key.ts` | Session key resolution |
| `server-wizard-sessions.ts` | Wizard session management |
| `server-reload-handlers.ts` | Config reload handlers |

#### Startup & Configuration
| File | Purpose |
|------|---------|
| `startup-control-ui-origins.ts` | Seed `gateway.controlUi.allowedOrigins` for non-loopback installs at startup (new v2026.3.1) |

#### Node Management
| File | Purpose |
|------|---------|
| `node-registry.ts` | Mobile/remote node registry |
| `server-mobile-nodes.ts` | Mobile node connection handling |
| `server-node-events.ts` | Node event handling (expanded with typed node event dispatch in v2026.3.1) |
| `server-node-events-types.ts` | Node event types |
| `server-node-subscriptions.ts` | Node event subscriptions |
| `node-command-policy.ts` | Node command security policy |
| `node-invoke-sanitize.ts` | Sanitize node invoke requests |
| `node-invoke-system-run-approval.ts` | System run approval for nodes |

#### Authentication
| File | Purpose |
|------|---------|
| `auth.ts` | Gateway auth (token/password/device) |
| `auth-rate-limit.ts` | Auth rate limiting |
| `device-auth.ts` | Device-based authentication |
| `probe-auth.ts` | Probe auth for health checks |
| `origin-check.ts` | Origin header validation |

#### Sessions & Chat
| File | Purpose |
|------|---------|
| `session-utils.ts` | Session CRUD operations |
| `session-utils.fs.ts` | Session filesystem operations |
| `session-utils.types.ts` | Session utility types |
| `sessions-patch.ts` | Session patching |
| `sessions-resolve.ts` | Session resolution |
| `chat-abort.ts` | Abort active chat runs |
| `chat-attachments.ts` | Handle chat attachments |
| `chat-sanitize.ts` | Sanitize chat input |

#### Misc
| File | Purpose |
|------|---------|
| `client.ts` | Gateway WebSocket client |
| `call.ts` | Gateway RPC call helper |
| `boot.ts` | Boot checklist runner |
| `config-reload.ts` | Config file watcher + reload |
| `control-ui.ts` | Control UI state management |
| `control-ui-shared.ts` | Shared control UI helpers |
| `agent-prompt.ts` | Agent prompt handling |
| `assistant-identity.ts` | Assistant identity resolution |
| `exec-approval-manager.ts` | Exec approval workflow |
| `live-image-probe.ts` | Probe image URLs for validity |
| `net.ts` | Network utilities (LAN IP resolution) |
| `probe.ts` | Gateway probe (health/status) |
| `server/health-state.ts` | Health state tracking |
| `server/close-reason.ts` | Close reason formatting |
| `server/hooks.ts` | Server hook integration |

### Key Types

| Type | File | Description |
|------|------|-------------|
| `GatewayServer` | server.impl.ts | Main server object |
| `GatewayServerOptions` | server.impl.ts | Server startup options |
| `GatewayClient` | client.ts | WebSocket client class |
| `CallGatewayOptions` | call.ts | RPC call options |
| `ResolvedAgentRoute` | (from routing) | Resolved agent for incoming message |
| `EventFrame` | protocol/schema/frames.ts | WebSocket event frame |
| `RequestFrame` | protocol/schema/frames.ts | WebSocket request frame |
| `ConnectParams` | protocol/schema/types.ts | Client connection parameters |
| `ChatRunEntry` | server-chat.ts | Active chat run tracking |
| `NodeRegistry` | node-registry.ts | Connected node management class |
| `ExecApprovalManager` | exec-approval-manager.ts | Exec approval workflow class |

### Data Flow (Message Lifecycle)

```
Channel (Telegram/Discord/...) 
  → Channel Plugin 
    → Gateway WebSocket server 
      → resolveAgentRoute() [routing module]
        → Session resolution [gateway/session-utils]
          → runEmbeddedPiAgent() [agents module]
            → System prompt construction
            → Model selection + auth
            → LLM API call (streaming)
              → Tool calls → Tool execution → LLM continuation
            → pi-embedded-subscribe (stream processing)
              → Block chunking
              → Message delivery to channel
            → Session file persistence
          → Broadcast to connected clients
```

### Internal Dependencies
- `agents/*` — Agent runtime (the primary consumer)
- `routing/*` — Route resolution
- `sessions/*` — Session utilities
- `hooks/*` — Hook triggering, Gmail watcher
- `config/*` — Configuration
- `channels/*` — Channel plugins
- `auto-reply/*` — Reply dispatching
- `infra/*` — Infrastructure (device identity, TLS, updates, etc.)
- `plugins/*` — Plugin registry and services
- `canvas-host/*` — Canvas server
- `logging/*` — Subsystem logging
- `wizard/*` — Onboarding wizard

### External Dependencies
- `ws` — WebSocket library
- `@sinclair/typebox` — Schema validation
- Various Node built-ins (`http`, `https`, `crypto`, `fs`, `path`, `net`)

### Configuration (major keys)
- `gateway.port` — Server port (default: 18789)
- `gateway.auth` — Auth token/password
- `gateway.tls` — TLS configuration
- `gateway.discovery` — mDNS discovery
- `gateway.tailscale` — Tailscale integration
- `gateway.lanes` — Concurrency configuration
- `gateway.nodes` — Node management config
- `gateway.browser` — Browser control config

### Test Coverage
83 test files. Heavy e2e testing of server methods, protocol handling, and integration scenarios.

### Known Patterns
- **RPC/Method dispatch** — JSON-RPC-style method handlers
- **Pub/Sub** — Server broadcast to connected WebSocket clients
- **Registry** — Node registry, chat run registry, exec approval manager
- **Middleware** — Auth, rate limiting, origin checking
- **Observer** — Config reload, agent events
- **Process management** — Gmail watcher, browser control server
- **Protocol versioning** — Client/server protocol negotiation

### Recent Changes

- **v2026.3.1:** Health probes: `/health`, `/healthz` (liveness) and `/ready`, `/readyz` (readiness) endpoints added to `server-http.ts` via `GATEWAY_PROBE_STATUS_BY_PATH` map and `handleGatewayProbeRequest()` for container health checks (Kubernetes, Docker, Fly.io). Control UI method guard: POST requests to `/plugins/*` and `/api/*` paths fall through to their respective handlers instead of being caught by the SPA fallback, preventing untrusted plugins from claiming arbitrary UI paths. WS security: origin allowlist in `origin-check.ts` upgraded from array `includes` to `Set.has`; wildcard `["*"]` in `gateway.controlUi.allowedOrigins` now explicitly accepted. Control UI CSP: `style-src` includes `https://fonts.googleapis.com` and `font-src` includes `https://fonts.gstatic.com` for Google Fonts support. Control UI origins: `startup-control-ui-origins.ts` seeds `gateway.controlUi.allowedOrigins` for non-loopback installs upgrading to v2026.2.26+ at startup. macOS supervised restart: `restartGatewayProcessWithFreshPid()` uses `launchctl kickstart -k` (via `triggerOpenClawRestart()`) on macOS under launchd to bypass ThrottleInterval delays. macOS TLS certs: `NODE_EXTRA_CA_CERTS` added to LaunchAgent environment for custom CA certificate support. Node exec approval: `systemRunPlanV2` renamed to `systemRunPlan` (breaking: old node clients sending `systemRunPlanV2` payloads will fail approval matching). `system.run` pins to canonical `realpath`: `resolveCommandResolution()` now resolves `resolvedRealPath` via `fs.realpathSync`, used in approval binding for symlink-resistant command identity. `node.canvas.capability.refresh` added to `NODE_ROLE_METHODS` in method-scopes. Canvas auth helpers extracted from `server-http.ts` into `server/http-auth.ts`.
- **v2026.2.23:** WS: repeated unauthorized request floods closed per-connection with sampled rejection logging. Config Write: `unsetPaths` applied with immutable path-copy updates; prototype-key traversal rejected in `config get/set/unset`.
- **v2026.2.22:** Auth: unified credential-source precedence via shared resolver helpers. Pairing: `operator.admin` satisfies `operator.*` scope checks; loopback scope-upgrade auto-approved; default scope bundles include `operator.read`/`operator.write`.

---

## 7. Cross-Module Data Flow

### Incoming Message Flow
```
External Channel → Channel Plugin → Gateway Server
  ↓
routing/resolve-route.ts: resolveAgentRoute(cfg, channel, account, peer)
  → Returns: { agentId, sessionKey, matchedBy }
  ↓
gateway/session-utils.ts: Load/create session entry
  ↓
sessions/send-policy.ts: resolveSendPolicy() → allow/deny
  ↓
sessions/input-provenance.ts: Tag message provenance
  ↓
agents/pi-embedded-runner/run.ts: runEmbeddedPiAgent()
  ├── agents/system-prompt.ts: Build system prompt
  ├── agents/model-selection.ts: Select model + auth
  ├── agents/pi-tools.ts: Register available tools
  └── Stream response → agents/pi-embedded-subscribe.ts
       ├── Tool calls → agents/bash-tools.exec.ts / agents/tools/*.ts
       ├── Chunk text → agents/pi-embedded-block-chunker.ts
       └── Deliver → agents/pi-embedded-messaging.ts → Channel
```

### Configuration Cascade
```
config/config.ts (openclaw.json)
  → agents/agent-scope.ts (resolve agent config)
    → agents/model-selection.ts (resolve model)
      → agents/auth-profiles.ts (resolve credentials)
        → pi-ai API call
```

### Session Key Construction
```
routing/session-key.ts: buildAgentPeerSessionKey()
  Format: agent:<agentId>:<channel>:<peerKind>:<peerId>
  DM:     agent:<agentId>:main (default) or agent:<agentId>:direct:<peerId>
  Thread: agent:<agentId>:<channel>:<peerKind>:<peerId>:thread:<threadId>
  Subagent: agent:<agentId>:subagent:<uuid>
  Cron:   agent:<agentId>:cron:<name>:run:<runId>
```

### Hook Trigger Points
```
Gateway startup → hooks: "gateway:startup"
/new command    → hooks: "command:new" (session memory hook)
Agent bootstrap → hooks: "agent:bootstrap" (extra files, boot checklist)
```

---

## v2026.2.15 Changes (2026-02-16)

~816 commits across core engine modules in the v2026.2.15 release window. Key changes by sub-module:

### Agents
- **Nested subagent orchestration** — depth-2 nesting with max 5 children per agent; deterministic idempotency keys to prevent duplicate announces; announce queue retry on send failure (`b8f66c260`, `ade11ec89`, `bbbec7a5c`, `2a8360928`)
- **before_tool_call hook double-fire fix** — deduplicated hook dispatch in `toToolDefinitions` and embedded runtime; hooks now fire from both tool execution paths without duplication (`8c3cc793b`, `534e4213a`, `d34138dfe`)
- **Compaction improvements** — prevent double compaction from cache-TTL guard bypass; stabilize overflow compaction retries and session context accounting; preserve per-agent exec overrides after compaction; session lock deadlock prevention during compaction timeout (`dcb921944`, `957b88308`, `3b5a9c14d`, `e6f67d5f3`)
- **Agent-isolated QMD collections** — prevent QMD scope deny bypass; eager-init QMD backend on startup; reuse default model cache; throttle embed + citations auto + restore --force; clamp QMD citations to injected budget (`f9bb748a6`, `efc79f69a`, `e4651d6af`, `9df78b337`, `1861e7636`)
- **Skill-filter normalization** — normalize skill-filter snapshots in cron; split isolated run helpers (`aef1d5530`)
- **Transcript hardening** — harden transcript tool-call block sanitization; resolve transcript paths with explicit agent context (`aa56045b4`, `cab0abf52`)
- **Empty-chunk timeout failover** — classify empty-chunk stream failures as timeout for proper failover behavior (`eb846c95b`)
- **Sandbox improvements** — centralize sha256 helpers, replace deprecated SHA-1; preserve array order in config hashing; clarify container-vs-host workspace paths in prompt; allow registry entries without agent scope (`d1fca442b`, `559c8d993`, `41ded303b`, `799049f58`, `b567ba5df`)

### Gateway
- **Sandbox bind validation** — tighten sandbox bind validation and harden docker config validation (`a7cbce1b3`, `887b209db`)
- **Control UI XSS fix** — serve Control UI bootstrap config and lock down CSP; share bootstrap contract; preserve control-ui scopes in bypass mode (`adc818db4`, `c6e6023e3`, `eed02a2b5`)
- **Session tool/webhook scoping** — scope session tools and webhook secret fallback for security (`c6c53437f`)
- **Sensitive field redaction** — redact sensitive status details for non-admin scopes (`fac040cb1`)
- **Per-channel ackReaction** — support per-channel `ackReaction` config (`b6069fc68`, community contrib @zerone0x)
- **Cron webhook** — add cron finished-run webhook (`115cfb443`)
- **Session persistence** — preserve session mapping across gateway restarts; keep boot sessions ephemeral (`b562aa662`, `a90e007d5`)
- **Control UI partial output** — preserve partial output on abort (`14fb2c05b`)
- **Prompt path sanitization** — harden prompt path sanitization for security (`6254e96ac`)

### Sessions
- **Transcript improvements** — resolve transcript paths with explicit agent context; archive old transcript files on `/new` and `/reset`; fix non-default agent session transcript path resolution; split access and resolution helpers (`cab0abf52`, `31537c669`, `ac4117653`, `1a03aad24`)
- **Session list caching** — cache session list transcript fields for performance (`3bbd29bef`)

### Hooks
- **Plugin LLM hooks** — expose LLM input/output hook payloads for plugins (`7c822d039`, community contrib @SecondThread)
- **Compaction/reset hooks** — Plugin API compaction/reset hooks, bootstrap file globs, memory plugin status (`ab71fdf82`)

### Providers
- **model_not_found failover** — handle 400 status in failover to enable model fallback (`71b4be879`)
- **OpenAI reasoning replay IDs** — preserve openai reasoning replay ids (`68ea06395`)

### Routing
- **Binding matching normalization** — normalize binding matching and harden QMD boot-update tests (`2583de530`)

### Cross-cutting
- **Security hardening** — restrict skill download target paths; harden untrusted web tool transcripts; sandbox allow-all semantics preservation (`2363e1b08`, `da55d70fb`, `4d9e310da`)
- **Discord** — component v2 UI tool support; preserve channel session keys via `channel_id` fallbacks; apply historyLimit to channel/group sessions (`a61c2dc4b`, `09566b169`, `5378583da`)
- **Skills** — cross-platform install fallback for non-brew environments (`d19b74692`)
- **Memory** — harden context window cache collisions (`cbf58d993`)
- **Test infrastructure** — massive test consolidation and deduplication (~100+ commits); shared env snapshot helpers; folded mini-suites into larger test files for faster CI

## v2026.2.19 Changes (2026-02-19)

### Agents
- **Read tool auto-paging** — `read` tool auto-pages based on model `contextWindow`; larger-context models read more before context guards kick in
- **Sub-agent context guard** — Truncates oversized tool outputs and compacts oldest tool-result messages before model calls; explicit recovery guidance for `[compacted: ...]` markers
- **Exec preflight guard** — Detects shell env var injection patterns in Python/Node scripts before execution. See DEVELOPER-REFERENCE.md §9 (gotcha 40)
- **YAML 1.2 frontmatter** — Core schema parsing; `on`/`off`/`yes`/`no` no longer coerced to booleans. See DEVELOPER-REFERENCE.md §9 (gotcha 37)

### Gateway
- **Auth defaults to token mode** — Unresolved auth defaults to token with auto-generated token; explicit `mode: "none"` required for open loopback. See DEVELOPER-REFERENCE.md §6
- **hooks.token ≠ gateway.auth.token** — Startup validation rejects matching tokens
- **Rate-limited control-plane RPCs** — `config.apply`, `config.patch`, `update.run` limited to 3/min per device+IP; config change audit logging
- **Security headers** — `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer` on HTTP responses
- **Plaintext ws:// blocked** — Non-loopback plaintext WebSocket connections rejected
- **Browser relay auth** — `/extension` and `/cdp` endpoints require `gateway.auth.token`
- **Canvas node-scoped sessions** — Canvas capabilities node-scoped, replacing shared-IP fallback

### Sessions
- **Heartbeat skip on empty** — Interval heartbeats skipped when `HEARTBEAT.md` missing/empty and no tagged cron events queued

### Routing
- **Telegram topic delivery** — Cron/heartbeat `<chatId>:topic:<threadId>` targets now route correctly

---

## v2026.2.21 Changes (2026-02-21)

<!-- v2026.2.21 -->

### commands-subagents Refactor

- **File**: `src/auto-reply/reply/commands-subagents.ts` (727-line change) + new `src/auto-reply/reply/commands-subagents/shared.ts` (432 lines)
- **What changed**: The monolithic `commands-subagents.ts` was split into a barrel dispatcher plus a `commands-subagents/shared.ts` module. The shared module contains all types (`SubagentsCommandContext`, `SubagentsAction`), helper functions (`resolveRequesterSessionKey`, `resolveHandledPrefix`, `resolveSubagentsAction`, `stopWithText`, `formatSubagentListLine`, `resolveFocusTargetSession`), command constants (`COMMAND`, `COMMAND_KILL`, `COMMAND_FOCUS`, etc.), and utilities that all action handlers (`action-list.ts`, `action-focus.ts`, `action-spawn.ts`, etc.) depend on.
- **Operational impact**: Internal refactor only — behavior of `/subagents`, `/focus`, `/unfocus`, `/kill`, `/steer`, `/tell`, `/agents` commands is unchanged. The dispatcher `handleSubagentsCommand` remains the single entry point. Per-action modules import shared helpers from `shared.ts` rather than duplicating logic.

### tool-display-common Compound Command Fix

- **File**: `src/agents/tool-display-common.ts` (224 lines)
- **What changed**: `resolveExecDetail` now correctly handles compound shell commands such as `cd ~/dir && npm install`. Previously, the summarizer extracted only the first stage (showing `list files in ~/dir` for a `cd && npm install` sequence). The fix:
  1. `stripShellPreamble()` strips leading `cd`/`pushd`/`export`/`set`/`unset` preamble stages and captures the inferred working directory (`chdirPath`).
  2. `splitTopLevelStages()` splits the remaining command on `&&`, `||`, `;` respecting quoting.
  3. Each stage is summarized independently via `summarizePipeline()` and joined with ` → `.
  4. If all stages produce generic summaries, the compact raw command is shown instead.
- **Operational impact**: The tool approval UI and activity feed now show the full compound command (e.g. `install dependencies (in ~/dir)\n\n\`cd ~/dir && npm install\``) rather than truncating to the first stage.

### invoke / system-run Hardening

- **Files**: `src/node-host/invoke-system-run.ts` (new file, 422 lines), `src/node-host/invoke.ts` (385-line change)
- **What changed**: System run command resolution was extracted from `invoke.ts` into a dedicated `invoke-system-run.ts` module via `handleSystemRunInvoke()`. Key security hardening applied:
  - `sanitizeSystemRunEnvOverrides()` from `src/infra/host-env-security.ts` blocks startup-file env injection (e.g. `BASH_ENV`, `ENV`, `NODE_OPTIONS` that point to files) and heredoc command substitution in shell wrappers.
  - `resolveSystemRunCommand()` centralizes argv/shell command resolution from `src/infra/system-run-command.ts`.
  - Exec approvals (`resolveExecApprovals`, `evaluateShellAllowlist`, `evaluateExecAllowlist`) are resolved per-agent from config and enforced before any command runs.
  - Runtime command override via injected env is prevented.
- **Operational impact**: Tighter security posture for node-host-executed commands. The extracted module makes approval and allowlist logic independently testable.

### Status-Reactions Controller

- **File**: `src/channels/status-reactions.ts` (new file, 390 lines)
- **What changed**: A new channel-agnostic `createStatusReactionController()` function provides a unified lifecycle reaction state machine used by both Telegram and Discord channel plugins. Features:
  - `StatusReactionController` API: `setQueued()`, `setThinking()`, `setTool(toolName?)`, `setDone()`, `setError()`, `clear()`, `restoreInitial()`.
  - Promise chain serialization prevents concurrent API calls.
  - Debounced intermediate state transitions (`debounceMs`, default 700 ms); terminal states (`done`/`error`) are immediate.
  - Stall timers: soft stall (`stallSoftMs` 10 s, default `🥱`) and hard stall (`stallHardMs` 30 s, default `😨`) fire automatically on inactivity.
  - Tool-type emoji routing: `CODING_TOOL_TOKENS` (exec, read, write, edit, bash) → coding emoji; `WEB_TOOL_TOKENS` (web_search, browser) → web emoji; all others → generic tool emoji.
  - Configurable emoji overrides via `StatusReactionEmojis` and timing overrides via `StatusReactionTiming`.
  - `StatusReactionAdapter` interface (`setReaction`, optional `removeReaction`) abstracts platform differences (Telegram sets atomically; Discord-style platforms need explicit remove).
- **Operational impact**: Reaction update logic is now shared across channels. Both Telegram and Discord status reactions go through this controller, fixing timing/dedup inconsistencies that existed when each channel had its own ad-hoc reaction management.

### subagent-spawn: senderIsOwner Forwarded to Embedded Runner

- **File**: `src/agents/subagent-spawn.ts` (236 lines)
- **What changed**: `spawnSubagentDirect()` now forwards the `senderIsOwner` flag into the `agent` gateway RPC call parameters. Previously, subagent runs launched via `sessions_spawn` did not propagate owner context, causing owner-only tools (e.g. tools gated on `tool-policy.ts` owner checks) to be unavailable inside subagent sessions even when spawned by an authorized owner.
- **Operational impact**: Owner-only tools now function correctly inside subagent sessions when the spawn request originates from an owner.

### subagent-announce: Improved Announce Format

- **File**: `src/agents/subagent-announce.ts` (319 lines)
- **What changed**: `buildCompletionDeliveryMessage()` distinguishes between `run` and `session` spawn modes in the header text (e.g. "session remains active" appended for `mode=session` spawns). `buildAnnounceReplyInstruction()` updated to include `expectsCompletionMessage` awareness, routing to the correct instruction variant depending on whether the requester is a subagent, has remaining active siblings, or expects a formatted user-facing completion message.
- **Operational impact**: Cleaner announce messages for both ephemeral run-mode subagents and persistent session-mode subagents.

---

## v2026.2.22 Changes (2026-02-23)

### Sessions
- **`session.dmScope` default** — `per-channel-peer` is now the default on new CLI installs.
- **Transcript path symlink resolution** — Symlinked state-dir aliases resolved during transcript-path validation.

### Providers
- **Mistral provider** — Embeddings and voice support added.
- **Grounded Gemini web search** — Available via `tools.webSearch.provider: "gemini"`.
- **Google Vertex AI** — Available as a routing target for Claude models.
- **OpenRouter cache_control** — `cache_control` injected on system prompts for Anthropic models.

### Agents
- **Moonshot** — `supportsDeveloperRole=false` forced.
- **Kimi context overflow** — Kimi token limit errors classified as context overflow.
- **Google thought_signature** — Non-base64 `thought_signature` sanitized from replay transcripts.
- **Mistral tool-call IDs** — Tool-call IDs sanitized.
- **Ollama integer args** — Large integer args preserved as exact strings.
- **Transcript validation** — Tool-call names validated before persistence.

### Gateway
- **Auth credential-source precedence** — Unified via shared resolver helpers across call/probe/status/auth entrypoints.
- **Pairing scope checks** — `operator.admin` satisfies `operator.*` scope checks; loopback scope-upgrade auto-approved; default scope bundles include `operator.read`/`operator.write`.

---

## v2026.2.23 Changes (2026-02-24)

### Sessions
- **Session key canonicalization** — Session keys canonicalized to lowercase; legacy case-variant entries migrated automatically. (`sessions/store.ts`)

### Providers
- **Kilocode provider** — First-class `kilocode` provider: auth, onboarding, implicit provider detection, model defaults (`kilocode/anthropic/claude-opus-4.6`), transcript/cache-ttl handling. (#20212)
- **Vercel AI Gateway** — Normalizes `vercel-ai-gateway/claude-*` shorthand refs to canonical Anthropic-routed IDs.
- **Anthropic OAuth tokens** — `sk-ant-oat-*` tokens skip `context-1m-*` beta injection.
- **OpenRouter reasoning_effort** — Conflicting top-level `reasoning_effort` removed when injecting `reasoning.effort`.
- **Groq TPM limit** — TPM limit errors no longer classified as context overflow.

### Agents
- **Auto-reasoning with thinking=low** — When `thinking=low` (model-default thinking), auto-reasoning stays disabled.
- **Reasoning-required errors** — No longer classified as context overflow.
- **Context overflow detection** — Detect additional error shapes + Chinese-language patterns.
- **HTTP 502/503/504 failover** — Treated as failover-eligible transient timeouts.

### Gateway
- **WS flood protection** — Repeated unauthorized request floods closed per-connection with sampled rejection logging.
- **Config write unsetPaths** — `unsetPaths` applied with immutable path-copy updates.
- **Prototype-key traversal** — Rejected in `config get/set/unset`.

---

## v2026.2.24 Changes (2026-02-25)

### Sessions
- **Tool-result guard** (#25429): synthetic `toolResult` entries are no longer generated for assistant turns that ended with `stopReason: "aborted"` or `"error"`, preventing orphaned tool-use IDs from triggering downstream API validation errors. Contributor: @mikaeldiakhate-cell.

### Providers
- **Reasoning override preservation** (#25314): explicit user `reasoning` overrides in config are preserved when merging provider model config with built-in catalog metadata, so `reasoning: false` is no longer silently overwritten by catalog defaults. Contributor: @lbo728.

### Agents
- **Tool dispatch** (#25427): block-reply flush is awaited before tool execution starts, preserving message ordering around tool calls. Contributor: @SidQin-cyber.

### Usage Accounting
- **Moonshot/Kimi cache metrics** (#25436): `cached_tokens` and `prompt_tokens_details.cached_tokens` fields from Moonshot/Kimi responses are now parsed into normalized cache-read usage metrics. Contributor: @Elarwei001.

---

## v2026.3.1 Changes (2026-03-02)

~588 commits across core engine modules in the v2026.3.1 release window. Key changes by sub-module:

### Agents
- **Subagent runtime events** — New `internal-events.ts` introduces typed `AgentInternalEvent` union (currently `task_completion` type) replacing ad-hoc system-message handoff for subagent and cron completion announces. Events carry structured fields (`source`, `childSessionKey`, `status`, `result`, `replyInstruction`) and are formatted via `formatAgentInternalEventsForPrompt()`. `AnnounceQueueItem` carries `internalEvents[]` through both queued and direct delivery paths. Cron completions skip the subagent status header in `buildCompletionDeliveryMessage()`.
- **Subagent Slack thread delivery fix** — `threadId` in `resolveSubagentCompletionOrigin()` no longer blindly copies `conversationId`; only explicit requester thread hints are preserved, fixing invalid `thread_ts` on Slack DM/top-level delivery. (`6a1eedf10`, #31105)
- **Thinking defaults** — Claude 4.6 defaults to `adaptive` thinking. `thinkingDefault` config now accepts `"adaptive"` level. Per-model thinking defaults take precedence over agent-level defaults. Session entry `thinkingLevel`/`verboseLevel`/`reasoningLevel` are checked before agent defaults in directive resolution. (`37d036714`, `0f2dce048`, `c9f0d6ac8`)
- **sessions_spawn delivery param rejection** — `sessions_spawn` tool rejects unsupported channel-delivery params (`target`, `transport`, `channel`, `to`, `threadId`, `replyTo`) with `ToolInputError`, directing callers to use `message` or `sessions_send` instead. New `sandbox` param with `"inherit"` | `"require"` modes. (`b0c7f1ebe`, #31000, #31110)
- **Model failover expansion** — `ECONNREFUSED`, `ENETUNREACH`, `EHOSTUNREACH`, `ENETRESET`, `EAI_AGAIN` classified as failover-worthy network errors alongside existing codes. (`76ed274aa`)
- **Failover reason classification** — `hasRateLimitTpmHint()` uses `\btpm\b` word-boundary regex instead of substring `includes("tpm")`, preventing false rate-limit classification from error messages that happen to contain `tpm` as a substring (e.g. `untpm`, `atpm`). Rate-limit error patterns updated to use regex for `tpm`.
- **Copilot token refresh** — Proactively refresh GitHub Copilot API tokens before expiry and retry on 401 auth errors during long-running embedded turns, preventing mid-session auth failures. (`2dcd2f909`)
- **Sessions list transcript paths** — `sessions-list-tool.ts` resolves `transcriptPath` per-agent using `resolveSessionFilePathOptions()` with `{agentId}` expansion and `~` home-dir expansion, instead of requiring a pre-resolved store path. (`53d6e07a6`, @martinfrancois)
- **Model overrides** — `applyModelOverrideToSessionEntry()` clears stale runtime `model`/`modelProvider` fields when overrides change, so status surfaces immediately reflect the selected model. Clears fallback notice fields on update.

### Gateway
- **Health probes** — `/health`, `/healthz` (liveness) and `/ready`, `/readyz` (readiness) HTTP endpoints added via `GATEWAY_PROBE_STATUS_BY_PATH` map and `handleGatewayProbeRequest()` in `server-http.ts` for container health checks (Kubernetes, Docker, Fly.io). (`eeb72097b`, #31272)
- **Control UI method guard** — POST requests to `/plugins/*` and `/api/*` paths fall through to their respective handlers instead of being caught by the SPA fallback, preventing untrusted plugins from claiming arbitrary UI paths.
- **WS security** — Origin allowlist in `origin-check.ts` upgraded from array `includes()` to `Set.has()`; wildcard `["*"]` in `gateway.controlUi.allowedOrigins` now explicitly accepted.
- **Control UI CSP** — `style-src` includes `https://fonts.googleapis.com` and `font-src` includes `https://fonts.gstatic.com` for Google Fonts support.
- **Control UI origins** — New `startup-control-ui-origins.ts` seeds `gateway.controlUi.allowedOrigins` for non-loopback installs upgrading to v2026.2.26+ at startup. Persists to config file on success.
- **macOS supervised restart** — `restartGatewayProcessWithFreshPid()` uses `launchctl kickstart -k` via `triggerOpenClawRestart()` on macOS under launchd (`OPENCLAW_LAUNCHD_LABEL` env) to bypass ThrottleInterval delays for intentional restarts. (`process-respawn.ts`)
- **macOS TLS certs** — `NODE_EXTRA_CA_CERTS` added to LaunchAgent environment for custom CA certificate support. (`d33f24c4e`)
- **Canvas auth refactor** — `authorizeCanvasRequest()`, `enforcePluginRouteGatewayAuth()`, and `isCanvasPath()` extracted from `server-http.ts` into `server/http-auth.ts`.
- **Node method scopes** — `node.canvas.capability.refresh` added to `NODE_ROLE_METHODS` in `method-scopes.ts`.

### Sessions
- **Internal routing** — `preserve external lastTo routing for internal turns`: internal turns no longer overwrite `lastTo`/`lastChannel` with internal values, so external delivery continues to target the correct channel. (`95db5bb5e`)

### Routing
- **Peer kind equivalence** — `peerKindMatches()` treats `group` and `channel` peer kinds as equivalent for binding scope matching, fixing bindings that targeted group peers but received channel-typed inbound messages (or vice versa). (`70ee256ae`, @Sid-Qin)
- **Multi-account default routing** — `channels.<channel>.defaultAccount` config key added for selecting a preferred account when multiple accounts are configured. (`41537e930`)
- **Inbound metadata** — `account_id` included in trusted inbound metadata context. (`0202d79df`, @Stxle2)

### Breaking Changes
- **Node exec approval payloads** — `systemRunPlanV2` renamed to `systemRunPlan`; `systemRunBindingV1` renamed to `systemRunBinding`. Old node clients sending `V2`/`V1`-suffixed payloads will fail approval matching. (`155118751`)
- **Node `system.run` pins to canonical `realpath`** — `resolveCommandResolution()` resolves `resolvedRealPath` via `fs.realpathSync()` for symlink-resistant command identity in approval bindings.

## v2026.3.7 Delta Notes

- **New `src/context-engine/` module** (5 files, now tracked in module inventory):
  - `types.ts` (167 lines) — ContextEngine lifecycle hook type definitions.
  - `registry.ts` (67 lines) — slot-based plugin registry for context engine plugins.
  - `legacy.ts` (115 lines) — LegacyContextEngine wrapper for backward compatibility.
  - `index.ts` (19 lines) — public API barrel export.
  - `init.ts` (23 lines) — initialization entry point.
- Compaction post-context configurability (PR #34556): compaction behavior after context window use is now configurable per-agent.
- Compaction safeguard pre-check additions: pre-compaction checks added to `pi-extensions/compaction-safeguard.ts` to block unsafe compaction conditions earlier.
- Bootstrap truncation warning handling: oversized bootstrap files now produce a structured warning rather than silently truncating context.
- Session startup date grounding: agent sessions inject the current date into the system prompt at startup for temporal grounding.
- Tool-result truncation with head+tail strategy: large tool results truncated using a head+tail window preserving both ends of output.
- Tool-result cleanup timeout hardening: cleanup jobs for tool results apply bounded timeouts preventing stalled cleanup from blocking shutdown.
- Failover overload vs rate-limit classification: attempt logic in `pi-embedded-runner/run/attempt.ts` now classifies provider overload vs rate-limit for smarter failover target selection.

---
