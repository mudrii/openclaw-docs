# OpenClaw — Master Architecture Document

> Generated: 2026-02-15 | Comprehensive reference for contributors

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Architecture Diagram](#2-system-architecture-diagram)
3. [Module Catalog](#3-module-catalog)
4. [Dependency Graph](#4-dependency-graph)
5. [Data Flow: Message Lifecycle](#5-data-flow-message-lifecycle)
6. [Data Flow: Cron Job Execution](#6-data-flow-cron-job-execution)
7. [Data Flow: Tool Execution](#7-data-flow-tool-execution)
8. [Configuration Architecture](#8-configuration-architecture)
9. [Security Model](#9-security-model)
10. [Storage & Persistence](#10-storage--persistence)
11. [Plugin System](#11-plugin-system)
12. [Provider System](#12-provider-system)
13. [Channel Abstraction](#13-channel-abstraction)
14. [Key Design Patterns](#14-key-design-patterns)
15. [Cross-Module Statistics](#15-cross-module-statistics)

---

## 1. Executive Summary

**OpenClaw** is an open-source, self-hosted AI agent platform that connects Large Language Models to messaging channels (Telegram, Discord, Slack, WhatsApp, Signal, iMessage, LINE, IRC, and more). It runs as a persistent gateway daemon on macOS/Linux/Windows, accepting messages from any connected channel, routing them to configured AI agents, executing tool calls on behalf of the agent, and delivering responses back to users. OpenClaw supports multi-agent configurations, per-channel routing, sandboxed execution environments (Docker), browser automation, semantic memory search, scheduled cron jobs, mobile node pairing, and a rich plugin/extension ecosystem.

Architecturally, OpenClaw follows a **hub-and-spoke model**: the `gateway` module is the central server process that orchestrates all subsystems. It exposes a WebSocket JSON-RPC API for CLI/TUI clients, an OpenAI-compatible HTTP API, and channel plugin connections. The `config` module provides the foundation — nearly every module depends on it for typed configuration. The `agents` module (the largest at ~530 files) contains the AI runtime: model selection, system prompt construction, tool registration, streaming response processing, sandbox management, and the embedded pi-agent integration (`@mariozechner/pi-ai`). The `auto-reply` module (~223 files) is the message processing pipeline that sits between channels and agents — handling commands, directives, session management, model routing, queue management, and reply delivery.

The codebase is written entirely in TypeScript (Node.js), uses Vitest for testing, and employs a plugin architecture where messaging channels, LLM providers, and feature extensions are loaded dynamically from an `extensions/` directory. Configuration is stored in `openclaw.json` (JSON5), validated via Zod schemas, and supports hot-reload. The system is designed for single-user or small-team self-hosting with strong security defaults: exec approval workflows, tool policies, SSRF protection, timing-safe auth, and filesystem permission hardening.

---

## 2. System Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL SERVICES                                   │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │ Telegram  │ │ Discord  │ │  Slack   │ │ WhatsApp │ │  Signal  │ │  LINE    │ │
│  │  Bot API  │ │ Gateway  │ │ Socket   │ │ Baileys  │ │signal-cli│ │  SDK     │ │
│  └────┬──────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ │
└───────┼──────────────┼───────────┼────────────┼────────────┼────────────┼────────┘
        │              │           │            │            │            │
        ▼              ▼           ▼            ▼            ▼            ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         CHANNEL PLUGINS (extensions/)                             │
│  Each implements ChannelPlugin interface: monitor, send, normalize, onboard      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │telegram/ │ │discord/  │ │ slack/   │ │whatsapp/ │ │ signal/  │ │  line/   │ │
│  │(grammY)  │ │(carbon)  │ │(web-api) │ │(baileys) │ │(RPC/SSE) │ │(bot-sdk) │ │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ │
└───────┼──────────────┼───────────┼────────────┼────────────┼────────────┼────────┘
        │              │           │            │            │            │
        └──────────────┴───────────┴─────┬──────┴────────────┴────────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        GATEWAY SERVER (src/gateway/)                             │
│                                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │ HTTP Server  │  │ WebSocket    │  │ OpenAI-compat│  │  Server Methods      │ │
│  │ (Express)    │  │ JSON-RPC     │  │ HTTP API     │  │  (30+ RPC handlers)  │ │
│  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘ │
│         │                │                  │                     │             │
│  ┌──────┴──────────────────────────────────────────────────────────┐            │
│  │                    SERVER SUBSYSTEMS                             │            │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────────┐ │            │
│  │  │ Cron   │ │Browser │ │ Nodes  │ │Plugins │ │  Auth/Rate   │ │            │
│  │  │Service │ │Control │ │Registry│ │Lifecycle│ │  Limiting    │ │            │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └──────────────┘ │            │
│  └────────────────────────────────┬────────────────────────────────┘            │
└───────────────────────────────────┼─────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌──────────────────────┐  ┌──────────────┐  ┌──────────────────────┐
│  ROUTING (src/routing)│  │  AUTO-REPLY   │  │   SESSIONS           │
│  resolveAgentRoute()  │  │ (src/auto-    │  │ (src/sessions/)      │
│  buildSessionKey()    │  │  reply/)      │  │ send-policy, keys,   │
│  bindings, DM scope   │──▶ Pipeline:    │  │ transcript events    │
└──────────────────────┘  │ dispatch →    │  └──────────────────────┘
                          │ directives →  │
                          │ commands →    │
                          │ agent run →   │
                          │ delivery      │
                          └───────┬───────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      AGENTS MODULE (src/agents/)  ~530 files                    │
│                                                                                 │
│  ┌─────────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ PI Embedded      │  │ System Prompt│  │Model Selection│  │  Auth Profiles  │ │
│  │ Runner (LLM call)│  │ Builder      │  │& Fallback    │  │  & OAuth        │ │
│  └────────┬────────┘  └──────────────┘  └──────────────┘  └──────────────────┘ │
│           │                                                                     │
│  ┌────────▼────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ Tool Registry    │  │  Sandbox     │  │  Skills      │  │  Subagent       │ │
│  │ 40+ tools:       │  │  (Docker)    │  │  System      │  │  Registry       │ │
│  │ exec, browser,   │  └──────────────┘  └──────────────┘  └──────────────────┘ │
│  │ web, memory,     │                                                           │
│  │ message, cron,   │                                                           │
│  │ canvas, nodes... │                                                           │
│  └─────────────────┘                                                            │
└──────────────────────────────────────┬──────────────────────────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                  ▼
           ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
           │   MEMORY      │  │    CRON       │  │    HOOKS         │
           │ (src/memory/) │  │ (src/cron/)   │  │ (src/hooks/)     │
           │ SQLite+FTS5   │  │ Scheduled     │  │ Event bus,       │
           │ Vector search │  │ agent jobs    │  │ lifecycle events │
           │ Embeddings    │  │               │  │ Gmail watcher    │
           └──────────────┘  └──────────────┘  └──────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      INFRASTRUCTURE LAYER                                       │
│                                                                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐ │
│  │ config/  │ │ infra/   │ │plugins/  │ │security/ │ │ logging/ │ │process/ │ │
│  │ (load,   │ │ (retry,  │ │(discover,│ │(audit,   │ │(tslog,   │ │(exec,   │ │
│  │ validate,│ │ restart, │ │ load,    │ │ SSRF,    │ │ redact,  │ │ lanes,  │ │
│  │ migrate) │ │ outbound,│ │ hooks,   │ │ scan,    │ │ subsys)  │ │ spawn)  │ │
│  │          │ │ heartbeat│ │ HTTP)    │ │ fix)     │ │          │ │         │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ └─────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           CLI & UI LAYER                                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐             │
│  │  cli/    │ │commands/ │ │  tui/    │ │ browser/ │ │ daemon/  │             │
│  │(Commander│ │(agent,   │ │(pi-tui,  │ │(Playwright│ │(launchd, │             │
│  │ argv,    │ │ doctor,  │ │ gateway  │ │ CDP,     │ │ systemd, │             │
│  │ routing) │ │ onboard) │ │ chat)    │ │ Chrome)  │ │ schtasks)│             │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘             │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Module Catalog

| Module | Files (src) | Lines (approx) | Purpose | Key Dependencies |
|--------|-------------|-----------------|---------|------------------|
| `agents/` | ~530 | ~60,000+ | AI agent runtime: model selection, tool execution, system prompt, sandbox, skills, subagents, auth profiles | config, routing, sessions, hooks, channels, infra, pi-ai |
| `auto-reply/` | ~223 | ~25,000+ | Message processing pipeline: dispatch, directives, commands, model routing, queue, reply delivery | agents, config, channels, routing, tts, media-understanding |
| `gateway/` | ~228 | ~83,000 | HTTP/WS server, RPC methods, protocol schema, cron service, node management, browser control | agents, config, routing, channels, plugins, infra |
| `config/` | ~85 | ~48,500 | Config loading, Zod schemas, session store, legacy migration, path resolution | channels (types), infra |
| `infra/` | ~130 | ~71,000 | Utilities: retry, restart, outbound delivery, heartbeat, exec approvals, device pairing, updates | config, agents, process |
| `channels/` | ~60 | ~8,000 | Channel plugin abstraction, registry, dock, normalization, outbound adapters | config, plugins |
| `telegram/` | ~45 | ~8,000 | Telegram Bot API via grammY: long-poll/webhook, topics, reactions, streaming | grammY, config, auto-reply, channels |
| `discord/` | ~41 | ~8,000 | Discord bot via @buape/carbon: guilds, threads, reactions, presence, admin | carbon, config, auto-reply, channels |
| `slack/` | ~34 | ~6,000 | Slack via Socket Mode: channels, threads, slash commands, file uploads | @slack/web-api, config, auto-reply |
| `signal/` | ~17 | ~3,000 | Signal via signal-cli REST API (JSON-RPC + SSE) | config, auto-reply, channels |
| `line/` | ~30 | ~5,000 | LINE via @line/bot-sdk: Flex Messages, Rich Menus, webhook | @line/bot-sdk, config, auto-reply |
| `imessage/` | ~13 | ~2,000 | iMessage via custom `imsg` CLI (JSON-RPC over stdin/stdout) | config, auto-reply, channels |
| `whatsapp/` | ~2 | ~300 | WhatsApp target normalization (bulk via src/web/) | utils, infra |
| `web/` | ~30 | ~5,000 | WhatsApp Web via Baileys: session, QR login, media, auto-reply | @whiskeysockets/baileys, config |
| `memory/` | ~61 | ~8,000 | Semantic search: SQLite + sqlite-vec + FTS5, embeddings (OpenAI/Gemini/Voyage/local) | config, agents, logging |
| `cron/` | ~52 | ~6,000 | Scheduled jobs: cron/interval/one-shot, isolated agent sessions, delivery | config, agents, routing |
| `hooks/` | ~32 | ~5,000 | Event-driven hooks: lifecycle events, Gmail integration, slug generation | config, agents, plugins |
| `plugins/` | ~30 | ~5,000 | Plugin discovery, loading (jiti), registry, hook runner, HTTP routes, services | config, agents, channels |
| `plugin-sdk/` | ~7 | ~500 | SDK re-exports for plugin authors | channels, config, routing |
| `cli/` | ~130 | ~22,000 | CLI entry point (Commander.js), subcommands, argument parsing | commands, config, agents |
| `commands/` | ~160 | ~32,000 | Command implementations: agent, doctor, onboard, configure, models, status | cli, config, agents, gateway, daemon |
| `tui/` | ~22 | ~4,500 | Terminal UI (pi-tui): interactive chat, slash commands, streaming | @mariozechner/pi-tui, gateway |
| `browser/` | ~55 | ~12,000 | Browser automation: Express server, CDP, Playwright, Chrome extension relay | playwright, express, ws |
| `media/` | ~27 | ~2,500 | Media handling: MIME detection, store with TTL, SSRF-safe fetch, image ops | file-type, sharp, express |
| `media-understanding/` | ~43 | ~3,600 | AI media analysis: audio transcription, image/video description, multi-provider | agents (model-auth), media |
| `link-understanding/` | ~7 | ~300 | URL extraction and CLI-based content summarization | media-understanding (scope), process |
| `tts/` | ~4 | ~1,600 | Text-to-speech: Edge TTS, OpenAI, ElevenLabs; auto-mode, directives | node-edge-tts, agents (model-auth) |
| `markdown/` | ~15 | ~1,600 | Markdown IR parser/renderer, WhatsApp conversion, frontmatter, tables | markdown-it, yaml |
| `canvas-host/` | ~3 | ~700 | Canvas/A2UI web server for node displays with live-reload | ws, chokidar |
| `security/` | ~15 | ~2,500 | Audit framework, content sanitization, code scanning, SSRF, permission fix | config, agents, channels |
| `logging/` | ~20 | ~3,000 | Structured logging (tslog), subsystem filtering, redaction, diagnostics | config, tslog |
| `process/` | ~11 | ~1,500 | Process execution, lane-based command queue, signal bridging | globals, logging |
| `daemon/` | ~25 | ~5,000 | OS service management: launchd (macOS), systemd (Linux), schtasks (Windows) | infra, cli |
| `routing/` | ~5 | ~1,600 | Agent route resolution, session key construction, binding matching | config, channels, sessions |
| `sessions/` | ~9 | ~1,200 | Session utilities: key parsing, send policy, transcript events, provenance | config, channels |
| `providers/` | ~9 | ~1,500 | Provider-specific auth: GitHub Copilot OAuth, Qwen refresh | config, agents (auth-profiles) |
| `acp/` | ~10 | ~1,500 | Agent Client Protocol bridge for IDE/editor integration | @agentclientprotocol/sdk, gateway |
| `pairing/` | ~5 | ~500 | Device pairing: code generation, approval, channel allowlists | channels, config, infra |
| `node-host/` | ~7 | ~1,000 | Remote node agent: gateway connection, command invocation, browser proxy | gateway, browser, config |
| `shared/` | ~16 | ~1,500 | Pure types/constants: frontmatter, requirements, reasoning tags | compat |
| `utils/` | ~22 | ~2,000 | General utilities: boolean parse, delivery context, usage format, shell argv | channels, config |
| `terminal/` | ~11 | ~1,000 | ANSI styling, tables, progress lines for CLI output | chalk |
| `compat/` | ~1 | ~50 | Legacy project name constants | — |
| `types/` | ~9 | — | Ambient TypeScript declarations for untyped npm packages | — |
| `macos/` | ~4 | ~300 | macOS app integration entry points | infra |
| `wizard/` | ~8 | ~1,200 | Interactive setup wizard via @clack/prompts | cli, config, channels |
| `extensions/` | ~45 dirs | — | Channel plugins, provider auth plugins, tool/feature plugins | plugin-sdk |

---

## 4. Dependency Graph

### Hierarchy (top = most depended-upon)

```
                         ┌──────────┐
                Level 0  │  config/  │  ← Foundation (700+ imports across codebase)
                         └────┬─────┘
                              │
                Level 1  ┌────┴────┐  ┌──────────┐  ┌──────────┐
                         │ infra/  │  │ shared/  │  │ logging/ │
                         │(400+imp)│  │          │  │          │
                         └────┬────┘  └──────────┘  └──────────┘
                              │
                Level 2  ┌────┴────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
                         │channels/│  │ routing/ │  │sessions/ │  │ process/ │
                         └────┬────┘  └────┬─────┘  └──────────┘  └──────────┘
                              │            │
                Level 3  ┌────┴────────────┴────┐
                         │       agents/        │  ← Core runtime (28+ modules depend on it)
                         │    (530 files)        │
                         └──────────┬───────────┘
                                    │
                Level 4  ┌──────────┴──────────┐
                         │     auto-reply/      │  ← Message pipeline (23+ dependents)
                         │    (223 files)        │
                         └──────────┬───────────┘
                                    │
                Level 5  ┌──────┬───┴───┬──────┬──────┬───────┐
                         │memory│ cron  │hooks │ tts  │media- │
                         │      │       │      │      │undrst │
                         └──────┘───────┘──────┘──────┘───────┘
                                    │
                Level 6  ┌──────────┴──────────┐
                         │      gateway/        │  ← Server orchestrator
                         │    (228 files)        │
                         └──────────┬───────────┘
                                    │
                Level 7  ┌──────┬───┴───┬──────┐
                         │ cli/ │cmds/  │daemon/│
                         └──────┘───────┘──────┘
```

### Notable Dependency Patterns

- **config/** is imported by virtually every module — it's the single source of truth
- **agents/** and **auto-reply/** form a bidirectional dependency (agents provides runtime, auto-reply orchestrates it)
- **gateway/** consumes most modules but is consumed only by cli/commands/daemon
- **plugins/** has cross-cutting dependencies: loads from config, registers into agents/channels/hooks
- **No significant circular dependencies** detected — the bidirectional agents↔auto-reply is by design (pipeline ↔ runtime)

---

## 5. Data Flow: Message Lifecycle

**Path: User sends message in Telegram → Response delivered back**

```
Step 1: TELEGRAM API → grammY Bot (long-poll or webhook)
        └─ src/telegram/bot-handlers.ts registers middleware

Step 2: bot-message.ts → Access check (allowlist, bot detection)
        └─ bot-message-context.ts → Build MsgContext (Body, From, ChatType, MediaPaths)

Step 3: bot-message-dispatch.ts → dispatchInboundMessage()
        └─ src/auto-reply/dispatch.ts

Step 4: routing/resolve-route.ts → resolveAgentRoute(cfg, channel, account, peer)
        └─ Tiered binding match: peer → guild+roles → channel → default
        └─ Returns: { agentId, sessionKey, accountId }

Step 5: auto-reply/reply/dispatch-from-config.ts → dispatchReplyFromConfig()
        └─ Handles TTS auto-mode, fast abort, dedup
        └─ Creates ReplyDispatcher with typing indicator

Step 6: auto-reply/reply/get-reply.ts → getReplyFromConfig()  [MAIN ORCHESTRATOR]
        ├─ 6a. Resolve agent workspace, model defaults
        ├─ 6b. media-understanding/apply.ts → Process image/audio/video attachments
        ├─ 6c. link-understanding/apply.ts → Extract and process URLs
        ├─ 6d. command-auth.ts → Check sender authorization
        ├─ 6e. reply/session.ts → initSessionState() (create/resume/reset)
        ├─ 6f. directive-handling.parse.ts → Parse /model, /think, /verbose directives
        ├─ 6g. get-reply-inline-actions.ts → Handle slash commands (/new, /status, etc.)
        └─ 6h. stage-sandbox-media.ts → Copy media to agent workspace

Step 7: auto-reply/reply/get-reply-run.ts → runPreparedReply()
        └─ Assembles FollowupRun config

Step 8: auto-reply/reply/agent-runner.ts → runReplyAgent()
        ├─ Queue check: if active run, enqueue followup
        ├─ Memory flush if near compaction threshold
        └─ Creates block reply pipeline

Step 9: auto-reply/reply/agent-runner-execution.ts → runAgentTurnWithFallback()
        └─ agents/pi-embedded-runner/run.ts → runEmbeddedPiAgent()
            ├─ 9a. system-prompt.ts → Build system prompt (identity, skills, tools, memory, time)
            ├─ 9b. model-selection.ts → Resolve model + auth profile
            ├─ 9c. pi-tools.ts → Register 40+ tools (exec, browser, web, memory, etc.)
            ├─ 9d. pi-ai API call → LLM provider (streaming SSE)
            ├─ 9e. pi-embedded-subscribe.ts → Process stream chunks
            │       ├─ Tool calls → Tool execution (see §7) → LLM continuation
            │       ├─ Text blocks → pi-embedded-block-chunker.ts
            │       └─ Reasoning blocks → onReasoningStream callback
            └─ 9f. Error recovery: context overflow → reset, transient → retry

Step 10: auto-reply/reply/block-reply-pipeline.ts → Coalesce blocks
         └─ reply-delivery.ts → Parse reply directives (MEDIA:, [[reply_to:id]])

Step 11: auto-reply/reply/reply-dispatcher.ts → Buffer, normalize, add human delay
         └─ channels/plugins/outbound/telegram.ts → Telegram-specific formatting

Step 12: telegram/send.ts → sendMessageTelegram()
         └─ Markdown → HTML conversion, chunking (4096 chars), media, retry
         └─ Telegram Bot API → User sees response
```

**Total modules touched:** telegram → channels → auto-reply → routing → sessions → agents → memory → media-understanding → config → infra → plugins → hooks → logging

---

## 6. Data Flow: Cron Job Execution

```
Step 1: User creates job via /cron command or cron tool
        └─ agents/tools/cron-tool.ts → gateway server-methods/cron.ts
        └─ cron/service.ts → CronService.add()

Step 2: cron/service/store.ts → Persist to ~/.openclaw/cron/jobs.json (atomic write)
        └─ computeNextRunAtMs() from schedule (cron expr, interval, or one-shot)

Step 3: cron/service/timer.ts → setTimeout fires when nextRunAtMs reached
        └─ locked() → Serialize concurrent access

Step 4: Job execution (depends on sessionTarget):
        ├─ "main" → Enqueue system event → heartbeat → agent processes in main session
        └─ "isolated" → cron/isolated-agent/run.ts → runCronIsolatedAgentTurn()
            ├─ 4a. isolated-agent/session.ts → Create isolated session (agent:<id>:cron:<name>:run:<uuid>)
            ├─ 4b. Run embedded agent turn (same as message lifecycle Step 9)
            └─ 4c. Collect output payloads

Step 5: cron/delivery.ts → resolveCronDeliveryPlan()
        ├─ mode "none" → Log only
        └─ mode "announce" → Resolve channel + target
            └─ infra/outbound/deliver.ts → deliverOutboundPayloads()
                └─ Channel plugin → Send to Telegram/Discord/etc.

Step 6: Post-execution:
        ├─ cron/run-log.ts → Append JSONL run log
        ├─ Update job state (lastRunAtMs, lastStatus, consecutiveErrors)
        ├─ Rearm timer for next run (or delete if one-shot deleteAfterRun)
        └─ cron/session-reaper.ts → Prune expired cron session entries
```

---

## 7. Data Flow: Tool Execution

```
Step 1: LLM decides to call a tool (during streaming response)
        └─ pi-embedded-subscribe.handlers.tools.ts → Extract tool call from stream

Step 2: agents/pi-tools.ts → Tool registry lookup
        ├─ SDK tools (from @mariozechner/pi-coding-agent): read, write, edit, exec, process
        └─ OpenClaw tools (from openclaw-tools.ts): browser, canvas, cron, message,
           nodes, web_search, web_fetch, image, memory, tts, sessions, subagents

Step 3: agents/pi-tools.before-tool-call.ts → Pre-tool-call hooks
        └─ Typing indicator, tool tracking

Step 4: agents/tool-policy-pipeline.ts → Policy evaluation
        ├─ tool-policy.ts → Check owner-only, profile policy, explicit allowlist
        ├─ pi-tools.policy.ts → Group/subagent restrictions
        └─ Decision: "allow" → execute | "deny" → block | "ask" → prompt user

Step 5: Tool execution (example: exec tool)
        └─ agents/bash-tools.exec.ts
            ├─ 5a. Sandbox check → Docker container or host execution
            ├─ 5b. Exec approval check → gateway/exec-approval-manager.ts
            │       ├─ "allowlist" match → auto-approve
            │       ├─ "ask" → Forward to user for approval (via channel)
            │       └─ "deny" → Block execution
            ├─ 5c. process/exec.ts → runCommandWithTimeout()
            │       └─ process/command-queue.ts → Lane-based serialization
            └─ 5d. Collect stdout/stderr, exit code

Step 6: agents/pi-embedded-runner/tool-result-truncation.ts → Truncate large results
        └─ agents/session-tool-result-guard.ts → Guard against oversized results

Step 7: Tool result returned to LLM → Continues generation
        └─ pi-embedded-subscribe.ts → Resume stream processing

Step 8: agents/tool-display.ts → Format tool result for user display
        └─ agents/tool-mutation.ts → Track if tool caused file mutations
```

---

## 8. Configuration Architecture

### Config File

- **Path:** `~/.openclaw/openclaw.json` (JSON5 format)
- **Loading:** `config/io.ts` → read → `parseConfigJson5()` → merge includes → Zod validation → `OpenClawConfig` object
- **Caching:** In-memory with `clearConfigCache()` invalidation
- **Hot reload:** `gateway/config-reload.ts` watches file via chokidar → broadcasts update to connected clients
- **Migration:** `config/legacy-migrate.ts` handles automatic migration from older config versions (3 migration parts)
- **Includes:** `config/includes.ts` supports `$include` for splitting config across files
- **Env substitution:** `config/env-substitution.ts` expands `${ENV_VAR}` in config values
- **Backup:** `config/backup-rotation.ts` maintains rotating backups on write

### Config Sections

| Section | Purpose | Key Fields |
|---------|---------|------------|
| `agents` | Agent definitions | `list[]` (agentId, model, skills, tools, workspace, identity, sandbox), `defaults.*` |
| `agents.defaults` | Default agent settings | `model`, `provider`, `heartbeat.*`, `compaction.*`, `memorySearch.*`, `allowedModels`, `modelFallbacks`, `typingIntervalSeconds`, `envelopeTimezone` |
| `bindings[]` | Channel→agent routing | `agentId`, `channel`, `account`, `peer`, `guild`, `roles`, `team` |
| `session` | Session behavior | `dmScope`, `identityLinks`, `resetTriggers`, `sendPolicy`, `store`, `mainKey` |
| `gateway` | Server config | `port`, `auth`, `tls`, `discovery`, `tailscale`, `lanes`, `nodes`, `browser` |
| `models` | Model/provider config | Model definitions, aliases, provider-specific settings |
| `authProfiles` | Auth credentials | Per-provider API keys, OAuth tokens, custom base URLs |
| `<channel>` | Per-channel config | `telegram.*`, `discord.*`, `slack.*`, `signal.*`, `whatsapp.*`, `imessage.*`, etc. |
| `hooks` | Hook system | `enabled`, `internal.*`, `gmail.*`, `path`, `token`, `presets` |
| `cron` | Cron jobs | `enabled`, `storePath`, `sessionRetention` |
| `messages` | Message handling | `tts.*`, `inbound.debounceMs`, `inbound.byChannel` |
| `tools` | Tool config | `media.*` (image/audio/video), `links.*`, `exec.*` |
| `sandbox` | Sandbox config | Docker image, mount paths, env, network |
| `memory` | Memory backend | `backend` (builtin/qmd), `citations`, `qmd.*` |
| `plugins` | Plugin config | Plugin entries, allow/deny, slots |
| `logging` | Logging config | `level`, `consoleLevel`, `consoleStyle`, `file`, `redact.mode` |
| `approvals` | Exec approvals | `mode` (off/allowlist/always), `allowlist[]`, security policy |

### Zod Schema

- **Main schema:** `config/zod-schema.ts` → `OpenClawSchema`
- **Sub-schemas:** `zod-schema.agents.ts`, `.channels.ts`, `.providers.ts`, `.session.ts`, `.hooks.ts`, `.approvals.ts`, `.allowdeny.ts`, `.sensitive.ts`
- **JSON Schema generation:** `config/schema.ts` → `buildConfigSchema()` (for editor autocomplete)
- **Validation:** `config/validation.ts` → `validateConfigObject()`, `validateConfigObjectWithPlugins()`

### Config Cascade (for model resolution)

```
openclaw.json defaults → per-agent config → session entry override → inline /model directive → heartbeat override
```

---

## 9. Security Model

### Authentication & Authorization

| Layer | Mechanism | Location |
|-------|-----------|----------|
| **Gateway auth** | Token/password in config | `gateway/auth.ts` |
| **Device auth** | Per-device tokens with roles/scopes | `gateway/device-auth.ts`, `infra/device-auth-store.ts` |
| **Rate limiting** | Per-IP auth attempt rate limit | `gateway/auth-rate-limit.ts` |
| **Origin checking** | Validate HTTP Origin header | `gateway/origin-check.ts` |
| **Channel allowlists** | Per-channel `allowFrom` sender filtering | `channels/allowlist-match.ts` |
| **Command authorization** | Owner-only commands, sender checks | `auto-reply/command-auth.ts` |
| **Tool policy** | Per-tool allow/deny/ask | `agents/tool-policy-pipeline.ts` |

### Exec Approval System

- **Modes:** `off` (no approval), `allowlist` (auto-approve matched commands), `always` (approve everything)
- **Flow:** Agent requests exec → `agents/bash-tools.exec.ts` → `gateway/exec-approval-manager.ts` → check allowlist → if not matched, forward to user via channel → user approves/denies → result returned
- **Command risk analysis:** `infra/exec-approvals-analysis.ts` classifies command risk
- **Dangerous tools:** `security/dangerous-tools.ts` lists tools requiring approval (e.g., ACP tools)

### Sandboxing (Docker)

- **Module:** `agents/sandbox/` (~18 files)
- **Behavior:** When enabled, agent exec commands run inside Docker containers
- **Config:** `sandbox.image`, mount paths, env, network settings
- **Tool policy:** `sandbox/tool-policy.ts` enforces sandbox-specific allow/deny
- **FS bridge:** `sandbox/fs-bridge.ts` maps host ↔ container paths

### SSRF Protection

- **Module:** `infra/net/ssrf.ts`, `infra/net/fetch-guard.ts`
- **Behavior:** All media fetching, web_fetch, and URL access goes through SSRF guard
- **Blocks:** Private IPs (10.x, 172.16-31.x, 192.168.x, 127.x), link-local, cloud metadata endpoints

### Content Security

- **External content wrapping:** `security/external-content.ts` → `wrapExternalContent()` adds boundary markers for LLM input
- **Prompt injection detection:** `detectSuspiciousPatterns()` flags suspicious content
- **Channel metadata:** `security/channel-metadata.ts` wraps untrusted usernames/bios in safety boundaries

### Secret Handling

- **Timing-safe comparison:** `security/secret-equal.ts` → `safeEqualSecret()` via `crypto.timingSafeEqual`
- **Config redaction:** `config/redact-snapshot.ts` strips sensitive fields from config snapshots
- **Log redaction:** `logging/redact.ts` → regex patterns for API keys, tokens, PEM blocks
- **Auth profiles:** Stored in `~/.openclaw/auth/` with filesystem permission hardening

### Audit & Remediation

- **Security audit:** `security/audit.ts` → `runSecurityAudit()` scans config + filesystem + channels
- **Auto-fix:** `security/fix.ts` → `runSecurityFix()` — chmod state dirs to safe permissions
- **Code scanning:** `security/skill-scanner.ts` scans skill/plugin code for dangerous patterns (eval, exec, fetch)
- **Filesystem inspection:** `security/audit-fs.ts` checks POSIX mode bits and Windows ACLs

---

## 10. Storage & Persistence

### SQLite Databases

| Database | Location | Purpose | Schema |
|----------|----------|---------|--------|
| **Memory index** | `<workspace>/memory-index.sqlite` | Semantic memory search | `meta`, `files`, `chunks` (with embeddings), `chunks_vec` (sqlite-vec), `chunks_fts` (FTS5), `embedding_cache` |
| **LanceDB** | `~/.openclaw/state/<agent>/lancedb/` | Long-term memory (memory-lancedb plugin) | Vector table with auto-recall |

### JSON File Stores

| File | Location | Purpose | Format |
|------|----------|---------|--------|
| **Config** | `~/.openclaw/openclaw.json` | Main configuration | JSON5 |
| **Sessions** | `~/.openclaw/state/sessions.json` | Session entries (model, usage, state) | JSON, file-locked |
| **Cron jobs** | `~/.openclaw/cron/jobs.json` | Scheduled job definitions | JSON, atomic write with .bak |
| **Cron run logs** | `~/.openclaw/cron/runs/<jobId>.jsonl` | Per-job execution history | JSONL, auto-pruned at 2MB/2000 lines |
| **Auth profiles** | `~/.openclaw/auth/profiles.json` | API keys, OAuth tokens | JSON, permission-hardened |
| **Copilot token cache** | `~/.openclaw/state/credentials/github-copilot.token.json` | Cached Copilot API token | JSON |
| **TTS preferences** | `~/.openclaw/settings/tts.json` | User TTS preferences | JSON |
| **Device auth** | `~/.openclaw/state/device-auth.json` | Device tokens and roles | JSON |
| **Pairing store** | `~/.openclaw/state/pairing.json` | Pending pairing requests | JSON, file-locked |
| **Node config** | `~/.openclaw/node.json` | Remote node identity and gateway connection | JSON |
| **Update offset** | `~/.openclaw/state/telegram-update-offset.json` | Telegram getUpdates offset | JSON |

### Session Transcript Files

- **Location:** `~/.openclaw/state/sessions/<sessionKey>.jsonl`
- **Format:** JSONL — one JSON object per turn (user message, assistant response, tool calls/results)
- **Managed by:** `@mariozechner/pi-coding-agent` SessionManager
- **Events:** `sessions/transcript-events.ts` emits updates for memory indexing and UI refresh

### Media Files

- **Location:** `~/.openclaw/media/`
- **Naming:** `<prefix>---<uuid>.<ext>`
- **TTL:** 2 minutes default, cleaned on access or periodic sweep
- **Served:** Express routes at `GET /media/:id`

### Log Files

- **Location:** `~/.openclaw/logs/openclaw.log`
- **Rotation:** 24-hour rotation
- **Format:** Structured JSON (tslog)

### Agent Workspaces

- **Location:** Configurable per agent, default `~/.openclaw/workspace-<agentId>/`
- **Contains:** `AGENTS.md`, `TOOLS.md`, `SOUL.md`, `memory/`, `hooks/`, session data

---

## 11. Plugin System

### Discovery

Plugins are discovered from multiple directories in order:

1. **Bundled** — `extensions/` shipped with OpenClaw (bundled-by-default: `device-pair`, `phone-control`, `talk-voice`)
2. **Global** — `~/.openclaw/plugins/` (user-installed)
3. **Workspace** — Agent workspace `plugins/` directory
4. **Config paths** — Explicit paths in `plugins` config section

Discovery is handled by `plugins/discovery.ts` → `discoverOpenClawPlugins()`.

### Manifest

Each plugin has `openclaw.plugin.json`:
```json
{
  "id": "plugin-name",
  "kind": "channel" | "provider" | "tool" | "feature",
  "configSchema": { ... },
  "channels": ["telegram", "discord"]
}
```
Parsed by `plugins/manifest.ts`.

### Loading

1. `plugins/loader.ts` → `loadOpenClawPlugins()`
2. Validates manifest → loads module via `jiti` (dynamic import)
3. Plugin exports `definePlugin()` returning: tools, hooks, channels, providers, HTTP routes, services, CLI commands
4. Built into `PluginRegistry` → stored as global singleton via `plugins/runtime.ts`

### Registry

`plugins/registry.ts` → `PluginRegistry`:
- **tools** — Agent tools provided by plugins
- **hooks** — Lifecycle hook handlers
- **channels** — Channel plugin implementations
- **providers** — LLM provider extensions
- **httpRoutes** — Custom HTTP endpoints
- **services** — Long-running background services
- **commands** — CLI commands

### Hook Runner

`plugins/hooks.ts` → `createHookRunner()`:
- Priority ordering of hook handlers
- Error isolation (one hook failure doesn't crash others)
- Async support
- Lifecycle events: `onLoad`, `onUnload`, `onConfigChange`, `onSessionStart`, `onSessionEnd`, `onAgentBootstrap`

### Extension Types

| Type | Count | Examples |
|------|-------|---------|
| **Channel plugins** | 21 | telegram, discord, slack, signal, whatsapp, line, irc, matrix, msteams, nostr, twitch, zalo |
| **Provider plugins** | 5 | copilot-proxy, google-antigravity-auth, gemini-cli-auth, minimax-portal-auth, qwen-portal-auth |
| **Tool/Feature plugins** | 10 | memory-core, memory-lancedb, llm-task, lobster, open-prose, diagnostics-otel, thread-ownership |

---

## 12. Provider System

### Architecture

OpenClaw delegates LLM API calls to `@mariozechner/pi-ai` — the external AI agent framework. OpenClaw's role is:

1. **Configuration** — Define available providers and models in `openclaw.json`
2. **Authentication** — Resolve API keys via auth profiles (`agents/auth-profiles/`)
3. **Model selection** — Choose model based on config cascade (see §8)
4. **Custom auth flows** — Implement provider-specific OAuth (GitHub Copilot, Qwen Portal)

### Auth Profiles (`agents/auth-profiles/`)

- **Store:** `~/.openclaw/auth/profiles.json`
- **Operations:** CRUD via `auth-profiles/profiles.ts`
- **Ordering:** Priority-ordered per provider (`auth-profiles/order.ts`)
- **OAuth:** Provider-specific OAuth flows (`auth-profiles/oauth.ts`)
- **Health:** Automatic health checking and repair (`auth-profiles/doctor.ts`, `repair.ts`)
- **Session override:** Per-session auth profile selection (`auth-profiles/session-override.ts`)
- **External sync:** Sync with external CLI credentials (`auth-profiles/external-cli-sync.ts`)

### Model Resolution Pipeline

```
1. agents/model-selection.ts → resolveModel()
   ├─ Check session override (persisted /model directive)
   ├─ Check agent config (agents.<id>.model)
   ├─ Check defaults (agents.defaults.model)
   ├─ Resolve aliases (config-defined shortcuts)
   ├─ Fuzzy match (Levenshtein + variant tokens)
   └─ Check allowlist (agents.defaults.allowedModels)

2. agents/model-auth.ts → resolveModelAuth()
   ├─ Find matching auth profile for provider
   ├─ Check live key validity (agents/live-auth-keys.ts)
   └─ Return API key + base URL

3. agents/model-fallback.ts → If primary fails, try fallback chain
   └─ agents.defaults.modelFallbacks or per-agent

4. agents/models-config.ts → Generate models.json for pi-agent
   └─ agents/models-config.providers.ts → Provider-specific config
```

### Supported Providers

Via pi-ai and auth profiles: **Anthropic**, **OpenAI**, **Google (Gemini)**, **xAI (Grok)**, **AWS Bedrock**, **Azure OpenAI**, **Ollama** (local), **Together.ai**, **Venice.ai**, **HuggingFace**, **MiniMax**, **Qwen**, **OpenCode/Zen**, **GitHub Copilot**, **Cloudflare AI Gateway**, **Chutes**, and any OpenAI-compatible API.

### Provider-Specific Auth

| Provider | Auth Method | Module |
|----------|-------------|--------|
| GitHub Copilot | Device code OAuth → token exchange | `providers/github-copilot-auth.ts` |
| Qwen Portal | OAuth2 refresh token | `providers/qwen-portal-oauth.ts` |
| Google Antigravity | OAuth (plugin) | `extensions/google-antigravity-auth/` |
| Gemini CLI | OAuth (plugin) | `extensions/google-gemini-cli-auth/` |
| MiniMax Portal | OAuth (plugin) | `extensions/minimax-portal-auth/` |

---

## 13. Channel Abstraction

### ChannelPlugin Interface

Every channel implements `ChannelPlugin` (defined in `channels/plugins/types.plugin.ts`) with ~20 adapter slots:

| Adapter | Purpose |
|---------|---------|
| `config` | Config schema and resolution |
| `setup` | Interactive onboarding wizard |
| `pairing` | Device pairing support |
| `security` | DM policy, sender verification |
| `groups` | Group membership, roles |
| `outbound` | Message sending (text, media, polls, reactions) |
| `status` | Account status and diagnostics |
| `gateway` | Gateway integration (start/stop monitor) |
| `auth` | Channel-specific authentication |
| `elevated` | Elevated permission handling |
| `commands` | Native slash command registration |
| `streaming` | Edit-in-place streaming delivery |
| `threading` | Thread/topic management |
| `messaging` | Message CRUD (send, edit, delete, react) |
| `directory` | User/channel directory access |
| `resolver` | Target name/ID resolution |
| `actions` | Channel-specific message actions |
| `heartbeat` | Keep-alive/heartbeat |
| `agentTools` | Channel-specific agent tools |

### Channel Dock

`channels/dock.ts` provides lightweight behavior metadata without loading heavy plugin code:
- Capabilities (threading, reactions, streaming, markdown support)
- Text chunk limits per channel
- Mention format
- Streaming coalesce settings

### Common Patterns Across Channels

1. **Account resolution:** `resolve<Channel>Account({ cfg, accountId })` → typed account config
2. **Monitor:** `monitor<Channel>Provider(opts)` → starts listening for inbound messages
3. **Send:** `sendMessage<Channel>(to, text, opts)` → delivers outbound message
4. **Probe:** `probe<Channel>(opts)` → verifies connectivity/credentials
5. **Normalization:** `plugins/normalize/<channel>.ts` → platform-specific → unified `MsgContext`
6. **Outbound:** `plugins/outbound/<channel>.ts` → unified send → platform-specific API

### Channel-Specific Features

| Channel | Framework | Special Features |
|---------|-----------|------------------|
| Telegram | grammY | Forum topics, inline buttons, draft streaming, reactions, proxy support |
| Discord | @buape/carbon | Guilds, threads, polls, PluralKit, presence, admin actions |
| Slack | @slack/web-api | Socket Mode, slash commands, file uploads, pins |
| Signal | signal-cli REST | JSON-RPC + SSE, groups, reactions, daemon management |
| LINE | @line/bot-sdk | Flex Messages, Rich Menus, markdown→flex conversion |
| iMessage | imsg CLI | JSON-RPC over stdin/stdout, macOS native |
| WhatsApp | Baileys | QR login, media conversion, broadcast groups |

---

## 14. Key Design Patterns

### 1. Pipeline / Chain of Responsibility
**Where:** `auto-reply/` — The entire message processing pipeline: dispatch → directives → commands → agent run → delivery. Each stage can short-circuit with a reply or pass to the next.

### 2. Observer / Event Bus
**Where:** `hooks/internal-hooks.ts` (register/trigger), `sessions/transcript-events.ts` (pub/sub), `infra/system-events.ts`, `infra/heartbeat-events.ts`, `gateway/server-broadcast.ts` (WS event broadcast).

### 3. Registry Pattern
**Where:** `plugins/registry.ts` (PluginRegistry), `agents/bash-process-registry.ts` (running processes), `agents/subagent-registry.ts`, `gateway/node-registry.ts`, `auto-reply/dispatcher-registry.ts`, `auto-reply/commands-registry.ts`.

### 4. Factory Pattern
**Where:** `memory/embeddings.ts` (createEmbeddingProvider), `plugins/registry.ts` (createPluginRegistry), `auto-reply/reply-dispatcher.ts` (createReplyDispatcher), `agents/pi-tools.ts` (createOpenClawTools).

### 5. Strategy / Tiered Matching
**Where:** `routing/resolve-route.ts` — Binding resolution uses ordered predicate tiers (peer → guild+roles → channel → default). `agents/model-selection.ts` — Model resolution with alias → fuzzy → allowlist strategies.

### 6. Middleware Pattern
**Where:** `agents/tool-policy-pipeline.ts` (tool call policies), `agents/pi-tools.before-tool-call.ts` (pre-tool hooks), `gateway/server-middleware.ts` (HTTP auth), `telegram/bot.ts` (grammY middleware).

### 7. Builder Pattern
**Where:** `agents/system-prompt.ts` — System prompt construction assembles sections: identity, skills, memory, tools, workspace, runtime, time. `agents/workspace.ts` — Bootstrap file assembly.

### 8. Guard / Circuit Breaker
**Where:** `agents/context-window-guard.ts` (context overflow), `agents/session-tool-result-guard.ts` (oversized results), `agents/pi-extensions/compaction-safeguard.ts` (infinite compaction loops), `auto-reply/agent-runner-execution.ts` (error recovery with session reset).

### 9. Barrel / Re-export Pattern
**Where:** Extensively used for public API surfaces: `agents/pi-embedded.ts` → `pi-embedded-runner.ts` → individual modules. `auto-reply/reply.ts`, `hooks/hooks.ts`, `config/config.ts`, etc.

### 10. WeakMap Caching
**Where:** `routing/resolve-route.ts` — Config-scoped binding evaluation cache (max 2000 keys). Avoids memory leaks when config objects are garbage-collected.

### 11. Discriminated Unions / Result Types
**Where:** Throughout: `{ kind: "reply", reply } | { kind: "result", result }`, `{ ok: true, value } | { ok: false, error }`, `SessionKeyShape`, `AgentRunLoopResult`.

### 12. Normalization Pattern
**Where:** Every input boundary: `normalizeAccountId()`, `normalizeAgentId()`, `normalizeChatChannelId()`, `normalizeThinkLevel()`, `normalizeSecretInput()`, etc. Defensive trim/lowercase/sanitize.

### 13. Mixin Pattern
**Where:** `memory/manager.ts` — `MemoryIndexManager` is extended by `manager-embedding-ops.ts` and `manager-sync-ops.ts` as mixed-in method groups.

### 14. Lane-Based Concurrency
**Where:** `process/command-queue.ts` — Commands are serialized per lane (Main, Cron, Subagent, Nested) with configurable concurrency per lane. `gateway/server-lanes.ts` manages gateway-level concurrency.

### 15. Dependency Injection (for testing)
**Where:** `providers/github-copilot-token.ts` accepts `fetchImpl`, `loadJsonFileImpl` etc. `cron/service.ts` accepts `CronServiceDeps`. Most modules accept config objects rather than reading globals.

---

## 15. Cross-Module Statistics

### Overall Codebase

| Metric | Value |
|--------|-------|
| **Total src/ modules** | ~45 directories |
| **Total source files** | ~2,500+ .ts files |
| **Total test files** | ~800+ .test.ts files |
| **Estimated total lines** | ~350,000+ |
| **External framework** | @mariozechner/pi-ai, pi-agent-core, pi-coding-agent |
| **Language** | TypeScript (Node.js) |
| **Test framework** | Vitest |
| **Build** | TypeScript compiler (tsc) |

### Largest Modules (by file count)

| Rank | Module | Source Files | Test Files |
|------|--------|-------------|------------|
| 1 | `agents/` | ~263 | ~267 |
| 2 | `auto-reply/` | ~129 | ~86 |
| 3 | `gateway/` | ~145 | ~83 |
| 4 | `commands/` | ~160 | ~30 |
| 5 | `cli/` | ~130 | ~20 |
| 6 | `infra/` | ~130 | ~40 |
| 7 | `config/` | ~85 | ~50 |
| 8 | `memory/` | ~30 | ~20 |
| 9 | `browser/` | ~55 | ~15 |
| 10 | `cron/` | ~25 | ~20 |

### Most-Imported Modules

| Rank | Module | Approximate Import Count |
|------|--------|-------------------------|
| 1 | `config/` | 700+ |
| 2 | `infra/` | 400+ |
| 3 | `agents/` | 300+ |
| 4 | `auto-reply/` | 200+ |
| 5 | `channels/` | 150+ |
| 6 | `routing/` | 100+ |
| 7 | `logging/` | 80+ |
| 8 | `utils/` | 70+ |
| 9 | `plugins/` | 60+ |
| 10 | `sessions/` | 50+ |

### Extension Ecosystem

| Category | Count |
|----------|-------|
| Channel plugins | 21 |
| Provider plugins | 5 |
| Tool/Feature plugins | 10 |
| **Total extensions** | **36** |

### Key External Dependencies

| Package | Purpose | Used By |
|---------|---------|---------|
| `@mariozechner/pi-ai` | Core AI agent framework | agents |
| `@mariozechner/pi-agent-core` | Agent core types | agents, auto-reply |
| `@mariozechner/pi-coding-agent` | Coding agent tools + session manager | agents |
| `@sinclair/typebox` | JSON schema builder | agents, gateway |
| `grammy` | Telegram Bot API | telegram |
| `@buape/carbon` | Discord REST API | discord |
| `@slack/web-api` | Slack API | slack |
| `@whiskeysockets/baileys` | WhatsApp Web | web |
| `@line/bot-sdk` | LINE Messaging API | line |
| `commander` | CLI framework | cli |
| `@clack/prompts` | Interactive CLI prompts | commands, wizard |
| `express` | HTTP server | media, browser, gateway |
| `ws` | WebSocket | gateway, browser |
| `playwright` | Browser automation | browser |
| `tslog` | Structured logging | logging |
| `zod` | Schema validation | config |
| `markdown-it` | Markdown parsing | markdown |
| `chokidar` | File watching | memory, canvas-host, gateway |
| `croner` | Cron expressions | cron |
| `file-type` | MIME detection | media |
| `sharp` | Image processing | media |
| `node-edge-tts` | Microsoft Edge TTS | tts |
| `jiti` | Dynamic module loading | plugins |

---

*End of OpenClaw Master Architecture Document*
