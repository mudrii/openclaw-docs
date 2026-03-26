# OpenClaw — Master Architecture Document

> Updated: 2026-03-27 (docs snapshot: v2026.3.24-1) | Released baseline: GitHub `v2026.3.24`

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

**OpenClaw** is an open-source, self-hosted AI agent platform that connects Large Language Models to messaging channels (Telegram, Discord, Slack, WhatsApp, Signal, iMessage, BlueBubbles, LINE, IRC, Synology Chat, and more). It runs as a persistent gateway daemon on macOS/Linux/Windows, accepting messages from any connected channel, routing them to configured AI agents, executing tool calls on behalf of the agent, and delivering responses back to users. OpenClaw supports multi-agent configurations, per-channel routing, sandboxed execution environments (Docker), browser automation, semantic memory search, scheduled cron jobs, mobile node pairing, and a rich plugin/extension ecosystem.

Architecturally, OpenClaw follows a **hub-and-spoke model**: the `gateway` module is the central server process that orchestrates all subsystems. It exposes a WebSocket JSON-RPC API for CLI/TUI clients, an OpenAI-compatible HTTP API, and channel plugin connections. The `config` module provides the foundation — nearly every module depends on it for typed configuration. The `agents` module (the largest at ~720 `.ts` files) contains the AI runtime: model selection, system prompt construction, tool registration, streaming response processing, sandbox management, and the embedded pi-agent integration (`@mariozechner/pi-ai`). The `auto-reply` module (~260 `.ts` files) is the message processing pipeline that sits between channels and agents — handling commands, directives, session management, model routing, queue management, and reply delivery.

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
│  ┌──────────┐                                                              │       │
│  │synology/ │                                                              │       │
│  │(webhook) │                                                              │       │
│  └────┬─────┘                                                              │       │
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
│  │  │ Cron   │ │Browser │ │ Nodes  │ │Plugins │ │  Auth/Rate   │ │  APNs/   │ │
│  │  │Service │ │Control │ │Registry│ │Lifecycle│ │  Limiting    │ │  Push    │ │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └──────────────┘ └──────────┘ │
│  │  ┌──────────────────┐                                                       │
│  │  │ Health Probes    │  /health, /healthz (live), /ready, /readyz (ready)    │
│  │  └──────────────────┘                                                       │
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
│                      AGENTS MODULE (src/agents/)  ~720 files                    │
│                                                                                 │
│  ┌─────────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ PI Embedded      │  │ System Prompt│  │Model Selection│  │  Auth Profiles  │ │
│  │ Runner (LLM call)│  │ Builder      │  │& Fallback    │  │  & OAuth        │ │
│  └────────┬────────┘  └──────────────┘  └──────────────┘  └──────────────────┘ │
│           │                                                                     │
│  ┌────────▼────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ Tool Registry    │  │  Sandbox     │  │  Skills      │  │  Subagent       │ │
│  │ 75+ tools:       │  │  (Docker)    │  │  System      │  │  Registry       │ │
│  │ exec, browser,   │  └──────────────┘  └──────────────┘  └──────────────────┘ │
│  │ web, memory,     │                                                           │
│  │ message, cron,   │                                                           │
│  │ canvas, nodes,   │                                                           │
│  │ diffs (plugin).. │                                                           │
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
| `agents/` | 720 | ~130,000+ | AI agent runtime: model selection, tool execution, system prompt, sandbox, skills, subagents (nested orchestration with typed `task_completion` events), auth profiles, OpenAI WS transport | config, routing, sessions, hooks, channels, infra, pi-ai |
| `auto-reply/` | 260 | ~49,000+ | Message processing pipeline: dispatch, directives, commands, model routing, queue, reply delivery | agents, config, channels, routing, tts, media-understanding |
| `gateway/` | 310 | ~66,000+ | HTTP/WS server, RPC methods, protocol schema, cron service, node management, browser control, health probes (`/health`, `/healthz`, `/ready`, `/readyz`) | agents, config, routing, channels, plugins, infra |
| `config/` | 191 | ~34,000+ | Config loading, Zod schemas, session store, legacy migration, path resolution | channels (types), infra |
| `infra/` | 297 | ~53,000+ | Utilities: retry, restart, outbound delivery, heartbeat, exec approvals, device pairing, updates | config, agents, process |
| `channels/` | 137 | ~17,000+ | Channel plugin abstraction, registry, dock, normalization, outbound adapters | config, plugins |
| `channels/status-reactions.ts` | 2 | ~850 | Shared lifecycle reaction controller for Telegram and Discord status events | channels, config, infra |
| `extensions/telegram/` | 96 | ~24,000+ | Telegram Bot API via grammY: long-poll/webhook, topics (including per-DM topics with routing/authorization), reactions, streaming | grammY, config, auto-reply, channels |
| `extensions/discord/` | 120 | ~30,000+ | Discord bot via @buape/carbon: guilds, threads, reactions, presence, admin, Component v2 UI | carbon, config, auto-reply, channels |
| `extensions/discord/voice/` | ~6 | ~800 | Discord voice channel join/leave management, auto-join, realtime conversation | discord, config, channels |
| `extensions/discord/monitor/thread-bindings/` | ~4 | ~500 | Thread-bound subagent session management with inactivity lifecycle (`idleHours` default 24h, `maxAgeHours`) for Discord | discord, sessions, config |
| `extensions/slack/` | 84 | ~14,500+ | Slack via Socket Mode: channels, threads, slash commands, file uploads | @slack/web-api, config, auto-reply |
| `extensions/signal/` | 31 | ~5,000+ | Signal via signal-cli REST API (JSON-RPC + SSE) | config, auto-reply, channels |
| `extensions/line/` | 45 | ~7,800+ | LINE via @line/bot-sdk: Flex Messages, Rich Menus, webhook | @line/bot-sdk, config, auto-reply |
| `extensions/imessage/` | 22 | ~3,000+ | iMessage via custom `imsg` CLI (JSON-RPC over stdin/stdout) | config, auto-reply, channels |
| `extensions/whatsapp/` | 4 | ~500+ | WhatsApp target normalization (bulk via src/web/) | utils, infra |
| `web/` | 80 | ~12,600+ | WhatsApp Web via Baileys: session, QR login, media, auto-reply | @whiskeysockets/baileys, config |
| `memory/` | 84 | ~17,000+ | Semantic search: SQLite + sqlite-vec + FTS5, embeddings (OpenAI/Gemini/Voyage/local) | config, agents, logging |
| `cron/` | 71 | ~14,800+ | Scheduled jobs: cron/interval/one-shot, isolated agent sessions, delivery | config, agents, routing |
| `hooks/` | 38 | ~6,600+ | Event-driven hooks: lifecycle events, Gmail integration, slug generation | config, agents, plugins |
| `plugins/` | 62 | ~12,000+ | Plugin discovery, loading (jiti), registry, hook runner, HTTP routes, services | config, agents, channels |
| `plugin-sdk/` | 36 | ~2,400+ | SDK re-exports for plugin authors | channels, config, routing |
| `cli/` | 254 | ~35,900+ | CLI entry point (Commander.js), subcommands, argument parsing | commands, config, agents |
| `commands/` | 304 | ~52,400+ | Command implementations: agent, doctor, onboard, configure, models, status | cli, config, agents, gateway, daemon |
| `tui/` | 45 | ~7,500+ | Terminal UI (pi-tui): interactive chat, slash commands, streaming | @mariozechner/pi-tui, gateway |
| `browser/` | 117 | ~19,100+ | Browser automation: Express server, CDP, Playwright, Chrome extension relay | playwright, express, ws |
| `media/` | 30 | ~3,800+ | Media handling: MIME detection, store with TTL, SSRF-safe fetch, image ops | file-type, sharp, express |
| `media-understanding/` | 51 | ~6,300+ | AI media analysis: audio transcription, image/video description, multi-provider | agents (model-auth), media |
| `link-understanding/` | 6 | ~300+ | URL extraction and CLI-based content summarization | media-understanding (scope), process |
| `tts/` | 4 | ~2,200+ | Text-to-speech: Edge TTS, OpenAI, ElevenLabs; auto-mode, directives | node-edge-tts, agents (model-auth) |
| `markdown/` | 14 | ~2,600+ | Markdown IR parser/renderer, WhatsApp conversion, frontmatter, tables | markdown-it, yaml |
| `canvas-host/` | 5 | ~1,100+ | Canvas/A2UI web server for node displays with live-reload | ws, chokidar |
| `security/` | 29 | ~10,800+ | Audit framework, content sanitization, code scanning, SSRF, permission fix | config, agents, channels |
| `logging/` | 24 | ~2,700+ | Structured logging (tslog), subsystem filtering, redaction, diagnostics | config, tslog |
| `process/` | 24 | ~3,100+ | Process execution, lane-based command queue, signal bridging | globals, logging |
| `daemon/` | 40 | ~5,700+ | OS service management: launchd (macOS), systemd (Linux), schtasks (Windows) | infra, cli |
| `routing/` | 10 | ~1,800+ | Agent route resolution, session key construction, binding matching | config, channels, sessions |
| `sessions/` | 8 | ~600+ | Session utilities: key parsing, send policy, transcript events, provenance | config, channels |
| `providers/` | 11 | ~1,200+ | Provider-specific auth: GitHub Copilot OAuth, Qwen refresh | config, agents (auth-profiles) |
| `acp/` | 43 | ~2,800+ | Agent Client Protocol bridge for IDE/editor integration | @agentclientprotocol/sdk, gateway |
| `pairing/` | 7 | ~1,500+ | Device pairing: code generation, approval, account-scoped channel allowlists | channels, config, infra |
| `node-host/` | 11 | ~2,300+ | Remote node agent: gateway connection, command invocation, browser proxy, APNs wake support | gateway, browser, config |
| `node-host/invoke-system-run` | ~2 | ~300 | Extracted system.run hardened command resolver (security-critical) | node-host, config, process |
| `shared/` | 36 | ~2,700+ | Pure types/constants: frontmatter, requirements, reasoning tags | compat |
| `utils/` | 28 | ~2,000+ | General utilities: boolean parse, delivery context, usage format, shell argv | channels, config |
| `terminal/` | 16 | ~1,200+ | ANSI styling, tables, progress lines for CLI output | chalk |
| `compat/` | 1 | ~20 | Legacy project name constants | — |
| `types/` | 8 | ~100+ | Ambient TypeScript declarations for untyped npm packages | — |
| `wizard/` | 13 | ~2,500+ | Interactive setup wizard via @clack/prompts | cli, config, channels |
| `extensions/` | 80+ dirs | — | Channel plugins, provider auth plugins, tool/feature plugins (including new `diffs` diff-viewer plugin) | plugin-sdk |
| `extensions/diffs/` | — | — | Read-only diff viewer plugin: renders before/after text or unified patches as gateway viewer URLs and PNG images; configurable theme/layout/font defaults | plugin-sdk, playwright |
| `extensions/synology-chat/` | — | — | Synology Chat channel plugin: webhook ingress, DM routing, outbound send/media, per-account config, DM policy controls | plugin-sdk, channels, config |
| `packages/` | — | — | Workspace compatibility shims (clawdbot, moltbot) for legacy package name consumers | — |
| `context-engine/` | — | — | Plugin slot for custom context management strategies; wraps core context assembly, compaction, and subagent spawn hooks | config, agents |

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
                         │    (720 files)        │
                         └──────────┬───────────┘
                                    │
                Level 4  ┌──────────┴──────────┐
                         │     auto-reply/      │  ← Message pipeline (23+ dependents)
                         │    (260 files)        │
                         └──────────┬───────────┘
                                    │
                Level 5  ┌──────┬───┴───┬──────┬──────┬───────┐
                         │memory│ cron  │hooks │ tts  │media- │
                         │      │       │      │      │undrst │
                         └──────┘───────┘──────┘──────┘───────┘
                                    │
                Level 6  ┌──────────┴──────────┐
                         │      gateway/        │  ← Server orchestrator
                         │    (310 files)        │
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
        └─ extensions/telegram/src/bot-handlers.ts registers middleware

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
            ├─ 9c. pi-tools.ts → Register 40+ tools (exec, browser, web, memory, etc.) + plugin tools
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
        ├─ OpenClaw tools (from openclaw-tools.ts): browser, canvas, cron, message,
        │    nodes, web_search, web_fetch, image, memory, tts, sessions, subagents
        └─ Plugin tools (from extensions): diffs (read-only diff viewer with gateway URLs)

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
| `agents.defaults` | Default agent settings | `model`, `provider`, `heartbeat.*` (incl. `lightContext`), `compaction.*`, `memorySearch.*`, `allowedModels`, `modelFallbacks`, `typingIntervalSeconds`, `envelopeTimezone` |
| `bindings[]` | Channel→agent routing | `agentId`, `channel`, `account`, `peer`, `guild`, `roles`, `team` |
| `session` | Session behavior | `dmScope`, `identityLinks`, `resetTriggers`, `sendPolicy`, `store`, `mainKey` |
| `gateway` | Server config | `port`, `auth`, `tls`, `discovery`, `tailscale`, `lanes`, `nodes`, `browser` |
| `models` | Model/provider config | Model definitions, aliases, provider-specific settings |
| `authProfiles` | Auth credentials | Per-provider API keys, OAuth tokens, custom base URLs |
| `<channel>` | Per-channel config | `telegram.*`, `discord.*`, `slack.*`, `signal.*`, `whatsapp.*`, `imessage.*`, etc. |
| `hooks` | Hook system | `enabled`, `internal.*`, `gmail.*`, `path`, `token`, `presets` |
| `cron` | Cron jobs | `enabled`, `storePath`, `sessionRetention` |
| `messages` | Message handling | `tts.*`, `inbound.debounceMs`, `inbound.byChannel` |
| `tools` | Tool config | `media.*` (image/audio/video, files with `pdf.*` sub-config for `maxPages`/`maxPixels`/`minTextChars`), `links.*`, `exec.*` |
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

### Shell Environment Markers

- **Marker:** `OPENCLAW_SHELL` is set across shell-like runtimes (`exec`, `acp`, `acp-client`, `tui-local`) so shell startup/config rules can detect and adapt to OpenClaw-spawned contexts
- **Module:** `agents/bash-tools.exec-runtime.ts` (sets `OPENCLAW_SHELL=exec`), `acp/client.ts` (sets `OPENCLAW_SHELL=acp-client`)

### WS Transport Security

- **Plaintext ws:// loopback-only** — WebSocket connections over plaintext `ws://` are blocked to non-loopback addresses (CWE-319); applies to both `gateway/call.ts` (outbound self-connect) and `gateway/client.ts` (CLI/TUI client connect)
- **Enforcement:** `gateway/net.ts` classifies addresses and rejects non-loopback `ws://`; only `127.x.x.x`, `::1`, and `localhost` are permitted without TLS

### Node Exec Security (v2026.3.1 Breaking)

- **`systemRunPlan` required** — Node (`host=node`) exec approval payloads must include a `systemRunPlan` object; requests without it are rejected. See `agents/bash-tools.exec-approval-request.ts` and `gateway/node-invoke-system-run-approval.ts`
- **`system.run` realpath pinning** — Path-token commands on nodes are pinned to the canonical executable path via `realpath` in both allowlist and approval execution flows. For example, `tr` resolves to `/usr/bin/tr`; integrations must accept canonical paths. See `node-host/invoke-system-run`

### Audit & Remediation

- **Security audit:** `security/audit.ts` → `runSecurityAudit()` scans config + filesystem + channels
- **Auto-fix:** `security/fix.ts` → `fixSecurityFootguns()` — chmod state dirs to safe permissions
- **Code scanning:** `security/skill-scanner.ts` scans skill/plugin code for dangerous patterns (eval, exec, fetch)
- **Filesystem inspection:** `security/audit-fs.ts` checks POSIX mode bits and Windows ACLs

### v2026.2.15 Security Hardening

- **Sandbox docker config validation** — Hardened validation of Docker sandbox configuration to prevent config injection
- **Prompt path sanitization** — Hardened path sanitization to prevent directory traversal in prompt paths
- **Skill download path restriction** — `infra/install-safe-path.ts` restricts skill download target paths to prevent writes outside allowed directories
- **Session tool scoping** — Session tools and webhook secret fallback are now properly scoped
- **Control-UI scope preservation** — Scopes are preserved in bypass mode; XSS fix via JSON endpoint + CSP lockdown
- **Account-scoped pairing** — `pairing/pairing-store.ts` scopes pairing stores by account; allowlists for Telegram/WhatsApp are account-scoped
- **Token redaction** — Telegram bot tokens are redacted in error messages; sensitive status details redacted for non-admin scopes
- **Input sanitization** — `chat.send` message input sanitization hardened
- **LINE webhook auth** — LINE webhook fails closed when authentication is missing (previously could pass through)

### v2026.2.19 Security Hardening

- **Gateway auth defaults** — Unresolved auth now defaults to token mode with auto-generated token; explicit `mode: "none"` required for open loopback
- **hooks.token ≠ gateway.auth.token** — Gateway refuses to start if hook token and gateway auth token match, preventing credential reuse
- **SSRF hardening** — NAT64/6to4/Teredo IPv6 transition addresses and octal/hex/short/packed IPv4 representations blocked in SSRF guard
- **safeBins trusted dirs** — Binaries must resolve from trusted bin directories; untrusted paths rejected
- **Rate-limited control-plane RPCs** — `config.apply`, `config.patch`, and `update.run` limited to 3/min per device+IP
- **Plugin discovery hardening** — Blocks unsafe plugin candidates (root escapes, world-writable directories, suspicious ownership)
- **YAML 1.2 core schema** — Frontmatter parsing uses YAML 1.2 core schema; no implicit `on`/`off` boolean coercion
- **Plaintext ws:// blocked** — WebSocket connections over plaintext `ws://` blocked to non-loopback hosts
- **Security headers** — `X-Content-Type-Options: nosniff` and `Referrer-Policy: no-referrer` added to gateway HTTP responses
- **Browser SSRF** — Browser navigation routed through SSRF guard (configurable via `browser.ssrfPolicy`)
- **Canvas node-scoped sessions** — Canvas session capabilities are now node-scoped, replacing shared-IP fallback
- **Cron webhook SSRF guard** — Cron webhook delivery URLs validated through SSRF guard
- **Discord moderation permissions** — Moderation permission checks enforced on trusted sender actions
- **ACP hardening** — Session rate limiting, idle reaping, and prompt size bounds (2 MiB) for Agent Client Protocol
- **Plugin/hook path containment** — Plugin and hook paths validated with `realpath` checks to prevent symlink escapes
- **Windows daemon cmd injection** — Hardened Windows daemon service commands against command injection

### v2026.2.21 Security Hardening

- **SHA-1 → SHA-256 synthetic IDs** — Internal synthetic IDs (session keys, content addressing, etc.) migrated from SHA-1 to SHA-256. External systems storing or comparing OpenClaw-generated synthetic IDs must regenerate after upgrade; old SHA-1 IDs will not match.
- **`--no-sandbox` disabled by default in containers** — Chrome/Chromium sandbox is enabled by default in containerized runs. Restricted container environments (e.g., nested Docker without `SYS_ADMIN` capability) may need capability grants or explicit sandbox opt-out.
- **Prototype-chain traversal blocked in webhook templates** — `getByPath` in webhook template evaluation now rejects expressions traversing `__proto__`, `constructor`, or other prototype-chain keys, preventing template-injection prototype pollution.
- **Heredoc command substitution blocked** — The exec tool preflight guard now rejects `$(cmd)` and backtick substitutions in unquoted heredoc bodies, preventing shell command injection through heredoc expansion.
- **noVNC observer requires token auth** — noVNC observer sessions now use one-time token authentication; unauthenticated external noVNC connections are rejected.
- **Tailscale tokenless auth restricted to WebSocket** — Tailscale-based tokenless auth is accepted only on WebSocket connections; HTTP API calls via Tailscale require explicit token auth.
- **WhatsApp JID allowlist enforced on all send paths** — All outbound WhatsApp sends (including tool-initiated) now validate the target JID against the configured allowlist.

### v2026.2.23 Security Hardening

- **Exec: obfuscated command detection** — The exec preflight guard now detects obfuscated commands (e.g., base64-encoded payloads, variable-expansion tricks) before consulting the allowlist; obfuscated commands are blocked regardless of allowlist entries.
- **Exec: safe-bin PATH trust removed** — The implicit trust of `safeBins` resolved via `PATH` has been removed. Explicit trust must be declared via `tools.exec.safeBinTrustedDirs`; binaries outside declared trusted directories are rejected even if their names appear in the safe-bin list.
- **Exec: shell env sanitization** — Shell execution now sanitizes the environment before spawning child processes; `HOME`, `ZDOTDIR`, `SHELLOPTS`, and `PS4` overrides from untrusted input are blocked to prevent shell startup file hijacking and debug-hook injection.
- **SSRF: expanded RFC special-use ranges** — The SSRF guard now blocks benchmarking (`198.18.0.0/15`), TEST-NET (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`), and multicast (`224.0.0.0/4`) address ranges in addition to existing private/link-local ranges.
- **SSRF: IPv6 dotted-quad transition literal normalization** — IPv6 addresses embedding dotted-quad IPv4 literals (e.g., `::ffff:192.168.1.1`) are now fully normalized and checked against private-range rules, closing a bypass via IPv6 transition syntax.
- **Archive: zip symlink escape blocking** — Archive extraction (skill/plugin install) now rejects zip entries whose resolved target path escapes the extraction root via symlink traversal.

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

1. **Config paths** — Explicit paths in `plugins.load.paths`
2. **Workspace** — Agent workspace `.openclaw/extensions/` directory
3. **Bundled** — `extensions/` shipped with OpenClaw (bundled-by-default: `device-pair`, `phone-control`, `talk-voice`)
4. **Global** — `~/.openclaw/extensions/` (auto-discovered user-installed extensions, kept behind bundled entries unless explicitly configured)

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
- Lifecycle events: `onLoad`, `onUnload`, `onConfigChange`, `onSessionStart`, `onSessionEnd`, `onAgentBootstrap`, `before_dispatch`

### Extension Types

| Type | Count | Examples |
|------|-------|---------|
| **Channel plugins** | 21 | telegram, discord, slack, signal, whatsapp, line, irc, matrix, msteams (official Teams SDK), nostr, twitch, zalo |
| **Provider plugins** | 7 | copilot-proxy, google-gemini-cli-auth, minimax-portal-auth, qwen-portal-auth, ollama, vllm, sglang |
| **Tool/Feature plugins** | 11 | memory-core, memory-lancedb, llm-task, lobster, open-prose, diagnostics-otel, thread-ownership, diffs |

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

Via pi-ai and auth profiles: **Anthropic**, **OpenAI**, **Google (Gemini, incl. Gemini 3.1)**, **xAI (Grok)**, **AWS Bedrock**, **Azure OpenAI**, **Ollama** (local), **Together.ai**, **Venice.ai**, **HuggingFace**, **MiniMax** (including image generation), **Qwen**, **Volcengine/BytePlus (Doubao)**, **OpenCode/Zen**, **GitHub Copilot**, **Cloudflare AI Gateway**, **Chutes**, **Mistral**, **SiliconFlow**, and any OpenAI-compatible API.

### Provider-Specific Auth

| Provider | Auth Method | Module |
|----------|-------------|--------|
| GitHub Copilot | Device code OAuth → token exchange | `providers/github-copilot-auth.ts` |
| Qwen Portal | OAuth2 refresh token | `providers/qwen-portal-oauth.ts` |
| Gemini CLI | OAuth (plugin) | `extensions/google-gemini-cli-auth/` |
| MiniMax Portal | OAuth (plugin) | `extensions/minimax-portal-auth/` |
| Mistral | API key | auth profiles |
| Vercel AI Gateway | API key, shorthand normalization, implicit provider synthesis, and catalog/static fallback | auth profiles, `agents/models-config.providers.ts` |
| Google Vertex AI | OAuth / service account | auth profiles |

### Provider Auto-Detection and Normalization

- **Vercel AI Gateway provider synthesis** — Model refs of the form `vercel-ai-gateway/claude-*` still normalize to the canonical Anthropic ID, but the provider path now also performs implicit provider synthesis, catalog discovery, and static fallback modeling (including GPT-5.4) rather than acting as shorthand-only sugar.
- **Google Vertex AI routing for Claude** — Claude models can be routed through Google Vertex AI by selecting the `vertex` provider in auth profiles; model IDs are translated to Vertex-compatible publisher/model paths automatically.
- **Grounded Gemini web search support** — Google Gemini models accessed via Vertex AI or the standard Gemini API support grounded web search responses; OpenClaw surfaces the grounding metadata in tool results.
- **Mistral embeddings and voice support** — Mistral models are available for both text generation and embeddings (via `memory/embeddings.ts` provider selection); Mistral voice/audio models are accessible through the standard TTS pipeline.

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
| Telegram | grammY | Forum topics, per-DM topic routing/authorization, inline buttons, draft streaming, reactions, proxy support |
| Discord | @buape/carbon | Guilds, threads (with inactivity lifecycle: `idleHours` default 24h, `maxAgeHours`), polls, PluralKit, presence, admin actions, Component v2 UI (`components.ts`, `components-registry.ts`, `send.components.ts`) |
| Slack | @slack/web-api | Socket Mode, slash commands, file uploads, pins |
| Signal | signal-cli REST | JSON-RPC + SSE, groups, reactions, daemon management |
| LINE | @line/bot-sdk | Flex Messages, Rich Menus, markdown→flex conversion |
| iMessage | imsg CLI | JSON-RPC over stdin/stdout, macOS native |
| BlueBubbles | BlueBubbles server (HTTP/WS) | iMessage via BlueBubbles, macOS server required |
| WhatsApp | Baileys | QR login, media conversion, broadcast groups |
| Feishu | Feishu Open Platform SDK | Docx tables/uploads, reactions, chat tools, reply-in-thread, group session scopes (`group`/`group_sender`/`group_topic`/`group_topic_sender`), multi-account with `defaultAccount` routing, typing backoff |
| Synology Chat | Webhook (HTTP) | Webhook ingress, DM routing, outbound send/media, per-account config, DM policy controls |

---

## 14. Key Design Patterns

### 1. Pipeline / Chain of Responsibility
**Where:** `auto-reply/` — The entire message processing pipeline: dispatch → directives → commands → agent run → delivery. Each stage can short-circuit with a reply or pass to the next.

### 2. Observer / Event Bus
**Where:** `hooks/internal-hooks.ts` (register/trigger), `sessions/transcript-events.ts` (pub/sub), `infra/system-events.ts`, `infra/heartbeat-events.ts`, `gateway/server-broadcast.ts` (WS event broadcast).

### 3. Registry Pattern
**Where:** `plugins/registry.ts` (PluginRegistry), `agents/bash-process-registry.ts` (running processes), `agents/subagent-registry.ts`, `gateway/node-registry.ts`, `auto-reply/reply/dispatcher-registry.ts`, `auto-reply/commands-registry.ts`.

### 4. Factory Pattern
**Where:** `memory/embeddings.ts` (createEmbeddingProvider), `plugins/registry.ts` (createPluginRegistry), `auto-reply/reply/reply-dispatcher.ts` (createReplyDispatcher), `agents/pi-tools.ts` (createOpenClawTools).

### 5. Strategy / Tiered Matching
**Where:** `routing/resolve-route.ts` — Binding resolution uses ordered predicate tiers (peer → guild+roles → channel → default). `agents/model-selection.ts` — Model resolution with alias → fuzzy → allowlist strategies.

### 6. Middleware Pattern
**Where:** `agents/tool-policy-pipeline.ts` (tool call policies), `agents/pi-tools.before-tool-call.ts` (pre-tool hooks), `gateway/http-auth-helpers.ts` (HTTP auth boundary helpers), `telegram/bot.ts` (grammY middleware).

### 7. Builder Pattern
**Where:** `agents/system-prompt.ts` — System prompt construction assembles sections: identity, skills, memory, tools, workspace, runtime, time. `agents/workspace.ts` — Bootstrap file assembly.

### 8. Guard / Circuit Breaker
**Where:** `agents/context-window-guard.ts` (context overflow), `agents/session-tool-result-guard.ts` (oversized results), `agents/pi-extensions/compaction-safeguard.ts` (infinite compaction loops), `auto-reply/reply/agent-runner-execution.ts` (error recovery with session reset).

### 16. Plugin Slot / Registry Pattern (v2026.3.7+)
**Where:** `context-engine/` — `registry.ts` defines the `ContextEnginePlugin` interface; `legacy.ts` wraps the built-in context assembly as the default plugin. In `v2026.3.8`, runtime plugin loading was widened so context-engine plugins are resolved before compaction and subagent lifecycle boundaries, and the registry is shared across duplicated bundled chunks via a process-global singleton. Lifecycle hooks (`bootstrap`, `ingest`, `assemble`, `compact`, `afterTurn`, `prepareSubagentSpawn`, `onSubagentEnded`) allow full replacement of context management behavior without touching core agent code.

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
| **Total src/ modules** | 70 directories |
| **Total source files** | 2,600+ `.ts` files (non-test) |
| **Total test files** | 1,600+ `.test.ts` files |
| **Total src lines** | 780,000+ (`src/**/*.ts`) |
| **External framework** | @mariozechner/pi-ai, pi-agent-core, pi-coding-agent |
| **Language** | TypeScript (Node.js) |
| **Test framework** | Vitest |
| **Build** | TypeScript compiler (tsc) |

### Largest Modules (by file count)

| Rank | Module | Source Files | Test Files |
|------|--------|-------------|------------|
| 1 | `agents/` | 370 | 350 |
| 2 | `commands/` | 220 | 110 |
| 3 | `infra/` | 205 | 130 |
| 4 | `gateway/` | 200 | 110 |
| 5 | `auto-reply/` | 185 | 75 |
| 6 | `cli/` | 180 | 88 |
| 7 | `config/` | 125 | 80 |
| 8 | `channels/` | 100 | 50 |
| 9 | `discord/` | 80 | 52 |
| 10 | `browser/` | 78 | 46 |

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

| Metric | Count |
|--------|-------|
| Extension directories (`extensions/*`) | 45 |
| Extension packages (`extensions/*/package.json`) | 33 |
| Bundled skills (`skills/*`) | 52 |

> Current analyzed source package: released `v2026.3.23` with shipped correction tag `v2026.3.23-2`. Counts measured from the `v2026.3.23-2` released tree.

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

## v2026.2.15 Changes (2026-02-16)

### Security (7 fixes)

1. **Sandbox bind validation** — Docker bind-mount paths are now validated against an allowlist; rejects traversal attempts and symlink escapes in `agents/sandbox/`
2. **Prompt path sanitization** — Directory traversal sequences (`../`) in prompt file paths are stripped before resolution, preventing reads outside agent workspaces
3. **Control UI XSS** — The gateway Control UI no longer renders untrusted HTML; status endpoint switched to JSON-only + Content-Security-Policy headers locked down
4. **Skill download path restriction** — `infra/install-safe-path.ts` confines skill artifact extraction to `~/.openclaw/skills/`; writes outside are rejected
5. **Sensitive field redaction** — Telegram bot tokens and other secrets are now redacted in error messages and non-admin status responses (`config/redact-snapshot.ts`, `logging/redact.ts`)
6. **Session tool scoping** — Tools and webhook-secret fallback are scoped to the owning session, preventing cross-session leakage
7. **Pairing account isolation** — `pairing/pairing-store.ts` scopes stores and channel allowlists (Telegram, WhatsApp) per account ID

### New Features

- **Discord Components v2 UI** — Rich interactive messages via `discord/components.ts`, `components-registry.ts`, and `send.components.ts`; supports buttons, selects, modals, media galleries, and containers
- **Per-channel `ackReaction`** — Configurable acknowledgment reaction emoji per channel in `messages.inbound.byChannel`
- **Plugin LLM hooks** — Plugins can register `before_llm_call` / `after_llm_call` hooks for request/response interception (e.g., logging, guardrails)
- **Multi-image tool calls** — The `image` tool now accepts an array of up to 20 images in a single call for batch vision analysis
- **Nested subagent orchestration** — Subagents can spawn their own children (depth 2, max 5 per parent) via `agents/subagent-registry.ts`; results auto-announce upward
- **Account selector for pairing** — Device pairing flow now prompts for target account when multiple accounts exist on a channel
- **Cross-platform skill install fallback** — Skill install detects OS package manager (brew/apt/choco) and falls back gracefully with manual instructions
- **`messages.suppressToolErrors`** — New config flag to hide tool-call error details from end users (errors still logged)

### Major Refactor

- **Channel deduplication** — Shared helpers extracted from per-channel send/normalize code into `channels/plugins/outbound/direct-text-media.ts` and `channels/plugins/normalize/shared.ts`; reduces duplicated media-upload, markdown-conversion, and chunk-splitting logic across Telegram, Discord, Slack, LINE, and Signal

### Performance

- **Cache-busting skip for bundled hooks** — Bundled hook files (shipped with OpenClaw) skip filesystem mtime checks on load, avoiding unnecessary `stat()` calls
- **Mtime-based workspace hook caching** — User workspace hooks are cached by file mtime; only re-evaluated when the file actually changes, eliminating redundant `jiti` loads

### Bug Fixes

- **`before_tool_call` hook double-fire** — Guard added in `agents/pi-tools.before-tool-call.ts` to prevent the hook from firing twice per tool invocation
- **Cron spin-loop floor** — `cron/service/timer.ts` enforces a minimum 1-second floor on `setTimeout` delay, preventing CPU spin when `nextRunAtMs` is in the past
- **Stale SQLite WAL connection** — `memory/` now detects and reconnects stale WAL-mode SQLite connections that silently stop returning results after long idle periods

---

## v2026.2.21 Changes (2026-02-21)

### New Modules

- **`discord/voice/`** — Voice channel management: join/leave/status, auto-join on trigger, realtime voice conversation support
- **`channels/status-reactions.ts`** — Shared lifecycle reaction controller extracted from Telegram/Discord; centralizes status-emoji reactions (typing, thinking, error) across both channels
- **`discord/monitor/thread-bindings/`** — Thread-bound subagent session management; tracks and reuses Discord thread ↔ subagent session bindings across interactions
- **`node-host/invoke-system-run.ts`** — Extracted from `invoke.ts`; hardened `system.run` command resolver with stricter validation (security-critical)

### New Channels

- **BlueBubbles** — iMessage via BlueBubbles server (HTTP/WebSocket); requires BlueBubbles running on macOS; added as a first-class channel alongside the existing `imsg` CLI path

### New Providers

- **Gemini 3.1** — Added to the Google provider family; accessible via standard `google/` auth profile
- **Volcengine/BytePlus (Doubao)** — ByteDance's Doubao model family added via OpenAI-compatible endpoint

### Security (7 fixes)

See [§9 v2026.2.21 Security Hardening](#v20262121-security-hardening) for details.

1. **SHA-1 → SHA-256 synthetic IDs** — Internal ID generation migrated; external systems must regenerate stored IDs
2. **`--no-sandbox` disabled in containers** — Chrome sandbox enabled by default; container environments need `SYS_ADMIN` or explicit opt-out
3. **Prototype-chain traversal blocked** — Webhook template `getByPath` rejects `__proto__`/`constructor` traversal
4. **Heredoc command substitution blocked** — Exec preflight guard rejects `$(cmd)` and backticks in heredoc bodies
5. **noVNC token auth required** — noVNC observer sessions require one-time token; unauthenticated connections rejected
6. **Tailscale tokenless auth WebSocket-only** — HTTP API calls via Tailscale now require explicit token auth
7. **WhatsApp JID allowlist enforced on all send paths** — Tool-initiated WhatsApp sends now validate JID against allowlist

### Bug Fixes

- **`cron.maxConcurrentRuns` now enforced** — Timer loop now enforces the `maxConcurrentRuns` limit; previously silently ignored
- **`senderIsOwner` propagated to subagent runners** — Ownership context is now forwarded to embedded/subagent runners; owner-only tools no longer fail silently in subagent contexts
- **`channels.telegram.streaming` simplified to boolean** — Legacy `streamMode` enum replaced with boolean; auto-mapper handles old values but explicit config should be updated

---

## v2026.2.22 Changes (2026-02-23)

### New Channel

- **Synology Chat** — Native webhook-based channel plugin with DM routing, outbound send/media, per-account config, and DM policy controls (`extensions/synology-chat/`)

### New Provider

- **Mistral** — Full provider support including memory embeddings and voice capabilities; use `model: "mistral/<model-id>"`

### Major Features

- **Grounded Gemini Web Search** — Web search with grounding via Gemini provider; enable with `tools.webSearch.provider: "gemini"`
- **Full Control UI Cron Edit Parity** — Complete web cron management: clone, validation, run history with pagination/search/sort/multi-filter
- **Optional Auto-Updater** — Built-in auto-updater (`update.auto.*`), default-off; `openclaw update --dry-run` for previews
- **Memory FTS Multilingual Expansion** — Full-text search now supports Spanish, Portuguese, Japanese, Korean, and Arabic stop-word filtering and tokenization
- **Tools Panel Data-Driven** — Control UI Tools panel driven from runtime `tools.catalog` with per-tool provenance labels

### Breaking Changes

1. **Google Antigravity Provider Removed** — `google-antigravity/*` model refs no longer work; migrate to `google-gemini-cli`
2. **Tool-Failure Replies Hide Raw Errors** — Detailed error suffixes hidden by default; use `/verbose on` or `/verbose full`
3. **`session.dmScope` Defaults to `per-channel-peer`** — New installs default to per-channel-peer; set `"session": {"dmScope": "main"}` for shared DM continuity
4. **Unified Streaming Config + Device Auth v1 Removed** — `channels.<channel>.streaming` uses enum `off | partial | block | progress`; device auth v1 signatures rejected

### Security Hardening (30+ fixes)

#### Exec Approval System
- Safe-bin PATH hijacking prevention — `tools.exec.safeBinTrustedDirs` for explicit trusted directories
- Wrapper-path bypass fix — Inner executable persisted, not wrapper binary
- Shell line continuation blocking — `\\\n`/`\\\r\n` fail closed
- `env` wrapper transparency
- Safe-bin profiles required for custom binaries
- `sort --compress-program` bypass blocked
- macOS app basename matching hardened
- Sandbox fail-closed when unavailable
- Shell exec env sanitization — `HOME`/`ZDOTDIR`/`SHELLOPTS`/`PS4` blocked
- Shell startup injection prevention

#### SSRF Hardening
- Expanded IPv4 fetch guard to RFC special-use ranges
- IPv6 dotted-quad transition literal normalization
- `autoSelectFamily` for IPv6 fallback
- MSTeams SharePoint allowlist enforcement

#### Symlink Escape Prevention
- Browser uploads accept in-root symlink paths
- Zip extraction blocks symlink escapes
- Media sandbox enforces symlink checks
- Control UI blocks symlink-based out-of-root reads
- Gateway avatars block symlink traversal
- Hooks transforms enforce symlink-safe containment

#### Auth and Identity
- Elevated scope bypass fix — `tools.elevated.allowFrom` matches sender only
- Feishu display-name collision prevention — ID-only matching
- Group policy `toolsBySender` requires explicit sender-key types
- Discord allowlist canonicalization
- Prototype pollution prevention in config merge

### Channel Updates

**Telegram:**
- WSL2 `autoSelectFamily` disabled by default
- Node 22+ `ipv4first` DNS default
- Forward burst coalescing
- Streaming preview preservation
- Polling offset watermark and stuck-runner restart
- Webhook keepalive and `webhookPort` config

**Slack:**
- Threading beyond first turn
- `replyToMode` respect with auto-populated `thread_ts`
- Extension thread forwarding
- Upload user ID resolution for DM channels

**Discord:**
- Allowlist canonicalization to IDs
- Security audit warnings for name/tag entries

---

## v2026.2.24 Changes (2026-02-25)

### Security Model

- **Multi-user heuristic** — New `security.trust_model.multi_user_heuristic` config key flags likely shared-user ingress patterns and clarifies the personal-assistant trust model. Hardening guidance for intentional multi-user deployments: `sandbox.mode="all"`, workspace-scoped filesystem, reduced tool surface, no personal/private identities on shared runtimes.
- **Native image ingestion boundary** — `tools.fs.workspaceOnly` is now enforced for native prompt image auto-load (including message history refs), closing an implicit out-of-workspace mount vector in the vision pipeline.
- **Exec approval argv contract** — `system.run` approval text is bound to the full argv for shell-wrapper payloads; rawCommand mismatches are rejected, strengthening the approval integrity model.
- **Bind-mount canonicalization** — Bind-mount source paths are canonicalized via `realpath` before allowed-source-root and blocked-path checks, closing the symlink-parent bypass.

### Gateway

- **Trusted-proxy Control UI auth** — Trusted-proxy authenticated WebSocket sessions can skip device pairing when device identity is absent, enabling use behind reverse proxies without false pairing failures.

---

## v2026.2.23 Changes

### New Providers

- **Kilo Gateway** — `kilocode` provider with auth, onboarding, implicit detection; default model `kilocode/anthropic/claude-opus-4.6`
- **Vercel AI Gateway provider synthesis** — shorthand normalization remains, but provider discovery/static fallback now synthesize richer model entries including GPT-5.4-compatible catalog behavior

### Major Features

- **Moonshot Video Provider** — Native video understanding with auto key detection
- **Kimi Web Search** — `provider: "kimi"` with citation extraction
- **Session Maintenance Hardening** — `openclaw sessions cleanup`, disk-budget controls, per-agent targeting
- **HSTS Support** — Optional `gateway.http.securityHeaders.strictTransportSecurity`
- **Per-Agent Params Overrides** — `params` merged on top of model defaults including `cacheRetention`
- **Bootstrap File Caching** — Per-session snapshots reduce prompt-cache invalidations

### Breaking Changes

1. **Browser SSRF Policy Default** — Browser SSRF policy now defaults to trusted-network mode (`browser.ssrfPolicy.dangerouslyAllowPrivateNetwork=true` when unset); canonical config key is `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` (replaces `browser.ssrfPolicy.allowPrivateNetwork`). `openclaw doctor --fix` migrates the legacy key automatically.

### Security Fixes

- **Exec Approvals:** Node-bound approvals, two-phase registration, canonical wrapper plans, busybox/toybox handling
- **Safe-Bins:** Long-option validation, sort flags denied, shell env hardening
- **WhatsApp:** JID allowlist on all sends enforced
- **Reasoning:** Payload suppression in shared channel dispatch
- **Config:** Prototype pollution prevention

### Key Fixes

- Telegram reactions soft-fail, polling offset scoping, reasoning suppression
- WhatsApp group policy, DM routing, `selfChatMode`, logging redaction
- Discord thread parent ID recovery
- Web UI locale hydration
- Browser control reliability, relay port derivation
- Isolated cron full prompt mode
- Plugin config schema fallback

---

## v2026.3.1 Changes (2026-03-02)

### New Tools & Plugins

- **Diffs plugin** (`extensions/diffs/`) — Read-only diff viewer and image renderer for agents; renders before/after text or unified patches, with gateway viewer URLs for canvas and PNG image output. Configurable defaults for theme, layout, font, line numbers, word wrap, and diff indicators. Registered as an optional plugin tool.
- **Browser `pdf` action** — The browser tool now includes a `pdf` action for saving the current page as a PDF file, with proxy request support for node-hosted browsers.

### New Gateway Features

- **Container health probes** — Built-in HTTP liveness/readiness endpoints at `/health`, `/healthz`, `/ready`, `/readyz` for Docker/Kubernetes health checks. Returns `{ ok: true, status: "live"|"ready" }`. Supports GET and HEAD methods. Fallback routing preserves existing handlers on those paths. Implemented in `gateway/server-http.ts` with `GATEWAY_PROBE_STATUS_BY_PATH` map.
- **Health RPC handlers** — `gateway/server-methods/health.ts` provides cached health snapshots with configurable refresh intervals and probe-on-demand support.

### New CLI Commands

- **`openclaw config file`** — Print the active config file path resolved from `OPENCLAW_CONFIG_PATH` or the default location. Implemented in `cli/config-cli.ts`.

### Transport Changes

- **OpenAI Responses WebSocket-first** — OpenAI Responses API transport defaults to `"auto"` (WebSocket-first with SSE fallback). Shared OpenAI WS stream/connection runtime with per-session cleanup. Server-side compaction payload mutation (`store` + `context_management`) preserved on the WS path. Implemented in `agents/openai-ws-connection.ts` and `agents/openai-ws-stream.ts`.
- **OpenAI WS warm-up** — Optional `response.create` with `generate:false` pre-loads connections without generating output. Enabled by default for `openai/*` models. Controllable via `params.openaiWsWarmup` per model.

### Channel Changes

- **Telegram DM topics** — Per-DM `direct` + topic config with allowlists, `dmPolicy`, `skills`, `systemPrompt`, and `requireTopic`. DM topics route as distinct inbound/outbound sessions with topic-aware authorization and debounce.
- **Discord thread bindings lifecycle** — Replaced fixed TTL with inactivity-based lifecycle: `idleHours` (default 24h) plus optional hard `maxAgeHours` cap. Added `/session idle` and `/session max-age` commands for focused thread-bound sessions. Legacy `ttlHours` key auto-migrated to `idleHours`.
- **Feishu enhancements** — Docx table creation/cell writing (`create_table`, `write_table_cells`, `create_table_with_values`) and file uploads (`upload_image`, `upload_file`). Inbound reaction handling with configurable `reactionNotifications` (`off|own|all`). Chat tools (`feishu_chat` for chat info/member queries). Reply-in-thread config (`disabled|enabled`). Group session scopes (`group`, `group_sender`, `group_topic`, `group_topic_sender`). Multi-account with `defaultAccount` routing.
- **Android nodes** — New node-tool actions: `camera.list`, `device.permissions`, `device.health`, `notifications.actions` (`open`/`dismiss`/`reply`), `system.notify`, `photos.latest`, `contacts.search`/`contacts.add`, `calendar.events`/`calendar.add`, `motion.activity`/`motion.pedometer`, and voice TTS via ElevenLabs WebSocket in Talk Mode.
- **Cross-channel multi-account** — `defaultAccount` routing via `channels/<channel>.defaultAccount` config with per-channel account helpers (`channels/plugins/account-helpers.ts`). Inbound metadata includes `account_id` for session disambiguation.

### Agent System

- **Subagent runtime events** — Ad-hoc subagent completion handoff replaced with typed internal completion events (`task_completion`). Events rendered consistently across direct and queued announce paths. Gateway/CLI plumbing for structured `internalEvents`. Implemented in `agents/internal-events.ts` and `agents/subagent-announce.ts`.
- **Thinking defaults** — Claude 4.6 models (including Bedrock Claude 4.6 refs) default to `adaptive` thinking level; other reasoning-capable models default to `low` unless explicitly configured.
- **Cron/heartbeat light bootstrap context** — Opt-in lightweight bootstrap mode for automation runs: `--light-context` for cron agent turns and `agents.*.heartbeat.lightContext` for heartbeat. Heartbeat runs keep only `HEARTBEAT.md`; cron lightweight runs skip bootstrap-file injection.

### Security

- **Shell env markers** — `OPENCLAW_SHELL` set across shell-like runtimes (`exec`, `acp`, `acp-client`, `tui-local`) so shell startup/config rules can detect OpenClaw-spawned contexts.
- **Node exec `systemRunPlan` required (BREAKING)** — `host=node` approval requests without `systemRunPlan` are rejected. See `agents/bash-tools.exec-approval-request.ts`.
- **Node `system.run` realpath pinning (BREAKING)** — Path-token commands on nodes are pinned to canonical executable paths via `realpath` in both allowlist and approval execution flows. For example, `tr` resolves to `/usr/bin/tr`.
- **WS plaintext loopback-only** — Plaintext `ws://` connections blocked to non-loopback addresses (CWE-319, CVSS 9.8). Enforced in `gateway/call.ts`, `gateway/client.ts`, and `gateway/net.ts`.

### Stats Update

- **5,102 TypeScript files** (was 4,875 at v2026.2.26)
- **938,374 lines** (was 887,472)
- **40 extension directories** (was 39), **33 packages** (was 32), **52 skills** (same)

---

## v2026.3.7 Changes (2026-03-08)

### New Module

- **`context-engine/`** — Plugin slot for custom context management strategies. Defines the `ContextEnginePlugin` interface with lifecycle hooks: `bootstrap`, `ingest`, `assemble`, `compact`, `afterTurn`, `prepareSubagentSpawn`, `onSubagentEnded`. Key files: `types.ts`, `registry.ts`, `legacy.ts` (built-in default), `index.ts`, `init.ts`. Low-medium blast radius; changes require testing compaction behavior and any plugins that implement the interface.

### New Channels & Features

- **ACP persistent bindings** — Agent Client Protocol now supports persistent session bindings across reconnects
- **Telegram topic routing** — Enhanced per-topic routing with fine-grained `requireTopic` and per-topic skills/system prompts
- **Spanish locale** — i18n pipeline extended with `es` locale support
- **Mattermost model picker** — Model selection UI added to Mattermost channel extension
- **Discord slash commands** — Native slash command registration via the `commands` adapter slot
- **Docker slim / Podman** — Sandbox image support extended to slim Docker images and Podman runtimes
- **systemd WSL2 hardening** — Daemon installer hardens systemd unit files for WSL2 environments

### New Providers

- **Gemini 3.1 Flash-Lite** — Added to the Google provider family; accessible via `google/gemini-3.1-flash-lite`
- **Venice kimi-k2-5** — `venice/kimi-k2-5` added to Venice.ai provider

### Breaking Changes

- **`gateway.auth.mode` required when both token and password are configured** — If both `gateway.auth.token` and `gateway.auth.password` are set, `gateway.auth.mode` must be explicitly declared (`"token"` or `"password"`). Omitting it causes a startup validation error. Run `openclaw doctor --fix` to detect and migrate.

### Stats Update

- **5,696 TypeScript files** (was 5,414 at v2026.3.2; +282 files, 893 commits this window)
- **40 extension directories** (stable-tag count), **33 packages**, **52 skills**

---

## v2026.3.11 Changes (2026-03-12)

### Core Runtime

- **Cron isolation hardening (BREAKING)** — cron jobs can no longer send ad hoc agent notifications or fallback main-session summaries. `openclaw doctor --fix` migrates legacy cron state and legacy notify/webhook delivery metadata.
- **Memory/multimodal indexing** — opt-in image and audio indexing for `memorySearch.extraPaths` using `gemini-embedding-2-preview` with configurable output dimensions and automatic reindexing on dimension change.
- **ACP session resume** — `sessions_spawn` with `runtime: “acp”` accepts `resumeSessionId` to resume existing ACPX/Codex sessions; `main` alias canonicalized to rehydrate restarted ACP main sessions; `loadSession` replays stored user/assistant text.
- **ACP tool streaming enrichment** — `tool_call` and `tool_call_update` events now carry best-effort text content and file-location hints.
- **Node pending-work primitives** — `node.pending.enqueue` / `node.pending.drain` in-memory queue added for dormant-node work delivery.
- **Gateway runtime version** — gateway status endpoint now exposes the running server version.
- **Control token stripping** — leaked model control tokens (`<|...|>` and `<｜...｜>` variants) are stripped from user-facing assistant text.
- **Context engine compaction guard** — overflow compaction attempts in `ownsCompaction` engines are now guarded against throws; compaction hooks fire for plugin-owned engines.

### Browser, Tools, and Platforms

- **iOS Home canvas overhaul** — bundled welcome screen with live agent overview (refreshes on connect, reconnect, and foreground return); floating controls replaced with a docked toolbar; chat opens in resolved main session.
- **macOS chat model picker** — new model picker in the macOS chat UI with persistent explicit thinking-level selections and provider-aware session model sync.
- **macOS remote onboarding shared-token detection** — the remote-gateway onboarding flow now detects when a gateway needs the shared auth token and explains where to find it.
- **Ollama onboarding** — first-class wizard with Local or Cloud + Local modes, browser-based cloud sign-in, and curated model suggestions.
- **OpenCode Go provider** — new OpenCode Go provider; Zen and Go share a single setup key while runtime providers remain split.
- **iOS local TestFlight beta flow** — Fastlane prepare/archive/upload flow added for local beta distribution, with canonical beta bundle IDs and watch-app archive fixes.
- **Brave LLM Context fix** — `llm-context` grounding snippets treated as plain strings so `web_search` no longer returns empty snippet arrays.
- **Kimi Coding tool format fix** — `kimi-coding` tools sent in native Anthropic format again, fixing degradation to XML/plain-text pseudo-invocations.
- **Discord `autoArchiveDuration`** — channel-level config for auto-created thread archiving (1h, 1d, 3d, 1w).
- **Telegram delivery hardening** — HTML-mode sends now chunk safely with preserved fallback/silent params, and final-preview cleanup no longer triggers duplicate fallback finals.

### Operations

- **macOS/launchd restart v2** — LaunchAgent stays registered during explicit restarts; self-restarts handed off through a detached launchd helper; config/hot reload paths recovered without unloading the service. Fixes #43311, #43406, #43035, #43049.
- **LaunchAgent install permissions** — LaunchAgent directory and plist permissions tightened during install to prevent launchd bootstrap failures from group/world-writable modes.
- **CLI/skills JSON** — ANSI and C1 control bytes stripped from `skills list|info|check --json` output for machine-readable validity.
- **CLI/tables legacy Windows** — ASCII borders default on legacy GBK/936 consoles; Unicode borders preserved on modern Windows terminals.

### Security

- **GHSA-5wcw-8jjv-m286 (WebSocket origin)** — browser origin validation enforced for all browser-originated connections regardless of proxy headers, closing cross-site WebSocket hijacking path in `trusted-proxy` mode.
- **Symlink-safe `*File` secret reads** — CLI and channel credential `*File` reads require direct regular files; symlink-backed secret paths are rejected.
- **TAR/bz2 extraction staging** — archives extracted into staging before merging into canonical destination; destination symlink and child-symlink escapes hardened.
- **Exec SecretRef traversal rejection** — exec SecretRef traversal IDs rejected across schema, runtime, and gateway.
- **fs-bridge write pinning** — staged writes pinned to verified parent directories before atomic replace.
- **Gateway auth fail-closed** — fails closed when local `gateway.auth.*` SecretRefs are configured but unavailable.
- **Session-reset auth split** — `/new` and `/reset` conversation paths split from the admin-only `sessions.reset` RPC.
- **Plugin HTTP route scope** — unauthenticated plugin HTTP routes cannot inherit synthetic admin gateway scopes for `runtime.subagent.*` calls.
- **`session_status` sandbox guards** — session-tree visibility and agent-to-agent access guards enforced before reads or mutations.
- **`nodes` tool owner-only policy** — `nodes` agent tool treated as owner-only fallback policy.

---

## v2026.3.8 Changes (2026-03-09)

### Core Runtime

- **Context-engine runtime loading widened** — context-engine plugins are now bootstrapped from the active runtime workspace before compaction and subagent lifecycle resolution, not just at gateway startup. The registry is also shared across duplicated bundled chunks via a process-global singleton so plugin-provided engines resolve consistently.
- **ACP provenance and lineage** — provenance now includes `originSessionId`, ACP can inject visible receipt text, and ACP-spawned sessions persist `spawnedBy` lineage plus best-effort transcript/session metadata.
- **Model/runtime follow-up hardening** — `openai-codex/gpt-5.4` now resolves with a 1,050,000-token context window and 128,000 max tokens, stale Codex configs snap back to `chatgpt.com/backend-api` with `openai-codex-responses`, and session model switches clear stale cached `contextTokens`.

### Browser, Tools, and Platforms

- **Remote CDP parity** — direct `ws://` / `wss://` CDP endpoints are first-class, wildcard debugger URLs from remote `/json/version` responses are rewritten to the external host/port, and host-local signed-in browser attach now lives on Chrome MCP `existing-session` rather than the removed extension relay path.
- **Strict browser SSRF redirect enforcement** — strict browser navigation flows now validate redirect hops and fail closed when remote tab-open flows cannot inspect them.
- **Backup CLI** — new `openclaw backup create` / `openclaw backup verify` commands create and validate local state archives, including config-only mode and workspace exclusion controls.
- **Talk/platform changes** — top-level `talk.silenceTimeoutMs` config landed with platform defaults (`700 ms` on macOS/Android, `900 ms` on iOS); macOS remote mode now exposes `gateway.remote.token`; Android Play-distributed builds remove self-update, background-location “always”, screen-record, and background mic capture behavior.

### Operations

- **launchd restart semantics changed** — supervised restart now exits and relies on launchd `KeepAlive`; `XPC_SERVICE_NAME` is treated as a supervision marker; LaunchAgent repair/restart paths `enable` before `bootstrap`.
- **Control UI asset serving refined** — bundled/package-proven hardlinked asset roots are allowed for auto-detected installs, while configured/custom roots remain on the strict hardlink boundary.
- **Podman and Docker follow-ups** — Podman bind mounts now add SELinux `:Z` relabeling when needed, inaccessible-cwd setup paths fall back safely, and the runtime Docker image is smaller.

---

## v2026.3.12 Changes (2026-03-12)

### New Tools

- **`sessions_yield` tool** — new cooperative turn-ending primitive for orchestrators. Calling `sessions_yield` ends the current turn immediately, skips any remaining queued tool work, and optionally carries a hidden follow-up payload for the next turn. Enables agents to yield control back to the caller without completing their full tool queue.

### Provider Plugin Architecture

- **Ollama, vLLM, SGLang modularized** — these three local/inference providers are now managed as provider plugins (`extensions/ollama/`, `extensions/vllm/`, `extensions/sglang/`), allowing operator-managed updates and configuration independent of the core package. No behavior change for existing configs; provider resolution and auth profiles remain unchanged.

### Dashboard-v2

- Modular UI architecture: overview, chat, config, agent, and session views are independently loaded
- Command palette, mobile bottom tabs, slash commands, search, message export, and pinned messages
- Block Kit messages in Slack shared reply delivery
- Slack opt-in interactive replies via `channels.slack.capabilities.interactiveReplies`

### Core Runtime

- **`/fast` mode toggle** — `service_tier` fast-mode for OpenAI and Anthropic; per-model config defaults; supported in TUI, ACP, and isolated cron runs
- **Context engine `sessionKey` forwarded** — `sessionKey` now flows through all ContextEngine lifecycle calls (`bootstrap`, `ingest`, `assemble`, `compact`, `afterTurn`, `prepareSubagentSpawn`, `onSubagentEnded`)
- **ACP final snapshot** — final assistant text is snapshotted before `end_turn` events are emitted, ensuring completeness in ACP session logs
- **Mattermost `replyToMode`** — `off`/`first`/`all` reply-thread modes for Mattermost channel extension
- **Compaction:** post-compaction memory sync and transcript updates; status reaction emitted during compaction; double `cache-ttl` write eliminated
- **Hooks:** fail closed on unreadable loader paths; idempotency key deduplication prevents duplicate hook invocations
- **Memory:** post-compaction session reindexing via `agents.defaults.compaction.postIndexSync` and `agents.defaults.memorySearch.sync.sessions.postCompactionForce`
- **Zalo:** markdown-to-Zalo text style parsing; stable group ID enforcement
- **Terminal:** grapheme display width and table rendering fixes

### Operations

- **Node 24 default; Node 22.14 minimum** — fresh installs default to Node 24; minimum supported runtime is now Node 22.14.0 (lowered from 22.16 in v2026.3.24)
- **Bootstrap `/pair` tokens** — `/pair` setup codes switched from shared credentials to short-lived tokens; previous shared-credential pairing codes are no longer accepted
- **Kubernetes starter path** — raw Kubernetes manifests for a minimal OpenClaw deployment added to `docs/install/kubernetes/`
- **`OPENCLAW_TZ` Docker timezone** — container deployments can pin timezone via `OPENCLAW_TZ` environment variable

### Security (20+ GHSAs)

- **GHSA-99qw:** workspace plugin trust boundary enforcement
- **GHSA-pcqg:** exec format character escape hardening
- **GHSA-9r3v:** Unicode obfuscation normalization in exec preflight
- **GHSA-f8r2:** exec allowlist glob pattern escape hardening
- **GHSA-r7vr:** `/config` and `/debug` endpoints restricted to owner-only scope
- **GHSA-rqpp:** WebSocket unbound scope strip hardening
- **GHSA-vmhq:** browser profile create/delete restricted
- **GHSA-2rqg:** spawned workspace boundary enforcement
- **GHSA-wcxr:** `session_status` sandbox visibility restricted
- **GHSA-2pwv:** device token scope capped per device
- **GHSA-jv4g + GHSA-xwx2:** WebSocket preauth connection limits
- **GHSA-6rph:** media store size cap enforcement
- **GHSA-jf5v:** `GIT_EXEC_PATH` env var blocked in exec context
- **GHSA-57jw + GHSA-jvqh + GHSA-x7pp + GHSA-jc5j:** exec approval inline loader, pnpm, npx hardening
- **GHSA-g353:** Feishu `encryptKey` validation hardened
- **GHSA-m69h:** Feishu reaction handling hardened
- **GHSA-mhxh:** LINE webhook signature validation hardened
- **GHSA-5m9r:** Zalo rate limiting enforced

---

## v2026.3.22-v2026.3.23 Released Changes (2026-03-23 to 2026-03-24 docs snapshot)

### Browser / Sandbox / Web

- **Legacy Chrome extension relay removed in released code** — `v2026.3.22` removed `driver: "extension"`, bundled relay assets, and `browser.relayBindHost`. The current host-local attach path is Chrome MCP `existing-session`, with the built-in `user` profile and optional `browser.profiles.<name>.userDataDir` for Brave, Edge, Chromium, and non-default Chrome profiles.
- **Chrome MCP readiness hardened** — `v2026.3.23` waits for attached tabs to become usable before treating an existing-session browser as ready, and reuses already-running loopback browsers more reliably on slower hosts.
- **Control UI CSP hashes** — inline bootstrap scripts are now explicitly hashed and allowed via `script-src` SHA-256 entries instead of relying on looser treatment.
- **Pluggable sandbox backends** — the released architecture now includes OpenShell and SSH sandbox backends alongside Docker-oriented flows.

### Plugins / Skills / Packaging

- **ClawHub-first install surface** — bare `openclaw plugins install <package>` now prefers ClawHub before npm, and the stable CLI has first-class `openclaw skills search|install|update`.
- **Plugin SDK migration** — the public released SDK surface is `openclaw/plugin-sdk/*`; `openclaw/extension-api` remains only as a deprecated compatibility bridge.
- **Bundled runtime sidecars restored** — `v2026.3.23` restores packaged plugin runtime sidecars in published npm installs so bundled channel/provider plugins work from packed builds again.
- **ClawHub compatibility is runtime-version aware** — the shipped correction build `v2026.3.23-2` validates package compatibility against the active runtime version instead of a stale fixed `1.2.0` constant.

### Gateway / Agents / Ops

- **Gateway auth tightened** — canvas routes now require auth and agent session reset requires admin scope.
- **Channel auth UX fixed** — single-channel `channels login` / `logout` flows auto-select the configured login-capable channel and harden channel-id handling.
- **Cron timezone correctness** — one-shot `--at ... --tz <iana>` scheduling now preserves the requested wall-clock time across DST and offset-less datetimes.
- **Same-base correction-version handling** — configs written by `2026.3.23-2` no longer spuriously warn when read by `2026.3.23`.

### Stats Update (v2026.3.24-1 docs snapshot)

- **7,627 TypeScript files** analyzed (`src/`, `extensions/`, `ui/`, `test/`, `scripts/`)
- **504,909 lines of TypeScript** (same scope)
- **80+ extension packages** and **51 bundled skills** in the released tree

---
