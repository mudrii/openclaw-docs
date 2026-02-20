# OpenClaw Codebase Analysis — Part 2: Agent System

> Updated: 2026-02-20 | Version: v2026.2.19

## 1. `src/agents/` — Agent Execution, Tool System, PI Tools

### Purpose
The core engine of OpenClaw. Handles LLM agent execution (the "PI embedded runner"), tool definitions and policy enforcement, model selection/auth/fallback, system prompt construction, sandbox management, session management primitives, skills, subagent orchestration, and workspace management. This is the largest module (~190+ files).

### Key Subsystems & Files

#### PI Embedded Runner (Agent Execution Core)
| File | Role |
|------|------|
| `pi-embedded-runner.ts` | Barrel re-export of runner subsystem |
| `pi-embedded-runner/run.ts` | Main agent run loop — sends messages to LLM, processes responses |
| `pi-embedded-runner/run/attempt.ts` | Single LLM call attempt with retry/failover |
| `pi-embedded-runner/run/params.ts` | Build LLM API call parameters |
| `pi-embedded-runner/run/payloads.ts` | Construct request payloads for different providers |
| `pi-embedded-runner/run/images.ts` | Image attachment handling in runs |
| `pi-embedded-runner/run/types.ts` | Run-related type definitions |
| `pi-embedded-runner/run/compaction-timeout.ts` | Timeout handling during compaction |
| `pi-embedded-runner/runs.ts` | Run lifecycle — queue, abort, check active/streaming state |
| `pi-embedded-runner/model.ts` | Model resolution for a run |
| `pi-embedded-runner/history.ts` | Conversation history management, turn limits |
| `pi-embedded-runner/compact.ts` | Session compaction (summarize old turns) |
| `pi-embedded-runner/system-prompt.ts` | System prompt override creation |
| `pi-embedded-runner/extensions.ts` | PI runner extension points |
| `pi-embedded-runner/lanes.ts` | Lane resolution (fast/normal execution lanes) |
| `pi-embedded-runner/logger.ts` | Runner-specific logging |
| `pi-embedded-runner/abort.ts` | Abort signal management |
| `pi-embedded-runner/cache-ttl.ts` | Response cache TTL |
| `pi-embedded-runner/extra-params.ts` | Extra LLM parameters (temperature, etc.) |
| `pi-embedded-runner/google.ts` | Google-specific turn ordering fixes |
| `pi-embedded-runner/session-manager-cache.ts` | Session manager caching |
| `pi-embedded-runner/session-manager-init.ts` | Session manager initialization |
| `pi-embedded-runner/sandbox-info.ts` | Sandbox environment info for agent |
| `pi-embedded-runner/tool-result-truncation.ts` | Truncate large tool results |
| `pi-embedded-runner/tool-split.ts` | Split SDK vs OpenClaw tools |
| `pi-embedded-runner/wait-for-idle-before-flush.ts` | Wait for idle state before flushing |
| `pi-embedded-runner/types.ts` | Core runner types (EmbeddedPiRunMeta, etc.) |
| `pi-embedded-runner/utils.ts` | Runner utilities |

#### PI Embedded Helpers (LLM API Adapters)
| File | Role |
|------|------|
| `pi-embedded-helpers.ts` | Barrel for helpers |
| `pi-embedded-helpers/bootstrap.ts` | Bootstrap file injection into context |
| `pi-embedded-helpers/errors.ts` | Error handling/classification |
| `pi-embedded-helpers/google.ts` | Google/Gemini API specifics |
| `pi-embedded-helpers/images.ts` | Image content handling |
| `pi-embedded-helpers/messaging-dedupe.ts` | Deduplicate messaging events |
| `pi-embedded-helpers/openai.ts` | OpenAI API specifics |
| `pi-embedded-helpers/thinking.ts` | Thinking/reasoning block handling |
| `pi-embedded-helpers/turns.ts` | Turn manipulation utilities |
| `pi-embedded-helpers/types.ts` | Helper types |

#### PI Embedded Subscribe (Streaming Response Processing)
| File | Role |
|------|------|
| `pi-embedded-subscribe.ts` | Main stream subscription entry |
| `pi-embedded-subscribe.raw-stream.ts` | Raw SSE/stream processing |
| `pi-embedded-subscribe.tools.ts` | Tool call extraction from stream |
| `pi-embedded-subscribe.handlers.ts` | Handler registry for stream events |
| `pi-embedded-subscribe.handlers.compaction.ts` | Compaction events in stream |
| `pi-embedded-subscribe.handlers.lifecycle.ts` | Lifecycle events (start/end) |
| `pi-embedded-subscribe.handlers.messages.ts` | Message content events |
| `pi-embedded-subscribe.handlers.tools.ts` | Tool call/result events |
| `pi-embedded-subscribe.handlers.types.ts` | Handler type definitions |
| `pi-embedded-subscribe.types.ts` | Subscribe type definitions |

#### PI Extensions
| File | Role |
|------|------|
| `pi-extensions/compaction-safeguard.ts` | Prevent infinite compaction loops |
| `pi-extensions/compaction-safeguard-runtime.ts` | Runtime state for safeguard |
| `pi-extensions/context-pruning/` | Context window pruning (extension.ts, pruner.ts, runtime.ts, settings.ts, tools.ts) |

#### Tool System
| File | Role |
|------|------|
| `pi-tools.ts` | Master tool assembly — combines coding tools + OpenClaw tools, applies policies |
| `pi-tools.types.ts` | Tool type definitions (AnyAgentTool) |
| `pi-tools.abort.ts` | Wrap tools with abort signal support |
| `pi-tools.before-tool-call.ts` | Pre-tool-call hook wrapper |
| `pi-tools.policy.ts` | Tool policy enforcement (allowlist/denylist per group/subagent) |
| `pi-tools.read.ts` | Read tool variants (sandboxed, workspace-guarded) |
| `pi-tools.schema.ts` | Schema normalization (Gemini compat, parameter cleanup) |
| `pi-tool-definition-adapter.ts` | Adapt tool definitions across providers |
| `openclaw-tools.ts` | Creates all OpenClaw-specific tools (browser, canvas, cron, message, nodes, etc.) |
| `tool-policy.ts` | Owner-only policy, profile policy, explicit allowlist |
| `tool-policy-pipeline.ts` | Pipeline for applying multiple tool policies |
| `tool-policy.conformance.ts` | Tool policy conformance checking |
| `tool-call-id.ts` | Tool call ID generation |
| `tool-display.ts` | Tool result display formatting |
| `tool-display-common.ts` | Shared display utilities |
| `tool-images.ts` | Image handling in tool results |
| `tool-mutation.ts` | Track tool mutations |
| `tool-summaries.ts` | Tool result summarization |

#### Individual Tools (`tools/` subdirectory)
| File | Role |
|------|------|
| `tools/common.ts` | Shared tool types |
| `tools/browser-tool.ts` | Browser automation tool |
| `tools/browser-tool.schema.ts` | Browser tool JSON schema |
| `tools/canvas-tool.ts` | Canvas presentation tool |
| `tools/cron-tool.ts` | Cron job management tool |
| `tools/gateway-tool.ts` | Gateway control tool |
| `tools/gateway.ts` | Gateway helpers |
| `tools/image-tool.ts` | Vision/image analysis tool |
| `tools/image-tool.helpers.ts` | Image tool utilities |
| `tools/memory-tool.ts` | Memory search/get tool |
| `tools/message-tool.ts` | Channel message sending tool |
| `tools/nodes-tool.ts` | Node device control tool |
| `tools/nodes-utils.ts` | Node tool utilities |
| `tools/session-status-tool.ts` | Session status inspection tool |
| `tools/sessions-*.ts` | Session management tools (list, history, send, spawn, announce, helpers) |
| `tools/subagents-tool.ts` | Subagent management tool |
| `tools/agents-list-tool.ts` | List configured agents |
| `tools/tts-tool.ts` | Text-to-speech tool |
| `tools/web-fetch.ts` | URL content fetching tool |
| `tools/web-search.ts` | Web search tool |
| `tools/web-tools.ts` | Web tool factories |
| `tools/web-shared.ts` | Shared web utilities |
| `tools/web-fetch-utils.ts` | Fetch utilities |
| `tools/agent-step.ts` | Agent step tracking |
| `tools/discord-actions*.ts` | Discord-specific actions (guild, messaging, moderation, presence) |
| `tools/slack-actions.ts` | Slack-specific actions |
| `tools/telegram-actions.ts` | Telegram-specific actions |
| `tools/whatsapp-actions.ts` | WhatsApp-specific actions |

#### Model Management
| File | Role |
|------|------|
| `model-selection.ts` | Model selection logic (alias resolution, provider routing) |
| `model-auth.ts` | Model authentication mode resolution |
| `model-catalog.ts` | Known model catalog |
| `model-compat.ts` | Model capability compatibility |
| `model-fallback.ts` | Failover/fallback chain |
| `model-forward-compat.ts` | Forward-compatible model handling |
| `model-scan.ts` | Scan available models |
| `model-alias-lines.ts` | Model alias parsing |
| `models-config.ts` | Models.json configuration |
| `models-config.providers.ts` | Provider-specific model config |
| `live-model-filter.ts` | Runtime model filtering |
| `live-auth-keys.ts` | Live API key validation |
| `pi-model-discovery.ts` | Dynamic model discovery |
| `synthetic-models.ts` | Synthetic/virtual model definitions |
| `bedrock-discovery.ts` | AWS Bedrock model discovery |
| `huggingface-models.ts` | HuggingFace model integration |
| `ollama-stream.ts` | Ollama local model streaming |
| `together-models.ts` | Together.ai models |
| `venice-models.ts` | Venice.ai models |
| `opencode-zen-models.ts` | OpenCode/Zen models |
| `minimax-vlm.ts` | MiniMax vision-language model |
| `cloudflare-ai-gateway.ts` | Cloudflare AI Gateway routing |

#### Auth Profiles (`auth-profiles/`)
| File | Role |
|------|------|
| `auth-profiles.ts` | Barrel export |
| `auth-profiles/profiles.ts` | Profile CRUD operations |
| `auth-profiles/store.ts` | Profile persistence |
| `auth-profiles/types.ts` | Profile type definitions |
| `auth-profiles/oauth.ts` | OAuth flow handling |
| `auth-profiles/doctor.ts` | Profile health diagnostics |
| `auth-profiles/display.ts` | Profile display formatting |
| `auth-profiles/order.ts` | Profile ordering |
| `auth-profiles/repair.ts` | Profile auto-repair |
| `auth-profiles/session-override.ts` | Per-session profile override |
| `auth-profiles/usage.ts` | Usage tracking per profile |
| `auth-profiles/external-cli-sync.ts` | Sync with external CLI credentials |
| `auth-profiles/paths.ts` | Profile file paths |
| `auth-profiles/constants.ts` | Constants |

#### Sandbox System (`sandbox/`)
| File | Role |
|------|------|
| `sandbox.ts` | Barrel export |
| `sandbox/config.ts` | Sandbox configuration |
| `sandbox/context.ts` | Sandbox execution context |
| `sandbox/docker.ts` | Docker container management |
| `sandbox/manage.ts` | Sandbox lifecycle (create/destroy) |
| `sandbox/registry.ts` | Active sandbox registry |
| `sandbox/runtime-status.ts` | Runtime status checking |
| `sandbox/fs-bridge.ts` | File system bridge (host↔sandbox) |
| `sandbox/fs-paths.ts` | Path mapping |
| `sandbox/browser-bridges.ts` | Browser bridge URLs |
| `sandbox/browser.ts` | Browser in sandbox |
| `sandbox/workspace.ts` | Workspace mounting |
| `sandbox/prune.ts` | Sandbox cleanup |
| `sandbox/tool-policy.ts` | Sandbox-specific tool policies |
| `sandbox/config-hash.ts` | Config change detection |
| `sandbox/types.ts` | Types |
| `sandbox/types.docker.ts` | Docker-specific types |
| `sandbox/constants.ts` | Constants |
| `sandbox/shared.ts` | Shared utilities |

#### Skills System (`skills/`)
| File | Role |
|------|------|
| `skills.ts` | Barrel export |
| `skills/config.ts` | Skills configuration |
| `skills/types.ts` | Skill type definitions |
| `skills/workspace.ts` | Workspace skill discovery |
| `skills/bundled-dir.ts` | Bundled skills directory |
| `skills/bundled-context.ts` | Bundled skill context |
| `skills/plugin-skills.ts` | Plugin-provided skills |
| `skills/frontmatter.ts` | SKILL.md frontmatter parsing |
| `skills/serialize.ts` | Skill serialization for system prompt |
| `skills/refresh.ts` | Skill refresh/reload |
| `skills/env-overrides.ts` | Environment-based skill overrides |

#### Subagent System
| File | Role |
|------|------|
| `subagent-registry.ts` | Registry of active subagents |
| `subagent-registry.store.ts` | Persistent subagent store |
| `subagent-announce.ts` | Subagent result announcement |
| `subagent-announce-queue.ts` | Queue for announcements |
| `subagent-depth.ts` | Nesting depth tracking |

#### System Prompt
| File | Role |
|------|------|
| `system-prompt.ts` | Main system prompt builder — assembles identity, tools, memory, skills, workspace, runtime sections |
| `system-prompt-params.ts` | Parameters for system prompt generation |
| `system-prompt-report.ts` | System prompt reporting/debugging |

#### Session & Workspace
| File | Role |
|------|------|
| `workspace.ts` | Workspace bootstrap files (AGENTS.md, TOOLS.md, etc.) |
| `workspace-dir.ts` | Workspace root resolution |
| `workspace-dirs.ts` | Multi-workspace directory handling |
| `workspace-run.ts` | Workspace run context |
| `workspace-templates.ts` | Workspace templates |
| `session-slug.ts` | Session slug generation |
| `session-write-lock.ts` | Session write locking |
| `session-file-repair.ts` | Repair corrupted session files |
| `session-transcript-repair.ts` | Repair transcript files |
| `session-tool-result-guard.ts` | Guard against invalid tool results |
| `session-tool-result-guard-wrapper.ts` | Wrapper for guard |

#### Other
| File | Role |
|------|------|
| `context.ts` | Agent context creation (loads config, resolves dirs) |
| `agent-paths.ts` | Agent directory path resolution |
| `agent-scope.ts` | Agent scope/config resolution |
| `identity.ts` | Agent identity (name, persona) |
| `identity-file.ts` | Identity from file |
| `identity-avatar.ts` | Avatar handling |
| `defaults.ts` | Default configuration values |
| `lanes.ts` | Execution lane definitions |
| `compaction.ts` | Compaction entry point |
| `context-window-guard.ts` | Context window overflow protection |
| `current-time.ts` | Current time injection |
| `date-time.ts` | Date/time formatting |
| `timeout.ts` | Agent timeout management |
| `usage.ts` | Token usage tracking |
| `bash-tools*.ts` | Shell execution tools (exec, process, PTY) |
| `pty-dsr.ts` | PTY device status report |
| `pty-keys.ts` | PTY key mapping |
| `shell-utils.ts` | Shell utilities |
| `glob-pattern.ts` | Glob pattern matching |
| `cache-trace.ts` | Cache tracing |
| `announce-idempotency.ts` | Idempotent announcements |
| `anthropic-payload-log.ts` | Anthropic API payload logging |
| `apply-patch.ts` / `apply-patch-update.ts` | Unified diff/patch application tool |
| `auth-health.ts` | Auth health checking |
| `bootstrap-files.ts` | Bootstrap file loading |
| `bootstrap-hooks.ts` | Bootstrap hook execution |
| `channel-tools.ts` | Channel-specific tool listing |
| `chutes-oauth.ts` | Chutes OAuth |
| `cli-backends.ts` | CLI backend resolution |
| `cli-credentials.ts` | CLI credential management |
| `cli-runner.ts` / `cli-runner/helpers.ts` | External CLI runner (Claude CLI, etc.) |
| `cli-session.ts` | CLI session management |
| `docs-path.ts` | Documentation path resolution |
| `failover-error.ts` | Failover error types |
| `memory-search.ts` | Memory search config resolution |
| `pi-auth-json.ts` | PI auth JSON handling |
| `pi-embedded-block-chunker.ts` | Block response chunking |
| `pi-embedded-messaging.ts` | Messaging tool for embedded runner |
| `pi-embedded-utils.ts` | Embedded runner utilities |
| `pi-embedded.ts` | Top-level embedded PI entry |
| `pi-settings.ts` | PI settings |
| `sandbox-paths.ts` | Sandbox path resolution |
| `schema/clean-for-gemini.ts` | Schema cleaning for Gemini |
| `schema/typebox.ts` | TypeBox schema utilities |
| `skills-install.ts` | Skill installation |
| `skills-status.ts` | Skill status reporting |
| `transcript-policy.ts` | Transcript policy |

### Dependencies (imports from)
`config`, `channels`, `hooks`, `plugins`, `providers`, `routing`, `sessions`, `auto-reply`, `infra`, `logging`, `utils`, `shared`, `security`, `media`, `markdown`, `gateway`, `cli`, `commands`

### Dependents (imported by)
`auto-reply`, `channels`, `cli`, `commands`, `config`, `cron`, `discord`, `gateway`, `hooks`, `imessage`, `infra`, `media-understanding`, `memory`, `node-host`, `plugin-sdk`, `plugins`, `providers`, `routing`, `security`, `signal`, `slack`, `telegram`, `tts`, `tui`, `utils`, `web`, `wizard`

**Nearly every module in the system depends on `src/agents/`.**

### Data Flow
```
Inbound message → auto-reply → agents/pi-embedded-runner/run.ts
  → builds system prompt (system-prompt.ts)
  → resolves model (model-selection.ts)
  → assembles tools (pi-tools.ts + openclaw-tools.ts)
  → calls LLM API (pi-embedded-runner/run/attempt.ts)
  → streams response (pi-embedded-subscribe.ts)
  → executes tool calls (tool handlers in tools/)
  → returns text/media reply → auto-reply for delivery
```

---

## 2. `src/auto-reply/` — Message Handling, Reply Generation, Dispatch

### Purpose
The message processing pipeline. Receives inbound messages, decides whether/how to reply, manages the reply queue, runs the agent, processes directives (slash commands, model overrides, thinking modes), handles streaming/block delivery, and dispatches replies back to channels.

### Key Files

#### Top-Level
| File | Role |
|------|------|
| `reply.ts` | Barrel export — `getReplyFromConfig`, directive extractors |
| `dispatch.ts` | `dispatchInboundMessage()` — entry point for processing an inbound message |
| `types.ts` | Core types: `GetReplyOptions`, `ReplyPayload`, `BlockReplyContext` |
| `envelope.ts` | Message envelope construction |
| `templating.ts` | `MsgContext`, `FinalizedMsgContext` — message context templating |
| `chunk.ts` | Message chunking for long replies |
| `tokens.ts` | Special tokens (SILENT_REPLY_TOKEN, etc.) |
| `thinking.ts` | Think/reasoning level types and extraction |
| `model.ts` | Model override parsing |
| `status.ts` | Reply status tracking |
| `send-policy.ts` | Send policy evaluation |
| `media-note.ts` | Media attachment notes |
| `tool-meta.ts` | Tool metadata for replies |
| `group-activation.ts` | Group chat activation logic (when to respond) |
| `inbound-debounce.ts` | Debounce rapid inbound messages |
| `heartbeat-reply-payload.ts` | Heartbeat reply construction |
| `heartbeat.ts` | Heartbeat scheduling/triggering |
| `skill-commands.ts` | Skill-related command handling |

#### Command System
| File | Role |
|------|------|
| `command-detection.ts` | Detect slash commands in messages |
| `command-auth.ts` | Command authorization |
| `commands-registry.ts` | Command registry |
| `commands-registry.data.ts` | Built-in command definitions |
| `commands-registry.types.ts` | Command type definitions |
| `commands-args.ts` | Command argument parsing |

#### Reply Pipeline (`reply/`)
| File | Role |
|------|------|
| `reply/get-reply.ts` | `getReplyFromConfig()` — main reply orchestrator |
| `reply/get-reply-run.ts` | Run the agent for a reply |
| `reply/get-reply-directives.ts` | Extract and apply directives |
| `reply/get-reply-directives-apply.ts` | Apply parsed directives |
| `reply/get-reply-directives-utils.ts` | Directive utilities |
| `reply/get-reply-inline-actions.ts` | Inline action handling |
| `reply/dispatch-from-config.ts` | Dispatch reply using config |
| `reply/agent-runner.ts` | Core agent runner wrapper |
| `reply/agent-runner-execution.ts` | Agent execution logic |
| `reply/agent-runner-helpers.ts` | Runner helpers |
| `reply/agent-runner-memory.ts` | Memory integration in agent runs |
| `reply/agent-runner-payloads.ts` | Build agent payloads |
| `reply/agent-runner-utils.ts` | Runner utilities |
| `reply/followup-runner.ts` | Follow-up message handling |

#### Command Handlers (`reply/commands-*.ts`)
| File | Role |
|------|------|
| `commands.ts` | Command dispatcher |
| `commands-core.ts` | Core commands (/new, /reset, /stop, etc.) |
| `commands-session.ts` | Session commands (/session, /sessions) |
| `commands-models.ts` | Model commands (/model, /models) |
| `commands-config.ts` | Config commands |
| `commands-status.ts` | Status commands (/status) |
| `commands-info.ts` | Info commands |
| `commands-bash.ts` | Bash execution commands |
| `commands-compact.ts` | Compaction commands |
| `commands-context.ts` | Context commands |
| `commands-context-report.ts` | Context window report |
| `commands-approve.ts` | Approval commands |
| `commands-allowlist.ts` | Allowlist commands |
| `commands-plugin.ts` | Plugin commands |
| `commands-ptt.ts` | Push-to-talk commands |
| `commands-tts.ts` | TTS commands |
| `commands-subagents.ts` | Subagent commands |
| `commands-types.ts` | Command handler types |
| `config-commands.ts` | Config-specific commands |
| `config-value.ts` | Config value display |
| `debug-commands.ts` | Debug commands |

#### Directive System (`reply/directive-*.ts`)
| File | Role |
|------|------|
| `directives.ts` | Directive extraction (elevated, reasoning, think, verbose) |
| `directive-parsing.ts` | Directive parsing from message text |
| `directive-handling.ts` | Directive handling entry |
| `directive-handling.impl.ts` | Implementation |
| `directive-handling.auth.ts` | Auth directives |
| `directive-handling.fast-lane.ts` | Fast lane directive |
| `directive-handling.model.ts` | Model directive |
| `directive-handling.model-picker.ts` | Model picker UI |
| `directive-handling.params.ts` | Parameter directives |
| `directive-handling.parse.ts` | Directive parsing |
| `directive-handling.persist.ts` | Persistent directives |
| `directive-handling.queue-validation.ts` | Queue validation |
| `directive-handling.shared.ts` | Shared directive utilities |
| `streaming-directives.ts` | Streaming-related directives |
| `line-directives.ts` | Per-line directives |

#### Reply Delivery
| File | Role |
|------|------|
| `reply-delivery.ts` | Reply delivery orchestration |
| `reply-dispatcher.ts` | `ReplyDispatcher` — buffered, ordered reply sending |
| `reply-directives.ts` | Reply-embedded directives |
| `reply-payloads.ts` | Reply payload construction |
| `reply-reference.ts` | Reply-to reference handling |
| `reply-threading.ts` | Thread/topic routing |
| `reply-elevated.ts` | Elevated reply handling |
| `reply-inline.ts` | Inline reply mode |
| `reply-tags.ts` | Reply tag extraction |
| `route-reply.ts` | Route reply to correct channel |
| `normalize-reply.ts` | Normalize reply text |
| `provider-dispatcher.ts` | Provider-specific dispatch |
| `dispatcher-registry.ts` | Dispatcher registry |

#### Queue System (`reply/queue/`)
| File | Role |
|------|------|
| `queue.ts` | Queue entry point |
| `queue/enqueue.ts` | Enqueue messages |
| `queue/drain.ts` | Drain/process queue |
| `queue/cleanup.ts` | Queue cleanup |
| `queue/directive.ts` | Queue directives |
| `queue/normalize.ts` | Queue normalization |
| `queue/settings.ts` | Queue settings |
| `queue/state.ts` | Queue state management |
| `queue/types.ts` | Queue types |

#### Other Reply Files
| File | Role |
|------|------|
| `block-reply-coalescer.ts` | Coalesce multiple block replies |
| `block-reply-pipeline.ts` | Block reply processing pipeline |
| `block-streaming.ts` | Streaming block delivery |
| `body.ts` | Reply body construction |
| `session.ts` | Session context for reply |
| `session-updates.ts` | Session state updates |
| `session-reset-model.ts` | Model reset on session change |
| `session-reset-prompt.ts` | Prompt reset |
| `session-run-accounting.ts` | Run accounting |
| `session-usage.ts` | Usage tracking |
| `history.ts` | Conversation history |
| `model-selection.ts` | Model selection for reply |
| `mentions.ts` | @mention handling |
| `groups.ts` | Group chat handling |
| `typing.ts` | Typing indicator management |
| `typing-mode.ts` | Typing mode settings |
| `inbound-context.ts` | Inbound message context building |
| `inbound-dedupe.ts` | Inbound deduplication |
| `inbound-meta.ts` | Inbound metadata |
| `inbound-text.ts` | Inbound text processing |
| `memory-flush.ts` | Memory flush after reply |
| `abort.ts` | Reply abort handling |
| `audio-tags.ts` | Audio tag parsing |
| `bash-command.ts` | Bash command in reply |
| `exec.ts` | Exec directive extraction |
| `exec/directive.ts` | Exec directive types |
| `elevated-unavailable.ts` | Elevated mode unavailable |
| `response-prefix-template.ts` | Response prefix templating |
| `stage-sandbox-media.ts` | Stage media from sandbox |
| `subagents-utils.ts` | Subagent utilities |
| `untrusted-context.ts` | Untrusted content handling |

### Dependencies
`agents`, `channels`, `config`, `infra`, `markdown`, `media-understanding`, `plugins`, `routing`, `telegram`, `tts`, `utils`

### Dependents
`agents`, `channels`, `commands`, `config`, `cron`, `discord`, `gateway`, `imessage`, `infra`, `line`, `link-understanding`, `markdown`, `media-understanding`, `plugin-sdk`, `plugins`, `sessions`, `signal`, `slack`, `telegram`, `tts`, `tui`, `web`

### Data Flow
```
Channel plugin (telegram/discord/etc.) → dispatchInboundMessage(ctx, cfg, dispatcher)
  → finalizeInboundContext() — normalize message, extract metadata
  → dispatchReplyFromConfig() — check triggers, commands, directives
    → command detected? → commands.ts handles it directly
    → agent reply needed? → getReplyFromConfig()
      → agent-runner.ts → agents/pi-embedded-runner
      → response streams back via onBlockReply/onPartialReply callbacks
  → ReplyDispatcher buffers and sends via channel plugin
```

---

## 3. `src/sessions/` — Session Management, State, Utilities

### Purpose
Lightweight session utility module providing session key parsing, send policies, model/level overrides, transcript events, and input provenance tracking. **Not** the session store itself (that's in `config/sessions`); this module provides cross-cutting session utilities.

### Key Files (7 files)

| File | Role |
|------|------|
| `session-key-utils.ts` | Parse agent session keys (`agent:<agentId>:<rest>`), detect cron/subagent keys |
| `send-policy.ts` | Evaluate whether a session is allowed to send messages (allow/deny per channel/chat type) |
| `model-overrides.ts` | Per-session model override resolution |
| `level-overrides.ts` | Per-session level/tier overrides |
| `input-provenance.ts` | Track where input came from (user, cron, subagent, etc.) |
| `transcript-events.ts` | Event emitter for session transcript updates (pub/sub pattern) |
| `session-label.ts` | Session label/display name resolution |

### Exported Functions
- `parseAgentSessionKey(key)` → `{agentId, rest}` — parse canonical session keys
- `isCronRunSessionKey(key)` → boolean
- `normalizeSendPolicy(raw)` → `"allow" | "deny"`
- `evaluateSessionSendPolicy(...)` — check if session can send to a target
- `onSessionTranscriptUpdate(listener)` → unsubscribe fn
- `emitSessionTranscriptUpdate(sessionFile)` — notify listeners

### Dependencies
`auto-reply`, `channels`, `config`

### Dependents
`agents`, `auto-reply`, `cli`, `commands`, `config`, `cron`, `gateway`, `hooks`, `infra`, `memory`, `routing`

### Data Flow
```
Session keys flow in from routing/config → parsed here
Transcript events: agents write session files → emitSessionTranscriptUpdate() → listeners (memory sync, UI)
Send policy: evaluated during reply dispatch to gate outbound messages
```

---

## 4. `src/memory/` — Memory Backends, QMD, Embeddings, Search

### Purpose
Semantic memory system. Indexes workspace files (MEMORY.md, memory/*.md, session transcripts) into a SQLite database with vector embeddings + full-text search. Supports hybrid search (BM25 + cosine similarity), multiple embedding providers (OpenAI, Gemini, Voyage, local llama), batch embedding, and QMD (Query-Memory-Document) scoped search.

### Key Files

#### Core
| File | Role |
|------|------|
| `index.ts` | Barrel export — `MemoryIndexManager`, `getMemorySearchManager` |
| `manager.ts` | `MemoryIndexManager` class — main manager with init, search, sync, embedding ops |
| `types.ts` | Core types: `MemorySearchResult`, `MemorySource`, `MemoryProviderStatus`, etc. |
| `internal.ts` | Internal utilities (path validation, extra memory path normalization) |

#### Search
| File | Role |
|------|------|
| `search-manager.ts` | `getMemorySearchManager()` — factory for search managers, caching |
| `manager-search.ts` | `searchKeyword()`, `searchVector()` — actual search implementations |
| `manager-cache-key.ts` | Cache key generation for manager instances |
| `hybrid.ts` | Hybrid search: BM25 ranking, FTS query building, result merging |
| `status-format.ts` | Format memory status for display |

#### Embedding Providers
| File | Role |
|------|------|
| `embeddings.ts` | `createEmbeddingProvider()` — provider factory (OpenAI/Gemini/Voyage/local) |
| `embeddings-openai.ts` | OpenAI embedding client |
| `embeddings-gemini.ts` | Gemini embedding client |
| `embeddings-voyage.ts` | Voyage embedding client |
| `node-llama.ts` | Local llama.cpp embeddings |
| `embedding-chunk-limits.ts` | Per-provider chunk size limits |
| `embedding-input-limits.ts` | Input length limits |
| `embedding-model-limits.ts` | Model-specific dimension limits |
| `manager-embedding-ops.ts` | Embedding operations on the manager |
| `provider-key.ts` | Provider key resolution |
| `headers-fingerprint.ts` | Auth header fingerprinting for cache invalidation |

#### Batch Embedding
| File | Role |
|------|------|
| `batch-http.ts` | HTTP-based batch embedding |
| `batch-openai.ts` | OpenAI batch API |
| `batch-gemini.ts` | Gemini batch API |
| `batch-voyage.ts` | Voyage batch API |
| `batch-output.ts` | Batch output processing |
| `openai-batch.ts` | OpenAI batch file API |

#### Sync & Indexing
| File | Role |
|------|------|
| `manager-sync-ops.ts` | Sync operations (index files, detect changes) |
| `sync-index.ts` | File indexing pipeline |
| `sync-memory-files.ts` | Sync memory markdown files |
| `sync-session-files.ts` | Sync session transcript files |
| `sync-stale.ts` | Detect and remove stale entries |
| `session-files.ts` | Session file discovery for indexing |
| `memory-schema.ts` | SQLite schema definitions |

#### QMD (Query-Memory-Document)
| File | Role |
|------|------|
| `qmd-manager.ts` | QMD manager — scoped memory queries |
| `qmd-query-parser.ts` | Parse QMD query syntax |
| `qmd-scope.ts` | QMD scope resolution |

#### Storage
| File | Role |
|------|------|
| `sqlite.ts` | SQLite database management |
| `sqlite-vec.ts` | sqlite-vec extension loading for vector search |
| `backend-config.ts` | Backend configuration |

### Dependencies
`agents`, `cli`, `config`, `infra`, `logging`, `sessions`, `utils`

### Dependents
`agents` (memory-tool), `cli`, `commands`, `gateway`

### Data Flow
```
Workspace files (MEMORY.md, memory/*.md, session transcripts)
  → sync-memory-files / sync-session-files discovers files
  → sync-index chunks text, generates embeddings via provider
  → stored in SQLite (chunks table + sqlite-vec for vectors + FTS5 for text)

Agent tool call (memory_search "query")
  → agents/tools/memory-tool.ts → getMemorySearchManager()
  → MemoryIndexManager.search(query)
    → parallel: searchVector (cosine similarity) + searchKeyword (BM25/FTS)
    → mergeHybridResults() → ranked MemorySearchResult[]
  → returned to agent as tool result
```

---

## 5. `src/cron/` — Scheduled Jobs, Timer, Delivery

### Purpose
Scheduled job system. Manages cron jobs that fire agent turns, system events, or announcements on schedules (cron expressions, intervals, one-shot "at" times). Runs jobs in isolated agent sessions or the main session. Handles delivery of results to channels.

### Key Files

#### Service
| File | Role |
|------|------|
| `service.ts` | `CronService` class — public API (start, stop, list, add, remove, patch, trigger) |
| `service/state.ts` | Service state management, `CronServiceDeps` |
| `service/ops.ts` | Core operations (start, stop, list, add, remove, run) |
| `service/timer.ts` | Timer loop — schedules next job execution |
| `service/store.ts` | Persistence (read/write cron store file) |
| `service/jobs.ts` | Job manipulation utilities |
| `service/locked.ts` | Lock management for concurrent access |
| `service/normalize.ts` | Job normalization |

#### Execution
| File | Role |
|------|------|
| `isolated-agent.ts` | Barrel — `runCronIsolatedAgentTurn()` |
| `isolated-agent/run.ts` | Run a cron job as an isolated agent turn |
| `isolated-agent/session.ts` | Session setup for isolated cron runs |
| `isolated-agent/helpers.ts` | Execution helpers |
| `isolated-agent/delivery-target.ts` | Resolve delivery target for cron results |

#### Scheduling & Parsing
| File | Role |
|------|------|
| `schedule.ts` | Schedule computation (next run time) |
| `parse.ts` | Parse cron expressions and schedule strings |
| `normalize.ts` | Normalize schedule inputs |
| `validate-timestamp.ts` | Timestamp validation |

#### Delivery & Storage
| File | Role |
|------|------|
| `delivery.ts` | `resolveCronDeliveryPlan()` — determine how/where to deliver results |
| `store.ts` | Low-level store operations |
| `run-log.ts` | Execution run logging |
| `payload-migration.ts` | Migrate legacy payload formats |
| `session-reaper.ts` | Clean up old cron session files |
| `types.ts` | All cron types: `CronJob`, `CronSchedule`, `CronPayload`, `CronDelivery`, etc. |

### Dependencies
`agents`, `channels`, `cli`, `config`, `gateway`, `infra`, `plugins`, `routing`, `sessions`

### Dependents
`agents` (cron-tool), `gateway`

### Data Flow
```
User creates cron job via /cron command or cron tool
  → CronService.add() → persisted to cron-store.json
  → timer.ts polls for next due job
  → job fires → isolated-agent/run.ts
    → creates isolated session
    → runs agent turn (via agents/pi-embedded-runner)
    → collects response
  → delivery.ts resolves where to send
  → announces result to channel (if delivery mode = "announce")
```

---

## 6. `src/hooks/` — Hook System, Plugin Hooks, Lifecycle

### Purpose
Event-driven hook system for agent lifecycle events. Hooks are JavaScript/TypeScript handlers triggered by events like `command:new`, `session:start`, `agent:bootstrap`. Supports bundled hooks, workspace hooks, and plugin-provided hooks. Hooks can inject bootstrap files, log commands, manage session memory, etc.

### Key Files

#### Core
| File | Role |
|------|------|
| `hooks.ts` | Barrel — re-exports internal-hooks with friendlier names (`registerHook`, `triggerHook`, etc.) |
| `internal-hooks.ts` | Core hook registry — Map-based handler storage, `registerInternalHook()`, `triggerInternalHook()`, `createInternalHookEvent()` |
| `types.ts` | All hook types: `Hook`, `HookEntry`, `HookSource`, `OpenClawHookMetadata`, `HookSnapshot` |
| `config.ts` | Hook configuration (shouldIncludeHook, filtering) |

#### Loading & Installation
| File | Role |
|------|------|
| `loader.ts` | Load hooks from directories and files |
| `install.ts` | Hook installation logic |
| `installs.ts` | Multi-hook installation management |
| `workspace.ts` | Discover hooks from workspace directory |
| `frontmatter.ts` | Parse HOOK.md frontmatter |
| `bundled-dir.ts` | Resolve bundled hooks directory |
| `hooks-status.ts` | Hook status reporting |

#### Plugin Integration
| File | Role |
|------|------|
| `plugin-hooks.ts` | `registerPluginHooksFromDir()` — load and register plugin-provided hooks |

#### Bundled Hooks (`bundled/`)
| File | Role |
|------|------|
| `bundled/boot-md/handler.ts` | Injects boot.md content into agent context |
| `bundled/bootstrap-extra-files/handler.ts` | Loads extra bootstrap files |
| `bundled/command-logger/handler.ts` | Logs slash commands |
| `bundled/session-memory/handler.ts` | Saves session context to memory on session events |

#### Gmail Integration
| File | Role |
|------|------|
| `gmail.ts` | Gmail hook — watch for emails |
| `gmail-watcher.ts` | Gmail polling/watching |
| `gmail-ops.ts` | Gmail operations (read, send) |
| `gmail-setup-utils.ts` | Gmail OAuth setup utilities |

#### Other
| File | Role |
|------|------|
| `llm-slug-generator.ts` | Generate slugs using LLM |

### Dependencies
`agents`, `cli`, `config`, `infra`, `logging`, `markdown`, `plugins`, `process`, `shared`

### Dependents
`agents`, `auto-reply`, `cli`, `commands`, `config`, `gateway`, `plugin-sdk`, `plugins`

### Data Flow
```
Events emitted throughout the system:
  auto-reply/commands → triggerHook("command", {action: "new", ...})
  session lifecycle → triggerHook("session", {action: "start", ...})
  agent bootstrap → triggerHook("agent", {action: "bootstrap", ...})

Hook handlers registered at startup:
  gateway starts → loads bundled hooks + workspace hooks + plugin hooks
  each hook registers for specific event keys (e.g., "command:new")

When event fires:
  triggerInternalHook(event) → looks up handlers by "type" and "type:action"
  → calls each handler with the event
  → handlers can push messages to event.messages[] for user feedback
  → handlers can modify bootstrap context, log data, update memory
```

---

## Cross-Module Dependency Summary

```
                    ┌──────────┐
                    │  agents  │ ◄── Core (everything depends on this)
                    └────┬─────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌──────────┐    ┌──────────┐    ┌──────────┐
   │auto-reply│    │  memory  │    │  hooks   │
   └────┬─────┘    └──────────┘    └──────────┘
        │
   ┌────┴─────┐
   │ sessions │ (utilities)
   └──────────┘
        │
   ┌────┴─────┐
   │   cron   │
   └──────────┘
```

- **agents** is the foundation — 28+ modules depend on it
- **auto-reply** is the second most depended-on (23+ dependents) — it's the message processing hub
- **sessions** provides lightweight utilities used by 11 modules
- **memory** is relatively isolated — only 4 modules import from it directly
- **cron** is consumed only by agents (cron-tool) and gateway
- **hooks** is consumed by 8 modules, primarily for registration and event triggering

---

## v2026.2.15 Changes

### Nested Subagent Orchestration
- **`subagent-depth.ts`** (new): Tracks nesting depth for subagents — enables depth-2 nested spawning (subagents can spawn their own children, max 5 children per agent)
- **`subagent-announce-queue.ts`** (new): Queue system for subagent result announcements — preserves queued items on send failure, uses deterministic idempotency keys to prevent duplicate announces (#17150)
- Related commit: `b8f66c260` — Agents: add nested subagent orchestration controls and reduce subagent token waste (#14447)

### Model Fallback in sessions_spawn (#17197)
- **`tools/sessions-spawn-tool.ts`**: Model fallback support when spawning subagent sessions — if the requested model fails, the spawn can fall back through configured fallback chains
- **`model-fallback.ts`**: Refactored — deduped model fallback candidate logic (`ed03b834d`)

### Multi-Image Support in Image Tool (#17512)
- **`tools/image-tool.ts`** + **`tools/image-tool.helpers.ts`**: Now accepts an array of up to 20 images in a single tool call (previously single image only). Commit `ff4f59ec9`
- Propagates workspace root for image allowlist (#16722), allows workspace and sandbox media paths (#15541)
- Increased maxTokens from 512 to 4096 (#11770)

### Context Window & Model Behavior
- Honor configured `contextWindow` overrides — respects per-model context window settings from config rather than only using catalog defaults
- Force `store=true` for direct OpenAI responses — ensures OpenAI Responses API calls persist via **`pi-embedded-helpers/openai.ts`**
- `messages.suppressToolErrors` config option — new config key to suppress tool error messages from being shown to users

### Reply & Tool Error Handling
- Suppress `NO_REPLY` when message tool already sent — if the agent used the message tool to send output, the runner no longer emits a redundant NO_REPLY token
- Mark required-param tool errors as non-retryable — prevents infinite retry loops when tools are called with missing required parameters

### Refactoring (Massive)
- **Dedupe exec spawn**: `process/spawn-utils.ts` — consolidated spawn logic with EBADF retry and fallback
- **Tool stubs**: Normalized tool stub generation across the codebase
- **Session write lock**: `session-write-lock.ts` — deduped session write lock release logic (`f4782e1e7`)
- **Memory tool config**: `memory-search.ts` — centralized memory search config resolution
- **Subagent announce**: Consolidated announce logic with queue persistence
- **Model fallback logic**: Deduped candidate selection in `model-fallback.ts`
- **Turn validation**: Improved turn validation in `pi-embedded-helpers/turns.ts`

## v2026.2.19 Changes

### Read Tool Auto-Paging
- **`read` tool auto-pages** based on model context window — scales per-call output budget from model's `contextWindow`; larger-context models read more before context guards kick in. See DEVELOPER-REFERENCE.md §9 (gotcha 38)

### Sub-Agent Context Guard
- **Context guard before model calls** — truncates oversized tool outputs and compacts oldest tool-result messages to avoid context-window overflow crashes
- **Compacted-output recovery** — Explicit guidance for recovering from `[compacted: ...]` / `[truncated: ...]` markers by re-reading with smaller chunks

### Exec Preflight Guard
- **Shell env var injection detection** — Preflight guard detects likely shell env var injection patterns (`$DM_JSON`, `$TMPDIR`) in Python/Node scripts before execution. See DEVELOPER-REFERENCE.md §9 (gotcha 40)

### YAML 1.2 Frontmatter
- **Core schema parsing** — Agent prompt frontmatter uses YAML 1.2 core schema; `on`/`off`/`yes`/`no` no longer coerced to booleans. Use `true`/`false` explicitly. See DEVELOPER-REFERENCE.md §9 (gotcha 37)