# OpenClaw Codebase Analysis — Part 2: Agent System
<!-- markdownlint-disable MD024 -->

> Updated: 2026-04-23 | Version: v2026.4.21 | Codebase: OpenClaw release tag `v2026.4.21`

## 1. `src/agents/` — Agent Execution, Tool System, PI Tools

### Purpose
The core engine of OpenClaw. Handles LLM agent execution (the "PI embedded runner"), tool definitions and policy enforcement, model selection/auth/fallback, system prompt construction, sandbox management, session management primitives, skills, subagent orchestration, and workspace management. This is the largest module (~850 tracked files, 75+ tools).

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
| `pi-extensions/compaction-instructions.ts` | `resolveCompactionInstructions()` — precedence-based resolver for compaction summary instructions (SDK event → config → default); `DEFAULT_COMPACTION_INSTRUCTIONS` for language/persona continuity |
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
| `tools/sessions-*.ts` | Session management tools (list, history, send, spawn, yield, announce, helpers) |
| `tools/sessions-yield-tool.ts` | `sessions_yield` tool — end current turn and pass optional message into next turn |
| `tools/subagents-tool.ts` | Subagent management tool |
| `tools/agents-list-tool.ts` | List configured agents |
| `tools/tts-tool.ts` | Text-to-speech tool |
| `tools/web-fetch.ts` | URL content fetching tool |
| `tools/web-search.ts` | Web search tool |
| `tools/web-tools.ts` | Web tool factories |
| `tools/web-shared.ts` | Shared web utilities |
| `tools/web-fetch-utils.ts` | Fetch utilities |
| `tools/agent-step.ts` | Agent step tracking |
| `tools/image-generate-tool.ts` | Image generation tool |
| `tools/pdf-tool.ts` | PDF reading/extraction tool |
| `tools/pdf-tool.helpers.ts` | PDF tool helper utilities |
| `tools/web-search-citation-redirect.ts` | Web search citation redirect resolution |
| `tools/web-search-provider-config.ts` | Web search provider configuration |
| `tools/web-guarded-fetch.ts` | SSRF-guarded web fetch |
| `tools/sessions-resolution.ts` | Session resolution for tools |
| `tools/sessions-access.ts` | Session access control for tools |

> **Note:** Channel-specific actions (Discord, Slack, Telegram, WhatsApp) live in `src/channels/plugins/actions/`, not in `src/agents/tools/`.

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
| `commands-session.ts` | Session commands (/session) |
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

## 4. Released Memory Surfaces — `src/memory-host-sdk/` + `extensions/memory-core/src/memory/`

### Purpose
Semantic memory system. On `v2026.4.9`, the released tree splits memory responsibilities between `src/memory-host-sdk/host/` (host/runtime helpers, embeddings, multimodal file handling) and `extensions/memory-core/src/memory/` (QMD, search managers, sync/indexing, and SQLite-backed memory orchestration). Together they index workspace files (`MEMORY.md`, `memory/*.md`, session transcripts) into SQLite with vector embeddings + full-text search. Supports hybrid search (BM25 + cosine similarity), multiple embedding providers (OpenAI, Gemini, Voyage, local llama), batch embedding, and QMD (Query-Memory-Document) scoped search.

### Key Files

> File basenames below map to the released `v2026.4.9` split: host/runtime helpers live under `src/memory-host-sdk/host/`; search, QMD, and sync managers live under `extensions/memory-core/src/memory/`.

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
| `hybrid.ts` | Hybrid search: BM25 ranking, FTS query building, result merging |
| `temporal-decay.ts` | Time-based relevance decay for search results |
| `mmr.ts` | Maximal Marginal Relevance diversification |
| `prompt-section.ts` | Memory prompt section builder |
| `multimodal.ts` | Multimodal memory content handling |
| `query-expansion.ts` | Query expansion for improved recall |
| `status-format.ts` | Format memory status for display |

#### Embedding Providers
| File | Role |
|------|------|
| `embeddings.ts` | `createEmbeddingProvider()` — provider factory (OpenAI/Gemini/Voyage/local) |
| `embeddings-openai.ts` | OpenAI embedding client |
| `embeddings-gemini.ts` | Gemini embedding client |
| `embeddings-voyage.ts` | Voyage embedding client |
| `embeddings-ollama.ts` | Ollama local embedding client |
| `node-llama.ts` | Local llama.cpp embeddings |
| `embedding-chunk-limits.ts` | Per-provider chunk size limits |
| `embedding-input-limits.ts` | Input length limits |
| `embedding-model-limits.ts` | Model-specific dimension limits |
| `manager-embedding-ops.ts` | Embedding operations on the manager |

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
| `manager-sync-ops.ts` | Sync operations — consolidated sync logic (index files, detect changes, memory/session sync, stale entry removal) |
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
| `src/plugins/hooks.ts` | `createHookRunner()` — execute plugin-provided hook handlers for lifecycle/tool/message events |

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

---

## v2026.2.21 Changes

<!-- v2026.2.21 -->

### Thread-Bound Subagents on Discord

- **Files** (all new): `src/discord/monitor/thread-bindings.manager.ts` (515 lines), `src/discord/monitor/thread-bindings.state.ts` (444 lines), `src/discord/monitor/thread-bindings.lifecycle.ts` (225 lines), `src/discord/monitor/thread-bindings.discord-api.ts` (289 lines)
- **What changed**: A complete Discord thread-binding subsystem was added, enabling per-thread subagent sessions on Discord. Architecture is a four-layer chain:

  | Layer | File | Responsibility |
  |-------|------|----------------|
  | Manager | `thread-bindings.manager.ts` | Public API (`createThreadBindingManager`); per-account `ThreadBindingManager` instances; integrates with `session-binding-service` adapter |
  | State | `thread-bindings.state.ts` | In-process + on-disk global state (`BINDINGS_BY_THREAD_ID`, `BINDINGS_BY_SESSION_KEY`); stored on `globalThis` so Jiti and ESM loader paths share one registry |
  | Lifecycle | `thread-bindings.lifecycle.ts` | Session-level helpers: `autoBindSpawnedDiscordSubagent`, `unbindThreadBindingsBySessionKey`, `setThreadBindingTtlBySessionKey` |
  | Discord API | `thread-bindings.discord-api.ts` | Low-level Discord REST calls: `createThreadForBinding`, `createWebhookForChannel`, `resolveChannelIdForBinding`, `maybeSendBindingMessage`, `isDiscordThreadGoneError` |

- **`ThreadBindingRecord`** fields: `accountId`, `channelId`, `threadId`, `targetKind` (`"subagent" | "acp"`), `targetSessionKey`, `agentId`, `label`, `webhookId`/`webhookToken`, `boundBy`, `boundAt`, `expiresAt`.
- **Persistence**: Bindings are serialized to `<stateDir>/discord/thread-bindings.json` (version 1 JSON format) and loaded on first manager creation via `ensureBindingsLoaded()`.
- **Sweeper**: Each `ThreadBindingManager` runs a periodic interval that probes thread status via Discord REST. Archived threads trigger `unbindThread(reason: "thread-archived")`; deleted/403 threads trigger silent removal (`sendFarewell: false`). TTL-expired bindings send a farewell message.
- **Webhook reuse**: Per `(accountId, channelId)` pair, a reusable webhook is cached in `REUSABLE_WEBHOOKS_BY_ACCOUNT_CHANNEL` to avoid creating a new webhook for every binding on the same parent channel.
- **Unbound echo suppression**: After unbinding, `rememberRecentUnboundWebhookEcho()` records the last webhook ID briefly so the Discord monitor can suppress echoed webhook messages from recently removed bindings.
- **Focus/unfocus integration**: `thread-bindings.lifecycle.ts` exposes `listThreadBindingsBySessionKey` so `/focus` and `/unfocus` commands can resolve the bound thread for a subagent session.
- **Completion routing**: Thread-bound sessions integrate with `BoundDeliveryRouter` (via the `session-binding-service` adapter registered in `createThreadBindingManager`) so subagent completion announcements are routed to the bound thread rather than the original channel.
- **Operational impact**: When `sessions_spawn` is called with `thread=true` on Discord, a new thread is automatically created and bound to the child session key. Follow-up messages from that subagent are routed into the thread. When the session ends (killed, TTL-expired, or session-mode farewell), the thread receives a farewell message and the binding is removed.

### sessions-spawn-hooks

- **File**: `src/agents/subagent-spawn.ts` (550 lines in `v2026.2.26` — located under `src/agents/` per test file `sessions-spawn-hooks.test.ts`)
- **What changed**: Session-spawn hook integration now runs directly in `subagent-spawn.ts`. `ensureThreadBindingForSubagentSpawn()` invokes `subagent_spawning` on the global hook runner before a thread-bound spawn proceeds. On spawn failure, `runSubagentEnded` is emitted so the thread binding is cleaned up even if the gateway `agent` RPC fails.
- **Hook contract**: The `subagent_spawning` hook must return `{ status: "ok", threadBindingReady: true }` for `thread=true` spawns to proceed. Any other return value or error causes the provisional child session to be deleted and an error returned to the tool caller.
- **Operational impact**: Channel plugins (Discord) register `subagent_spawning` hooks to create the thread and return binding confirmation before the agent run is dispatched.

### subagent-registry Refactor

- **File**: `src/agents/subagent-registry.ts` (590-line change)
- **What changed**: Major internal refactor with several behavioral fixes:
  - **Raised dynamic retry cap budget**: `MAX_ANNOUNCE_RETRY_COUNT` governs how many times a failed `runSubagentAnnounceFlow` is retried before giving up. Previously unbounded; now capped to prevent infinite retry loops (`#18264`).
  - **Announce expiry**: `ANNOUNCE_EXPIRY_MS` (5 minutes) force-expires announces that were never delivered, guarding against stale registry entries surviving gateway restarts.
  - **Capped embedded runner retry loop**: `waitForSubagentCompletion` (gateway RPC-based) and the in-process lifecycle listener are now properly coordinated. `resumeSubagentRun` respects the retry cap and expiry before re-attempting cleanup.
  - **Backoff on retry**: `resolveAnnounceRetryDelayMs()` implements exponential backoff (1s → 2s → 4s → max 8s) for retry scheduling.
  - **Completion hook deferral**: `completeSubagentRun` defers `subagent_ended` hook emission until after announce delivery completes when `expectsCompletionMessage=true`, so the farewell message arrives in the thread before the binding is torn down.
  - **Module split**: Queries (`subagent-registry-queries.ts`), state helpers (`subagent-registry-state.ts`), cleanup logic (`subagent-registry-cleanup.ts`), and completion logic (`subagent-registry-completion.ts`) extracted from the monolithic file. Public API (`registerSubagentRun`, `listSubagentRunsForRequester`, `countActiveRunsForSession`, etc.) unchanged.
- **Operational impact**: Eliminates stuck registry entries on gateway restart; prevents announce storm after transient failures; ensures thread-unbind and farewell happen in the correct order relative to delivery.

### QMD Manager Improvements

- **File**: `extensions/memory-core/src/memory/qmd-manager.ts` (326-line change)
- **What changed**:
  - **Han script BM25 normalization**: `normalizeHanBm25Query()` decomposes CJK queries into keyword tokens (capped at `QMD_BM25_HAN_KEYWORD_LIMIT = 12`) before passing to BM25 search, improving recall for Chinese/Japanese/Korean memory searches.
  - **NUL byte filtering**: `NUL_MARKER_RE` detects and filters NUL-byte markers in QMD output that caused parse errors in downstream processing.
  - **Embed queue serialization**: `qmdEmbedQueueTail` serializes embedding calls to prevent concurrent embed operations from exceeding provider rate limits; uses `QMD_EMBED_BACKOFF_BASE_MS` / `QMD_EMBED_BACKOFF_MAX_MS` (1 min to 1 hour) for backoff on provider errors.
  - **Search pending update wait**: `SEARCH_PENDING_UPDATE_WAIT_MS` (500 ms) prevents searches from reading stale index state immediately after a file sync.
- **Operational impact**: More reliable memory search for CJK content; fewer embedding rate-limit errors under high load; eliminates parse failures caused by NUL bytes in QMD output.

### Per-Channel Model Overrides

- **Config key**: `channels.modelByChannel`
- **What changed**: Per-channel model override resolution was documented and wired into the agent pipeline. In the routing/session resolution path, `channels.modelByChannel` is read during `applyModelOverrideToSessionEntry()` (in `src/sessions/model-overrides.ts`) and applied before the embedded runner selects its model. This allows different channels (e.g. Discord vs Telegram) to be pinned to different models without modifying per-agent config.
- **Resolution order** (lowest to highest precedence): agent default model → `channels.modelByChannel[channel]` → per-session override → in-message directive override.
- **Operational impact**: Operators can now set `channels.modelByChannel.discord = "google/gemini-3-flash-preview"` to use a different model for Discord traffic without affecting Telegram or other channels.

---

## v2026.2.22 Changes

<!-- v2026.2.22 -->

### Compaction

- **Count auto-compactions correctly** — Auto-compaction counters now increment only after a non-retry `auto_compaction_end` event, preventing inflated counts from retry attempts.
- **Restore embedded compaction safeguard** — The embedded compaction safeguard was broken in production builds due to bundler resolution order; now correctly restored.
- **Strip stale assistant usage snapshots** — Pre-compaction turns now have stale assistant usage snapshots stripped to prevent the compaction logic from immediately re-triggering on the next turn.
- **Pass model metadata through embedded runtime** — Model metadata is now propagated through the embedded runtime for safeguard summarization when `ctx.model` is unavailable.

### Subagents

- **`alsoAllow` and explicit `allow` entries honored** — `tools.subagents.tools.alsoAllow` and explicit `allow` entries are now correctly honored when resolving built-in subagent deny defaults.
- **Configurable announce call timeouts** — Announce call timeouts are now configurable via `agents.defaults.subagents.announceTimeoutMs` (default 60s restored after regression).

### Auto-reply

- **Default completion acknowledgement scoped to direct/private tool-only runs** — The default `✅ Done.` completion acknowledgement is now emitted only for direct/private tool-only completions. It is suppressed for channel/group sessions and for runs that delivered output via messaging tools.

---

## v2026.2.24 Changes

<!-- v2026.2.24 -->

### Auto-reply

- **Multilingual stop phrases** (#25103) — Standalone stop phrases expanded (`stop openclaw`, `stop action`, `stop run`, `stop agent`, `please stop` + variants); trailing punctuation accepted (e.g. `STOP OPENCLAW!!!`); multilingual stop keywords added (ES/FR/ZH/HI/AR/JP/DE/PT/RU forms) for reliable emergency stops. Contributors: @steipete, @vincentkoc.
- **Reset hooks** (#25459) — Native `/new` and `/reset` flows now emit command/reset hooks even on early-return command paths, with dedupe protection against double emission. Contributor: @chilu18.

### Hooks

- **Slug generator** (#25485) — Session slug resolved from agent's *effective* model (including defaults/fallback chain) instead of raw agent-primary config only. Contributor: @SudeepMalipeddi.

### Agents

- **Tool dispatch** (#25427) — Block-reply flush is awaited before tool execution starts, so buffered block replies preserve message ordering around tool calls. Contributor: @SidQin-cyber.

---

## v2026.2.23 Changes

<!-- v2026.2.23 -->

### Agent Reasoning

- **Model-default thinking guard** — When model-default thinking is active (`thinking=low`), auto-reasoning stays disabled unless explicitly enabled — prevents `Reasoning:` block leakage in channel replies.
- **Reasoning-required provider errors reclassified** — Reasoning-required provider errors are no longer classified as context overflow, fixing compaction-style recovery triggering incorrectly.
- **OpenRouter `/think` level mapping** — `/think` levels are now mapped to `reasoning.effort` in embedded runs; default reasoning is enabled when the model advertises `reasoning: true`.

### Compaction

- **`agentDir` passed into manual `/compact` runs** — `agentDir` is now passed into manual `/compact` runs for correct auth/profile resolution.
- **Cancel safeguard compaction on summary failure** — When summary generation cannot run, safeguard compaction is now cancelled — preserving conversation history instead of truncating to "Summary unavailable".

### Context Overflow

- **Additional provider error shapes detected** — `input length` + `max_tokens` exceed-context error shapes are now detected as context overflow.
- **Chinese context-overflow patterns** — Chinese-language context-overflow patterns are now added to `isContextOverflowError` detection.
- **HTTP 502/503/504 treated as failover-eligible** — HTTP 502, 503, and 504 responses are now treated as failover-eligible transient timeouts.
- **Groq TPM limit errors no longer classified as overflow** — Groq TPM (tokens-per-minute) rate limit errors are no longer misclassified as context overflow.

### Auto-reply

- **Direct-chat metadata selectively hidden** — `message_id`/`message_id_full` and sender metadata are hidden from the normalized chat type only (not sender-id sentinels) — preserves group metadata visibility for correct routing.

---

## v2026.3.1 Changes

<!-- v2026.3.1 -->

### Agent Reasoning & Thinking

- **Claude 4.6 defaults to `adaptive` thinking** — `resolveThinkingDefault()` in `model-selection.ts` now returns `"adaptive"` for Claude Opus 4.6 and Sonnet 4.6 models (matched via `CLAUDE_46_MODEL_RE`), letting the model dynamically decide when and how much to reason. Commit `37d036714`.
- **`thinkingDefault: "adaptive"` config support** (#31227) — The `ThinkLevel` union type in `auto-reply/thinking.ts` gains `"adaptive"` as a recognized level. `normalizeThinkLevel()` maps both `"adaptive"` and `"auto"` to the canonical `"adaptive"` value. `listThinkingLevels()` includes it in available levels. Commit `c9f0d6ac8`.
- **Per-model thinking defaults prioritized** (#30439) — Per-model thinking overrides (`agents.defaults.models.<key>.params.thinking`) are now evaluated before the global `thinkingDefault`, giving operators fine-grained control per model entry.
- **Thinking fallback: retry with `think=off`** — `pickFallbackThinkingLevel()` in `pi-embedded-helpers/thinking.ts` now falls back to `"off"` when a provider error includes `"not supported"` but does not list supported values, allowing requests to succeed on providers that reject thinking levels entirely.

### Model Fallback & Failover

- **Additional network errors as failover-worthy** — `resolveFailoverReasonFromError()` in `failover-error.ts` now classifies `ECONNREFUSED`, `ENETUNREACH`, `EHOSTUNREACH`, `ENETRESET`, and `EAI_AGAIN` error codes as `"timeout"` failover reasons, joining the existing `ETIMEDOUT`/`ESOCKETTIMEDOUT`/`ECONNRESET`/`ECONNABORTED` set.
- **`session_expired` failover reason** — New `"session_expired"` failover reason (status 410 Gone) for CLI-provider sessions that no longer exist, with dedicated `isCliSessionExpiredErrorMessage()` classifier.
- **Avoid false rate-limit from `tpm` substrings** — `hasRateLimitTpmHint()` now uses word-boundary matching (`\btpm\b`) instead of substring includes, preventing error messages containing "tpm" as part of longer words (e.g. model names) from being misclassified as rate-limit errors. The `ERROR_PATTERNS.rate_limited` entry for TPM is also a regex with `\b` boundaries.
- **Copilot token refresh** — GitHub Copilot API tokens are now refreshed 5 minutes before expiry during long-running turns, with automatic retry on auth errors. Commit `2dcd2f909`.
- **Preserve reasoning in provider fallback resolution** — `resolveModel()` in `pi-embedded-runner/model.ts` now resolves the `reasoning` flag from the matching configured model entry (`configuredModel?.reasoning`) instead of hardcoding `false`, and likewise resolves `contextWindow` and `maxTokens` from the matching entry before falling back to the first provider model.

### Runtime Events & Subagents

- **Typed `task_completion` internal events** — New file `internal-events.ts` introduces a typed `AgentInternalEvent` system. `AgentTaskCompletionInternalEvent` carries structured fields (`source`, `childSessionKey`, `announceType`, `taskLabel`, `status`, `result`, `replyInstruction`) replacing the previous ad-hoc `[System Message]` text format for subagent/cron completion handoff. `formatAgentInternalEventsForPrompt()` renders events into a trusted system context block.
- **Subagent announce uses structured events** — `subagent-announce.ts` now builds `AgentInternalEvent[]` arrays and threads them through `deliverSubagentAnnouncement()`, `maybeQueueSubagentAnnounce()`, and `sendSubagentAnnounceDirectly()`. Announce queue items carry `internalEvents` for coalesced batch delivery. The `steerMessage` parameter is separated from `triggerMessage` to support different content for PI steering vs. queue delivery. Commit `4c43fccb3`.
- **Reject unsupported `sessions_spawn` delivery params** — `sessions-spawn-tool.ts` now declares `UNSUPPORTED_SESSIONS_SPAWN_PARAM_KEYS` (`target`, `transport`, `channel`, `to`, `threadId`, `thread_id`, `replyTo`, `reply_to`) and throws a `ToolInputError` if any are present, directing agents to use `message` or `sessions_send` for channel delivery instead.
- **Slack announce threadId fix** (#31105) — `resolveSubagentCompletionOrigin()` no longer blindly copies `conversationId` into `threadId`, which caused invalid `thread_ts` for Slack DM/top-level delivery. Only explicit requester thread hints are preserved. Commit `6a1eedf10`.
- **Sandbox mode for `sessions_spawn`** — New `sandbox` parameter (`"inherit"` | `"require"`) in the sessions spawn tool schema, passed through to the subagent spawn flow.

### Diffs Plugin (New Extension)

- **`extensions/diffs/`** (new) — Read-only diff viewer and PNG renderer plugin. Registered as plugin `"diffs"` with `createDiffsTool()` for agents, HTTP handler for browser-based viewing, and `before_prompt_build` hook injecting `DIFFS_AGENT_GUIDANCE`. Supports rendering diffs from before/after text or unified patches, with configurable defaults via `diffsPluginConfigSchema`. Artifacts stored in a `DiffArtifactStore` with TTL and max pixel cap enforcement.

### OpenAI Responses API

- **Rewritten store patches** — `shouldForceResponsesStore()` in `pi-embedded-runner/extra-params.ts` now respects `model.compat.supportsStore === false` to avoid forcing `store=true` on providers that do not support it (e.g. Azure OpenAI Responses). Also fixes `isDirectOpenAIBaseUrl()` returning `true` for empty base URLs (now returns `false`).
- **Auto-inject `context_management` for compatible models** — OpenAI Responses API integration now enables server-side compaction (`context_management`) via `shouldEnableOpenAIResponsesServerCompaction()` with a compact threshold of 70% of `contextWindow` (min 1,000 tokens, default 80,000).
- **Function call reasoning pair downgrade** — `downgradeOpenAIFunctionCallReasoningPairs()` strips `|fc_*` suffixes from tool call IDs when the matching `reasoning` item with a valid signature is absent from the same assistant turn, preventing OpenAI rejection of replayed `function_call` items.

### Prompt Spoofing Hardening

- **Stop injecting queued runtime events into user-role prompt text** — Completion events are now routed through the structured `AgentInternalEvent` system and rendered via `formatAgentInternalEventsForPrompt()` into trusted system-prompt context, rather than being injected as `[System Message]` blocks in user-facing text.
- **Neutralize spoof markers in untrusted content** — `sanitizeInboundSystemTags()` in `auto-reply/reply/inbound-text.ts` replaces `[System Message]`, `[System]`, `[Assistant]`, `[Internal]` brackets with parenthesized forms and rewrites `System:` line prefixes to `System (untrusted):`. Suspicious patterns added to `security/external-content.ts` detection. Commit `5b8f492a4`.
- **System prompt messaging section updated** — Removed references to `[System Message]` blocks in `system-prompt.ts`; replaced with guidance about "runtime-generated completion events" in assistant voice.

### Auto-reply

- **NO_REPLY: strip token from mixed-content messages** (#30916, #30955) — `normalizeReplyPayload()` in `normalize-reply.ts` now calls `stripSilentToken()` to remove a trailing `NO_REPLY` from mixed-content text (e.g. "result text NO_REPLY") so the token never leaks to end users. If stripping leaves nothing, the message is treated as silent.
- **`stripSilentToken()` utility** (new) — `tokens.ts` exports `stripSilentToken(text, token)` that strips a trailing silent reply token from mixed content, returning the remaining text trimmed.
- **Block reply timeout path: normalize through `Promise.resolve`** — `createBlockReplyPipeline()` in `block-reply-pipeline.ts` wraps `onBlockReply()` return value in `Promise.resolve()` before passing to `withTimeout()`, handling cases where `onBlockReply` returns `undefined` instead of a Promise.

### Compaction

- **Remove post-compaction audit injection message** — `post-compaction-audit.ts` deleted entirely (111-line file). The post-compaction audit that checked for required startup file reads (WORKFLOW_AUTO.md, daily memory files) is removed.
- **Identifier preservation instructions** — `compaction.ts` gains `IDENTIFIER_PRESERVATION_INSTRUCTIONS` and `buildCompactionSummarizationInstructions()` supporting configurable `identifierPolicy` (`"strict"` | `"custom"` | `"off"`) from `AgentCompactionIdentifierPolicy`. Strict mode (default) preserves UUIDs, hashes, IDs, tokens, API keys, hostnames, IPs, ports, URLs, and file names exactly as written during compaction summarization.

### Cron

- **Heartbeat light bootstrap context: opt-in `--light-context`** (#26064) — `runCronIsolatedAgentTurn()` checks `agentPayload.lightContext` and sets `bootstrapContextMode: "lightweight"` for heartbeat/cron runs, reducing the bootstrap context injected into isolated sessions. Commit `0f2dce048`.
- **Delivery mode `none`: disable messaging tool** — When `deliveryPlan.mode === "none"`, the message tool is now disabled (`disableMessageTool: true`) in isolated cron runs, preventing cron jobs with no delivery target from sending messages.
- **One-shot reschedule re-arm** (#28915) — Completed `at` (one-shot) jobs can now be rescheduled to run again. `computeNextRunAtMs()` also accepts `schedule.cron` as an alias for `schedule.expr` for broader compatibility. Croner year-rollback bug workaround added with tomorrow-UTC retry. Commit `08c35eb13`.
- **Cron list: rename Agent to Agent ID, add Model column** (#26259) — `printCronList()` now shows `"Agent ID"` header (was `"Agent"`) and adds a `"Model"` column displaying `payload.model` for `agentTurn` jobs. Missing `agentId` shows `"-"` instead of `"default"`. Commit `cb6f993b4`.
- **Cron run exit code: return 0 only for `ok:true, ran:true`** (#31121) — `cron run` CLI command now exits with code 0 only when the gateway returns `{ ok: true, ran: true }`. Any other outcome (including `ok:true, ran:false`) exits with code 1. Commit `ffe1937b9`.
- **Subagent model for isolated cron** (#11474) — `runCronIsolatedAgentTurn()` now resolves `subagents.model` (agent-level or global) and applies it as the preferred model for isolated cron sessions, subject to the model allowlist.
- **Per-job payload fallbacks** (#26120) — `payload.fallbacks` array in cron job definitions takes priority over agent-level fallback chains.
- **Fresh CLI session ID for new cron sessions** (#29774) — Fresh isolated cron sessions no longer reuse a stored CLI session ID, which was incorrectly activating the resume watchdog profile (timeout ~1/3 of configured).

### Sessions

- **Sessions list transcript paths handling** — `sessions-list-tool.ts` now resolves transcript paths more robustly: handles `storePath` placeholders containing `{agentId}` or `~` prefixes, skips the `"(multiple)"` sentinel, and uses `resolveSessionFilePathOptions()` for proper agent-scoped path resolution.

### Cron Tool

- **Flat-params recovery for `patch` action** — `cron-tool.ts` now accepts `additionalProperties: true` in the tool schema and recovers flat patch parameters (`name`, `schedule`, `payload`, `delivery`, `enabled`, etc.) into a synthetic `patch` object when the LLM passes them at the top level instead of nested.

## v2026.3.7 Delta Notes

- Heartbeat requests-in-flight scheduling: heartbeat runs now gate on in-flight request counts to avoid overloading busy agents.
- Memory QMD collection safety: QMD collection management hardened against race conditions during collection access.
- SQLite contention resilience: agent-side SQLite operations now retry on contention, complementing memory-layer fixes.
- QMD search result decoding with `qmd://` URIs: search results carrying `qmd://` scheme URIs are decoded correctly in agent tool responses.
- Memory hybrid search BM25 ordering: BM25 keyword ranking order is now stable in hybrid search results surfaced by the memory tool.
- Memory flush daily file canonicalization: daily flush files are canonicalized before write, preventing duplicate flush artifacts.
- QMD collection-name conflict recovery: startup detects and recovers from conflicting QMD collection names without requiring manual intervention.
- Tool-result truncation with head+tail strategy (PR context): large tool results are now truncated using a head+tail window rather than hard-cutoff, preserving both the start and end of output.
- Tool-result cleanup timeout hardening: tool-result cleanup jobs apply a bounded timeout to prevent stalled cleanup blocking agent shutdown.
- Failover overload vs rate-limit classification: the agent runner now distinguishes provider overload errors from rate-limit errors when selecting failover targets.

## v2026.3.8 Delta Notes

- Context-engine plugins now bootstrap from the active runtime workspace before compaction and subagent boundaries, and the registry is shared across duplicated bundled chunks through a process-global singleton.
- Compaction can now use `agents.defaults.compaction.model` as an override instead of always reusing the session's primary model.
- Session model switches now clear stale cached `contextTokens` in addition to stale runtime model/fallback fields.
- ACP child runs persist transcript/session metadata and `spawnedBy` lineage more reliably without blocking successful execution on storage failures.
- Rate-limit classification now catches Bedrock-style `Too many tokens per day` quota errors without confusing them with context-window failures.
- Cron text-only announce delivery now routes through the real outbound adapters, and restart catch-up staggering prevents missed-job replays from flooding the gateway after restart.

## v2026.3.11 Delta Notes

### ACP / Session Changes

- **ACP main session alias** (#43285, fixes #25692): `"main"` is now canonicalized before ACP session lookup so restarted ACP main sessions rehydrate into the existing session instead of failing with `Session is not ACP-enabled: main`.
- **ACP `sessions_spawn` resume** (#41847): `sessions_spawn` accepts an optional `resumeSessionId` parameter when `runtime: "acp"`, enabling spawned ACP sessions to resume an existing ACPX/Codex conversation thread.
- **ACP `spawnedBy`/`spawnDepth` lineage on session keys** (#40995): `sessions.patch` now accepts `spawnedBy` and `spawnDepth` lineage fields on ACP session key payloads.
- **ACP stop reason: `error` → `end_turn`** (#41187): gateway chat `state: "error"` is now mapped to ACP `end_turn` instead of `refusal`, matching actual chat-error semantics.
- **ACP `setSessionMode` failure propagation** (#41185): `sessions.patch` failures from the gateway are now propagated back to ACP clients rather than silently swallowed.
- **ACP bridge mode hardening** (#41424): unsupported per-session MCP server configuration is now rejected at session setup; rejected session-mode changes are propagated to the caller.
- **ACP `loadSession` UX** (#41425): stored user and assistant text turns are replayed on `loadSession`; session controls and metadata are exposed in the response.
- **ACP tool streaming enrichment** (#41442): `tool_call` and `tool_call_update` events are enriched with best-effort text content and file-location hints.
- **ACP runtime image attachments** (#41427): normalized inbound image attachments are forwarded into ACP runtime turns.
- **ACP implicit stream for `mode="run"` spawns** (#42404): `mode="run"` ACP spawns implicitly stream to the parent session when the requester is an eligible subagent orchestrator.
- **ACP ACPX runtime RPC coverage** (#41456): gateway RPC now covers ACP lineage patching operations.
- **ACP context engine model auth** (#41090): `runtime.modelAuth` is exposed to plugins and the plugin-sdk provides auth helpers so plugins can resolve API keys through the normal auth pipeline.
- **ACP ACPX pin bump** (#41975): bundled `acpx` updated to `0.1.16`.

### Agent Fixes

- **Strip model control tokens** (#42173): leaked model control tokens (`<|...|>` ASCII and `<｜...｜>` full-width variants, e.g. from GLM-5/DeepSeek) are stripped from user-facing assistant text by `stripModelSpecialTokens()` in `pi-embedded-utils.ts`.
- **Azure OpenAI Responses API store** (#42934): `azure-openai` is included in the Responses API `store` override set alongside `openai` and `azure-openai-responses`.
- **Ignore stale `errorMessage` on successful turns** (#40616): stale `errorMessage` fields on assistant turns that ultimately succeeded are ignored during rendering.
- **Billing cooldown probing** (#41422): existing throttle checks are used to probe single-provider billing cooldowns.
- **HTTP 499 treated as transient** (#41468): HTTP 499 responses are classified as transient errors eligible for retry/failover.
- **Venice 402 billing error** (#43205): Venice 402 responses are recognized as billing errors in the fallback pipeline.
- **Poe 402 billing error** (#42278): Poe 402 responses are recognized as billing errors in the fallback pipeline.
- **Gemini `MALFORMED_RESPONSE` treated as retryable timeout** (#42292): the `MALFORMED_RESPONSE` stop reason from Gemini is treated as a retryable timeout rather than a hard failure.
- **Cooldown default: `unknown` instead of `rate_limit`** (#42911): unclassified cooldown reasons default to `"unknown"` rather than `"rate_limit"`, preventing incorrect rate-limit handling for unrelated failures.
- **Embedded runner compaction retry bound** (#40324): compaction retry wait time is bounded; SIGUSR1 restart drains pending work before exiting.
- **Fallback cooldown probing capped** (#41711): per-provider cooldown probing is capped to one attempt per provider per fallback run, preventing redundant probes.
- **Context pruning: image-only tool results** (#43045): image-only tool results are pruned during soft context trim, reducing token pressure when image content is no longer needed.
- **Session reset: clear stale model metadata** (#41173): stale runtime model and fallback metadata are cleared before session resets to prevent stale model state persisting into new sessions.
- **Subagent authority persistence** (#41711): leaf vs orchestrator control scope is persisted at spawn time, ensuring authority checks remain consistent for the lifetime of a subagent session.

---

## v2026.3.12 Delta Notes

### Agents / PI Embedded Runner

- **`sessions_yield` tool** (#36537): New `sessions_yield` tool added (`src/agents/tools/sessions-yield-tool.ts`). Orchestrators call it to end the current turn immediately, skip any queued tool work, and carry an optional message payload into the next turn. The tool schema accepts an optional `message` string; on success it returns `{ status: "yielded", message }`. The implementation relies on an `onYield` callback injected by the runner context; without it, the tool returns an error. This enables subagent-orchestration patterns where the parent yields after spawning children and resumes when their results arrive.

- **PI embedded runner / OAuth and payload hooks** (#pi-0.58): The pi-ai OAuth and payload hooks in `pi-embedded-runner` were adapted for the pi 0.58.0 API surface. All `@mariozechner/pi-*` packages bumped to `0.58.0`.

- **Compaction: skip double cache-ttl marker on same-attempt completion** (#28548): Post-compaction `cache-ttl` marker writes are now skipped when compaction completed in the same attempt, preventing the marker from being written twice and triggering a spurious second compaction.

- **Compaction: post-compaction memory sync + transcript update emission** (#25558, #25561): After compaction, the memory manager is now synced and `emitSessionTranscriptUpdate` is called so downstream subscribers (memory index, UI) see the updated transcript. Post-compaction session reindexing is gated by `agents.defaults.compaction.postIndexSync` and `agents.defaults.memorySearch.sync.sessions.postCompactionForce`.

- **Compaction safeguard: route warnings through SubsystemLogger** (#9974): Missing-model and API-key warnings emitted by the compaction safeguard are now routed through `createSubsystemLogger("compaction-safeguard")` instead of bare `console.warn`, keeping log output consistent with other subsystems.

- **Failover: z.ai `network_error` stop reason classified as retryable timeout** (#43884): The z.ai provider's `network_error` stop reason is now classified as a retryable timeout in `failover-error.ts`, allowing the fallback chain to engage instead of treating it as a hard failure.

- **Failover: ZenMux quota-refresh 402 classified as rate_limit** (#43917): ZenMux quota-refresh HTTP 402 responses are now classified as `rate_limit` failover reasons. HTTP 422 responses are classified as `format` errors. OpenRouter credit-exhaustion responses are classified as `billing` (#43823).

- **Kimi Coding: fix malformed Anthropic-compatible tool call args** (#42835): Malformed tool call argument strings from the Kimi Coding provider (Anthropic-compatible format) are now repaired in `pi-embedded-runner/run/attempt.ts`. Trailing garbage suffixes after the valid JSON are trimmed and a warning is logged.

- **Kimi Coding: clear invalid Kimi tool arg repair** (#43824): Invalid Kimi tool argument repair state is cleared between turns so stale repair context does not corrupt subsequent calls.

### Subagents

- **Completion announce timeout raised to 90 s; gateway-timeout retries disabled** (#41235): `DEFAULT_SUBAGENT_ANNOUNCE_TIMEOUT_MS` is now `90_000` ms (raised from 60 s). Gateway-timeout failures are no longer retried for external completion announces, preventing retry storms on slow external endpoints.

### Auto-reply / Compaction

- **Status reaction during context compaction pauses** (#35474): A status reaction is now shown to the user while a context compaction pause is in progress, providing feedback that the agent is working rather than idle.

### Delivery / Dedupe

- **Direct-cron delivery cache trimmed correctly** (#44666): The completed direct-cron delivery cache is now correctly trimmed, preventing stale entries from accumulating. Mirrored transcript deduplication remains active even when individual JSONL lines are malformed.

### ACP

- **Preserve terminal assistant text snapshot before `end_turn`** (#17615): The ACP layer now snapshots the terminal assistant text before emitting `end_turn`, ensuring the final message content is preserved for downstream consumers.

### Agents / Thinking Hints

- **`xhigh` thinking level added to `openclaw cron add`, `openclaw cron edit`, `openclaw agent`** (#44819): The `--thinking` option in the cron-add CLI, cron-edit CLI, and agent CLI now lists `xhigh` as a valid level in its help text, matching the full `ThinkLevel` union.

---

## v2026.3.22-v2026.3.23 Delta Notes

### Agents / Runtime Defaults

- **Per-agent reasoning defaults:** the released line now supports per-agent thinking, reasoning, and fast defaults, with disallowed model overrides reverting back to the agent default instead of drifting indefinitely.
- **Default agent timeout raised to `48h`:** long-running ACP and agent sessions now inherit a much longer default timeout unless explicitly configured shorter.

### Agents / Runtime Resolution

- **Active runtime `web_search` provider selection fixed** (`v2026.3.23`) — agent turns now use the configured runtime provider instead of a stale/default selection.
- **Skill SecretRef/env injection fixed** (`v2026.3.23`) — embedded skill config now resolves from the active runtime snapshot, so `skills.entries.<skill>.apiKey` SecretRefs reach the runtime correctly.
- **Malformed assistant replay canonicalization** (`v2026.3.23-2`) — corrupted assistant transcript content is normalized before replay so follow-up turns stop crashing recovery paths.
- **Transient `api_error` classification tightened** (`v2026.3.23`) — generic API errors only trigger retry/failover when they actually look transient.

### Agents / Custom Providers

- **Preserve blank API keys for loopback OpenAI-compatible custom providers** (#45631): Custom providers pointing at loopback addresses (`localhost`, `127.0.0.1`, `[::1]`, etc.) can now have a blank `apiKey` configured without auth resolution failing. The `isLocalBaseUrl()` guard in `model-auth.ts` synthesizes a local auth marker when the key is absent, so users who left the API key empty during onboarding are not blocked.

### Agents / Compaction

- **Post-compaction token sanity check against full-session totals** (#28347): The post-compaction token sanity check in `pi-embedded-runner/compact.ts` now compares against the full-session token count (`fullSessionTokensBefore`) rather than only the summarizable window. When token estimation fails, the sanity check is skipped entirely (`sanityCheckBaseline > 0` guard) instead of crashing compaction.

- **Compaction summary language continuity via `agents.defaults.compaction.customInstructions`** (#10456): A new `compaction-instructions.ts` module (`src/agents/pi-extensions/compaction-instructions.ts`) provides `resolveCompactionInstructions()` with a precedence chain (SDK event → runtime config → built-in default). The built-in `DEFAULT_COMPACTION_INSTRUCTIONS` instructs the model to write the summary in the primary conversation language and preserve code/path/identifier content unchanged. Operators can override via `agents.defaults.compaction.customInstructions` (string, max 800 characters). The function `composeSplitTurnInstructions()` merges SDK turn-prefix instructions with the resolved instructions.

### Agents / Tool Warnings

- **Distinguish gated core tools from plugin-only unknowns in `tools.profile` warnings**: The `tools.profile` warning path now separates gated core tools (present but access-restricted) from genuinely unknown tools (not found in any plugin), emitting distinct warning messages for each category.

### Agents / OpenAI-Compatible Compat Overrides

- **Respect explicit `models[].compat` opt-ins for non-native OpenAI completions endpoints** (#44432): When a model entry in `models[]` has an explicit `compat` object, those compat overrides are now applied even for non-native OpenAI completions endpoints, giving operators fine-grained control over per-model compatibility flags.

### Agents / Azure OpenAI

- **Rephrase built-in startup prompts to avoid HTTP 400 content filter** (#43403): Built-in startup prompts sent to Azure OpenAI are rephrased to avoid triggering Azure's content filter with HTTP 400 responses.

### Agents / Memory Bootstrap

- **Load only one root memory file; prefer `MEMORY.md`, fall back to `memory.md`** (#26054): `resolveMemoryBootstrapEntry()` in `workspace.ts` now returns at most one root memory file. It checks `MEMORY.md` first and only falls back to `memory.md` when `MEMORY.md` is absent. This prevents both files from being injected as bootstrap context when both exist (a common case on case-insensitive filesystems where Docker mounts expose both names).

### Agents / Anthropic Replay

- **Drop replayed thinking blocks for native Anthropic and Bedrock Claude providers** (#44843): When replaying session history for native Anthropic and Amazon Bedrock Claude providers, thinking blocks are now stripped from prior assistant turns. This prevents `400 Bad Request` errors caused by passing raw thinking block content back to providers that do not accept replayed thinking in non-streaming context.

### Cron / Isolated Sessions

- **Nested cron-triggered embedded runner work routed onto nested lane**: `runCronIsolatedAgentTurn()` now calls `resolveNestedAgentLane(params.lane)` when setting up the embedded runner, routing nested cron-triggered work (compaction, inner tool calls, etc.) onto a dedicated nested lane. This prevents deadlocks that could occur when compaction or other inner queued work ran on the same lane as the outer cron session.

### Dependencies

- **pi packages bumped to `0.58.0`**: All `@mariozechner/pi-*` packages (`pi-agent-core`, `pi-ai`, `pi-coding-agent`, `pi-tui`) are at `0.58.0` in `package.json`.

---

## v2026.3.24 Delta Notes

### Skills — Install Recipes

- **`SkillInstallSpec` type in `src/agents/skills/types.ts`** — New type for one-click skill install recipes, encapsulating install source, config, and dependencies.
- **`skills-install.ts` orchestration** — New skill install orchestration module handles end-to-end skill installation from install specs.
- **Skills CLI: "needs setup" label in `skills-cli.format.ts`** — Skills that require post-install configuration are now surfaced with a "needs setup" label in CLI output.

### Cron — Heartbeat Suppression

- **Embedded run heartbeat-prompt suppression for non-cron sessions** — Heartbeat prompts are no longer injected into embedded runs that are not cron-triggered, reducing unnecessary prompt noise in interactive sessions.

### Compaction — Late Auto-Compaction Reconciliation

- **`sessions.json.compactionCount` reconciliation after late embedded auto-compaction** — When an embedded auto-compaction completes after the session has already been updated, the `compactionCount` field in `sessions.json` is reconciled to reflect the actual compaction state.

### Embedded — SecretRef Crash Fix

- **SecretRef crash fix** — Fixed a crash in the embedded runner when resolving SecretRef entries from the active runtime snapshot.

### Agents `/tools` — Available Right Now

- **`/tools` shows currently usable tools with "Available Right Now" section** — The `/tools` command now displays an "Available Right Now" section listing tools that are currently usable in the active session context.

---

## v2026.3.28 Delta Notes

<!-- v2026.3.28 -->

### Memory / Plugins — Flush Plan Moved to `memory-core` Plugin Contract

- **`extensions/memory-core/src/flush-plan.ts`** now owns the pre-compaction memory flush plan: prompt text, system prompt, soft-threshold tokens, append-only guards, target path policy (`memory/YYYY-MM-DD.md`), and the `buildMemoryFlushPlan()` factory.
- The plugin registers its resolver with core via `api.registerMemoryFlushPlan(buildMemoryFlushPlan)` in `extensions/memory-core/index.ts`. Core resolves the active flush plan by calling `getMemoryFlushPlanResolver()` from `src/plugins/memory-state.ts` at runtime rather than executing hardcoded logic.
- `MemoryFlushPlanResolver` (`src/plugins/memory-state.ts`) is the plugin contract type: `(params: { cfg, nowMs }) => MemoryFlushPlan | null`. `registerMemoryFlushPlanResolver()` stores the active resolver on `memoryPluginState.flushPlanResolver`.
- Operators and third-party memory plugins can now replace flush prompt and target-path policy by implementing this contract and calling `registerMemoryFlushPlan` at registration time.

### Memory Flush — Append-Only During Embedded Attempts (PR #53725)

- `flush-plan.ts` enforces append-only semantics for daily flush files: `MEMORY_FLUSH_APPEND_ONLY_HINT` is included in both `DEFAULT_MEMORY_FLUSH_PROMPT` and `DEFAULT_MEMORY_FLUSH_SYSTEM_PROMPT`, instructing the model to append new content to `memory/YYYY-MM-DD.md` if the file already exists, never overwrite.
- This prevents compaction writes from clobbering earlier notes written in the same session day, ensuring all flush entries accumulate rather than the last one winning.
- `ensureMemoryFlushSafetyHints()` enforces these hints on any custom prompt/system-prompt supplied via config, keeping the append-only guarantee even for operator-customized flush turns.

### Agents / Compaction — Post-Compaction AGENTS Refresh on Preflight Compaction (PR #49479)

- **`src/auto-reply/reply/agent-runner-memory.ts`**: after a preflight (stale-usage budget-triggered) compaction completes, `appendPostCompactionRefreshPrompt()` reads critical sections from the workspace `AGENTS.md` via `readPostCompactionContext()` and injects them into the follow-up run's `extraSystemPrompt`.
- This preserves the post-compaction AGENTS refresh that was previously only applied to on-demand compactions, ensuring the agent re-reads session startup, daily memory paths, and red-line instructions even when compaction was triggered by the stale-usage preflight path.
- `readPostCompactionContext()` lives in `src/auto-reply/reply/post-compaction-context.ts` and substitutes `YYYY-MM-DD` placeholders with the real date so agents load the correct daily memory file after compaction.

### Agents / Compaction — Benign `/compact` No-Ops Labeled "skipped" Not "failed" (PR #51072)

- **`src/auto-reply/reply/commands-compact.ts`**: the compaction label displayed to the user is now `"Compaction skipped"` when `isCompactionSkipReason(result.reason)` is true (context already small enough or manual `/compact` was a no-op), instead of `"Compaction failed"`. `"Compaction failed"` is reserved for genuine errors only.
- Condition: `result.ok || isCompactionSkipReason(result.reason)` then `compacted ? "Compacted ..." : "Compaction skipped"`, otherwise `"Compaction failed"`.
- Compaction safeguard cancel reasons are surfaced via `cancelReason` on `CompactionSafeguardRuntime` (`src/agents/pi-hooks/compaction-safeguard-runtime.ts`): `setCancelReason()` stores a human-readable reason; `consumeCancelReason()` pops it for inclusion in user-facing output.

### Agents / Embedded Replies — Mid-Turn 429 and Overload Failures Surfaced (PR #50930)

- **`src/agents/pi-embedded-runner/run.ts`**: when a mid-turn 429/overload triggers an internal `pi-agent-core` retry, `waitForRetry()` can resolve prematurely before tool execution completes, producing an empty payload. The runner now detects this: if `payloads.length === 0` and `stopReason === "toolUse"` or `"error"` (and no deterministic approval prompt, no yield, no suppressed tool error), it surfaces an explicit error message to the user instead of silently discarding the reply.
- For runs where mutating tools executed before the interrupt, the error warns that tool actions may have already taken effect.
- The failing auth profile is marked for cooldown so multi-profile setups rotate away from the exhausted credential on the next turn.
- Successful media-only replies and approval-prompt turns with `stopReason=toolUse` are excluded and continue to pass through correctly.

### Agents / Errors — Provider Quota/Reset Details Surfaced; HTML/Cloudflare Filtered (PR #54512)

- **`src/agents/pi-embedded-helpers/errors.ts`**: `extractProviderRateLimitMessage()` checks whether a rate-limit error body contains actionable details (`RATE_LIMIT_SPECIFIC_HINT_RE`: reset times, plan names, quota info) and, if so, surfaces the provider's own message instead of the generic "API rate limit reached" fallback.
- HTML/Cloudflare error pages are explicitly excluded via `isCloudflareOrHtmlErrorPage()`, keeping these on the generic fallback even when the page text incidentally matches the pattern.
- Very long blobs (>300 chars) and raw JSON are also excluded, preventing unreadable provider payloads from leaking to end users.

### ACP / Channels — Current-Conversation ACP Binds for Discord, BlueBubbles, and iMessage

- **`src/auto-reply/reply/commands-acp/lifecycle.ts`**: `/acp spawn <codex> --bind here` now resolves a `placement: "current"` binding when issued inside an active conversation on supported channels, without creating a child thread.
- Channel binding capability registrations in **`src/channels/plugins/contracts/registry.ts`** confirm `"current"` placement support: `bluebubbles` (`placements: ["current"]`), `discord` (`placements: ["current", "child"]`), and `imessage` (`placements: ["current"]`). The generic current-conversation binding store lives in `src/infra/outbound/current-conversation-bindings.ts`.
- When `capabilities.placements` does not include `"current"` for the active channel, a clear error is returned: `Conversation bindings do not support current placement for ${channel}`.

### Plugins / Runtime — `runHeartbeatOnce` in Plugin Runtime `system` Namespace (PR #40299)

- **`src/plugins/runtime/runtime-system.ts`**: `createRuntimeSystem()` now includes `runHeartbeatOnce` in the `PluginRuntime["system"]` namespace, wrapping `runHeartbeatOnceInternal` from `src/infra/heartbeat-runner.ts`.
- The plugin-safe wrapper accepts `RunHeartbeatOnceOptions` (`{ reason, agentId, sessionKey, heartbeat }`) and explicitly restricts to the plugin-visible subset — `cfg` and internal `deps` injection are blocked at the plugin boundary.
- The `heartbeat` override allows callers to force delivery to a specific target (e.g., `{ target: "last" }`) for single-cycle heartbeat runs triggered from plugin code.
- Type contract in **`src/plugins/runtime/types-core.ts`**: `PluginRuntimeSystemCore.runHeartbeatOnce: (opts?: RunHeartbeatOnceOptions) => Promise<HeartbeatRunResult>`.

### Config / Doctor — Automatic Migrations Older Than Two Months Dropped

- The `openclaw doctor` config migration pipeline no longer applies automatic migrations whose introduction dates are more than two months old. This keeps the migration backlog bounded and prevents ancient migration paths from interfering with current config repair flows.
- Operators whose configs have not been touched in a long time should run `openclaw doctor --fix` promptly after upgrading to v2026.3.28 to ensure any remaining in-window migrations are applied before they age out.

## v2026.3.31 Delta Notes

### Background Tasks / Task Flows

- **Shared task registry now anchors detached work:** the stable line moves ACP, subagent, cron, and detached CLI lifecycle state onto `src/tasks/`, giving agent work a durable SQLite-backed control plane instead of ACP-only bookkeeping.
- **Task flow ownership matters to delivery:** one-task flow linkage and parent-owner routing mean detached work now returns through the intended requester/session path more reliably; agent-side regressions need attached and detached coverage.

### Agents / Runtime

- **Embedded Pi runs gain native Codex web search:** stable `v2026.3.31` includes Codex-native web search support for embedded Pi runs, plus matching runtime/config integration.
- **Idle stream timeout is now configurable:** stalled embedded model streams can abort cleanly instead of waiting for the broader run timeout, which changes how long-running agent failures surface.
- **OpenAI Responses verbosity is forwarded:** `text.verbosity` now travels across Responses transports and status surfaces, so runtime and docs should treat verbosity as a released config/runtime behavior.

## v2026.4.9 Delta Notes

### Agents / Runtime

- **Built-in `music_generate` and `video_generate`:** media generation is now part of the released agent tool surface rather than an external-only plugin path. The current release line expects tool registration, async completion, and outbound media delivery to work together across these new tools.
- **Structured plan updates:** long-running runs can now emit structured plan and execution-item events so compatible UIs can show clearer progress than raw commentary text.
- **Planning-text containment for OpenAI/GPT:** OpenAI/GPT runs now buffer commentary until `final_answer`, use shorter lower-verbosity defaults, and retry one planning-only turn before the run is treated as complete.

### Memory / Dreaming

- **Released memory surfaces split between `src/memory-host-sdk/` and `extensions/memory-core/src/memory/`:** current embeddings, dreaming, and host/runtime helpers live under `src/memory-host-sdk/`, while QMD, search, and sync/index orchestration live under `extensions/memory-core/src/memory/` rather than the older standalone `src/memory/` path referenced by previous docs snapshots.
- **Dreaming surfaces:** the released line adds weighted recall promotion, `/dreaming`, Dreams UI, REM preview tooling, replay-safe promotion, and `dreams.md` as the explicit dreaming trail file.

### ACPX / Runtime

- **Embedded ACPX runtime:** bundled ACPX now embeds its runtime directly instead of shelling out through an extra external ACP CLI hop.
- **Generic `reply_dispatch` hook:** bundled plugins such as ACPX can now intercept reply delivery through the generic reply-dispatch seam instead of hardcoded ACP-specific branches in core auto-reply routing.

### Providers / Model Runtime

- **Forward-compat GPT surface:** the released tree adds forward-compat `openai-codex/gpt-5.4-mini` support, provider-owned GPT-5 prompt contributions, and shared request-override policy across model and media paths.
- **Expanded bundled providers:** Qwen, Fireworks AI, and StepFun join the stable-line provider set, which materially expands the runtime/provider matrix agents can route across.

---

## Changes in v2026.4.15–v2026.4.21

### v2026.4.15

- **Unknown-tool stream guard enabled by default** (#67401): The unknown-tool guard (`resolveUnknownToolGuardThreshold` in `src/agents/pi-embedded-runner/run/attempt.ts`) now runs unconditionally regardless of `tools.loopDetection.enabled`. Previously it was opt-in via that flag. The guard rewrites assistant message content after a configurable number of consecutive unknown-tool attempts, breaking otherwise-infinite tool-not-found loops. The threshold is configurable via `tools.loopDetection.unknownToolThreshold`.

- **Skills snapshot version bumped on skills config writes** (#67401): When a config write touches `skills.*` keys, the skills snapshot version is incremented. This prevents disabled skills from remaining callable through stale snapshots that were captured before the config change took effect.

- **Trimmed default startup and skills prompt budgets** (`src/agents/`): Default token budgets for the startup context and skills prompt sections are reduced. The `memory_get` tool now caps excerpts and appends continuation metadata so agents know more content is available without exceeding the context budget on the first call.

- **`localModelLean` experimental flag** (#66495): Setting `agents.defaults.experimental.localModelLean = true` in config drops heavyweight tools (`browser`, `cron`, `message`) from the tool set assembled in `src/agents/pi-tools.ts`. Intended for weaker local-model setups where these tools are too expensive or unreliable to include by default.

### v2026.4.20

- **Compaction notices: opt-in `notifyOnStart` and `notifyOnCompletion`** (#67830): The `agents.defaults.compaction` config block gains a single opt-in flag. When enabled, the agent sends brief notices to the user when compaction starts and when it completes (e.g. "Compacting context..." / "Compaction complete"). Disabled by default to keep compaction silent and non-intrusive. Verified in `src/config/schema.base.generated.ts` help text.

- **Pi runner retries silent `stopReason=error` turns with no output** (#68310): `src/agents/pi-embedded-runner/run.ts` now retries turns that end with `stopReason="error"` and zero output tokens for non-frontier providers. Previously this resubmission path was gated on the strict-agentic (GPT-5 only) contract, so models like ollama/glm-5.1 that occasionally produce empty error turns fell through to an "incomplete turn detected" dead end. The new retry is model-agnostic and bounded. Covered by `run.empty-error-retry.test.ts`.

- **System prompt strengthened**: The agent system prompt gains completion-bias guidance, live-state check instructions, weak-result recovery strategies, and verification-before-final-answer directives to improve task completion reliability across providers.

- **Sessions maintenance enforced by default** (#69404): The session store now enforces built-in entry cap (`maxEntries`) and age prune (`pruneAfter`) limits by default. Oversized stores are pruned at load time. Previously maintenance was advisory; now enforcement runs unless explicitly set to `"warn"` mode. Verified in `src/config/sessions/store.pruning.integration.test.ts`.

- **`/new` and `/reset` clear auto-sourced model/provider/auth-profile overrides** (#69419): `src/auto-reply/reply/session-reset-model.ts` now clears model, provider, and auth-profile overrides that were applied automatically (e.g. from a channel binding or directive) when the user issues `/new` or `/reset`, while preserving overrides the user set explicitly. This prevents stale auto-resolved selections from carrying over into a new session.

- **Shell: ignore non-interactive placeholder shells** (#69308): `src/agents/shell-utils.ts` defines `NON_INTERACTIVE_SHELLS = new Set(["false", "nologin"])` and falls back to `sh` on PATH when `$SHELL` resolves to `/usr/bin/false` or `/sbin/nologin`. These are service-account shells used by macOS LaunchDaemon users that reject `-c`-style invocations.

### v2026.4.21

- **ACP: skip `sessions_send` A2A ping-pong for parent→own background oneshot child** (`src/acp/session-interaction-mode.ts`): `isRequesterParentOfBackgroundAcpSession()` identifies the case where a parent session sends directly to its own background oneshot ACP child. In this case the A2A ping-pong flow in `sessions_send` is skipped, preventing echo loops. Unrelated sessions with broad visibility that send to the same target still go through the normal A2A path.

- **Codex lifecycle hooks for native app-server turns**: `llm_input`, `llm_output`, and `agent_end` plugin hooks (defined in `src/plugins/hook-types.ts`) are now fired for native Codex app-server turns, giving plugins visibility into Codex-native LLM interactions that previously bypassed the hook pipeline.
