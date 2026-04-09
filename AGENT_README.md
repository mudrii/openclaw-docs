# OpenClaw Agent README

> Load this document before any PR, bug fix, refactor, or new feature work.
> Designed for AI agents and human contributors.
> This document **complements** `AGENTS.md` (the repo's canonical agent guidelines file, symlinked as `CLAUDE.md`). Load both before starting work. When build/test commands differ, `AGENTS.md` is authoritative.
> Tracks published OpenClaw releases. Current package version: check `package.json` (`"version"`). Gotchas are versioned — read only the sections that apply to the release you are targeting.
> **Current docs version: v2026.4.9-2 (2026-04-06).** Latest published upstream release: v2026.4.9 (published 2026-04-06 UTC).

---

## 0. Quick Start for Coding Agents

Use this routing first to avoid scanning the whole file.

| Task | Read in this order |
| --- | --- |
| Fix message delivery bug | §2 Message Lifecycle → §3 (`auto-reply/`, `channels/`) → §9 Telegram/Race gotchas → §5 checklist |
| Add or change config key | §6 config reference → §3 (`config/*`) → §5 checklist |
| Add a tool | §8 file placement → §2 Tool Execution → §3 (`agents/pi-tools.ts`) → §5 checklist |
| Change routing/session behavior | §2 Message Lifecycle → §3 (`routing/*`) → §9 session/race gotchas (#3, #17, #18, #20) |
| Fix session or memory bug | §2 Message Lifecycle → §3 (`sessions/`, `memory/`) → §9 gotchas #10 #12 #20 → §5 checklist |
| Add or modify a hook | §2 Hook Loading → §3 (`hooks/internal-hooks.ts`) → §8 new hook placement → §5 checklist |
| Add a new model or provider | §6 config (`authProfiles`, `models`) → §3 (`agents/model-selection.ts`) → §9 gotchas #59 #68 #71 #72 → §5 checklist |
| Fix a security issue | `SECURITY.md` → §1 blast radius → §5 checklist (run `openclaw security audit` before PR) |
| Add channel-specific behavior | §1 channel module risk → §2 Message Lifecycle → §3 channel impact rows → §5 checklist |
| Plugin/extension development | §2 Plugin Loading → §8 file placement (`extensions/`) → §5 checklist |
| Configure health probes (Docker/K8s) | §6 config (`gateway`) → §9 gotchas #96 → Gateway probe endpoints (`/health`, `/healthz`, `/ready`, `/readyz`) |
| Use PDF/diffs tool | §8 file placement (`extensions/`) → §2 Tool Execution → §3 (`agents/pi-tools.ts`) → §5 checklist |
| Fix task/flow lifecycle | §2 Task Flow Lifecycle → §3 (`src/tasks/`) → §9 gotchas #119 #126 #129 #130 → §5 checklist |
| Change provider transport/auth | §2 Tool Execution → §3 (`agents/model-selection.ts`) → §9 gotchas #131 #132 → §5 checklist |
| Fix config migration (xAI/Firecrawl) | §6 config → §9 gotchas #129 #130 → §5 checklist (`openclaw doctor --fix`) |
| Prepare PR | §4 testing → §5 checklist → §10 PR practices |
| Debug triage | §11 Debugging Workflow → §9 gotcha index → §2 critical paths |

Fast rule: identify module in §1, then run only the matching impact row in §3 plus relevant gotchas in §9.

---

## 1. Module Dependency Map

### Hierarchy (Level 0 = most depended upon)

| Level | Module        | Imports From                                                                    | Imported By                                | Risk        |
| ----- | ------------- | ------------------------------------------------------------------------------- | ------------------------------------------ | ----------- |
| 0     | `config/`     | `channels/registry`, `infra/`, `logging/`, `agents/defaults`                    | **Everything**                             | 🔴 CRITICAL |
| 0     | `shared/`     | `compat/`                                                                       | Many modules (pure types/constants)        | 🟡          |
| 1     | `infra/`      | `config/`, `logging/`, `process/`                                               | 400+ imports                               | 🔴 CRITICAL |
| 1     | `logging/`    | `config/`                                                                       | Most modules                               | 🟡          |
| 1     | `process/`    | `logging/`                                                                      | `infra/`, `agents/`, `hooks/`              | 🟡          |
| 2     | `channels/`   | `config/`, `infra/`, `plugins/`, `auto-reply/`                                  | `agents/`, `routing/`, `gateway/`          | 🔴 HIGH     |
| 2     | `routing/`    | `config/`, `sessions/`, `agents/agent-scope`, `channels/`                       | `gateway/`, `auto-reply/`, `cron/`, `web/` | 🔴 HIGH     |
| 2     | `sessions/`   | `config/`, `channels/`, `auto-reply/thinking`                                   | `routing/`, `gateway/`, `agents/`          | 🟡          |
| 3     | `agents/`     | `config/`, `routing/`, `sessions/`, `hooks/`, `channels/`, `infra/`, `plugins/` | 28+ modules                                | 🔴 CRITICAL |
| 4     | `auto-reply/` | `agents/`, `config/`, `channels/`, `routing/`, `tts/`, `media-understanding/`   | `gateway/`, channel monitors               | 🔴 HIGH     |
| 5     | `memory/`     | `config/`, `agents/`, `sessions/`, `logging/`                                   | `agents/tools/`, `gateway/`                | 🟡          |
| 5     | `cron/`       | `config/`, `agents/`, `routing/`, `infra/`, `auto-reply/`                       | `gateway/`, `agents/tools/`                | 🟡          |
| 5     | `hooks/`      | `config/`, `agents/`, `plugins/`, `markdown/`, `process/`                       | `gateway/`, `agents/`, `plugins/`          | 🟡          |
| 5     | `plugins/`    | `config/`, `agents/`, `channels/`, `hooks/`, `infra/`, `logging/`               | `gateway/`, `channels/`, `cli/`            | 🟡          |
| 5     | `tasks/`      | `config/`, `infra/`, `logging/`, `agents/`                                      | `gateway/`, `cron/`, `agents/tools/`, `cli/` | 🟡          |
| 5     | `security/`   | `agents/`, `browser/`, `channels/`, `config/`, `gateway/`, `infra/`, `plugins/` | `cli/`, `gateway/`, `acp/`                 | 🟢          |
| 5     | `browser/`    | `config/`, `infra/`                                                             | `agents/tools/`, `gateway/`, `cli/`        | 🟢          |
| 5     | `media/`      | `config/`, `infra/`                                                             | `agents/`, `auto-reply/`, channels         | 🟢          |
| 5     | `tts/`        | `agents/model-auth`                                                             | `auto-reply/`, `gateway/`                  | 🟢          |
| 6     | `gateway/`    | Almost everything                                                               | `cli/`, `commands/`, `daemon/`, `acp/`     | 🔴 HIGH     |
| 7     | `cli/`        | `config/`, `commands/`, `infra/`, `agents/`, `plugins/`                         | Package entry point                        | 🟡          |
| 7     | `commands/`   | `config/`, `cli/`, `infra/`, `agents/`, `channels/`, `daemon/`                  | `cli/`                                     | 🟡          |

Released channel implementations mostly live in `extensions/` (`telegram/`, `discord/`, `slack/`, `signal/`, `whatsapp/`, `imessage/`, `feishu/`, `matrix/`, `qqbot/`, etc.); `src/channels/`, `src/routing/`, `src/line/`, `src/tasks/`, `src/web-fetch/`, and `src/web-search/` remain the shared/core surfaces. Browser automation on the current release line lives in `extensions/browser/`. Channel/plugin implementations are leaf-heavy with 🟢 risk once you are below the shared routing/config layers.

**v2026.4.9 additions (current release line):**

- **Released memory runtime is split across `src/memory-host-sdk/` and `extensions/memory-core/src/memory/`:** dreaming, embeddings, host backends, and memory promotion live under `src/memory-host-sdk/`; QMD, search managers, and sync/index orchestration live under `extensions/memory-core/src/memory/`. If you touch memory, inspect both surfaces instead of older standalone `src/memory/` references from earlier releases.
- **Built-in media-generation tools:** `music_generate` and `video_generate` are now first-class tool surfaces, with supporting runtime code under `src/music-generation/`, `src/video-generation/`, `src/media-generation/`, and bundled Comfy workflow support in `extensions/comfy/`.
- **Provider/runtime expansion:** bundled Qwen, Fireworks AI, and StepFun providers are on the released line, alongside Bedrock Mantle discovery, shared model/media transport overrides, and forward-compat `openai-codex/gpt-5.4-mini`.
- **OpenAI/GPT reply-delivery changes:** commentary is buffered until `final_answer`, GPT-5/Codex defaults are shorter and lower-verbosity, and planning-only turns get a one-shot retry. If you touch reply streaming or block delivery, test that planning text does not leak to user-visible channels.
- **Structured plan updates:** compatible UIs can now receive structured plan/event progress from long-running runs. If you change run-event emission, verify both commentary and structured progress paths.
- **ACPX/runtime ownership:** the bundled ACPX plugin now embeds its runtime directly and uses the generic `reply_dispatch` hook to intercept reply delivery without hardcoded core ACP routing.
- **Approval UX surface expanded:** APNs modal approvals and Matrix-native exec approvals are now part of the released approval paths; approval changes must be tested beyond the older Web UI / Slack / Discord / Telegram paths.

**v2026.4.1 additions (historical):**

- **Chat-native `/tasks` board:** `/tasks` is now a first-class chat command exposing the background task board for the current session with agent-local fallback counts. Changes to task status, task-flow linkage, or `/status` rendering should test `/tasks` output alongside `openclaw tasks list|show|cancel`.
- **SearXNG web search:** a bundled SearXNG provider plugin is now part of the released `web_search` surface. If you touch `src/web-search/` or web search provider discovery, test SearXNG alongside existing providers (Brave, Perplexity, Pi/Codex native search).
- **Bedrock Guardrails:** the Amazon Bedrock extension now supports guardrail configuration. If you change Bedrock provider behavior or tool-use payloads, verify guardrail pass-through and rejection flows.
- **Voice Wake (macOS):** Voice Wake can now trigger Talk Mode. Changes to macOS voice or talk-mode paths should cover the wake-trigger entry point.
- **Feishu Drive comments:** Feishu now has a dedicated comment-event flow with thread context resolution and `feishu_drive` comment actions. If you touch Feishu inbound processing or plugin actions, test comment events alongside standard message events.
- **Exec approval persistence hardening:** `allow-always` is now durable trust (not one-shot), shell-wrapper paths reuse exact-command trust, static allowlist entries no longer bypass `ask:"always"`, and Windows builds an explicit approval plan. Slack and Discord have native approval routing. If you change exec approval logic, test persistence across restarts and cross-channel approval flows.
- **`agents.defaults.params`:** global default provider parameters are now configurable. If you change model selection or provider param injection, verify `agents.defaults.params` merging with per-agent and per-model overrides.
- **Cron `--tools` allowlist:** per-job tool allowlists via `openclaw cron --tools`. If you change cron job creation/update paths, test tool-filter persistence.

**v2026.3.31 additions (historical):**

- **Background task control plane:** the released line now uses `src/tasks/` as the shared SQLite-backed registry for ACP, subagents, cron, and detached CLI execution. If you touch task ownership, status, cleanup, or delivery, test both attached and detached flows.
- **`openclaw tasks` on the stable line:** operator-visible task control is now part of the released CLI surface. Changes in status/session reporting should be checked against `/status`, `openclaw tasks list`, `openclaw tasks show`, and detached-run delivery.
- **QQ Bot + expanded channel surface:** bundled QQ Bot is now part of the released tree, alongside stable-line updates for LINE outbound media + ACP binding, Matrix history/proxy/streaming/thread replies, Slack native exec approvals, Teams member-info, and WhatsApp reactions.
- **MCP remote transports:** `mcp.servers` now supports remote HTTP/SSE endpoints plus `streamable-http`, and bundled MCP tools use provider-safe names. If you touch MCP configs or tool registration, test both local stdio and remote transport paths.
- **Current browser/runtime layout:** browser automation is no longer a top-level `src/browser/` module on the released line. Inspect `extensions/browser/src/browser/*` and `extensions/browser/src/config/*` for current attach/session/runtime behavior.
- **Gateway trust hardening:** `trusted-proxy`, node pairing approval, node command gating, plugin-auth route scoping, and HTTP tool invoke auth all changed on this release line. Gateway/auth edits need explicit trust-boundary regression coverage.

**v2026.3.28 additions (historical):**

- **Gateway MCP Bridge** (`src/mcp/`): channel-server, channel-bridge, channel-tools. If you change `src/mcp/`, test Codex/Claude channel tool discovery and stdio lifecycle. Impact: `gateway/`, `plugins/`, any MCP client.
- **Plugin `requireApproval`** in `before_tool_call` hooks: async approval gate routing through exec overlay, Telegram, Discord, and `/approve`. If you change plugin hook `before_tool_call`, test `requireApproval` approval flow across exec overlay, Telegram, and Discord.
- **xAI plugin surface on Responses API:** `x_search` and `code_execution` are now plugin-owned; auto-enable from config. xAI provider now depends on Responses API transport.
- **CLI backend plugin surface:** Claude CLI, Codex CLI, and Gemini CLI inference on plugin architecture. If you change gateway startup, verify bundled plugin auto-load (`withBundledPluginAllowlistCompat`).
- **MiniMax `image-01`** generation provider in `extensions/minimax/`; catalog trimmed to M2.7 only.
- **Memory plugin flush plan contract:** `memory-core` owns flush prompts via `MemoryFlushPlanResolver`. If you change memory QMD chunking, test CJK and surrogate-pair handling.
- **Config/TTS auto-migration:** stale keys auto-migrated on startup. If you change `config/legacy-migrate.ts`, verify migrations older than 2 months are dropped.

**v2026.3.24 additions (historical):**

- **Gateway OpenAI compat:** `/v1/models`, `/v1/embeddings` endpoints, model override forwarding through `/v1/chat/completions` and `/v1/responses`
- **Microsoft Teams:** official Teams SDK migration with streaming 1:1 replies, welcome cards, prompt starters, feedback/reflection, typing indicators, native AI labeling (PR #51808). Message edit and delete support (PR #49925).
- **Skills:** one-click install recipes for bundled skills. `SkillInstallSpec` with kind brew/node/go/uv/download.
- **Control UI/skills:** status-filter tabs (All / Ready / Needs Setup / Disabled) with counts, click-to-detail dialog
- **Slack interactive replies:** restored rich parity, auto-render `Options:` lines as buttons/selects (PR #53389)
- **CLI `--container` / `OPENCLAW_CONTAINER`:** for Docker/Podman container commands (PR #52651)
- **Discord `autoThreadName: "generated"`:** for LLM-generated thread titles (PR #43366)
- **Plugin hook `before_dispatch`:** canonical inbound metadata and final-delivery routing (PR #50444)
- **Node 22.14 minimum floor** (lowered from 22.16). CLI update preflight node engine check.
- **Agents `/tools`:** now shows currently usable tools with "Available Right Now" UI section

**v2026.3.22-v2026.3.23 additions (historical):**

- **Chrome MCP existing-session only:** the released host-local attach path is `driver="existing-session"` with the built-in `user` profile; the legacy `chrome-relay` profile and `browser.relayBindHost` were removed in `v2026.3.22`.
- **Browser profile targeting:** `browser.profiles.<name>.userDataDir` now supports attaching Chrome MCP to Brave, Edge, Chromium, and non-default Chrome profiles.
- **ClawHub-first skills/plugins:** bare plugin installs prefer ClawHub, `openclaw skills search|install|update` is first-class, and `v2026.3.23` / `v2026.3.23-2` fix auth-path handling and runtime-version compatibility checks.
- **Plugin SDK migration:** current released plugin guidance is `openclaw/plugin-sdk/<subpath>`; `openclaw/extension-api` is deprecated compatibility only.
- **Gateway auth tightening:** canvas routes now require auth, and agent session reset now requires admin scope.
- **Cron timezone fix:** one-shot `openclaw cron add|edit --at ... --tz <iana>` now honors requested local wall-clock time for offset-less datetimes, including DST boundaries.
- **Packaged runtime sidecars restored:** bundled plugin runtime sidecars are present again in published npm installs after the `v2026.3.23` packaging fixes.

**v2026.2.21 additions (historical):**

| Level | Module | Imports From | Imported By | Risk |
| ----- | ------ | ------------ | ----------- | ---- |
| leaf | `discord/voice/` | `discord/`, `config/`, `channels/` | — | 🟢 |
| 2 | `channels/status-reactions` | `channels/`, `config/`, `infra/` | `telegram/`, `discord/` | 🟡 |
| leaf | `discord/monitor/thread-bindings.*` | `discord/`, `sessions/`, `config/` | — | 🟢 |
| 2 | `node-host/invoke-system-run` | `node-host/`, `config/`, `process/` | `node-host/invoke.ts` | 🟡 |

**v2026.3.2 additions (behavioral):**

- No new top-level modules were added in this release window; feature and security changes are in existing modules, including `cli/`, `channels/`, `agents/`, `gateway/`, `security/`, `plugins/`, and `extensions/`.

**v2026.3.13 additions (historical runtime/ops):**

- **Chrome DevTools MCP attach mode:** `v2026.3.13` introduced live signed-in Chrome attach flows that were later simplified in `v2026.3.22` to the current released `existing-session` / `user` path. Current released files to inspect are `src/browser/config.ts`, `src/browser/chrome-mcp.ts`, `src/browser/profile-capabilities.ts`, and `src/browser/server-context.availability.ts`.
- **`sessions_yield` safeguards:** compaction token sanity check validates against full pre-compaction totals; follow-up payload routing hardened. Key file: `src/agents/tools/sessions-yield-tool.ts`.
- **Compaction language continuity:** `agents.defaults.compaction.customInstructions` allows per-agent instructions to preserve language/style across compaction resets. Key file: `src/agents/pi-extensions/compaction-instructions.ts`.
- **Memory bootstrap preference:** `MEMORY.md` is now preferred over `memory.md` during workspace bootstrap. No config change needed.
- **Nested cron lane routing:** nested cron jobs route through their own lane to prevent cross-lane deadlocks.
- **Security — 6 exec hardening items:** pnpm, Perl `-M`/`-I`, PowerShell `-File`/`-f`, env wrappers, macOS line continuation, skill auto-allow all hardened; single-use bootstrap pairing codes enforced; external content ZWS stripping added.

**v2026.3.12 additions (runtime/ops):**

- **`sessions_yield` tool (new):** cooperative turn-ending primitive. Calling `sessions_yield` ends the turn immediately, skips queued tool work, and optionally carries a hidden follow-up payload. Key file: `src/agents/tools/sessions-yield-tool.ts`.
- **Provider plugins for Ollama/vLLM/SGLang:** these providers are now managed as plugins (`extensions/ollama/`, `extensions/vllm/`, `extensions/sglang/`). No behavioral change for existing configs.
- **`/fast` mode:** `service_tier` fast mode for OpenAI and Anthropic; per-model config defaults; TUI/ACP support; cron runs can use fast mode.
- **Node 24 default; Node 22.14 minimum:** upgrade runtime before upgrading OpenClaw on hosts below Node 22.14.
- **Security — 20+ GHSAs:** see CHANGELOG.md v2026.3.12 section for full list. Exec allowlist hardening (GHSA-pcqg, GHSA-f8r2, GHSA-57jw, GHSA-jvqh, GHSA-x7pp, GHSA-jc5j) is the most impactful for agent tooling workflows.

**v2026.3.11 additions (runtime/ops):**

- **Cron isolation (BREAKING):** cron jobs can no longer notify via ad hoc agent sends or fallback main-session summaries. Run `openclaw doctor --fix` to migrate legacy cron storage and delivery metadata.
- **Memory/multimodal:** opt-in multimodal image and audio indexing is now available for `memorySearch.extraPaths` using `gemini-embedding-2-preview`. Configurable output dimensions; reindexing triggers automatically on dimension changes.
- **ACP session resume:** `sessions_spawn` with `runtime: "acp"` now accepts `resumeSessionId` to resume an existing ACPX/Codex conversation. `main` alias is now canonicalized so restarted ACP main sessions rehydrate instead of failing.
- **macOS/launchd restart v2:** explicit restarts keep the LaunchAgent registered and hand off the actual restart through a detached launchd helper; config/hot reload paths recovered without unloading the service.
- **Node pending-work queue:** `node.pending.enqueue` / `node.pending.drain` primitives added for dormant-node work delivery.
- **Gateway runtime version:** `gateway status` now exposes the running server version.
- **Security — GHSA-5wcw-8jjv-m286:** browser origin validation now enforced for all browser-originated connections regardless of proxy headers; closes cross-site WebSocket hijack path in `trusted-proxy` mode.
- Additional security hardening: symlink-safe `*File` secret reads, TAR/bz2 extraction staging, exec SecretRef traversal rejection, fs-bridge staged-write pinning, gateway auth fail-closed on missing SecretRefs, `/new`/`/reset` split from admin RPC, plugin HTTP route scope isolation, `session_status` sandbox guards, `nodes` tool owner-only policy.

**v2026.3.8 additions (runtime/ops):**

- `openclaw backup create|verify` is now part of the top-level CLI. Use it before destructive flows or restart/debugging work that may touch local state.
- launchd-supervised restart no longer depends on in-process `launchctl kickstart -k`; the process exits and relies on supervisor restart, with `XPC_SERVICE_NAME` now treated as a supervision marker.
- `talk.silenceTimeoutMs` is a top-level runtime knob with current defaults of `700 ms` on macOS/Android and `900 ms` on iOS.
- ACP provenance now carries `originSessionId` and optional receipt text; spawned ACP child sessions persist lineage/transcript metadata more reliably.

**v2026.3.7 additions (new module):**

| Level | Module | Imports From | Imported By | Risk |
| ----- | ------ | ------------ | ----------- | ---- |
| 3 | `context-engine/` | `config/`, `agents/` | `agents/pi-embedded-runner/`, `gateway/` | 🟡 LOW-MEDIUM |

- `context-engine/` is a new plugin slot for custom context management strategies. Key files: `types.ts`, `registry.ts`, `legacy.ts`, `index.ts`, `init.ts`. Lifecycle hooks: `bootstrap`, `ingest`, `assemble`, `compact`, `afterTurn`, `prepareSubagentSpawn`, `onSubagentEnded`. When you change this module, test compaction behavior and any plugin implementing the `ContextEnginePlugin` interface.

### High-Blast-Radius Files

| File                                | What It Exports                                                    |
| ----------------------------------- | ------------------------------------------------------------------ |
| `config/config.ts`                  | `loadConfig()`, `OpenClawConfig`, `clearConfigCache()`             |
| `config/types.ts`                   | All config type re-exports                                         |
| `config/paths.ts`                   | `resolveStateDir()`, `resolveConfigPath()`, `resolveGatewayPort()` |
| `config/sessions.ts`                | Session store CRUD                                                 |
| `infra/errors.ts`                   | Error utilities                                                    |
| `infra/json-file.ts`                | `loadJsonFile()`, `saveJsonFile()`                                 |
| `agents/agent-scope.ts`             | `resolveDefaultAgentId()`, `resolveAgentWorkspaceDir()`            |
| `agents/defaults.ts`                | `DEFAULT_PROVIDER`, `DEFAULT_MODEL`                                |
| `channels/registry.ts`              | `CHAT_CHANNEL_ORDER`, `normalizeChatChannelId()`                   |
| `routing/session-key.ts`            | `buildAgentPeerSessionKey()`, `normalizeAgentId()`                 |
| `auto-reply/templating.ts`          | `MsgContext`, `TemplateContext` types                              |
| `auto-reply/thinking.ts`            | `ThinkLevel`, `VerboseLevel`, `normalizeVerboseLevel()`            |
| `logging/subsystem.ts`              | `createSubsystemLogger()`                                          |
| `agents/subagent-depth.ts`          | Nested subagent depth tracking and orchestration controls          |
| `agents/subagent-announce-queue.ts` | Subagent result announcement queueing                              |
| `discord/components.ts`             | Discord Component v2 UI rendering and registry                     |
| `infra/install-safe-path.ts`        | Restricted skill download target path validation                   |
| `pairing/pairing-store.ts`          | Account-scoped device pairing store                                |
| `channels/status-reactions.ts`      | Shared lifecycle reaction controller (used by Telegram and Discord) |
| `node-host/invoke-system-run.ts`    | system.run command resolution (security-critical)                  |
| `process/exec.ts`                   | `runCommandWithTimeout()` — used by every tool execution path      |
| `agents/model-selection.ts`         | Model resolution across all agents, directives, cron, subagents    |
| `agents/exec-approval-result.ts`   | Exec approval result types — used by approval pipeline and channels |
| `agents/internal-runtime-context.ts` | Internal agent runtime context — used by embedded runner and subagents |
| `tasks/task-flow-registry.ts`       | Task Flow registry — durable flow state, sync, lifecycle for all task orchestration |
| `tasks/task-flow-registry.store.sqlite.ts` | Task Flow SQLite persistence — backing store for durable flow state |
| `web-fetch/runtime.ts`             | Web fetch provider routing — shared fetch-provider boundary         |

### Risk Level Definitions

| Level | Meaning | Required Validation |
| --- | --- | --- |
| 🔴 CRITICAL | Cross-cutting core modules; failures can break startup/routing or multiple channels | Full test suite + impacted subsystem tests |
| 🔴 HIGH | Multi-subsystem module with large blast radius | Targeted subsystem tests + pre-PR checklist |
| 🟡 MEDIUM | Shared module with moderate import fan-out | Module tests + caller verification |
| 🟢 LOW | Leaf/isolated module | Local tests + smoke checks |

---

## 2. Critical Paths (Don't Break These)

### Message Lifecycle: Inbound → Response

```
Channel SDK event (grammY/Carbon/Bolt/SSE/RPC)
→ extensions/<channel>/bot-handlers.ts or monitor/*.ts
→ extensions/<channel>/bot-message-context.ts   # normalize to MsgContext
→ src/auto-reply/dispatch.ts                   # dispatchInboundMessage()
→ src/routing/resolve-route.ts                 # resolveAgentRoute() → {agentId, sessionKey}
→ src/auto-reply/reply/dispatch-from-config.ts # dispatchReplyFromConfig()
→ src/auto-reply/reply/get-reply.ts            # getReplyFromConfig() - MAIN ORCHESTRATOR
  ├─ media-understanding/apply.ts
  ├─ command-auth.ts
  ├─ reply/session.ts                          # initSessionState()
  ├─ reply/get-reply-directives.ts             # parse /model, /think, etc.
  ├─ reply/get-reply-inline-actions.ts         # handle /new, /status, etc.
  └─ reply/get-reply-run.ts                    # runPreparedReply()
→ src/auto-reply/reply/agent-runner.ts         # runReplyAgent()
→ src/auto-reply/reply/agent-runner-execution.ts   # runAgentTurnWithFallback()
→ src/agents/pi-embedded-runner/run.ts             # runEmbeddedPiAgent()
  ├─ pi-embedded-runner/system-prompt.ts           # build system prompt
  ├─ agents/model-selection.ts                     # resolve model + auth
  ├─ agents/pi-tools.ts                            # register tools
  ├─ pi-embedded-runner/run/attempt.ts             # single attempt execution
  ├─ pi-embedded-runner/run/payloads.ts            # build LLM payloads
  └─ agents/pi-embedded-subscribe.handlers.tools.ts # process LLM stream + tool calls
→ src/auto-reply/reply/block-reply-pipeline.ts # coalesce blocks
→ src/auto-reply/reply/reply-dispatcher.ts     # buffer + human delay
→ src/channels/plugins/outbound/<channel>.ts   # format + chunk
→ src/<channel>/send.ts                        # API call
```

### Tool Execution: Tool Call → Result

```
LLM stream → agents/pi-embedded-subscribe.handlers.tools.ts  # extract tool call
→ agents/pi-tools.ts                                          # registry lookup
→ agents/pi-tools.before-tool-call.ts                         # pre-call hooks
→ agents/tool-policy-pipeline.ts                              # allow/deny/ask
  └─ agents/tool-policy.ts + pi-tools.policy.ts
→ Tool implementation (e.g. agents/bash-tools.exec.ts)
  └─ process/exec.ts → runCommandWithTimeout()
  └─ gateway/exec-approval-manager.ts (if approval needed)
→ agents/pi-embedded-runner/tool-result-truncation.ts         # truncate if large
→ Result returned to LLM stream → continues generation
```

### Config Loading: JSON → Validated Object

```
config/paths.ts → resolveConfigPath()
→ config/io.ts → readFileSync (JSON5)
→ config/includes.ts → resolve $include directives
→ config/env-substitution.ts → expand ${ENV_VAR}
→ config/validation.ts → Zod parse against OpenClawSchema
→ config/legacy-migrate.ts → auto-migrate old config
→ config/defaults.ts → apply*Defaults() (models, agents, sessions, logging, compaction, pruning)
→ config/runtime-overrides.ts → env var overrides
→ config/normalize-paths.ts → normalize paths
→ Cache in memory (clearConfigCache() to invalidate)
```

### Hook Loading: Boot → Registered

```
gateway/server-startup.ts → loadInternalHooks()
→ hooks/workspace.ts → loadWorkspaceHookEntries()
  └─ Scan: extraDirs → bundled → managed → workspace (later overrides)
  └─ hooks/frontmatter.ts → parse HOOK.md metadata
  └─ hooks/config.ts → shouldIncludeHook() (OS/binary/config checks)
→ hooks/loader.ts → dynamic import() with buildImportUrl()
→ hooks/internal-hooks.ts → registerInternalHook()
```

### Plugin Loading: Discovery → Registered

```
gateway/server-plugins.ts
→ plugins/discovery.ts → discoverOpenClawPlugins()
  └─ Scan: config paths → workspace .openclaw/extensions → bundled extensions/ → ~/.openclaw/extensions
→ plugins/manifest.ts → parse openclaw.plugin.json
→ plugins/loader.ts → loadOpenClawPlugins()
  └─ jiti dynamic import for each plugin module
  └─ Plugin exports definePlugin() → {tools, hooks, channels, providers, httpRoutes, services}
→ plugins/registry.ts → PluginRegistry singleton
→ plugins/runtime.ts → setActivePluginRegistry()
```

### Task Flow Lifecycle: Create → Sync → Complete/Cancel

```
Plugin or agent tool → api.runtime.taskFlow (bound seam)
→ tasks/task-flow-registry.ts             # createFlow(), syncFlow(), cancelFlow()
  ├─ task-flow-registry.types.ts          # FlowState, SyncMode (managed|mirrored)
  ├─ task-flow-registry.store.ts          # store abstraction
  ├─ task-flow-registry.store.sqlite.ts   # SQLite durable persistence
  ├─ task-flow-registry.paths.ts          # flow state directory resolution
  ├─ task-flow-registry.audit.ts          # audit trail for flow transitions
  └─ task-flow-registry.maintenance.ts    # async sweep, stale flow cleanup
→ tasks/task-flow-runtime-internal.ts     # internal runtime wiring
→ gateway/ or agents/tools/               # flow status exposed via CLI and /tasks
→ openclaw tasks flow list|show|cancel    # operator inspection/recovery
```

### Provider Transport: Auth → Request → Response

```
agents/model-selection.ts → resolve model + auth profile
→ providers/transport-policy.ts           # centralized request policy
  ├─ HTTP path: shared request auth, proxy, TLS, headers
  ├─ Stream path: SSE streaming with attribution headers
  └─ WebSocket path: WS streaming with auth + proxy
→ providers/endpoint-resolver.ts          # native-vs-proxy classification
→ providers/<provider>/request.ts         # provider-specific request shaping
→ Response processing (streaming or buffered)
```

---

## 3. Change Impact Matrix

| If You Change...                 | You MUST Also Check/Test...                                                                                                                    |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `config/zod-schema*.ts`          | ALL config validation tests, `config/defaults.ts`, JSON schema generation (`config/schema.ts`), every module that reads the changed config key |
| `config/types*.ts`               | Every file that imports the changed type (grep!), Zod schema must match                                                                        |
| `config/io.ts`                   | Config loading, `$include`, env substitution, backup rotation, migration                                                                       |
| `config/sessions/store.ts`       | Session CRUD in gateway, agent runner, directive handling                                                                                      |
| `routing/resolve-route.ts`       | All channel monitors, gateway session resolution, cron delivery                                                                                |
| `routing/session-key.ts`         | Session key parsing everywhere, cron sessions, subagent sessions                                                                               |
| `agents/pi-tools.ts`             | ALL tool tests, tool policy, tool display, sandbox tool policy                                                                                 |
| `agents/pi-embedded-runner/run.ts` + `run/` | The entire agent execution path, fallback, compaction, streaming                                                                     |
| `agents/pi-embedded-runner/system-prompt.ts` | Agent behavior changes - test with actual LLM calls                                                                                  |
| `agents/model-selection.ts`      | Model resolution across all agents, directives, cron, subagents                                                                                |
| `agents/tool-policy*.ts`         | Tool access for all tools, sandbox, subagent restrictions                                                                                      |
| `auto-reply/dispatch.ts`         | All channel inbound paths                                                                                                                      |
| `auto-reply/reply/get-reply.ts`  | The entire reply pipeline - most impactful single file                                                                                         |
| `auto-reply/templating.ts`       | `MsgContext` type used by 15+ files                                                                                                            |
| `auto-reply/thinking.ts`         | `ThinkLevel`/`VerboseLevel` used across agents, directives, sessions                                                                           |
| `channels/plugins/types*.ts`     | ALL channel implementations, plugin SDK                                                                                                        |
| `channels/registry.ts`           | Channel normalization, routing, dock, all channel references                                                                                   |
| `gateway/server.impl.ts`         | Gateway startup, all server subsystems                                                                                                         |
| `gateway/protocol/schema/*.ts`   | WS protocol compat, CLI client, TUI                                                                                                            |
| `hooks/internal-hooks.ts`        | All hook handlers, gateway startup, session-memory hook                                                                                        |
| `plugins/loader.ts`              | ALL plugins, gateway startup                                                                                                                   |
| `infra/json-file.ts`             | Cron store, auth profiles, session store, device auth                                                                                          |
| `security/external-content.ts`   | Prompt injection defense, web_fetch, link understanding                                                                                        |
| `logging/subsystem.ts`           | Every module that creates loggers                                                                                                              |
| Any `index.ts` barrel            | All consumers of that module's exports                                                                                                         |
| `context-engine/registry.ts` or `context-engine/types.ts` | All plugins implementing `ContextEnginePlugin`, compaction behavior, subagent spawn/end hooks |
| Plugin hook `before_tool_call` | `requireApproval` approval flow across exec overlay, Telegram, Discord; verify `/approve` command routing |
| Gateway startup / plugin allowlist | Bundled plugin auto-load (`withBundledPluginAllowlistCompat`); verify Claude CLI, Codex CLI, Gemini CLI extensions load correctly |
| Memory QMD chunking | CJK boundary handling and surrogate-pair splitting; slugified path resolution for non-ASCII file names |
| `src/mcp/` (channel-server, channel-bridge, channel-tools) | Codex/Claude channel tool discovery; stdio lifecycle; test with a live MCP client connected to gateway |
| `src/web-search/` or web search provider plugins | SearXNG, Brave, Perplexity, Pi/Codex native search; verify provider discovery and fallback across all bundled providers |
| `extensions/amazon-bedrock/` guardrail config | Bedrock Guardrails pass-through and rejection flows; tool-use payloads with guardrail IDs |
| `src/tasks/` task-flow linkage or status rendering | `/tasks` chat command, `openclaw tasks list\|show\|cancel`, `/status` task cards, detached-run delivery, task-flow blocked/retry state |
| Exec approval persistence or `exec-approvals.json` | `allow-always` durability across restarts, shell-wrapper exact-command trust, Slack/Discord native approval routing, Windows approval plan generation |
| `agents.defaults.params` or provider param injection | Global default params merging with per-agent, per-model, and inline directive overrides |
| `src/tasks/task-flow-registry*` | Task Flow lifecycle (create, sync, cancel, maintenance sweep); `openclaw tasks flow` CLI output; `/tasks` chat command; durable flow state persistence; managed child task spawning |
| `src/tasks/task-flow-runtime-internal.ts` | Plugin `api.runtime.taskFlow` seam; managed Task Flow creation from plugins and authoring layers |
| Provider transport policy or auth/header injection | All three transport modes (HTTP, stream, websocket); native-vs-proxy endpoint classification; attribution headers; mTLS settings; proxy-hop TLS isolation |
| `src/web-fetch/runtime.ts` | Web fetch provider routing; Firecrawl and non-Firecrawl fallback paths; plugin-owned fetch provider boundary |
| `agents/exec-approval-result.ts` | Exec approval pipeline result types; channel-specific approval routing (Telegram, Slack, Discord) |
| `agents/internal-runtime-context.ts` | Agent embedded runner; subagent context propagation; runtime context threading |
| Plugin hook `before_agent_reply` | Synthetic reply short-circuit after inline actions; verify alongside `before_tool_call` and `before_dispatch` hooks |
| xAI `x_search` or Firecrawl `web_fetch` config paths | Plugin-owned config paths (`plugins.entries.xai.config.xSearch.*`, `plugins.entries.firecrawl.config.webFetch.*`); `openclaw doctor --fix` migration; legacy core config path removal |
| Exec defaults (`security`, `ask`, host approval) | YOLO mode (`security=full`, `ask=off`); cross-platform approval-file fallbacks; Windows and node host exec; doctor reporting |

### Cross-Module Side Effects (Non-Obvious)

- **`agents/` ↔ `auto-reply/`**: Bidirectional dependency by design. Changes to agent run result types break reply delivery.
- **`config/sessions/` → `routing/` → `gateway/`**: Session key format changes ripple to route resolution and gateway session management.
- **`channels/dock.ts`**: Returns channel metadata without importing heavy channel code. If you change a channel's capabilities, update the dock too.
- **`auto-reply/thinking.ts`**: `VerboseLevel` is used by `sessions/level-overrides.ts` - changing enum values breaks session persistence.
- **`infra/outbound/deliver.ts`**: Used by both cron delivery AND channel tool message sending. Changes affect both paths.
- **`tasks/task-flow-registry.*` → `gateway/` → `agents/tools/` → `cli/`**: Task Flow state changes ripple through gateway task status, agent tool task discovery, CLI `openclaw tasks flow`, and `/tasks` chat command rendering.
- **Provider transport → `agents/model-selection.ts` → all provider request paths**: Transport policy centralization means auth, proxy, and TLS changes affect HTTP, stream, and websocket paths simultaneously. Test all three.

---

## 4. Testing Guide

### Running Tests

```bash
# Full suite (parallel runner, matches CI)
pnpm test

# Single module (targeted vitest run)
pnpm exec vitest run src/config/
# or: bunx vitest run src/config/

# Single file
pnpm exec vitest run src/config/io.write-config.test.ts

# Watch mode
pnpm exec vitest src/config/io.write-config.test.ts

# With coverage (uses vitest.unit.config.ts)
pnpm test:coverage

# E2E tests only
pnpm test:e2e

# Memory-constrained hosts (non-Mac-Studio, CI stragglers)
OPENCLAW_TEST_PROFILE=low OPENCLAW_TEST_SERIAL_GATEWAY=1 pnpm test

# Live tests (requires real API keys)
CLAWDBOT_LIVE_TEST=1 pnpm test:live
```

### Test Config Files

| Config | Used for |
| --- | --- |
| `vitest.unit.config.ts` | Unit tests (`pnpm test:coverage`) |
| `vitest.e2e.config.ts` | End-to-end tests (`pnpm test:e2e`) |
| `vitest.gateway.config.ts` | Gateway integration tests |
| `vitest.live.config.ts` | Live tests against real APIs |
| `vitest.extensions.config.ts` | Extension/plugin tests |

`pnpm test` runs `scripts/test-parallel.mjs`, which spawns multiple vitest processes across these configs.

### Test Framework: Vitest

- Linter/formatter: Oxlint + Oxfmt (`pnpm check` = format + tsgo + oxlint + custom boundary lints)
- Mocking: `vi.mock()`, `vi.fn()`, `vi.spyOn()`
- Assertions: `expect()` with Vitest matchers
- Coverage threshold: 70% lines/branches/functions/statements (V8)
- **Do not set test workers above 16** — causes memory issues; already tried.
- Pre-commit hooks: `prek install` — runs same checks as CI before each commit.

### Test Patterns

| Pattern      | Example                                |
| ------------ | -------------------------------------- |
| Unit test    | `src/config/io.write-config.test.ts`                |
| E2E test     | `src/agents/bash-tools.exec.pty.e2e.test.ts` |
| Smoke test   | `src/cli/program.smoke.test.ts`    |
| Test harness | `src/cron/service.test-harness.ts`     |
| Test helpers | `src/test-helpers/`, `src/test-utils/` |
| Mock file    | `src/cron/isolated-agent.mocks.ts`     |

### Test Helpers

- `src/test-helpers/` - Shared test utilities
- `src/test-utils/` - Additional test utilities
- `src/config/test-helpers.ts` - Config-specific test helpers
- `src/cron/service.test-harness.ts` - Cron service test fixture
- `src/cron/isolated-agent.test-harness.ts` - Isolated agent test fixture
- `extensions/memory-core/src/memory/embedding-manager.test-harness.ts` - Embedding test fixture

### CI Pipeline

- **Fail-fast scoping gates run first**: `docs-scope` and `changed-scope` determine whether heavy Node/macOS/Android jobs run.
- **Docs-only PRs skip heavy lanes** (`check`, `checks`, `checks-windows`, `macos`, `android`) and run `check-docs` instead.
- **Node quality gate**: `pnpm check` remains the required type+lint+format gate.
- **Node test lane**: `checks` matrix runs Node tests, Bun tests, and `pnpm protocol:check`.
- **Build artifact reuse**: `build-artifacts` builds `dist/` once and downstream jobs (including release checks / Windows lint lane) consume it.
- **Windows lane**: runs `pnpm lint`, `pnpm test`, and `pnpm protocol:check` with constrained worker settings.
- **macOS lane**: consolidated single job runs TS tests, Swift lint/format, Swift build, and Swift tests sequentially.
- **Android lane**: runs Gradle unit tests and debug build when Android/shared paths change.
- **Push-to-main only**: `release-check` validates packed release contents.

---

## 5. Pre-PR Checklist

> Canonical command source: `CONTRIBUTING.md` + `.github/workflows/ci.yml`. If any checklist command diverges from those, update this document immediately.

```
□ pnpm build                        # TypeScript compilation (always)
□ pnpm check                        # Format + type check + lint (always)
□ pnpm test                         # Required by policy unless docs-only criteria pass
□ pnpm check:docs                   # Required when docs files changed
□ CI checks green                   # Required before merge
□ Branch up-to-date with main       # Required before merge
□ CHANGELOG.md update               # User-facing changes only; no internal/meta notes
                                     # Include `(#<PR>)` suffix + `thanks @<pr-author>` when available
                                     # Pure test additions/fixes do NOT need a CHANGELOG entry
□ git diff --stat                   # Review staged scope
□ grep all callers                  # If changing exported signatures
□ Squash fix-on-fix commits         # Keep logical commits only
□ Use scripts/committer "<msg>" <files>  # Do NOT use manual git add/commit — keeps staging scoped

Conditional checks:
□ If `config/*` changed: run config-focused tests + read-back validation of changed keys
□ If `routing/*` changed: verify session key parsing + cron + subagent routing
□ If `agents/pi-tools*` changed: run tool policy + tool execution paths
□ If `auto-reply/reply/get-reply.ts` changed: run reply pipeline checks across fallback paths
□ If `src/tasks/task-flow-registry*` changed: test task lifecycle (create, sync, cancel, maintenance sweep) + `openclaw tasks flow` CLI
□ If provider transport/auth changed: test HTTP, stream, and websocket paths + native-vs-proxy classification
□ If `gateway/protocol/schema/**` changed: run pnpm protocol:gen:swift && pnpm protocol:check
```

### Fix Commands (when pnpm check fails)

```bash
pnpm format:fix          # oxfmt --write — auto-fix formatting
pnpm lint:fix            # oxlint --fix + format — auto-fix lint + format
```

### Release-window Workflow Additions (v2026.4.9)

- **Treat released memory as a split surface:** if you change dreaming, embeddings, or promotion logic, start in `src/memory-host-sdk/`; if you change QMD, search, or memory sync/indexing, start in `extensions/memory-core/src/memory/`. Older `src/memory/` references in historical notes are no longer the current inspection path.
- **Media-generation changes span multiple modules:** `music_generate` and `video_generate` touch agent tools, provider/runtime policy, async completion, and outbound media delivery. Test generation, async completion, and final delivery together.
- **OpenAI reply delivery must hide planning text:** changes in reply streaming, block delivery, or `/v1/responses` compatibility need explicit checks that commentary stays buffered until `final_answer`.
- **`contextVisibility` changes channel context semantics:** if you touch channel context assembly or quoted/thread history paths, test all `contextVisibility` modes (`all`, `allowlist`, `allowlist_quote`) with both allowed and disallowed senders.
- **Approval flows are broader than before:** APNs modal approvals and Matrix-native approvals are on the current line. If you change approval state or exec approval delivery, include iOS reconnect and Matrix room/thread resolution in validation.
- **Bundled CLI backend assumptions changed:** the old bundled CLI text-provider surface is gone. If you touch CLI backend routing or onboarding, validate Claude CLI legacy-profile migration and ensure native bundled providers still work without the removed `agents.defaults.cliBackends` surface.

### Release-window Workflow Additions (v2026.4.2)

- **Task Flow registry is a durable substrate:** `src/tasks/task-flow-registry.*` now manages persistent flow state with managed-vs-mirrored sync modes. If you touch `src/tasks/`, test task lifecycle (create, sync, cancel, maintenance sweep), `openclaw tasks flow` CLI output, and `/tasks` chat rendering.
- **Provider transport is centralized:** request auth, proxy, TLS, and header shaping are shared across HTTP, stream, and websocket paths. If you change provider transport or request policy, test all three transport modes plus native-vs-proxy endpoint classification.
- **Exec defaults are YOLO by default:** gateway/node host exec now defaults to `security=full` with `ask=off`. If you change exec defaults or approval behavior, test cross-platform (Windows, node host) and verify doctor reporting.
- **`before_agent_reply` hook exists:** plugins can short-circuit LLM replies with synthetic responses. If you change plugin hook dispatch or the reply pipeline, verify `before_agent_reply` alongside other hooks.
- **xAI `x_search` config migrated to plugin-owned path:** `tools.web.x_search.*` is now `plugins.entries.xai.config.xSearch.*`. Test `openclaw doctor --fix` migration and verify xAI plugin auto-enable from config.
- **Firecrawl `web_fetch` config migrated to plugin-owned path:** `tools.web.fetch.firecrawl.*` is now `plugins.entries.firecrawl.config.webFetch.*`. Test `openclaw doctor --fix` migration and verify web fetch provider routing.
- **`web-fetch/runtime.ts` routes through provider boundary:** web fetch fallback is no longer Firecrawl-only. If you touch `src/web-fetch/`, test Firecrawl and non-Firecrawl provider paths.

### Release-window Workflow Additions (v2026.4.1)

- **`/tasks` is a chat-native surface:** the `/tasks` command now shows a session task board in chat. If you change task status, task-flow linkage, or `/status` rendering, verify `/tasks` output alongside CLI `openclaw tasks list|show|cancel`.
- **SearXNG is a bundled web search provider:** web search provider discovery must now test SearXNG alongside Brave, Perplexity, and native search. Config key: `webSearch.searxng.host`.
- **Bedrock Guardrails are first-class:** if you change the Amazon Bedrock provider extension, test with and without guardrail config. Guardrail IDs flow through tool-use payloads.
- **Exec approval `allow-always` is now durable:** previously behaved like `allow-once`. After this release, `allow-always` persists across restarts. Shell-wrapper paths reuse exact-command trust. Slack and Discord have native approval routing. Static allowlist entries no longer silently bypass `ask:"always"`.
- **`agents.defaults.params` merges globally:** global default provider params now apply unless overridden per-agent or per-model. Test param precedence when changing model selection or provider param injection.
- **Cron `--tools` per-job allowlists:** cron jobs can now restrict available tools. Test tool-filter persistence on `openclaw cron add|edit --tools`.
- **Feishu Drive comment events:** Feishu inbound now includes comment events with thread context. Test alongside standard Feishu message events.
- **Voice Wake triggers Talk Mode (macOS):** changes to macOS voice or talk-mode code paths should cover the Voice Wake entry point.
- **Auth failover cap:** rate-limit retries now cap same-provider auth-profile rotations before cross-provider fallback. Config: `auth.cooldowns.rateLimitedProfileRotations`. Test multi-profile setups.
- **Telegram `errorPolicy`:** configurable error suppression per account/chat/topic. If you change Telegram error delivery, test repeated failure suppression and cooldown reset.

### Release-window Workflow Additions (v2026.3.31)

- **Task surfaces must be tested together:** validate `openclaw tasks list|show|cancel`, `/status`, detached cron/subagent runs, and task cleanup/recovery as one released surface.
- **Node trust changed materially:** if you change node pairing, node commands, or node-originated exec flows, verify approval gating and reduced-trust behavior explicitly.
- **Remote MCP configs are now first-class:** test `mcp.servers` with URL-based transports, auth headers, and timeouts; do not assume stdio-only behavior.
- **Current browser code lives under `extensions/browser/`:** when auditing or changing browser automation, read the extension runtime instead of relying on historical `src/browser/` references.
- **Install security is now fail-closed:** plugin/skill installation workflows must cover scanner failures and the documented dangerous override path.

### Release-window Workflow Additions (v2026.3.28)

- **Qwen portal auth removed:** `qwen-portal-auth` is gone. Use `openclaw onboard --auth-choice modelstudio-api-key` for new Qwen setups. Existing configs with `qwen-portal-auth` will fail validation — run `openclaw doctor --fix` to migrate.
- **Config/Doctor migration window dropped:** migrations older than 2 months are no longer applied. Old config keys that required those migrations will fail validation. Audit configs and run `openclaw doctor --fix` before upgrading if you have not migrated recently.
- **`--claude-cli-logs` deprecated:** renamed to `--cli-backend-logs`, but retained as a compatibility alias on the released line. Update scripts and runbooks anyway.
- **MiniMax catalog reduced:** only M2.7 is available. Configs referencing M2, M2.1, M2.5, or VL-01 must be updated to `minimax/m2.7`.
- **`src/mcp/` is a new module:** gateway-backed channel MCP bridge. When touching gateway startup, verify MCP channel-server and channel-tools initialize correctly. `withBundledPluginAllowlistCompat` controls bundled plugin auto-load.
- **Plugin `requireApproval`:** the `before_tool_call` hook contract now supports async approval. Any plugin that implements `before_tool_call` should be reviewed against the new approval contract interface to avoid silent incompatibility.
- **xAI on Responses API:** xAI provider now uses the Responses API exclusively. If your config uses `x_search` or `code_execution` as standalone tools, these are now plugin-owned and require the xAI plugin to be enabled (auto-enabled from config in v2026.3.28).
- **`apply_patch` enabled by default for OpenAI/Codex:** review exec allowlists if you were explicitly blocking `apply_patch` for these model paths.

### Release-window Workflow Additions (v2026.3.13)

- **Exec approval hardening:** pnpm, Perl `-M`/`-I`, PowerShell `-File`/`-f`, env wrappers, macOS line continuation, and skill auto-allow are all hardened in the exec approval pipeline. Review any custom exec allowlist entries — previously-allowed invocations using these patterns may now be blocked by the preflight guard.
- **Single-use bootstrap pairing codes:** `/pair` pairing codes are now single-use (v2026.3.12 introduced short-lived tokens; v2026.3.13 enforces single-use). Automated pairing flows that reuse codes must be updated.
- **Memory bootstrap:** `MEMORY.md` is preferred over `memory.md`. If your workspace uses `memory.md` as the primary bootstrap file, rename it or ensure `MEMORY.md` exists.
- **`agents.defaults.compaction.customInstructions`:** add per-agent compaction instructions to preserve language/style across context resets. Key file: `src/agents/pi-extensions/compaction-instructions.ts`.
- **Chrome DevTools MCP (historical note):** `v2026.3.13` introduced live signed-in attach flows; by the current released line (`v2026.3.22+`), the supported host-local path is `driver: "existing-session"` with the built-in `user` profile. `chrome-relay` is no longer a built-in/current profile.
- **Windows gateway:** install/stop/status behavior changed on Windows. Verify gateway lifecycle commands after upgrading on Windows deployments.

### Release-window Workflow Additions (v2026.3.22-v2026.3.23)

- **Legacy Chrome relay removed:** `driver: "extension"` and `browser.relayBindHost` were removed in `v2026.3.22`. Current released browser guidance is Chrome MCP `existing-session`, optionally with `browser.profiles.<name>.userDataDir`.
- **ClawHub is the default first-party marketplace path:** bare `openclaw plugins install <package>` now prefers ClawHub before npm, and `openclaw skills search|install|update` is the released skills workflow.
- **Plugin SDK guidance changed:** plugin imports should target `openclaw/plugin-sdk/<subpath>`; treat `openclaw/extension-api` as deprecated compatibility only.
- **Canvas/session-reset auth tightened:** canvas routes require auth and agent session reset requires admin scope in the released gateway.
- **One-shot cron timezone semantics changed:** `openclaw cron add|edit --at ... --tz <iana>` now preserves the requested local wall-clock time for offset-less datetimes.

### Release-window Workflow Additions (v2026.3.12)

- **Node 22.14 minimum runtime:** upgrade to Node 22.14+ before upgrading OpenClaw. Node 24 is the new default for fresh installs.
- **`sessions_yield` tool added:** orchestrators can now call `sessions_yield` to end a turn immediately and skip queued tool work. Key file: `src/agents/tools/sessions-yield-tool.ts`. Update any orchestration logic that depends on queued tools completing before `end_turn`.
- **Provider plugins for Ollama/vLLM/SGLang:** these providers are now plugin-managed. Ensure no hardcoded references to old inline provider paths remain in config or tooling.
- **Exec allowlist GHSAs (20+):** multiple exec approval hardening advisories were fixed in this release (GHSA-pcqg, GHSA-f8r2, GHSA-57jw, GHSA-jvqh, GHSA-x7pp, GHSA-jc5j, and others). Review exec allowlists carefully after upgrading.
- **Bootstrap tokens:** `/pair` setup codes switched to short-lived tokens. Any automated pairing flow using the previous shared-credential approach must be updated.
- **Post-compaction reindexing:** enable `agents.defaults.compaction.postIndexSync: true` and `agents.defaults.memorySearch.sync.sessions.postCompactionForce: true` if memory freshness after compaction is required.
- **Hook loader paths:** unreadable hook loader paths now fail closed. Audit hook configurations to ensure loader paths are accessible.

### Release-window Workflow Additions (v2026.3.11)

- **Cron isolation — run `openclaw doctor --fix` (BREAKING):** cron jobs can no longer send ad hoc agent notifications or route through fallback main-session summaries. Any cron jobs relying on this behavior will silently stop delivering. Migration is mandatory; `openclaw doctor --fix` handles legacy cron storage and legacy notify/webhook delivery metadata.
- **macOS/launchd restart semantics v2:** explicit restarts no longer unload/reload the LaunchAgent — they hand off to a detached launchd helper. If you have custom restart scripts wrapping `launchctl bootout` + `bootstrap`, update them; the new path is `kickstart -k` via a helper. See `src/daemon/launchd-restart-handoff.ts`.
- **ACP `sessions_spawn` + session resume:** spawned ACP sessions now accept `resumeSessionId` to reattach to an existing ACPX/Codex conversation. If your orchestration code spawns fresh ACP sessions to continue prior conversations, use this field rather than passing prior context manually.
- **Memory multimodal reindexing:** if you enable `memorySearch.extraPaths` and configure `gemini-embedding-2-preview` with explicit `outputDimensions`, changing `outputDimensions` triggers a full reindex. Plan for that latency on first-run after config change.
- **WebSocket origin enforcement (GHSA-5wcw-8jjv-m286):** gateway now enforces browser-origin checks for all WebSocket connections regardless of proxy headers. If you run behind a reverse proxy and see new connection rejections, verify that your proxy passes a correct `Origin` header.
- **Gateway auth fail-closed:** gateway now fails closed when local `gateway.auth.*` SecretRefs are configured but unavailable (e.g. secrets backend offline). Previously it fell back silently to `gateway.remote.*` credentials. Ensure secrets backends are healthy before gateway startup.

### Release-window Workflow Additions (v2026.3.8)

- **Back up before destructive maintenance:** `openclaw backup create` / `openclaw backup verify` now exist specifically for state recovery. Use them before `reset`, uninstall, or risky config surgery.
- **launchd restart semantics changed:** supervised macOS restart now exits and relies on launchd `KeepAlive`; LaunchAgent repair/restart also `enable`s before `bootstrap`, and `XPC_SERVICE_NAME` now counts as a supervision hint.
- **Bundled channel plugins outrank duplicate npm installs during onboarding/update sync:** if a bundled plugin and an npm-installed plugin share the same ID, the bundled channel plugin now wins unless the operator intentionally overrides via explicit config paths.
- **Browser remote relay / WSL2 support changed (historical):** `browser.relayBindHost` existed in this release line, but it was removed in `v2026.3.22`. For the current released line, use Chrome MCP `existing-session` locally and direct WS/CDP profiles for remote/browserless setups.

### Release-window Workflow Additions (v2026.3.7)

- **`gateway.auth.mode` is now required when both auth methods are configured:** if both `gateway.auth.token` and `gateway.auth.password` are set, you must explicitly declare `gateway.auth.mode: "token"` or `gateway.auth.mode: "password"`. Omitting it causes a startup error. Run `openclaw doctor --fix` to migrate automatically.
- **`src/context-engine/` is a new module:** plugin slot for custom context management. When touching compaction, session assembly, or subagent spawn/end, check whether a `ContextEnginePlugin` is registered; the default is `legacy.ts`. Test lifecycle hooks (`bootstrap`, `ingest`, `assemble`, `compact`, `afterTurn`, `prepareSubagentSpawn`, `onSubagentEnded`) when modifying context paths.
- **Stable-tag extension inventory is 40 directories / 33 packages:** do not assume prior ad hoc counts or treat `context-engine/` as an extension directory.

### Release-window Workflow Additions (v2026.2.24)

- **Heartbeat delivery is now explicit and non-DM by default:** verify `heartbeat.delivery.target` for every environment. Default `none` means no external delivery unless configured, and DM/direct targets are now blocked.
- **Cross-channel shared-session routing is fail-closed:** when validating followup behavior, test explicit source-channel metadata paths and ensure no fallback to active dispatcher on route failure.
- **Sandbox namespace-join behavior changed:** if any workflow depended on `network: "container:<id>"`, confirm whether `agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin` must be explicitly enabled.
- **Security audit should be treated as a gate after upgrade:** this release introduced additional high-signal audit findings around trusted directories, trust model, and ingress hardening; resolve findings before shipping changes.
- **Long-running reply UX paths need keepalive coverage:** for channel runtime changes, include typing/keepalive assertions in tests so indicators remain active across extended inference/tool phases.

### Release-window Workflow Additions (v2026.2.19)

- **Gateway auth changes:** Before any gateway auth config edit, verify `gateway.auth.mode` explicitly. Default is now token mode. If touching `gateway.auth.token`, ensure `hooks.token` is set to a **different** value — identical values fail at startup.
- **Security audit as first step:** After upgrade or any security-adjacent config change, run `openclaw security audit` and triage all findings before writing new code or runbooks.
- **YAML frontmatter in prompts/cron:** Use explicit `true`/`false` for booleans — `on`/`off`/`yes`/`no` are now strings under YAML 1.2 core schema. Review all agent prompt frontmatter accordingly.
- **Cron webhook targets:** Verify all cron webhook URLs are publicly reachable. SSRF guard now blocks private/metadata destinations at dispatch time.
- **Plugin/hook installs:** Use `--pin` for npm plugin installs in automated flows. Unversioned installs now generate audit warnings. Record `name`, `version`, `spec`, and integrity in install records.
- **Canvas/A2UI automation:** Update any canvas proxy clients that used shared-IP auth to use scoped session capability URLs instead.

### Release-window Workflow Additions (v2026.2.17)

- **Config include changes:** If touching config loading, run at least one negative-path check for out-of-root `$include` and symlink escape behavior; do not assume legacy include layouts remain valid.
- **Cron schedule edits:** For any cron change, verify both expression and persisted `schedule.staggerMs`; include one exact (`staggerMs: 0`) and one staggered case in tests/validation.
- **Subagent UX changes:** Prefer announce-driven completion semantics in docs/tests. Polling-based examples should be marked as intervention/debug only.
- **Tool-loop safety:** Any process polling flow (`process poll/log`) must include progress checks and bounded backoff to avoid loop circuit-breaker triggers.

### Commit Message Conventions

- `feat:` - New feature
- `fix:` - Bug fix
- `perf:` - Performance improvement
- `refactor:` - Code restructuring
- `test:` - Test additions/changes
- `docs:` - Documentation

### Common Pitfalls

1. **Never call `loadConfig()` in render/hot paths** - it does sync `fs.readFileSync`. Thread config through params.
2. **Verify function is actually `async` before adding `await`** - causes `await-thenable` lint errors.
3. **Removing `async` from exported functions is BREAKING** - changes return type from `Promise<T>` to `T`. All `await` callers break.
4. **Primary operations must throw; only convenience ops get try/catch** - don't swallow errors on critical paths.
5. **Guard numeric comparisons against NaN** - use `Number.isFinite()` before `>` / `<`.
6. **Normalize paths before string comparison** - `path.resolve()` before `===`.
7. **Derive context from parameters, not global state** - use explicit paths, not env var fallbacks.
8. **Run FULL `pnpm check` before every push** - not just changed files. This is the canonical type+lint+format gate used in contributor and maintainer policy.

### Safety Invariants (Never Violate)

1. Never call `loadConfig()` in hot paths (`auto-reply/*`, channel send paths, streaming handlers).
2. Use subsystem logger instead of `console.*` in runtime paths.
3. Do exact identity checks (`a === b`), not truthy co-existence checks.
4. Treat primary operations as fail-fast (throw); only convenience wrappers should catch.
5. Keep file locking for sessions/cron stores; do not remove lock wrappers.
6. For security-sensitive changes (auth, SSRF, exec), run `openclaw security audit` before PR.
7. For config mutations, patch full nested path and verify by immediate read-back.

### Common Errors -> First Fix

| Error / Symptom | First Check | First Fix |
| --- | --- | --- |
| `SQLITE_CANTOPEN` in daemon mode | LaunchAgent env (`TMPDIR`) | Reinstall/restart gateway service to pick up env forwarding |
| `config.patch` "ok" but behavior unchanged | Wrong nesting path | Patch full nested key (`channels.telegram.*` etc.) + read-back verify |
| Startup fails with token conflict | `hooks.token` equals `gateway.auth.token` | Set different values; restart gateway |
| SSRF block on webhook/browser URL | Target is private/metadata/non-public | Use public HTTPS endpoint |
| `await-thenable` lint error | Function is not async | Fix signature or remove incorrect `await` |

---

## 6. Configuration Reference

### Root Config (`openclaw.json` - JSON5)

| Section             | Type File                          | Zod Schema                         |
| ------------------- | ---------------------------------- | ---------------------------------- |
| `agents`            | `types.agents.ts`                  | `zod-schema.agents.ts`             |
| `agents.defaults`   | `types.agent-defaults.ts`          | `zod-schema.agent-defaults.ts`     |
| `bindings[]`        | `types.agents.ts` (`AgentBinding`) | `zod-schema.agents.ts`             |
| `session`           | `types.base.ts` (`SessionConfig`)  | `zod-schema.session.ts`            |
| `gateway`           | `types.gateway.ts`                 | `zod-schema.ts`                    |
| `models`            | `types.models.ts`                  | `zod-schema.ts`                    |
| `authProfiles`      | `types.auth.ts`                    | `zod-schema.ts`                    |
| `tools`             | `types.tools.ts`                   | `zod-schema.agent-runtime.ts`      |
| `channels.telegram` | `types.telegram.ts`                | `zod-schema.providers.ts`          |
| `channels.discord`  | `types.discord.ts`                 | `zod-schema.providers.ts`          |
| `channels.slack`    | `types.slack.ts`                   | `zod-schema.providers.ts`          |
| `channels.signal`   | `types.signal.ts`                  | `zod-schema.providers.ts`          |
| `channels.whatsapp` | `types.whatsapp.ts`                | `zod-schema.providers-whatsapp.ts` |
| `channels.imessage` | `types.imessage.ts`                | `zod-schema.providers.ts`          |
| `hooks`             | `types.hooks.ts`                   | `zod-schema.hooks.ts`              |
| `cron`              | `types.cron.ts`                    | `zod-schema.ts`                    |
| `memory`            | `types.memory.ts`                  | `zod-schema.ts`                    |
| `messages`          | `types.messages.ts`                | `zod-schema.ts`                    |
| `approvals`         | `types.approvals.ts`               | `zod-schema.approvals.ts`          |
| `sandbox`           | `types.sandbox.ts`                 | `zod-schema.ts`                    |
| `logging`           | `types.base.ts`                    | `zod-schema.ts`                    |
| `plugins`           | `types.plugins.ts`                 | `zod-schema.ts`                    |
| `browser`           | `types.browser.ts`                 | `zod-schema.ts`                    |

All type files are in `src/config/`, all Zod schemas in `src/config/`.

> **v2026.2.15 additions:** `messages.suppressToolErrors` (bool) suppresses tool error display. Per-channel `ackReaction` config added to Telegram, Discord, Slack, WhatsApp type files.
>
> **v2026.2.17 additions:** Anthropic models support `params.context1m: true` (1M beta header), Z.AI models default to `params.tool_stream: true`, and recurring top-of-hour cron schedules now persist deterministic staggering via `schedule.staggerMs` unless explicitly disabled.
>
> **v2026.2.19 additions:** `gateway.auth.mode` defaults to `"token"` (was implicit open). `gateway.auth.token` is auto-generated and persisted on first start. `hooks.token` must differ from `gateway.auth.token` (startup validation). `browser.ssrfPolicy` controls SSRF behavior for browser URL navigation. `tools.exec.safeBins` now validates against trusted bin directories only. Cron webhook targets are SSRF-validated before dispatch. Plugin install records now include `name`, `version`, `spec`, integrity, and shasum (`--pin` flag for npm plugins).
>
> **v2026.3.12 additions:** `channels.slack.capabilities.interactiveReplies` (bool) opts into Block Kit interactive replies. `agents.defaults.compaction.postIndexSync` (bool) triggers post-compaction memory sync. `agents.defaults.memorySearch.sync.sessions.postCompactionForce` (bool) forces session reindexing after compaction. Per-model `service_tier` fast-mode config for OpenAI and Anthropic.
>
> **v2026.3.13 additions:** `agents.defaults.compaction.customInstructions` (string) — per-agent instructions for compaction to preserve language/style continuity. `OPENCLAW_TZ` environment variable for Docker timezone pinning. `agents.list[].params`, `tools.web.fetch.readability`, `tools.web.fetch.firecrawl`, `channels.signal.groups`, `discovery.wideArea.domain` are now accepted config keys (previously Zod-rejected as unknown).
>
> **v2026.4.9 additions:** `channels.<channel>.contextVisibility` (enum) — filter supplemental quote/thread/fetched context by allowlist. Shared model/media request transport overrides across OpenAI-, Anthropic-, Google-, and compatible providers. Dreaming config/operations on the released line now include user-facing `enabled` and optional `frequency` plus aging controls (`recencyHalfLifeDays`, `maxAgeDays`) and `dreams.md` trail output. Media generation surfaces add built-in `music_generate`, `video_generate`, and `tools.media.asyncCompletion.directSend`. `agents.defaults.videoGenerationModel` is part of the released config surface.
>
> **v2026.4.2 additions:** `plugins.entries.xai.config.xSearch.*` — plugin-owned xAI x_search config (migrated from `tools.web.x_search.*`). `plugins.entries.xai.config.webSearch.apiKey` — xAI web search API key. `plugins.entries.firecrawl.config.webFetch.*` — plugin-owned Firecrawl web_fetch config (migrated from `tools.web.fetch.firecrawl.*`). `agents.defaults.compaction.notifyUser` (bool) — opt-in compaction start notice. Provider transport policy: insecure TLS/runtime overrides blocked; proxy-hop TLS separated from target mTLS.
>
> **v2026.4.1 additions:** `agents.defaults.params` (object) — global default provider parameters applied to all model calls unless overridden per-agent or per-model. `gateway.webchat.chatHistoryMaxChars` (number) — configurable chat history text truncation for webchat. `auth.cooldowns.rateLimitedProfileRotations` (number) — cap for same-provider auth-profile retries before cross-provider fallback. `channels.telegram.errorPolicy` (string) and `channels.telegram.errorCooldownMs` (number) — per-account/chat/topic error suppression. `channels.whatsapp.reactionLevel` (string) — agent reaction guidance level. `webSearch.searxng.host` (string) — SearXNG instance host for bundled web search provider.

### How to Add a New Config Key

1. Add type to appropriate `config/types.*.ts` file
2. Add Zod schema to appropriate `config/zod-schema.*.ts` file - type and schema MUST match
3. Add default in `config/defaults.ts` if applicable
4. Update `config/schema.hints.ts` if it needs UI labels
5. Add test in `config/config.*.test.ts`
6. If migrating from old format: add rule in `config/legacy.migrations.part-*.ts`

---

## 7. Key Types Quick Reference

| Type                          | File                                 | Usage                                                          |
| ----------------------------- | ------------------------------------ | -------------------------------------------------------------- |
| `OpenClawConfig`              | `config/types.openclaw.ts`           | Root config - used everywhere                                  |
| `AgentConfig`                 | `config/types.agents.ts`             | Per-agent config                                               |
| `AgentBinding`                | `config/types.agents.ts`             | Channel→agent binding                                          |
| `SessionEntry`                | `config/sessions/types.ts`           | Persistent session state                                       |
| `MsgContext`                  | `auto-reply/templating.ts`           | Inbound message context                                        |
| `ReplyPayload`                | `auto-reply/types.ts`                | Reply output (text, media, replyTo)                            |
| `ChannelPlugin`               | `channels/plugins/types.plugin.ts`   | Channel plugin contract                                        |
| `ChannelId`                   | `channels/plugins/types.core.ts`     | `ChatChannelId | (string & {})` — extensible channel ID        |
| `ChatChannelId`               | `channels/registry.ts`               | Built-in channel IDs (re-exported via `types.core.ts`)         |
| `ChatType`                    | `channels/chat-type.ts`              | `"direct" \| "group" \| "channel"`                             |
| `ThinkLevel`                  | `auto-reply/thinking.ts`             | `"off" \| "minimal" \| "low" \| "medium" \| "high" \| "xhigh"` |
| `VerboseLevel`                | `auto-reply/thinking.ts`             | `"off" \| "on" \| "full"`                                      |
| `ToolPolicyLike`              | `agents/tool-policy.ts`              | `{ allow?: string[]; deny?: string[] }` — tool access policy   |
| `EmbeddedPiRunResult`         | `agents/pi-embedded-runner/types.ts` | Agent run result                                               |
| `ResolvedAgentRoute`          | `routing/resolve-route.ts`           | Routing result                                                 |
| `InputProvenance`             | `sessions/input-provenance.ts`       | Message origin tracking                                        |
| `HookSource`                  | `hooks/types.ts`                     | Hook source discriminator                                      |
| `CronJob`                     | `cron/types.ts`                      | Scheduled job definition                                       |
| `MemorySearchResult`          | `memory/types.ts`                    | Search result from memory index                                |

---

## 8. File Naming Conventions

### Within Modules

```
src/<module>/
├── index.ts                    # Barrel re-exports (public API)
├── types.ts                    # Type definitions
├── *.ts                        # Implementation files
├── *.test.ts                   # Co-located unit tests
├── *.e2e.test.ts               # End-to-end tests
├── *.test-harness.ts           # Reusable test fixtures
├── *.mocks.ts                  # Test mocks
```

### Naming Patterns

| Pattern             | Meaning                             | Example                     |
| ------------------- | ----------------------------------- | --------------------------- |
| `*.test.ts`         | Unit test                           | `io.write-config.test.ts`   |
| `*.e2e.test.ts`     | End-to-end test                     | `program.nodes-basic.e2e.test.ts` |
| `*.test-harness.ts` | Reusable test fixture               | `service.test-harness.ts`   |
| `*.mocks.ts`        | Test mock definitions               | `isolated-agent.mocks.ts`   |
| `*.impl.ts`         | Implementation (when barrel exists) | `auto-reply.impl.ts`        |
| `zod-schema.*.ts`   | Zod validation schema               | `zod-schema.agents.ts`      |
| `types.*.ts`        | Domain-specific types               | `types.telegram.ts`         |

### Where to Put New Files

| Adding a...            | Put it in...                                                           |
| ---------------------- | ---------------------------------------------------------------------- |
| New tool               | `src/agents/tools/<tool-name>.ts` + register in `openclaw-tools.ts` (e.g. `src/agents/tools/sessions-yield-tool.ts` for `sessions_yield`) |
| New channel            | `src/<channel>/` + `extensions/<channel>/` for plugin                  |
| New CLI command        | `src/cli/<command>-cli.ts` + `src/commands/<command>.ts`               |
| New config type        | `src/config/types.<section>.ts` + `src/config/zod-schema.<section>.ts` |
| New hook               | `src/hooks/bundled/<hook-name>/handler.ts` + `HOOK.md`                 |
| New gateway RPC method | `src/gateway/server-methods/<method>.ts`                               |
| New test               | Co-locate with source: `src/<module>/<file>.test.ts`                   |

---

## 9. Gotchas & Landmines

### Gotcha Index by Domain

- **Config & Auth:** #1, #6, #16, #33, #34, #37, #44, #61, #62, #67, #68, #73, #74, #82, #129, #130
- **Routing & Sessions:** #3, #6, #17, #18, #20, #67, #70, #99, #100, #105
- **Telegram/Channel Delivery:** #8, #9, #10, #12, #35, #36, #99, #104
- **Tooling & Agent Runtime:** #5, #19, #39, #40, #41, #51, #52, #63, #66, #79, #80, #84, #88, #89, #92, #93, #98, #102, #103, #132, #133, #134
- **Security/Network:** #38, #42, #43, #45, #46, #47, #48, #53, #54, #55, #56, #57, #58, #64, #65, #66, #73, #74, #75, #96, #97
- **Channel/Streaming Config:** #36, #49, #50, #62, #72, #81, #83, #94
- **Subagent/Ownership:** #48, #53, #98
- **Provider/Model:** #59, #68, #69, #71, #72, #94, #95, #131
- **Breaking Changes (v2026.2.22):** #59, #60, #61, #62, #63
- **Breaking Changes (v2026.3.1):** #92, #93
- **Breaking Changes (v2026.3.7):** #106
- **Breaking Changes (v2026.3.28):** #107, #108, #109, #110
- **Behavioral Shifts (v2026.4.1):** #121, #122, #123, #124, #125, #126, #127, #128
- **Breaking Changes (v2026.4.9):** #135, #136, #137
- **Behavioral Shifts (v2026.4.9):** #138, #139, #140
- **Breaking Changes (v2026.4.2):** #129, #130, #131, #132, #133, #134
- **Operational Notes (v2026.3.28):** #111, #112, #113
- **Exec/Shell Security (v2026.2.22):** #64, #65, #66
- **Feishu:** #101

(Use this index first, then read only the relevant gotchas for your change.)

### Things That Look Simple But Aren't

1. **`loadConfig()` is synchronous with caching** - First call reads disk (sync `fs.readFileSync`). Subsequent calls return cached. `clearConfigCache()` to invalidate. NEVER call in hot paths.

2. **Route resolution uses `WeakMap` cache on config object** - `routing/resolve-route.ts` caches bindings evaluation on the config object itself. If you spread/clone config, the cache misses.

3. **Session keys are hierarchical** - Format: `agent:<id>:<channel>:<kind>:<peerId>[:thread:<threadId>]`. Functions like `isSubagentSessionKey()`, `isCronSessionKey()` depend on exact format.

4. **`agents/` ↔ `auto-reply/` is bidirectional by design** - Not a circular dependency bug. `agents/` provides runtime, `auto-reply/` orchestrates it.

5. **`agents/pi-embedded-subscribe.ts` processes SSE stream chunks** - It's a streaming state machine. Adding/removing events here can break tool call parsing, block chunking, or reasoning block extraction.

### Modules With Surprising Coupling

- **`auto-reply/thinking.ts`** exports `VerboseLevel` used by `sessions/level-overrides.ts` - changing enum values breaks session persistence.
- **`channels/dock.ts`** returns lightweight metadata to avoid importing heavy channel code. Must be updated when channel capabilities change.
- **`infra/outbound/deliver.ts`** is used by both cron delivery AND message tool sends - test both.
- **`config/sessions/store.ts`** uses file locking - concurrent writes can deadlock if lock isn't released.

### Race Conditions to Watch

- **Cron service uses `locked()`** (in `cron/service/locked.ts`) to serialize operations. Removing this causes race conditions.
- **Session file writes**: `agents/session-write-lock.ts` provides file-based locking. Concurrent JSONL appends without locking corrupt files.
- **Gateway config reload**: `gateway/config-reload.ts` uses chokidar debounce. Rapid config changes can trigger multiple reloads.
- **Telegram media groups**: `bot-updates.ts` aggregates photos with a timeout window. Changing this can split or merge groups incorrectly.
- **Telegram draft stream cleanup vs fallback delivery**: `bot-message-dispatch.ts` has a `finally` block that calls `draftStream?.stop()`. The actual preview cleanup (`clear()`) must run **after** fallback delivery logic, but must still be guaranteed via `try/finally` wrapping the fallback. Cleanup in a `finally` that runs _before_ fallback logic executes too early - the preview gets deleted before fallback can send, causing silent message loss (#19001). Always use this pattern:

  ```ts
  try {
    await fallbackDelivery();
  } finally {
    clearDraftPreviewIfNeeded(); // runs even if fallback throws
  }
  ```

  If the fallback itself throws and cleanup isn't in `finally`, stale preview messages are left behind.

- **Telegram `disableBlockStreaming` evaluation order**: When `streamMode === "off"`, `disableBlockStreaming` must be `true` (not `undefined`). A ternary chain like `a ? x : b ? y : undefined` produces `undefined` instead of `true` when the priority condition isn't checked first. Code using `if (disableBlockStreaming)` treats `undefined` as falsy, silently allowing block streaming in off mode with `draftStream === undefined` → message loss. Always put the most restrictive condition first, and prefer explicit `true`/`false` over `undefined` in boolean ternaries.

### Other Landmines

- **JSON5 vs JSON**: Config files are JSON5 (comments, trailing commas). Session files, cron store, auth profiles are strict JSON. Don't mix parsers.
- **Telegram HTML formatting**: `telegram/format.ts` converts Markdown→Telegram HTML. Telegram's HTML subset is limited - broken HTML silently fails.
- **Discord 2000 char limit**: `discord/chunk.ts` enforces limits with fence-aware splitting. Don't bypass the chunker.
- **Signal styled text**: Uses byte-position ranges, not character positions. Multi-byte chars shift ranges.
- **WhatsApp target normalization**: Converts between E.164, JID (`@s.whatsapp.net`), and display formats. Getting this wrong means messages go nowhere silently.
- **`config.patch` path nesting matters**: `config.patch` with `{"telegram":{"streamMode":"off"}}` silently writes to an ignored top-level key. The correct path is `{"channels":{"telegram":{"streamMode":"off"}}}`. A "successful" patch that changes nothing is worse than an error - always verify the full nested structure before patching.
- **Gateway config patches need read-back verification**: After `config.patch`, always read back the config to confirm the change took effect. Silent success + wrong nesting path = hours of debugging the wrong code while the config was never actually changed.

### v2026.2.15 New Gotchas

9. **Pairing stores are now account-scoped** - `pairing/pairing-store.ts` scopes by account. Old unscoped pairing data requires migration via `legacy allowFrom migration` in Telegram.

10. **Nested subagent depth limits** - `agents/subagent-depth.ts` enforces max depth (default 2) and max children per agent (default 5). Exceeding these silently blocks spawning.

11. **Discord Component v2 UI** - `discord/components.ts` and `discord/components-registry.ts` handle new Discord components. The `send.components.ts` file handles outbound component messages separately from regular sends.

12. **Memory collections are now per-agent isolated** - Managed QMD collections are isolated per agent. Drifted collection paths are automatically rebound. Don't assume shared memory across agents.

13. **Cron skill-filter snapshots are normalized** - Cron service normalizes skill-filter snapshots. Treat missing `enabled` as `true` in cron job updates. Model-only update patches infer payload kind automatically.

14. **`sessions_spawn` supports model fallback** - The `model` parameter in `sessions_spawn` now supports fallback chains. Don't assume the spawned session uses exactly the requested model.

15. **Skill download paths are restricted** - `infra/install-safe-path.ts` validates target paths for skill downloads, preventing path traversal. Cross-platform fallback for non-brew installs added.

### v2026.2.17 New Gotchas

16. **Config include confinement is strict** - `$include` paths are now confined to the top-level config directory with traversal/symlink hardening. Old layouts that reached outside config root will fail and need explicit restructuring.

17. **Cron top-of-hour defaults are now staggered** - recurring cron schedules like `0 * * * *` persist deterministic `schedule.staggerMs` by default. If you need exact clock boundaries, set stagger to `0` (`--exact`).

18. **`sessions_spawn` is push-first, not poll-first** - one-off spawns return an accepted note and completion is auto-announced back to requester context. Busy polling can now trip loop protections and waste tokens.

19. **Tool-loop detection hard-blocks no-progress poll/log loops** - repeated `process(action=poll|log)` with no progress now escalates to warnings and eventually a circuit-breaker block. Always include progress checks/backoff/exit criteria.

20. **Read truncation markers are actionable, not fatal** - when output contains `[compacted: tool output removed to free context]` or `[truncated: output exceeded context limit]`, recover with smaller targeted reads (`offset`/`limit`) instead of full-file retries.

21. **Z.AI tool streaming defaults ON** - `tool_stream` is enabled by default for Z.AI models. If your automation assumes non-streamed tool behavior, explicitly set `params.tool_stream: false` and test both paths.

22. **Anthropic 1M context is explicit opt-in** — `params.context1m: true` controls the beta header (`anthropic-beta: context-1m-2025-08-07`). Don't assume larger windows without this flag and provider support.

### Additional Release-window Gotchas (between v2026.2.17 and v2026.2.19)

23. **Pass API tokens explicitly in every call** — When one call in a flow passes a token explicitly, all calls must. Missing token causes silent auth failure; SDK defaults aren't guaranteed to carry the right credentials.

24. **Use the repo's logging abstraction, not `console.*`** — Raw `console.debug()` bypasses user-controlled verbosity (`logVerbose`/`deps.log.*`) and spams stdout in hot paths. Every log call must go through the subsystem.

25. **Identity checks must compare exact values, not field existence** — `(message.bot_id && ctx.botUserId)` matches ANY bot. Correct: `message.bot_id === ctx.botUserId`. "Both truthy" ≠ identity match.

26. **Don't mix ID namespaces for message provenance** — `uploadSlackFile()` returns a file ID, not a message `ts`. Using it as a message key in an origin cache means lookups always fail. Verify what SDK methods actually return.

27. **Classify shell builtins by token list, not resolved PATH** — `resolveExecutablePath()` finds PATH-shadowed binaries for builtin names (e.g., `/usr/bin/echo`). Detecting builtins via `resolvedPath == null` fails. Use a known-builtins set.

28. **Close resource pools on every call, not just teardown** — `ProxyAgent` and connection pools instantiated per-call must be explicitly closed. Unclosed pools accumulate and leak.

29. **Validate URLs before constructing connection objects** — Invalid proxy URL throws in `ProxyAgent` constructor, crashing execution. Validate format first.

30. **Duplicate inverse conditions in sequence = dead code** — Block A skips non-gateway, Block B skips gateway → nothing runs. Two inverse conditions = contradiction bug.

31. **Consolidate duplicate functions into shared utilities immediately** — Same function body in 3+ files guarantees divergence. When flagged, fix it now.

32. **Run `pnpm protocol:gen:swift` after protocol schema changes** — Forgetting breaks `pnpm protocol:check` on all rebased PRs. Add to pre-commit checklist for `src/gateway/protocol/schema/**`.

### v2026.2.19 New Gotchas

33. **Gateway auth now defaults to token mode** — `gateway.auth.mode` is no longer implicitly open. A `gateway.auth.token` is auto-generated and persisted on first start. If you need an explicitly open loopback setup, set `gateway.auth.mode: "none"` explicitly. The new security audit fires a `gateway.http.no_auth` warning (loopback) or critical (remote exposure) when `mode="none"` is active.

34. **`hooks.token` must differ from `gateway.auth.token` — startup failure if equal** — New startup validation rejects configs where these two tokens are identical. The gateway refuses to start. Separate them before upgrading, or let gateway auto-generate a fresh `gateway.auth.token`.

35. **Heartbeat skips when HEARTBEAT.md is missing or empty** — Interval heartbeats now no-op automatically when `HEARTBEAT.md` is absent or empty and no tagged cron events are queued. Cron-event fallback for queued tagged reminders is still preserved. Don't rely on heartbeat firing to run "always-on" logic unless HEARTBEAT.md has content.

36. **Cron/heartbeat Telegram topic delivery now works — but requires correct target format** — Explicit `<chatId>:topic:<threadId>` targets in cron/heartbeat delivery now correctly route to the configured topic. The old behavior (falling back to last active thread) is fixed. If you were working around this with custom routing, remove the workaround.

37. **YAML frontmatter: `on`/`off`/`yes`/`no` are now strings, not booleans** — YAML 1.2 core schema is used for frontmatter parsing. `on` and `off` no longer coerce to `true`/`false`. Any agent prompt, cron payload, or config that relied on implicit YAML 1.1 boolean coercion must be updated to use explicit `true`/`false`.

38. **Browser relay requires gateway-token auth on `/extension` and `/cdp`** — Both the Chrome extension relay endpoint and CDP endpoint now require `gateway.auth.token` authentication. External browser clients (non-tool-path access) must supply the token. The `browser` tool handles this automatically.

39. **`read` tool auto-pages using model contextWindow** — When no explicit `limit` is provided, `read` now auto-pages and scales its per-call output budget from the model's configured `contextWindow`. Large-context models get proportionally more output before guards kick in. This is a behavior change: previously `read` without `limit` was equivalent to a bounded single-call read.

40. **exec preflight guard for env var injection patterns** — The exec tool now runs a preflight check that detects shell env var injection patterns (e.g., `$DM_JSON`, `$TMPDIR`, `$HOME`) in Python/Node scripts before execution. Scripts that include env-var interpolation from surrounding shell context will be blocked with an early warning. Fix: use explicit Python/Node variable assignment instead of shell-sourced vars.

41. **`tools.exec.safeBins` now validates against trusted bin directories only** — Previously, `safeBins` checked allowlist membership by name. Now, the resolved binary path must be in trusted bin directories (system defaults + gateway startup `PATH`). A trojan binary shadowing a safe bin name in a later PATH directory will be rejected even if the name is in `safeBins`.

42. **Cron webhook delivery is now SSRF-guarded** — Cron webhook POST targets are validated via `fetchWithSsrFGuard` before dispatch. Private addresses (RFC1918, localhost, metadata endpoints) and non-standard IPv4 forms (octal, hex, short) are blocked. Only publicly reachable HTTPS endpoints are accepted.

43. **SSRF bypass via IPv6 transition addresses is now blocked** — NAT64 (`64:ff9b::/96`, `64:ff9b:1::/48`), 6to4 (`2002::/16`), Teredo (`2001:0000::/32`), and ISATAP embedded IPv4 addresses are now rejected by SSRF guards. Non-standard IPv4 forms (octal, hex, packed, short) are also blocked before DNS lookup.

44. **Canvas/A2UI now uses node-scoped session capability URLs** — The `/__openclaw__/canvas/*` and `/__openclaw__/a2ui/*` endpoints no longer use shared-IP fallback auth. Requests must use node-scoped session capability URLs. Trusted-proxy requests that omit forwarded client headers now fail closed. Update any canvas proxy automation that assumed shared-IP auth.

45. **macOS LaunchAgent SQLite fix: no action needed, but restart service after upgrade**

46. **Plaintext `ws://` to non-loopback hosts is blocked** — Only `wss://` is accepted for remote WebSocket connections. Loopback (`ws://localhost`) still works. Any remote integrations using `ws://` must switch to `wss://`.

47. **Control-plane RPCs are rate-limited (3/min per device+IP)** — `config.apply`, `config.patch`, `update.run` are rate-limited. Gateway restarts are coalesced with a 30-second cooldown. Rapid scripted config changes will hit 429 errors.

48. **Discord moderation actions enforce guild permissions on trusted sender** — `timeout`, `kick`, `ban` now check the calling sender's guild permissions. Untrusted `senderUserId` params are ignored. Ensure the bot has appropriate guild permissions for moderation. — `TMPDIR` is now forwarded into installed service environments. If the gateway daemon was hitting `SQLITE_CANTOPEN` errors under macOS LaunchAgent, this is fixed automatically. Re-run `openclaw gateway install` (or reinstall the service) to pick up the new environment forwarding.

### v2026.2.21 New Gotchas

49. **`channels.telegram.streaming` is now a boolean — old `streamMode` values don't map perfectly** — `channels.telegram.streaming` was simplified from an enum (`streamMode`) to a boolean. Legacy values are auto-mapped, but explicit `streamMode` config keys in any AGENTS.md or config files should be reviewed and updated to the boolean form to avoid silent behavior from the auto-mapper.

50. **`maxConcurrentRuns` in cron was NOT enforced before v2026.2.21** — The `cron.maxConcurrentRuns` config key was silently ignored in the timer loop prior to v2026.2.21. If you had slow cron jobs that were supposed to be limited to 1 concurrent run, they may have been accumulating silently. After upgrade, the limit is now enforced, which could surface queued-but-never-started jobs or change timing behavior.

51. **`senderIsOwner` was NOT propagated to embedded/subagent runners** — Before v2026.2.21, owner-only tools failed silently when invoked from subagents or embedded runners — the ownership context wasn't forwarded. If you have tools guarded by `senderIsOwner`, test them in subagent contexts after upgrade.

52. **Heredoc command substitution now blocked — may break exec tool scripts** — `$(cmd)` and backtick substitutions in unquoted heredoc bodies are now blocked by the exec tool preflight guard. Scripts that used heredocs with command substitution will be rejected with a preflight error instead of executing. Rewrite to use variable assignment or separate exec calls.

53. **`--no-sandbox` disabled in sandbox containers by default** — Chrome/Chromium sandbox is now enabled by default in container runs. This may cause failures in restricted container environments (e.g., nested Docker without `SYS_ADMIN` capability). If browser tasks fail in container context, check whether the container has the required Linux capabilities.

54. **noVNC observer requires password auth — external noVNC clients will fail** — noVNC observer sessions now require one-time token auth. Any external tooling that connected to noVNC observer endpoints without auth will need to be updated to use the token-based auth flow.

55. **Tailscale tokenless auth scoped to WebSocket only** — Tailscale-based tokenless auth is now only accepted on WebSocket connections. HTTP API calls via Tailscale must use explicit token auth. If any automation used HTTP API calls over Tailscale without explicit token auth, these will now be rejected.

56. **WhatsApp JID allowlist now enforced on all send paths** — All outbound WhatsApp sends now validate JID against the allowlist. Previously some send paths (e.g., via tools) bypassed the allowlist. If you have WhatsApp automation that sends to JIDs not in the allowlist, these will now return 403.

57. **Prototype-chain traversal blocked in webhook template `getByPath`** — Crafted template expressions using prototype-chain traversal (e.g., `__proto__`, `constructor`) in webhook templates now fail. If any webhook templates used these paths for legitimate reasons, they need to be rewritten.

58. **SHA-1 synthetic IDs migrated to SHA-256 — stored IDs may differ** — Synthetic IDs (used internally for session keys, content addressing, etc.) have migrated from SHA-1 to SHA-256. If any external system stores or compares synthetic IDs generated by OpenClaw, they will need to regenerate after upgrade — old SHA-1 IDs will not match new SHA-256 IDs.

### v2026.2.22–v2026.2.23 Specific

**BREAKING CHANGES (v2026.2.22):**

59. **`google-antigravity/*` model refs silently fail** — `google-antigravity/*` model refs silently fail. Migrate to `google-gemini-cli` or another supported Google provider. Run `openclaw doctor --fix` to detect.

60. **Device auth v1 removed** — Device-auth clients must sign `v2` payloads with the per-connection `connect.challenge` nonce. Nonce-less connects are rejected with `unauthorized`. Client libraries or scripts using v1 signing must be updated.

61. **`session.dmScope` default changed** — New installs default `session.dmScope` to `per-channel-peer`. If your setup depends on shared DM history across multiple senders, explicitly set `"session": {"dmScope": "main"}`.

62. **Streaming config unified** — `channels.<channel>.streaming` now uses enum `off | partial | block | progress`. Legacy `streamMode` and Slack boolean `streaming` still read but deprecated. Canonical key for Slack native streaming is `channels.slack.nativeStreaming`. Run `openclaw doctor --fix` to migrate.

63. **Tool-failure replies hide raw errors** — By default, detailed error suffixes (provider messages, local paths) are hidden. Use `/verbose on` or `/verbose full` for raw details. This affects debugging — don't be surprised when agents send generic failure messages.

**SECURITY (v2026.2.22):**

64. **Safe-bin PATH trust removed** — `tools.exec.safeBins` no longer trusts PATH-derived directories. Add `tools.exec.safeBinTrustedDirs` for explicit trusted directories. Existing safe-bin configs may fail if they relied on PATH resolution. Requires `tools.exec.safeBinProfiles` for custom binaries without built-in profiles.

65. **Shell env sanitization expanded** — Request-scoped `HOME`, `ZDOTDIR`, `SHELLOPTS`, `PS4` overrides are blocked in host exec env sanitizers. Shell-wrapper env restricted to explicit allowlist: `TERM`, `LANG`, `LC_*`, `COLORTERM`, `NO_COLOR`, `FORCE_COLOR`. Login-shell executable paths now validated against `/etc/shells`.

66. **Exec obfuscation detection** — Obfuscated commands (e.g., base64-encoded, hex-encoded payloads) are detected before exec allowlist decisions and require explicit approval. Affects scripts that use encoding as a convenience pattern.

**SESSION/PROVIDER (v2026.2.22–v2026.2.23):**

67. **Session keys are case-sensitive (historically) but now canonicalized** — v2026.2.23 canonicalizes inbound mixed-case session keys to lowercase and migrates legacy case-variant entries. If you have external tooling that reads session stores by key, be aware keys may change case after upgrade.

68. **Anthropic OAuth tokens cannot use context-1m beta** — `context-1m-*` beta header injection is skipped for OAuth/subscription tokens (`sk-ant-oat-*`). If `params.context1m=true` and you use an OAuth token, the beta is silently skipped to avoid 401 failures. Direct API key users are unaffected.

69. **OpenRouter `reasoning_effort` conflict** — If you inject `reasoning.effort` in OpenRouter payloads, the top-level `reasoning_effort` field is now removed automatically to avoid 400 errors. Do not set both — use only the nested `reasoning.effort`.

70. **Cron `maxConcurrentRuns` now actually enforced** — Previously `cron.maxConcurrentRuns` was honored in some paths but jobs always ran serially in the timer loop. Now enforced end-to-end. If you depend on serial execution with `maxConcurrentRuns=1` (which was the implicit default), behavior is unchanged. Higher values now actually run in parallel.

71. **Vercel AI Gateway Claude shorthand** — Use `vercel-ai-gateway/claude-opus-4-6` etc. — shorthand refs normalize to canonical Anthropic-routed IDs automatically. No config change needed.

72. **Reasoning thinking-block leak** — When a model has `thinking=low` (model-default thinking active), `auto-reasoning` is now kept disabled by default. If your agent config enables auto-reasoning AND the model has default thinking, the auto-reasoning was previously leaking `Reasoning:` blocks into channel replies. This is now fixed — but if you explicitly need reasoning output, enable it explicitly rather than relying on auto.

**BREAKING CHANGES (v2026.2.23):**

73. **Browser SSRF default flipped to trusted-network** — `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` now defaults to `true` (trusted-network mode) when unset, meaning browser navigation to private/LAN addresses is allowed by default. To enforce strict SSRF blocking, explicitly set `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: false`. The legacy key `browser.ssrfPolicy.allowPrivateNetwork` is still accepted but deprecated — `openclaw doctor --fix` migrates it automatically.

### v2026.2.24 Specific

**BREAKING / OPERATIONAL SHIFTS (v2026.2.24):**

74. **Heartbeat DM delivery now blocked** — direct/DM heartbeat targets are rejected. Heartbeat runs still execute, but only non-DM targets can receive external delivery.

75. **Heartbeat default target changed to `none`** — if you previously relied on implicit `last` routing, heartbeat outputs will now stay internal unless you explicitly configure an external target.

76. **Sandbox container namespace join is disabled by default** — Docker `network: "container:<id>"` in sandbox paths now requires explicit break-glass opt-in.

77. **Cross-channel shared-session routing now fails closed** — explicit cross-channel followups no longer fall back to dispatcher context on route failure, reducing misrouted reply risk.

78. **Stop/abort phrase matching expanded and stricter standalone semantics retained** — multilingual stop phrases with punctuation tolerance now trigger abort flows; regression tests should include both valid standalone triggers and false-positive guard cases.

79. **Typing keepalive is now lifecycle-managed across longer runs** — long inference/tool replies should keep typing indicators active; test channel-specific keepalive teardown/cleanup paths.

80. **OpenRouter auth profiles bypass cooldown persistence** — if you depended on cooldown behavior for OpenRouter profiles, selection/fallback behavior now differs from other providers by design.

81. **Allowlisted model refs can be used even when bundled catalog is stale** — `models.list` and model selection now synthesize explicit allowlisted refs; stale catalog absence is no longer a blocker.

82. **Exec safe-bin trust defaults tightened** — implicit trust in mutable PATH-derived directories is reduced; review `tools.exec.safeBinTrustedDirs` and audit findings before assuming previous behavior.

### v2026.2.25 Specific

**BREAKING / OPERATIONAL SHIFTS (v2026.2.25):**

83. **Heartbeat `directPolicy` opt-in to block DMs** — v2026.2.25 reverts the heartbeat DM default back to `allow`. If you were relying on v2026.2.24's blocked-DM behavior, you **must** set `agents.defaults.heartbeat.directPolicy: "block"` (or per-agent `agents.list[].heartbeat.directPolicy: "block"`) explicitly. Silent regression: heartbeat runs may now deliver to DM targets that were previously blocked.

84. **Gateway WebSocket origin enforcement is stricter** — origin checks now apply to all direct browser WebSocket clients beyond just Control UI/Webchat. Password-auth failure throttling applies to browser-origin loopback attempts including `localhost`. Cross-origin WebSocket handshakes that previously succeeded may now be rejected. Affects custom browser integrations connecting to the gateway directly.

85. **macOS beta OAuth path removed** — the Anthropic OAuth sign-in and legacy `oauth.json` onboarding path (which exposed the PKCE verifier via OAuth `state`) have been removed. The macOS beta onboarding path is now setup-token-only. Any automation or tooling relying on `oauth.json` or the OAuth-state onboarding flow will break silently.

86. **Exec approval matching now bound to exact argv identity** — `system.run` approval matching is bound to exact argv and whitespace in rendered command text. Trailing-space executable path swaps and payload-only `rawCommand` mismatches for wrapper-carrier forms are now rejected. Exec workflows that relied on flexible approval matching (e.g., shell wrapper payloads without positional argv) need updating; symlink `cwd` paths and non-canonical executable argv are also rejected at spawn time.

87. **Workspace FS hardlink rejection** — `tools.fs.workspaceOnly` and `tools.exec.applyPatch.workspaceOnly` boundary checks (including sandbox mount-root guards) now reject in-workspace hardlinked file aliases pointing outside the workspace. Scripts or sandboxes that use hardlinks to alias workspace paths for cross-boundary access will fail closed.

88. **Reaction events require channel authorization** — Signal, Discord, Slack, and Telegram reaction events are now gated through full sender authorization (DM `dmPolicy`/`allowFrom`, channel `users` allowlists, group `groupPolicy`) before system-event enqueue. Previously, reaction events bypassed the authorization preflight that normal messages required. Any reaction-triggered workflows that depended on unauthorized senders delivering reactions will silently stop working.

89. **MS Teams file consent bound to originating conversation** — `fileConsent/invoke` upload acceptance and decline are now bound to the originating conversation before consuming pending uploads. Cross-conversation pending-file upload or cancellation via a leaked `uploadId` is blocked. Integrations that manually route Teams file consent invocations across conversations will break.

90. **Slack channel allowlist matching is now case-insensitive** — channel IDs in `groupPolicy: "allowlist"` config are matched case-insensitively. If you used uppercase channel IDs (e.g., `C0ABC12345`) in config expecting case-sensitive matching to intentionally exclude channels, that behavior is now gone. Verify your allowlist entries still produce the intended filtering.

91. **Slack `session.parentForkMaxTokens` guard added** — oversized parent-session inheritance is now capped by `session.parentForkMaxTokens` (default `100000`; `0` disables). Thread sessions forked from very large parent sessions will no longer silently inherit unbounded context. Workflows that relied on inheriting large parent contexts into thread sessions will be cut off at the cap unless the value is increased.

### v2026.3.1 Specific

**BREAKING CHANGES (v2026.3.1):**

92. **Node exec approval payloads now require `systemRunPlan`** — `host=node` approval requests without a structured `systemRunPlan` are rejected with `execution-plan-miss`. Node clients that submit bare `system.run` requests without the plan payload will fail closed. Update node-host integrations to include the plan generated by `buildSystemRunApprovalPlan()` in `invoke-system-run-plan.ts`.

93. **Node `system.run` pins path-token commands to canonical `realpath`** — both allowlist and approval execution flows now resolve token-form executable names (e.g. `tr`, `grep`) to their canonical path via `fs.realpathSync()` before matching. Tests that asserted token-form argv (e.g. `tr`) must accept canonical paths (e.g. `/usr/bin/tr`). Symlink-based executable paths and non-canonical cwds are also rejected at spawn time.

**BEHAVIORAL SHIFTS (v2026.3.1):**

94. **OpenAI Responses transport is WebSocket-first by default** — OpenAI and Codex Responses API calls now default to `transport: "auto"` which tries WebSocket first with SSE fallback. This applies to all `openai/*` and `codex/*` model refs. If WebSocket connectivity is unreliable (corporate proxies, restrictive firewalls), set `transport: "sse"` explicitly via model params or agent config. Per-session WebSocket cleanup is handled automatically.

95. **Claude 4.6 defaults to `adaptive` thinking** — Anthropic Claude 4.6 models (matched by `/claude-(?:opus|sonnet)-4(?:\.|-)6/i`, including Bedrock refs) now default to `thinking: "adaptive"` when no explicit thinking level is configured. Other reasoning-capable models remain at `low`. Override via `agents.defaults.thinkingDefault` globally, or per-model via `agents.defaults.models.<provider/model>.params.thinkingDefault`. The `ThinkLevel` type now includes `"adaptive"` as a valid enum value.

96. **Gateway WS flood protection closes unauthorized request floods** — repeated post-handshake `unauthorized role:*` request floods from a single misbehaving WebSocket connection are now actively closed with sampled rejection logs, preventing gateway responsiveness degradation. This builds on the existing v2026.2.19 loopback-only `ws://` enforcement. Docker/Kubernetes deployments should use the new built-in HTTP health endpoints (`/health`, `/healthz`, `/ready`, `/readyz`) instead of WebSocket probes where possible.

97. **Prompt spoofing hardening: subagent events route through typed `internalEvents`** — queued subagent/cron completion announcements are no longer injected as `[System Message]` text in user-role prompt context. They now route through typed `task_completion` internal events rendered via structured `internalEvents` plumbing. Spoof markers like `[System Message]` and `System:` in untrusted user content can no longer trick the agent into treating injected text as trusted system context. Code that parsed `[System Message]` blocks from prompt text must migrate to consuming `internalEvents` from the gateway/CLI event stream.

98. **Subagent delivery params in `sessions_spawn` are rejected with explicit errors** — passing channel-delivery parameters (`target`, `channel`, `to`, `threadId`, `replyTo`, `transport`) to `sessions_spawn` now returns an explicit input error: `"sessions_spawn does not support '<param>'. Use 'message' or 'sessions_send' for channel delivery."` Previously these params were silently ignored but could leak delivery intent to the parent conversation. Use `sessions_send` or the `message` tool for explicit channel delivery.

99. **Telegram DM topic sessions are scoped by `chatId:threadId`** — DM topic thread session keys now include the chat ID prefix (`<chatId>:<threadId>`) instead of bare thread IDs. This prevents parallel DMs from different chats from colliding when they share the same topic thread ID. Outbound recovery also parses scoped thread IDs. Existing DM topic sessions will create new session files under the scoped key format.

100. **Discord thread bindings use inactivity lifecycle instead of fixed TTL** — thread-bound sessions now expire based on inactivity (`idleHours`, default 24h) with an optional hard `maxAgeHours` cap, replacing the previous fixed-TTL mechanism. Old TTL config keys no longer apply. Use `/session idle` and `/session max-age` commands to configure per-thread lifecycle. Auto-unfocus messages are sent when idle timeout or max-age expires.

101. **Feishu system preview messages no longer leak into agent prompt context** — inbound Feishu message previews were previously enqueued as system events, causing user preview text to be injected into later turns as trusted `System:` content. This is now fixed: previews are no longer enqueued. If any workflow depended on preview text appearing in agent context, it will stop working.

102. **Cron `delivery.mode: "none"` now actually disables the agent messaging tool** — previously, setting `delivery.mode: "none"` on a cron job left the messaging tool available, allowing cron agents to send messages despite the `none` config. The messaging tool is now disabled when `delivery.mode` is `"none"` or when any delivery is explicitly requested. The cron editor also correctly sends `delivery: { mode: "none" }` for both add and update flows.

103. **Post-compaction audit injection message has been removed** — the Layer 3 post-compaction read audit system that enqueued warning messages after session compaction has been deleted. `post-compaction-audit.ts` and its test file are removed. Do not rely on post-compaction audit warnings being present in session context. Any code that checked for or parsed these audit injection messages must be updated.

104. **Telegram multi-account fallback is fail-closed for non-default accounts** — when route resolution for a non-default Telegram account (`accountId !== DEFAULT_ACCOUNT_ID`) falls back to `matchedBy=default`, the message is now dropped with reason `"non-default account requires explicit binding"`. This prevents cross-account DM/session contamination. Non-default Telegram accounts must have explicit route bindings configured to receive messages.

105. **Sessions lock recovery detects stale gateway PIDs via port liveness** — gateway lock recovery uses port-reachability as a primary stale-lock liveness signal (via `findGatewayPidsOnPortSync`). Stale locks from crashed processes are reclaimed by detecting dead PIDs and unreachable ports. On Linux, process start time comparison (from `/proc/<pid>/stat`) prevents PID-recycling false positives in containerized environments. The `restart-stale-pids.ts` module handles SIGTERM/SIGKILL cleanup with bounded timeouts.

### v2026.3.7 Specific

**BREAKING CHANGES (v2026.3.7):**

106. **`gateway.auth.mode` is now required when both token and password are configured** — If `gateway.auth.token` and `gateway.auth.password` are both set, `gateway.auth.mode` must be explicitly declared (`"token"` or `"password"`). Omitting `gateway.auth.mode` in this configuration causes a startup validation error (introduced v2026.3.7). Run `openclaw doctor --fix` to detect and auto-migrate. Configs with only one auth method configured are unaffected.

### v2026.3.31 Specific

**BREAKING CHANGES (v2026.3.31):**

114. **`nodes run` wrapper removed** — current released node shell execution goes through `exec host=node` and `nodes invoke`. If a runbook still references `nodes run`, update it before release validation.

115. **Plugin/skill installs now fail closed on critical scan findings** — treat security scanner failures as blocking unless the operator explicitly chooses `--dangerously-force-unsafe-install`.

116. **`trusted-proxy` mixed-token configs are invalid** — configs that combine incompatible shared-token paths now fail validation, and local-direct fallback requires the configured token.

117. **Node commands need pairing approval** — device pairing alone is no longer enough to expose declared node commands or broader node trust.

118. **Current browser runtime is extension-owned** — inspect `extensions/browser/src/browser/config.ts`, `extensions/browser/src/browser/chrome-mcp.ts`, `extensions/browser/src/browser/profile-capabilities.ts`, and `extensions/browser/src/browser/server-context.availability.ts` for released attach/runtime behavior.

119. **Task state is now first-class runtime data** — status, recovery, and detached-delivery investigations should read `src/tasks/*` in addition to the old cron/ACP/subagent paths.

120. **Remote MCP transports are part of the stable surface** — `mcp.servers` URL configs, auth headers, and provider-safe tool naming now matter for correctness reviews.

### v2026.4.9 Specific

**BREAKING CHANGES (v2026.4.9):**

135. **Legacy public config aliases are removed from the documented surface** - old public keys such as `talk.voiceId`, `talk.apiKey`, `agents.*.sandbox.perSession`, `browser.ssrfPolicy.allowPrivateNetwork`, `hooks.internal.handlers`, and channel/group/room `allow` toggles must be migrated to canonical paths and `enabled`. `openclaw doctor --fix` remains the supported migration path.

136. **Bundled CLI text-provider backends are no longer the release-line contract** - the old bundled CLI text-provider surface and `agents.defaults.cliBackends` are removed from the public surface. Native bundled providers and ACP harness sessions remain, but new code should not depend on the removed CLI-backend configuration path.

137. **Anthropic onboarding no longer treats Claude CLI as a new default backend** - new onboarding removes the Claude CLI backend/setup token from fresh Anthropic setup; existing configured legacy profiles still run, but stale `anthropic:claude-cli` state now relies on `openclaw doctor` migration/repair behavior.

**BEHAVIORAL SHIFTS (v2026.4.9):**

138. **The released memory surface is split, with host/runtime code in `src/memory-host-sdk/`** - dreaming, embeddings, Bedrock embeddings, replay-safe promotion, `dreams.md`, and related host/runtime logic live under `src/memory-host-sdk/`, while QMD/search/sync logic lives under `extensions/memory-core/src/memory/`. Old `src/memory/` references in historical docs are no longer the current path to inspect.

139. **Media generation is now a first-class runtime path** - `music_generate` and `video_generate` are built-in tools with async completion and optional direct-send delivery. Changes now span agent tools, provider transport, media persistence, and outbound delivery, so testing only the tool schema is insufficient.

140. **OpenAI/GPT replies now buffer commentary until `final_answer`** - planning text should no longer leak into user-visible channels. GPT-5/Codex defaults are shorter and lower-verbosity, and planning-only turns get a one-shot retry. If a change affects reply delivery or Responses compatibility, verify commentary/final separation explicitly.

### v2026.4.1 Specific

**BEHAVIORAL SHIFTS (v2026.4.1):**

121. **Exec `allow-always` is now durable trust** — previously `allow-always` behaved as `allow-once` and was not persisted across restarts. After v2026.4.1, `allow-always` writes durable user-approved trust to `exec-approvals.json`. Shell-wrapper paths that cannot safely persist an executable allowlist entry reuse exact-command trust instead. Static allowlist entries no longer silently bypass `ask:"always"` policy. If your setup relied on `allow-always` being ephemeral, review persisted approvals after upgrade.

122. **Slack and Discord native exec approval routing** — exec approval prompts can now stay in Slack or Discord instead of falling back to Web UI or terminal. Approvers are inferred from existing owner config when `execApprovals.approvers` is unset, and the default approval window is extended to 30 minutes. If you have custom approval routing, verify it still takes precedence.

123. **`agents.defaults.params` applies globally** — global default provider parameters now merge into all provider calls unless overridden per-agent or per-model. If you have provider-specific params that should not apply globally, move them to per-agent or per-model config.

124. **Auth failover caps same-provider retries** — rate-limit failures now cap same-provider auth-profile rotations (via `auth.cooldowns.rateLimitedProfileRotations`) before triggering cross-provider model fallback. Multi-profile setups that previously retried indefinitely within one provider will now fall over sooner.

125. **Telegram `errorPolicy` suppresses repeated errors** — configurable `errorPolicy` and `errorCooldownMs` controls suppress repeated delivery errors per account, chat, and topic without muting distinct failures. If you relied on seeing every error in chat, set `errorPolicy: "always"`.

126. **Task-registry maintenance sweep is async** — the synchronous SQLite maintenance sweep that previously stalled the gateway event loop is now asynchronous with current-state rechecks. If you have custom task-registry extensions, ensure they do not assume synchronous sweep completion.

127. **Webchat exec approvals use native UI** — agent system prompts in webchat sessions now reference native approval UI instead of manual `/approve` commands. If you have custom system prompt templates that reference `/approve`, update them for webchat contexts.

128. **WhatsApp `reactionLevel` guidance** — agents can now react with emoji on incoming WhatsApp messages based on `reactionLevel` config. Default behavior may include reactions where none existed before. Set `reactionLevel: "none"` to suppress.

### v2026.4.2 Specific

**BREAKING CHANGES (v2026.4.2):**

129. **xAI `x_search` config path moved to plugin-owned** — `tools.web.x_search.*` is now `plugins.entries.xai.config.xSearch.*`. Legacy config at the old path will fail validation. Run `openclaw doctor --fix` to migrate. xAI `x_search` auth is standardized on `plugins.entries.xai.config.webSearch.apiKey` / `XAI_API_KEY`.

130. **Firecrawl `web_fetch` config path moved to plugin-owned** — `tools.web.fetch.firecrawl.*` is now `plugins.entries.firecrawl.config.webFetch.*`. Legacy config at the old path will fail validation. Run `openclaw doctor --fix` to migrate. Web fetch fallback now routes through the fetch-provider boundary instead of Firecrawl-only core branch.

**BEHAVIORAL SHIFTS (v2026.4.2):**

131. **Provider transport is centralized across HTTP/stream/websocket** — request auth, proxy, TLS, and header shaping are now shared across all three transport modes. Insecure TLS/runtime transport overrides are blocked. Proxy-hop TLS is separate from target mTLS settings. If you change provider request behavior, you must test all three transport paths (HTTP, SSE stream, websocket). Attribution headers are merged consistently across embedded-runner, proxy stream, and OpenAI websocket paths.

132. **Exec defaults to YOLO mode** — gateway/node host exec now defaults to `security=full` with `ask=off` (no approval prompt). Host approval-file fallbacks and doctor reporting align with this no-prompt default. If your setup previously relied on interactive approval prompts as a security gate, review exec policy after upgrade. Windows builds and node hosts are affected.

133. **`before_agent_reply` hook can short-circuit LLM** — plugins implementing `before_agent_reply` can return synthetic replies, bypassing the LLM entirely after inline actions. If you change reply pipeline dispatch, verify that `before_agent_reply` hooks are evaluated before LLM invocation and that short-circuit replies are delivered correctly.

134. **Task Flow managed child spawning with sticky cancel** — external orchestrators can stop scheduling immediately via sticky cancel intent, and parent Task Flows settle to `cancelled` once active child tasks finish. If you change task cancellation or flow lifecycle, verify that cancel intent propagates correctly through managed child hierarchies and that parent flows do not settle prematurely.

### v2026.3.28 Specific

**BREAKING CHANGES (v2026.3.28):**

107. **Qwen portal auth is gone — use Model Studio** — `qwen-portal-auth` plugin and associated OAuth flow are removed. Configs that used `qwen-portal-auth` will fail validation at startup. Migrate: run `openclaw onboard --auth-choice modelstudio-api-key` and remove any `qwen-portal-auth` plugin entry from your config.

108. **Doctor no longer auto-migrates configs older than 2 months** — migrations with a cutoff date older than 2 months from the current release are dropped. Config keys requiring those migrations will fail Zod validation. Run `openclaw doctor --fix` before upgrading if you haven't migrated recently; do not assume silent backward compat.

109. **`--claude-cli-logs` flag deprecated** — replaced by `--cli-backend-logs`. Any scripts, runbooks, or CI configs referencing `--claude-cli-logs` should be updated; the old flag still exists as a compatibility alias on the released line.

110. **MiniMax: only M2.7 model available** — M2, M2.1, M2.5, and VL-01 have been removed from the MiniMax catalog. Configs or agent definitions referencing these model IDs will fail model resolution. Update to `minimax/m2.7`.

**OPERATIONAL NOTES (v2026.3.28):**

111. **`src/mcp/` is a new gateway-backed module** — channel-server, channel-bridge, and channel-tools are the three components. If MCP client connections are failing, check that gateway startup completes before the MCP stdio listener is attached. The channel-tools surface is registered via the bundled plugin allowlist (`withBundledPluginAllowlistCompat`).

112. **xAI Responses API migration** — xAI/Grok provider now uses the Responses API exclusively. `x_search` and `code_execution` are plugin-owned tools (auto-enabled from config). If you have custom tool-policy rules that reference xAI tools by name, verify they still apply under the plugin-owned surface. The xAI plugin is auto-enabled when an xAI auth profile is configured.

113. **LINE timing-safe HMAC** — LINE webhook signature verification uses `crypto.timingSafeEqual`. Do not replace the comparison with string equality in any fork/patch; timing-oracle attacks against byte-length leakage are a real threat on HMAC verification.

---

## 10. PR & Bug Filing Best Practices

### Multi-Model Review

- Use multiple models for PR review when complexity warrants it. Different models catch different classes of bugs: one focuses on correctness (happy path), another on failure paths (what happens when recovery itself fails). At minimum, use one model for "does it work?" and another for "what breaks?"

### Test Matrices

- **Test across ALL config permutations.** The unified `streaming` config uses a 4-value enum (`off` / `partial` / `block` / `progress`) across all channels, each with different code paths. A fix for one mode can break another. Build a test matrix covering every mode × every failure scenario before claiming a fix is complete.
- File bugs with full test matrices. Include: reproduction steps for each mode, all root causes identified, and a gap matrix showing which PRs fix which scenarios. Detailed issues attract better reviews and prevent partial fixes (e.g., #19001).

### Issue & PR Workflow

- **Search existing issues before filing.** A quick `gh issue list --search "<keywords>"` surfaces prior analysis and avoids duplicate effort. Issue #18244 (Telegram message loss) was found only after deep investigation - a search would have saved hours.
- **One PR, multiple root causes = scope risk.** A PR fixing 3 distinct failure modes (eval order, failed delivery tracking, cleanup timing) is harder to review even if each fix is independently correct. Consider whether splitting gets faster review vs. the coherence benefit of a single fix.
- **Scope PRs to one logical change when possible.** If root causes are independent, separate PRs are easier to review, revert, and bisect.
- **Call out behavior-default shifts explicitly in PR descriptions.** If a release changes defaults (for example cron stagger, include confinement, tool streaming), include a short "old assumption vs new behavior" note so reviewers can validate migration risk quickly.
- **Maintainer flow is ordered and explicit.** Use `review-pr` -> `prepare-pr` -> `merge-pr`, and do not skip stages.
- **Use script-first wrappers in maintainer flow.** Prefer `scripts/pr-review`, `scripts/pr-prepare`, and `scripts/pr-merge`; treat manual low-level runs as debugging-only.
- **Rebase is mandatory before substantive review/prep.** Rebase PR branch onto current `main` first, resolve conflicts, then evaluate correctness.
- **Resolve all BLOCKER/IMPORTANT findings before merge.** Treat review artifacts as requirements, not suggestions.
- **Required maintainer artifacts must exist.** Ensure `.local/pr-meta.json`, `.local/pr-meta.env` (from review init), `.local/review.json`, `.local/review.md` (from review output), `.local/prep-context.env`, `.local/prep.md`, and `.local/prep.env` (from prepare) are present before merge.
- **Stop if you cannot verify the fix.** If the problem cannot be reproduced or there is no meaningful verification path, escalate instead of merging.
- **For AI-assisted PRs, require transparency.** Mark AI assistance in the PR title or description, state testing depth (untested / lightly tested / fully tested), include prompts or session logs when possible, and confirm you understand what the code does.

### Documentation Update Guardrails (from recent failures)

- **Run doc checks before push, not after CI fails.** For docs-only edits, run all three locally: `pnpm check:docs`, `pnpm format:check -- <changed-doc>`, and `npx markdownlint-cli2 <changed-doc>`.
- **Watch for markdownlint table/fence traps.** `MD060` (table pipe alignment) and `MD031` (blank lines around fenced blocks) are easy to miss in long docs and were recent CI failure causes.
- **Verify command names against source, never memory.** Before documenting commands, confirm in `package.json`, `CONTRIBUTING.md`, and `.github/workflows/ci.yml` to avoid stale/wrong instructions.
- **Keep comments shell-safe when posting with `gh pr comment`.** Backticks in inline shell strings can be evaluated by the shell; prefer plain text, single-quoted heredoc, or escaped backticks to avoid mangled comments and duplicate reposts.
- **After conflict resolution, run type checks.** Merge conflict fixes can drop `import type` lines; tests and lint may still pass. Run `pnpm check` to catch `tsgo` regressions before push.

### Documentation Link & i18n Guardrails

- **Never remove the only link to authoritative docs without replacement.** Deleting a cross-reference that is the sole path to a config reference orphans readers. Verify no other navigation path exists first.
- **Use heredoc or `--raw-field` for JSON in CLI examples.** `gh api graphql -F input='{}'` is brittle across shells. Prefer `@file.json` or heredoc.
- **Ensure trailing newline when writing/converting text files.** Symlink→regular-file conversion and programmatic writes often drop the POSIX newline, causing format failures and noisy diffs.
- **Apply style rules to ALL locale variants.** AGENTS.md rules (no emojis in headings, etc.) apply to `docs/zh-CN`, `docs/ja-JP`, etc. Run the same checks on all locales.
- **Edit i18n docs via pipeline, not directly.** `docs/zh-CN/**` is generated by `scripts/docs-i18n`. Manual edits get overwritten. Workflow: update English → glossary → i18n pipeline → targeted fixes.

### Code & Gateway Safety Patterns (from recent PR reviews)

- **Atomic writes for critical config files.** `writeFileSync()` truncates then writes —
  a SIGTERM mid-write leaves an empty file. Use write-to-tmp + `fdatasyncSync` + rename
  for all critical config files (`auth-profiles.json`, lock files, etc.). A backup `.bak`
  before write provides a recovery path. (ref: #23980)

- **Never mutate live state before validation passes.** `cron.update` mutated the live
  cron job object before patch validation succeeded, leaving the job in a partially mutated
  state when the patch was rejected. Always validate on a copy, then apply to live state
  atomically if validation passes. (ref: #26852)

- **Error catch blocks must enumerate all thrower message strings.** Gateway handlers that
  classify exceptions as `INVALID_REQUEST` vs `UNAVAILABLE` by string matching must cover
  every error message emitted by all called functions. A single missing pattern (e.g.
  `"unknown cron job id"` from `findJobOrThrow`) silently routes client errors as server
  errors — obscuring root cause for both client and operator. (ref: #26852)

- **Wide catch blocks in gateway handlers must not suppress server-side errors.** Converting
  all exceptions to `UNAVAILABLE` without logging or rethrowing non-client failures removes
  operator visibility into server faults. Only intercept known client validation error
  patterns; rethrow (or explicitly log) anything else so the outer WS handler can emit
  `request handler failed`. (ref: #26852)

- **Config-declared transport ≠ runtime-active trust.** Bypassing the `ws://` plaintext
  check based on `gateway.remote.transport === "ssh"` is only safe if a tunnel is actually
  established at runtime. In CLI mode SSH tunnels are NOT auto-started — only the macOS app
  manages `RemoteTunnelManager`. Any security-check bypass conditioned on a config flag must
  verify the corresponding invariant is operationally active, not merely declared. (ref: #26573)

- **Platform-guard Linux `/proc` paths in tests.** Tests that read
  `/proc/sys/kernel/random/boot_id`, `/proc/<pid>/stat`, or other procfs paths must guard
  with `if (process.platform !== "linux") { return; }`. On macOS `/proc` doesn't exist and
  PID 1 (launchd) is always alive, silently skewing stale-lock detection tests.
  Follow the pattern established in `pid-alive.test.ts`. (ref: #27272)

- **Test provider-specific params on ALL API mode paths.** When a provider supports multiple
  API modes (native + OpenAI-compat), provider-specific request params (e.g. `num_ctx` for
  Ollama, which prevents the server-side 4096-token cap) must be injected on each path
  independently — they share config but run through separate injection chains. (ref: #27292)

- **macOS LaunchAgent crash-only keepalive + proxy passthrough.** `KeepAlive: true` causes
  infinite restart loops on clean exits. Use `KeepAlive.SuccessfulExit: false` +
  `ThrottleInterval: 5` (crash-only restart with 5-second backoff). Additionally: macOS
  strips proxy env vars (`HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`, `ALL_PROXY` and lowercase
  variants) from LaunchAgent environments — forward them explicitly in the plist
  `EnvironmentVariables` block. (ref: #27276)

### Control UI Development

- **Use legacy decorators only.** The Control UI uses Lit with legacy decorators — the
  current Rollup build does not support `accessor` fields required by standard decorators.
  Use `@state() foo = "bar"` and `@property({ type: Number }) count = 0` syntax. The root
  `tsconfig.json` sets `experimentalDecorators: true` with `useDefineForClassFields: false`.
  Do not flip these settings unless you are also updating the UI build tooling to support
  standard decorators.

---

## 11. Debugging & Triage Quick Workflow

1. Reproduce once with minimal scope.
2. Map symptom to module using §0 + §1 + §2.
3. Check matching impact row in §3 and relevant gotchas in §9.
4. Run targeted tests first, then full suite if touching 🔴/high-blast modules.
5. Confirm fix did not regress fallback paths (cron, channel send, subagent completion).

For ambiguous runtime failures: capture exact error string, map via "Common Errors -> First Fix", then inspect nearest owning module.

---

## 12. Repository Structure

### Top-Level Layout

```
openclaw/
├── src/                     # Main TypeScript source (see §1 for module map)
├── extensions/              # Channel plugin extensions (pnpm workspace packages)
│   ├── bluebubbles/         # BlueBubbles iMessage
│   ├── msteams/             # Microsoft Teams
│   ├── matrix/              # Matrix
│   ├── zalo/ zalouser/      # Zalo
│   ├── voice-call/          # Voice call
│   ├── discord/ telegram/   # Additional channel extensions
│   ├── slack/ signal/       # Additional channel extensions
│   ├── whatsapp/            # WhatsApp
│   └── ...                  # 30+ total; full list: extensions/
├── apps/                    # Native apps
│   ├── ios/                 # iOS app (Swift/SwiftUI)
│   ├── macos/               # macOS menubar app (Swift/SwiftUI)
│   └── android/             # Android app (Kotlin/Gradle)
├── docs/                    # Mintlify documentation (docs.openclaw.ai)
│   ├── zh-CN/               # Auto-generated Chinese translation — do NOT edit directly
│   └── .i18n/               # i18n pipeline config and glossaries
├── ui/                      # Control UI (Lit web components, legacy decorators)
├── scripts/                 # Maintainer workflow and build scripts
│   ├── committer            # Scoped commit helper (use instead of git add/commit)
│   ├── pr-review            # PR review script (maintainer flow)
│   ├── pr-prepare           # PR prepare script
│   ├── pr-merge             # PR merge script
│   └── clawlog.sh           # Query macOS unified logs for OpenClaw subsystem
├── skills/                  # Bundled skills shipped with the package
├── packages/                # Internal workspace packages
├── vendor/                  # Vendored dependencies
├── dist/                    # Built output (gitignored; generated by pnpm build)
├── AGENTS.md                # Canonical agent/AI guidelines (see §15)
├── CLAUDE.md                # Symlink → AGENTS.md
├── CONTRIBUTING.md          # Contributor guide
├── CHANGELOG.md             # User-facing release notes
├── package.json             # Root package — version, scripts, deps
├── pnpm-workspace.yaml      # pnpm workspace config
├── vitest.config.ts         # Default vitest config
├── vitest.unit.config.ts    # Unit test config (used by pnpm test:coverage)
├── vitest.e2e.config.ts     # E2E test config
├── vitest.gateway.config.ts # Gateway integration test config
├── vitest.live.config.ts    # Live (real API) test config
├── vitest.extensions.config.ts # Extension/plugin test config
├── tsconfig.json            # TypeScript config (legacy decorators enabled for Control UI)
└── tsdown.config.ts         # Build bundler config
```

### Source Modules (`src/`)

Core built-in channels (leaf modules, 🟢 risk):

```
extensions/telegram/   extensions/discord/   extensions/slack/      extensions/signal/
extensions/imessage/   extensions/whatsapp/  extensions/feishu/     extensions/matrix/
extensions/qqbot/      extensions/browser/   src/channels/          src/routing/
src/line/              src/tasks/            src/web-search/        src/web-fetch/
```

Additional source modules beyond §1's map:

| Module | Purpose |
| --- | --- |
| `acp/` | Agent Communication Protocol server |
| `canvas-host/` | Canvas/A2UI host integration |
| `link-understanding/` | URL/link resolution and understanding |
| `markdown/` | Markdown processing and rendering |
| `node-host/` | Node.js host process management |
| `pairing/` | Account-scoped device pairing |
| `plugin-sdk/` | Public plugin SDK exports |
| `terminal/` | Terminal UI utilities (table, palette, progress) |
| `tts/` | Text-to-speech |
| `tui/` | Terminal UI (TUI) |
| `web-fetch/` | Web fetch provider routing and runtime |
| `wizard/` | Onboarding wizard |
| `daemon/` | Daemon management |

Extension channels live in `extensions/` as separate pnpm workspace packages. Keep extension-only deps in the extension's own `package.json`; do not add them to root `package.json`.

### Version Numbering

Format: `YYYY.M.D` — year, month, day of release.

- `v2026.2.26` = stable release
- `v2026.2.26-beta.1` = prerelease (npm dist-tag `beta`)
- `main` branch = dev head (no tag)

Version must be bumped in all locations simultaneously: `package.json`, `apps/android/app/build.gradle.kts`, `apps/ios/Sources/Info.plist`, `apps/macos/Sources/OpenClaw/Resources/Info.plist`, and `docs/install/updating.md`. See `AGENTS.md` for the full list.

---

## 13. Development Environment Setup

### Prerequisites

| Requirement | Notes |
| --- | --- |
| **Node.js 22+** | No `.nvmrc`; enforce via asdf or nvm. Both Node and Bun paths must stay working. |
| **pnpm** | Primary package manager. Lock file is `pnpm-lock.yaml`. |
| **Bun** | Preferred for dev/test script execution (`bunx vitest`, `pnpm dev`). Keep `pnpm-lock.yaml` in sync when touching deps. |
| **Swift toolchain** | Required for `pnpm check` (`check:host-env-policy:swift`) and macOS/iOS app builds. |
| **Xcode** | Required for iOS/macOS development. |

### Bootstrap

```bash
# 1. Install dependencies
pnpm install

# 2. Install pre-commit hooks (runs same checks as CI)
prek install

# 3. Build
pnpm build
```

If `node_modules` is missing or a command fails with "not found", run `pnpm install` first, then retry.

### Running the CLI Locally

```bash
# Run CLI via Bun (dev mode — no build step required)
pnpm openclaw <command>

# Example
pnpm openclaw config get agents

# Alt: run built output directly
node dist/index.js <command>
```

### Gateway (macOS)

The gateway runs as the macOS menubar app. Start/stop via the app, not ad-hoc tmux sessions.

```bash
# Restart gateway via helper script
scripts/restart-mac.sh

# Kill + restart manually on server/VM (not macOS app)
pkill -9 -f openclaw-gateway || true
nohup openclaw gateway run --bind loopback --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &

# Verify
openclaw channels status --probe
tail -n 120 /tmp/openclaw-gateway.log
```

### Full Quality Gate (CI Equivalent)

```bash
pnpm build          # TypeScript compilation + bundling
pnpm check          # Format + type check + lint (see breakdown below)
pnpm test           # Full test suite (parallel)
pnpm check:docs     # Required when any docs file changes
```

`pnpm check` runs this chain:

```
oxfmt --check                          # format check
pnpm tsgo                              # TypeScript type check
oxlint --type-aware                    # lint
scripts/check-no-random-messaging-tmp.mjs
scripts/check-channel-agnostic-boundaries.mjs
scripts/check-no-raw-channel-fetch.mjs
scripts/check-no-pairing-store-group-auth.mjs
scripts/check-pairing-account-scope.mjs
scripts/generate-host-env-security-policy-swift.mjs --check
```

Run `pnpm check` before every push — not just on changed files. This is the CI gate.

### Protocol Schema Sync

When changing `src/gateway/protocol/schema/**`:

```bash
pnpm protocol:gen        # Regenerate dist/protocol.schema.json
pnpm protocol:gen:swift  # Regenerate apps/macos/Sources/OpenClawProtocol/GatewayModels.swift
pnpm protocol:check      # Verifies both files match (fails CI if out of sync)
```

Forgetting `pnpm protocol:gen:swift` after a schema change will fail `protocol:check` on every PR that rebases onto your commit.

### macOS Log Query

```bash
./scripts/clawlog.sh          # Query unified macOS logs for OpenClaw subsystem
./scripts/clawlog.sh -f       # Follow (tail) mode
```

---

## 14. Key CLI Commands

Quick reference for commands mentioned throughout this document.

| Command | Purpose |
| --- | --- |
| `openclaw doctor` | Diagnose config and service issues; run first after any upgrade or when something breaks |
| `openclaw doctor --fix` | Auto-fix detected issues (config migration, path rebinding, stale state) |
| `openclaw security audit` | Security audit — required gate before any security-adjacent PR |
| `openclaw gateway install` | Install / reinstall gateway as a system service |
| `openclaw channels status --probe` | Verify channel connectivity |
| `openclaw config set <key> <value>` | Set a config key |
| `openclaw config get <key>` | Read a config key |
| `openclaw config patch <path> <json>` | Patch nested config path — always read-back verify after |
| `openclaw message send ...` | Send a test message |

### State Directory (`~/.openclaw/`)

| Path | Contents |
| --- | --- |
| `openclaw.json` | Main config (JSON5) |
| `openclaw.json.bak` | Auto-backup before patches |
| `agents/<id>/agent/` | Per-agent runtime state, auth profiles |
| `agents/<id>/sessions/` | Session JSONL logs (`*.jsonl`) |
| `agents/<id>/qmd/` | QMD memory collections (per-agent isolated) |
| `cron/` | Cron job state store |
| `credentials/` | Web provider credentials |
| `devices/` | Device pairing state |
| `delivery-queue/` | Outbound delivery queue |
| `exec-approvals.json` | Exec approval records |
| `memory/` | Memory / QMD index |
| `browser/` | Browser session data |
| `canvas/` | Canvas / A2UI state |
| `logs/` | Log files |
| `media/` | Media cache |
| `skills/` | Installed skills |
| `workspace/` | Agent workspace files |

### Commit Helper

Use `scripts/committer "<msg>" <file...>` instead of manual `git add` + `git commit`. This keeps staging scoped to your changes and avoids accidentally including unrelated files — especially important in multi-agent sessions.

```bash
scripts/committer "fix: correct session key parsing" src/routing/session-key.ts
```

---

## 15. AGENTS.md and CLAUDE.md

`AGENTS.md` at the repo root is the **canonical AI/agent guidelines file**. `CLAUDE.md` is a symlink pointing to it (`ln -s AGENTS.md CLAUDE.md`).

**Always load `AGENTS.md` before starting work.** It is the authoritative source for:

- Project structure and module organization
- Build, test, and dev commands (the source of truth over this document for commands)
- Coding style and naming conventions (TypeScript ESM, strict typing, no `any`, no `@ts-nocheck`)
- Commit and PR guidelines (pointer to `.agents/skills/PR_WORKFLOW.md` for full maintainer flow)
- Agent-specific operational notes: macOS gateway, multi-agent safety invariants, vocabulary
- Security and configuration tips
- Platform-specific notes (iOS/macOS Observation framework, SwiftUI, Android)

### Key Rules from AGENTS.md

- File references in chat/comments: repo-root relative only — never absolute paths or `~/...`
- GitHub issue/PR comments: use heredoc (`-F - <<'EOF'`) not `-b "..."` when body contains backticks
- Never edit `node_modules` — not even for a quick fix
- Never commit real phone numbers, API keys, or live config values
- Any dependency in `pnpm.patchedDependencies` must use an exact version (no `^`/`~`)
- When adding a new `AGENTS.md` in any sub-package, also create a `CLAUDE.md` symlink there

### i18n Docs

`docs/zh-CN/**` is auto-generated — do not edit directly. The same AGENTS.md style rules (no emojis in headings, no absolute paths, etc.) apply to all locale variants including `docs/zh-CN/` and `docs/ja-JP/`. Workflow: update English docs → adjust `docs/.i18n/glossary.zh-CN.json` → run `scripts/docs-i18n` → targeted fixes only if instructed.

---

## 16. Coding Style Rules

> Source of truth: `AGENTS.md`. Repeated here for quick reference.

### Language & Tooling

- **TypeScript (ESM)**. Strict typing. Avoid `any`.
- Formatting: **Oxfmt** (`pnpm format` / `pnpm format:fix`)
- Linting: **Oxlint** with `--type-aware` (`pnpm lint` / `pnpm lint:fix`)
- Never add `@ts-nocheck`. Never disable `no-explicit-any`. Fix root causes.
- Never share class behavior via prototype mutation. Use explicit inheritance/composition.

### Naming

- **OpenClaw** for product/app/docs headings
- **openclaw** for CLI command, package/binary, paths, config keys
- File references in chat/comments/PRs: **repo-root relative only** — never absolute paths or `~/...`

### Code Quality

- Add brief comments for tricky/non-obvious logic only — do not add comments to code you didn't change.
- Keep files under ~500 LOC (guideline). Split/refactor when it improves clarity or testability.
- CLI progress: use `src/cli/progress.ts` (`osc-progress` + `@clack/prompts`). Do not hand-roll spinners.
- Status output: use `src/terminal/table.ts` with shared palette from `src/terminal/palette.ts`.

### Dependencies

- Any dependency in `pnpm.patchedDependencies` must use **exact version** (no `^`/`~`).
- Patching dependencies (pnpm patches, overrides, vendored changes) requires **explicit approval**.
- **Never edit `node_modules`** — not even for a quick fix. Updates overwrite everything.
- **Never update the Carbon dependency** — explicit prohibition.

### Maintainers & Ownership

See `CONTRIBUTING.md` for the full maintainer list with GitHub handles and area ownership. Key areas:
- Shadow — Discord, community moderation
- Jos — Telegram, API, Nix
- Tyler — Agents/subagents, cron, BlueBubbles, macOS app
- Vincent — Agents, Telemetry, Hooks, Security
- Josh Avant — Core, CLI, Gateway, Security, Agents

---

## 17. Multi-Agent Safety Rules

> Critical for agents operating in parallel. Source: `AGENTS.md`.

### Invariants (Never Violate)

1. **Do NOT** create/apply/drop `git stash` entries unless explicitly requested (includes `git pull --rebase --autostash`).
2. **Do NOT** create/remove/modify `git worktree` checkouts unless explicitly requested.
3. **Do NOT** switch branches or check out a different branch unless explicitly requested.
4. When user says **"push"** → you may `git pull --rebase` to integrate latest changes. Never discard other agents' work.
5. When user says **"commit"** → scope to your changes only.
6. When user says **"commit all"** → commit everything in grouped chunks.
7. When you see **unrecognized files** → keep going. Focus on your changes and commit only those.

### Commit Workflow

Always use `scripts/committer "<msg>" <file...>` instead of `git add` + `git commit`. This keeps staging scoped to your specific changes and prevents accidentally including other agents' in-progress work.

### Focus Reports

- Focus reports on your edits only.
- Avoid guard-rail disclaimers unless truly blocked.
- When multiple agents touch the same file, continue if safe.
- End with a brief "other files present" note only if relevant.

### Lint/Format Churn

- If staged+unstaged diffs are formatting-only → auto-resolve without asking.
- If commit/push already requested → auto-stage formatting fixes in the same or a tiny follow-up commit.
- Only ask when changes are **semantic** (logic/data/behavior).
