# Utilities & Support Modules — Comprehensive Analysis

**Generated:** 2026-02-15  
**Cluster:** Utilities & Support Modules  
**Total files analyzed:** ~325 .ts files across 14 modules

---

## Table of Contents

1. [src/auto-reply (223 files)](#srcauto-reply)
2. [src/utils (22 files)](#srcutils)
3. [src/shared (16 files)](#srcshared)
4. [src/types (9 files)](#srctypes)
5. [src/logging (20 files)](#srclogging)
6. [src/process (11 files)](#srcprocess)
7. [src/pairing (5 files)](#srcpairing)
8. [src/node-host (7 files)](#srcnode-host)
9. [src/macos (4 files)](#srcmacos)
10. [src/compat (1 file)](#srccompat)
11. [src/scripts (1 file)](#srcscripts)
12. [src/docs (1 file)](#srcdocs)
13. [src/test-helpers (2 files)](#srctest-helpers)
14. [src/test-utils (4 files)](#srctest-utils)

---

## src/auto-reply

### Module Overview

The **auto-reply** module is the central brain of OpenClaw — the complete pipeline from receiving an inbound message to dispatching an LLM agent reply. It handles:

- **Inbound message processing** (envelope formatting, media notes, debouncing)
- **Session management** (init, reset, persistence, freshness evaluation)
- **Directive/command parsing** (`/model`, `/think`, `/verbose`, `/elevated`, `/queue`, etc.)
- **Model routing & selection** (aliases, allowlists, fuzzy matching, fallback chains)
- **Agent execution** (embedded pi-agent, CLI agent, model fallback with retry)
- **Reply delivery** (block streaming, chunking, coalescing, threading, TTS)
- **Queue management** (followup runs, steer mode, deduplication, drop policies)
- **Heartbeat system** (scheduled proactive agent runs)

**Architecture pattern:** Pipeline/chain — `dispatch` → `getReplyFromConfig` → `resolveReplyDirectives` → `handleInlineActions` → `runPreparedReply` → `runReplyAgent` → `runAgentTurnWithFallback` → delivery.

**Entry points:**
- `dispatch.ts` — `dispatchInboundMessage()`, `dispatchInboundMessageWithBufferedDispatcher()`
- `reply.ts` — re-exports `getReplyFromConfig()` and directive extractors
- `reply/get-reply.ts` — `getReplyFromConfig()` — the main orchestrator

### File Inventory (129 source files, 86 test files)

#### Top-level files

| File | Description |
|------|-------------|
| `types.ts` | Core types: `ReplyPayload`, `GetReplyOptions`, `BlockReplyContext`, `ModelSelectedContext` |
| `dispatch.ts` | Top-level dispatch functions: `dispatchInboundMessage()`, `withReplyDispatcher()` |
| `reply.ts` | Re-export barrel for `getReplyFromConfig` and directive extractors |
| `model.ts` | `extractModelDirective()` — parses `/model provider/name@profile` from message text |
| `envelope.ts` | `formatAgentEnvelope()`, `formatInboundEnvelope()` — wraps messages in `[Channel From Timestamp]` headers |
| `templating.ts` | `MsgContext`, `TemplateContext`, `FinalizedMsgContext` — the massive context object passed through the pipeline |
| `chunk.ts` | `resolveTextChunkLimit()`, `chunkText()` — splits outbound text for platform limits |
| `thinking.ts` | `ThinkLevel`, `VerboseLevel`, `ElevatedLevel`, `ReasoningLevel` enums + normalization; `XHIGH_MODEL_REFS` |
| `tokens.ts` | `HEARTBEAT_TOKEN`, `SILENT_REPLY_TOKEN`, `isSilentReplyText()` |
| `status.ts` | `buildStatusReply()` — generates `/status` output with model, session, usage info |
| `heartbeat.ts` | `HEARTBEAT_PROMPT`, `resolveHeartbeatPrompt()`, `stripHeartbeatToken()`, `isHeartbeatContentEffectivelyEmpty()` |
| `heartbeat-reply-payload.ts` | `resolveHeartbeatReplyPayload()` — extracts last meaningful payload from heartbeat reply |
| `send-policy.ts` | `normalizeSendPolicyOverride()`, `parseSendPolicyCommand()` — `/send allow|deny` |
| `tool-meta.ts` | `formatToolAggregate()`, `shortenPath()` — formats tool execution summaries |
| `skill-commands.ts` | `listSkillCommandsForWorkspace()`, `listSkillCommandsForAgents()` — discovers skill-based slash commands |
| `commands-registry.ts` | Command registration system: `listChatCommands()`, `normalizeCommandBody()`, `detectTextCommand()` |
| `commands-registry.types.ts` | Types: `ChatCommandDefinition`, `CommandScope`, `CommandCategory`, `CommandArgDefinition` |
| `commands-registry.data.ts` | All built-in command definitions via `defineChatCommand()` |
| `commands-args.ts` | `COMMAND_ARG_FORMATTERS` — per-command argument formatting |
| `command-auth.ts` | `resolveCommandAuthorization()` — checks sender against allowFrom lists |
| `command-detection.ts` | `hasControlCommand()`, `isControlCommandMessage()` — detects slash commands in text |
| `group-activation.ts` | `GroupActivationMode`, `parseActivationCommand()` — `/activation mention|always` |
| `inbound-debounce.ts` | `resolveInboundDebounceMs()`, `createInboundDebouncer()` — coalesces rapid inbound messages |
| `media-note.ts` | `buildInboundMediaNote()` — generates `[media attached: path (type)]` annotations |

#### reply/ subdirectory — Core Reply Pipeline

| File | Description |
|------|-------------|
| `get-reply.ts` | **Main orchestrator** `getReplyFromConfig()` — session init → directives → inline actions → agent run |
| `get-reply-directives.ts` | `resolveReplyDirectives()` — parses all inline directives from message |
| `get-reply-directives-apply.ts` | Applies parsed directives to session state |
| `get-reply-directives-utils.ts` | Utility functions for directive resolution |
| `get-reply-inline-actions.ts` | `handleInlineActions()` — processes commands, status requests, skill triggers |
| `get-reply-run.ts` | `runPreparedReply()` — assembles agent run config, invokes `runReplyAgent()` |
| `agent-runner.ts` | `runReplyAgent()` — full agent run lifecycle including queuing, memory flush, compaction, usage tracking |
| `agent-runner-execution.ts` | `runAgentTurnWithFallback()` — executes the LLM call with model fallback, transient error retry, session auto-reset |
| `agent-runner-helpers.ts` | Helper functions: `finalizeWithFollowup()`, `signalTypingIfNeeded()`, `isAudioPayload()` |
| `agent-runner-memory.ts` | `runMemoryFlushIfNeeded()` — pre-compaction memory dump to disk |
| `agent-runner-payloads.ts` | `buildReplyPayloads()` — assembles final reply payloads from agent output |
| `agent-runner-utils.ts` | `appendUsageLine()`, `formatResponseUsageLine()`, `buildThreadingToolContext()` |
| `session.ts` | `initSessionState()` — session creation/resumption/reset with full state management |
| `session-reset-model.ts` | `applyResetModelOverride()` — handles `/new model` syntax |
| `session-reset-prompt.ts` | Session reset prompt generation |
| `session-run-accounting.ts` | `persistRunSessionUsage()`, `incrementRunCompactionCount()` — tracks token usage per session |
| `session-updates.ts` | Session entry update helpers |
| `session-usage.ts` | Session usage aggregation |
| `history.ts` | `buildHistoryContext()`, `appendHistoryEntry()`, `evictOldHistoryKeys()` — group message history |
| `model-selection.ts` | `ModelSelectionState` — resolves model from directives/config with fuzzy matching, allowlist checking |
| `directive-handling.ts` | Barrel: exports directive parsing/persistence/fast-lane |
| `directive-handling.impl.ts` | Core directive application implementation |
| `directive-handling.parse.ts` | `parseInlineDirectives()`, `isDirectiveOnly()` — extracts `/think`, `/verbose`, `/model` etc. |
| `directive-handling.persist.ts` | `persistInlineDirectives()`, `resolveDefaultModel()` — saves directive state to session store |
| `directive-handling.shared.ts` | `formatDirectiveAck()` — builds acknowledgment messages for directives |
| `directive-handling.fast-lane.ts` | `applyInlineDirectivesFastLane()` — quick directive-only handling (no agent run) |
| `directive-handling.auth.ts` | Directive authorization checks |
| `directive-handling.model.ts` | Model-specific directive handling |
| `directive-handling.model-picker.ts` | Interactive model picker UI |
| `directive-handling.params.ts` | Directive parameter types |
| `directive-handling.queue-validation.ts` | Queue mode validation for directives |
| `directive-parsing.ts` | `skipDirectiveArgPrefix()`, `takeDirectiveToken()` — low-level tokenizer for directives |
| `directives.ts` | `extractElevatedDirective()`, `extractThinkDirective()`, `extractReasoningDirective()`, `extractVerboseDirective()` |
| `commands.ts` | Barrel: `buildCommandContext()`, `handleCommands()`, `buildStatusReply()` |
| `commands-types.ts` | `CommandContext`, `CommandHandlerResult`, `HandleCommandsParams` |
| `commands-core.ts` | `handleCommands()` — dispatches slash commands to handlers |
| `commands-context.ts` | `buildCommandContext()` — assembles command execution context |
| `commands-context-report.ts` | Context reporting for debugging |
| `commands-info.ts` | `/info` command handler |
| `commands-status.ts` | `/status` command handler |
| `commands-models.ts` | `/model`, `/model-list` command handlers |
| `commands-session.ts` | `/new`, `/reset` session command handlers |
| `commands-subagents.ts` | `/subagents` command handler |
| `commands-config.ts` | `/config` command handler |
| `commands-bash.ts` | Bash-related command handling |
| `commands-compact.ts` | `/compact` command handler |
| `commands-allowlist.ts` | Model allowlist management commands |
| `commands-approve.ts` | Approval-related commands |
| `commands-ptt.ts` | Push-to-talk commands |
| `commands-tts.ts` | `/tts` command handler |
| `commands-plugin.ts` | Plugin command handling |
| `commands-policy.test.ts` | Command policy tests |
| `commands-parsing.test.ts` | Command parsing tests |
| `config-commands.ts` | Configuration commands |
| `config-value.ts` | Config value resolution for commands |
| `debug-commands.ts` | Debug/diagnostic commands |
| `provider-dispatcher.ts` | `dispatchReplyWithBufferedBlockDispatcher()` — convenience wrappers |
| `dispatch-from-config.ts` | `dispatchReplyFromConfig()` — config-aware dispatch with TTS, abort, dedup |
| `dispatcher-registry.ts` | Active dispatcher tracking |
| `reply-dispatcher.ts` | `createReplyDispatcher()`, `createReplyDispatcherWithTyping()` — manages reply delivery with human delay, normalization, response prefix |
| `reply-delivery.ts` | `createBlockReplyDeliveryHandler()`, `normalizeReplyPayloadDirectives()` — processes inline directives in reply text |
| `reply-directives.ts` | `parseReplyDirectives()` — extracts `[[reply_to_current]]`, `MEDIA:`, `NO_REPLY` from agent output |
| `reply-threading.ts` | `resolveReplyToMode()`, `createReplyToModeFilter()` — controls reply threading per channel |
| `reply-payloads.ts` | Payload normalization and filtering |
| `reply-tags.ts` | `extractReplyToTag()` — parses `[[reply_to:id]]` tags |
| `reply-reference.ts` | Reply reference resolution |
| `reply-inline.ts` | Inline reply handling |
| `reply-elevated.ts` | Elevated mode reply handling |
| `normalize-reply.ts` | `normalizeReplyPayload()` — strips heartbeat tokens, silent replies, sanitizes text |
| `response-prefix-template.ts` | Template interpolation for response prefixes |
| `route-reply.ts` | `routeReply()`, `isRoutableChannel()` — cross-channel reply routing |
| `block-reply-pipeline.ts` | `createBlockReplyPipeline()` — manages block streaming delivery with coalescing and audio buffering |
| `block-reply-coalescer.ts` | `createBlockReplyCoalescer()` — merges rapid block replies |
| `block-streaming.ts` | `resolveBlockStreamingCoalescing()` — config resolution for block stream settings |
| `streaming-directives.ts` | Directive handling during streaming |
| `typing.ts` | `createTypingController()` — manages typing indicators with interval-based refresh |
| `typing-mode.ts` | `createTypingSignaler()` — typing signal dispatch based on mode (always/reasoning/message) |
| `queue.ts` | Barrel: queue directive extraction, followup run management |
| `queue/types.ts` | `QueueMode`, `QueueSettings`, `FollowupRun`, `QueueDropPolicy` |
| `queue/state.ts` | In-memory queue state |
| `queue/settings.ts` | `resolveQueueSettings()` |
| `queue/enqueue.ts` | `enqueueFollowupRun()`, `getFollowupQueueDepth()` |
| `queue/drain.ts` | `scheduleFollowupDrain()` — processes queued followup runs |
| `queue/directive.ts` | `extractQueueDirective()` |
| `queue/normalize.ts` | Queue input normalization |
| `queue/cleanup.ts` | `clearSessionQueues()` |
| `followup-runner.ts` | `createFollowupRunner()` — executes queued followup turns |
| `memory-flush.ts` | `resolveMemoryFlushSettings()`, `DEFAULT_MEMORY_FLUSH_PROMPT` — pre-compaction memory persistence |
| `abort.ts` | `tryFastAbortFromMessage()`, `formatAbortReplyText()` — `/stop` handling |
| `body.ts` | Body text processing |
| `audio-tags.ts` | Audio tag processing |
| `bash-command.ts` | Bash command handling |
| `exec.ts` | `/exec` directive handling |
| `exec/directive.ts` | Exec directive types |
| `elevated-unavailable.ts` | Elevated mode unavailability handling |
| `groups.ts` | Group-specific logic |
| `inbound-context.ts` | `finalizeInboundContext()` — normalizes MsgContext |
| `inbound-dedupe.ts` | `shouldSkipDuplicateInbound()` — deduplicates rapid message deliveries |
| `inbound-meta.ts` | Inbound message metadata |
| `inbound-text.ts` | `normalizeInboundTextNewlines()` |
| `line-directives.ts` | Per-line directive extraction |
| `mentions.ts` | `stripMentions()`, `stripStructuralPrefixes()` |
| `stage-sandbox-media.ts` | `stageSandboxMedia()` — copies inbound media into agent workspace |
| `subagents-utils.ts` | Subagent utility functions |
| `untrusted-context.ts` | Untrusted context handling |

### Key Types & Interfaces

| Type | File | Description |
|------|------|-------------|
| `ReplyPayload` | `types.ts` | Reply output: text, mediaUrl, mediaUrls, replyToId, audioAsVoice, isError, channelData |
| `GetReplyOptions` | `types.ts` | Options for reply generation: abortSignal, images, callbacks (onPartialReply, onBlockReply, onToolResult, onModelSelected, etc.) |
| `BlockReplyContext` | `types.ts` | Block reply delivery context with abort signal and timeout |
| `ModelSelectedContext` | `types.ts` | Provider, model, thinkLevel after model selection |
| `MsgContext` | `templating.ts` | Massive inbound message context (~100 fields): Body, From, To, SessionKey, MediaPaths, ChatType, GroupId, etc. |
| `TemplateContext` | `templating.ts` | Extended MsgContext with session info (SessionId, IsNewSession, BodyStripped) |
| `FinalizedMsgContext` | `templating.ts` | MsgContext after normalization |
| `OriginatingChannelType` | `templating.ts` | Union of ChannelId and InternalMessageChannel |
| `ThinkLevel` | `thinking.ts` | `"off" \| "minimal" \| "low" \| "medium" \| "high" \| "xhigh"` |
| `VerboseLevel` | `thinking.ts` | `"off" \| "on" \| "full"` |
| `ElevatedLevel` | `thinking.ts` | `"off" \| "on" \| "ask" \| "full"` |
| `ReasoningLevel` | `thinking.ts` | `"off" \| "on" \| "stream"` |
| `ChatCommandDefinition` | `commands-registry.types.ts` | Command spec: key, description, textAliases, args, scope, category |
| `CommandScope` | `commands-registry.types.ts` | `"text" \| "native" \| "both"` |
| `CommandCategory` | `commands-registry.types.ts` | `"session" \| "options" \| "status" \| "management" \| "media" \| "tools" \| "docks"` |
| `SessionInitResult` | `reply/session.ts` | Full session initialization result with sessionCtx, entry, store, key, flags |
| `SessionEntry` | (from config/sessions) | Persistent session state: sessionId, modelOverride, thinkingLevel, usage, etc. |
| `QueueMode` | `reply/queue/types.ts` | Queue behavior mode |
| `QueueSettings` | `reply/queue/types.ts` | Queue configuration: mode, debounceMs, cap, dropPolicy |
| `FollowupRun` | `reply/queue/types.ts` | Queued agent run with config and prompt |
| `CommandAuthorization` | `command-auth.ts` | Sender authorization result: providerId, ownerList, senderIsOwner |
| `GroupActivationMode` | `group-activation.ts` | `"mention" \| "always"` |
| `SendPolicyOverride` | `send-policy.ts` | `"allow" \| "deny"` |
| `ChunkMode` | `chunk.ts` | `"length" \| "newline"` |
| `EnvelopeFormatOptions` | `envelope.ts` | Timestamp/timezone/elapsed formatting options |
| `MemoryFlushSettings` | `reply/memory-flush.ts` | Memory flush config: enabled, threshold, prompt |
| `BlockReplyPipeline` | `reply/block-reply-pipeline.ts` | Pipeline interface: enqueue, flush, stop, hasBuffered, didStream |
| `AgentRunLoopResult` | `reply/agent-runner-execution.ts` | `{ kind: "success" \| "final" }` discriminated union |

### Data Flow — Auto-Reply Pipeline

```
Inbound Message (channel plugin)
  │
  ▼
dispatchInboundMessage() [dispatch.ts]
  │ Creates ReplyDispatcher with typing/human-delay
  ▼
dispatchReplyFromConfig() [reply/dispatch-from-config.ts]
  │ Handles TTS auto-mode, fast abort, dedup, routable channel
  ▼
getReplyFromConfig() [reply/get-reply.ts]  ◄── MAIN ORCHESTRATOR
  │
  ├─ 1. Resolve agent ID, workspace, model defaults
  ├─ 2. Apply media/link understanding
  ├─ 3. Resolve command authorization
  ├─ 4. initSessionState() → session create/resume/reset
  │     └─ Checks reset triggers (/new, /reset)
  │     └─ Evaluates session freshness
  │     └─ Forks from parent session if thread
  ├─ 5. applyResetModelOverride() → /new model syntax
  ├─ 6. resolveReplyDirectives() → parse /model, /think, /verbose, /elevated, /queue
  │     └─ Returns "reply" (directive-only ack) or "result" (continue)
  ├─ 7. handleInlineActions() → slash commands, status, skill triggers
  │     └─ Returns "reply" (command result) or continues
  ├─ 8. stageSandboxMedia() → copy inbound media to workspace
  ▼
runPreparedReply() [reply/get-reply-run.ts]
  │ Assembles FollowupRun config
  ▼
runReplyAgent() [reply/agent-runner.ts]
  │
  ├─ Queue check: if active run, enqueue followup
  ├─ Memory flush if near compaction threshold
  ├─ Create followup runner, block reply pipeline
  ▼
runAgentTurnWithFallback() [reply/agent-runner-execution.ts]
  │
  ├─ runWithModelFallback() → tries primary model, then fallbacks
  │   ├─ CLI agent path: runCliAgent()
  │   └─ Embedded path: runEmbeddedPiAgent() ← @mariozechner/pi-coding-agent
  │       ├─ Streaming: onPartialReply, onBlockReply, onToolResult, onReasoningStream
  │       ├─ Tool execution with typing signals
  │       └─ Auto-compaction tracking
  ├─ Error recovery:
  │   ├─ Context overflow → session reset + user warning
  │   ├─ Compaction failure → session reset + retry
  │   ├─ Role ordering conflict → session reset
  │   ├─ Session corruption (Gemini) → session reset
  │   └─ Transient HTTP error → retry after 2.5s delay
  ▼
Post-processing [agent-runner.ts continued]
  │
  ├─ Persist usage (tokens, cost, model) to session store
  ├─ Build reply payloads from agent output
  ├─ Filter duplicates from messaging tool sends
  ├─ Apply reply threading mode
  ├─ Append usage line if responseUsage enabled
  ├─ Append compaction/session verbose notices
  └─ finalizeWithFollowup() → drain queued followup runs
```

### Model Routing

1. **Default model**: from `agents.defaults.model` / `agents.defaults.provider` in config
2. **Per-agent override**: from `agents.<agentId>.model` in config
3. **Session override**: persisted `/model` directive in session entry
4. **Inline directive**: `/model provider/name` in current message
5. **Heartbeat override**: `agents.defaults.heartbeat.model` or per-agent
6. **Alias resolution**: config-defined aliases (`/gpt` → `openai/gpt-4.1`)
7. **Fuzzy matching**: Levenshtein distance + variant token matching
8. **Allowlist**: `agents.defaults.allowedModels` restricts available models
9. **Fallback chain**: `agents.defaults.modelFallbacks` or per-agent overrides

### Internal Dependencies

- `src/config` — `OpenClawConfig`, `loadConfig()`, `sessions.*`
- `src/agents` — agent scope, model selection, pi-embedded, workspace, skills, context, usage
- `src/channels` — channel plugins, dock system, sender labels, chat types
- `src/infra` — diagnostic events, agent events, file lock, gateway lock
- `src/routing` — session key normalization
- `src/tts` — TTS auto-mode, config resolution
- `src/media-understanding` — media understanding pipeline
- `src/link-understanding` — link understanding pipeline
- `src/plugins` — hook runner, plugin registry
- `src/markdown` — fence parsing for chunking
- `src/utils` — message-channel, usage-format, delivery-context, boolean
- `src/logging` — diagnostic logging
- `src/gateway` — session-utils.fs for transcript archiving

### External Dependencies

- `@mariozechner/pi-ai` — `ImageContent` type
- `@mariozechner/pi-coding-agent` — `SessionManager`, `CURRENT_SESSION_VERSION`

### Configuration Keys

- `agents.defaults.model`, `agents.defaults.provider` — default model
- `agents.defaults.heartbeat.*` — heartbeat config (model, prompt, every, ackMaxChars)
- `agents.defaults.compaction.*` — compaction settings (reserveTokensFloor, memoryFlush.*)
- `agents.defaults.typingIntervalSeconds` — typing indicator refresh
- `agents.defaults.envelopeTimezone`, `envelopeTimestamp`, `envelopeElapsed` — envelope formatting
- `agents.defaults.allowedModels` — model allowlist
- `agents.defaults.modelFallbacks` — fallback chain
- `agents.defaults.skipBootstrap` — skip workspace bootstrapping
- `session.resetTriggers` — triggers like `/new`, `/reset`
- `session.scope` — `per-sender` or other scope modes
- `session.store` — session store path
- `session.mainKey` — main session key mode
- `messages.inbound.debounceMs`, `messages.inbound.byChannel` — debouncing
- `<channel>.textChunkLimit`, `<channel>.chunkMode` — per-channel chunking
- `<channel>.blockStreamingCoalesce` — block streaming settings

### Test Coverage (86 test files)

Major test categories:
- **E2E directive tests** (~30 files): `reply.directive.directive-behavior.*.e2e.test.ts` — test each directive (model, think, verbose, elevated, reasoning, queue) end-to-end
- **E2E trigger tests** (~20 files): `reply.triggers.trigger-handling.*.e2e.test.ts` — test command handling, status, media staging, group activation
- **Unit tests**: `dispatch.test.ts`, `model.test.ts`, `status.test.ts`, `thinking.test.ts`, `inbound.test.ts`, `media-note.test.ts`, `chunk.test.ts`, `skill-commands.test.ts`, `command-control.test.ts`, `tool-meta.test.ts`
- **Reply sub-module tests**: `commands-policy.test.ts`, `commands-info.test.ts`, `commands-parsing.test.ts`, `line-directives.test.ts`, `agent-runner-utils.test.ts`, `followup-runner.test.ts`, `reply-delivery tests`
- **Test harnesses**: `reply.test-harness.ts`, `reply.directive.directive-behavior.e2e-harness.ts`, `reply.directive.directive-behavior.e2e-mocks.ts`, `reply/commands.test-harness.ts`, `reply/agent-runner.memory-flush.test-harness.ts`

---

## src/utils

### Module Overview

General-purpose utility functions used across the codebase. Small, focused helpers with no heavy dependencies.

### File Inventory

| File | Description |
|------|-------------|
| `account-id.ts` | `normalizeAccountId()` — trims/validates account ID strings |
| `boolean.ts` | `parseBooleanValue()` — parses "true"/"yes"/"1"/"on" etc. |
| `delivery-context.ts` | `DeliveryContext` type, `normalizeDeliveryContext()`, `mergeDeliveryContext()`, `deliveryContextKey()` — routing context for message delivery |
| `directive-tags.ts` | `parseInlineDirectives()` — extracts `[[audio_as_voice]]`, `[[reply_to_current]]`, `[[reply_to:id]]` from text |
| `fetch-timeout.ts` | `fetchWithTimeout()`, `bindAbortRelay()` — fetch wrapper with AbortController timeout |
| `message-channel.ts` | `resolveMessageChannel()`, `isMarkdownCapableMessageChannel()`, `INTERNAL_MESSAGE_CHANNEL`, gateway client utilities |
| `normalize-secret-input.ts` | `normalizeSecretInput()` — strips line breaks from copy-pasted API keys |
| `provider-utils.ts` | `isReasoningTagProvider()` — identifies providers needing `<think>`/`<final>` tags |
| `queue-helpers.ts` | `QueueState`, `applyQueueDropPolicy()`, `buildQueueSummaryLine()`, `elideQueueText()` |
| `reaction-level.ts` | `ReactionLevel`, `resolveReactionLevel()` — resolves reaction behavior (off/ack/minimal/extensive) |
| `safe-json.ts` | `safeJsonStringify()` — JSON.stringify with bigint/Error/Uint8Array support |
| `shell-argv.ts` | `splitShellArgs()` — shell argument parsing with quote handling |
| `transcript-tools.ts` | `extractToolCallNames()`, `hasToolCall()`, `countToolResults()` — transcript analysis |
| `usage-format.ts` | `formatTokenCount()`, `formatUsd()`, `estimateUsageCost()`, `resolveModelCostConfig()` |

### Key Types

| Type | File | Description |
|------|------|-------------|
| `DeliveryContext` | `delivery-context.ts` | `{ channel, to, accountId, threadId }` |
| `InlineDirectiveParseResult` | `directive-tags.ts` | Parsed `[[tag]]` directives from text |
| `ReactionLevel` | `reaction-level.ts` | `"off" \| "ack" \| "minimal" \| "extensive"` |
| `ResolvedReactionLevel` | `reaction-level.ts` | Resolved reaction config with ackEnabled, agentReactionsEnabled |
| `ModelCostConfig` | `usage-format.ts` | `{ input, output, cacheRead, cacheWrite }` per-million-tokens |
| `QueueState<T>` | `queue-helpers.ts` | Generic queue with drop policy and summary tracking |
| `BooleanParseOptions` | `boolean.ts` | Custom truthy/falsy string lists |

### Internal Dependencies

- `src/channels` — `ChannelId`, channel registry, gateway protocol types
- `src/config` — `OpenClawConfig`
- `src/agents` — `NormalizedUsage`
- `src/plugins` — plugin registry
- `src/routing` — `normalizeAccountId`

### External Dependencies

None (pure Node.js utilities).

### Test Coverage

- `boolean.test.ts` — truthy/falsy parsing
- `delivery-context.test.ts` — context normalization, merging, key generation
- `message-channel.test.ts` — channel resolution with plugin registry
- `provider-utils.test.ts` — reasoning tag provider detection
- `reaction-level.test.ts` — reaction level resolution
- `shell-argv.test.ts` — shell argument splitting
- `transcript-tools.test.ts` — tool call extraction from transcripts
- `usage-format.test.ts` — token/USD formatting, cost estimation

---

## src/shared

### Module Overview

Shared utilities used across both gateway and CLI/node-host contexts. Designed to be import-safe without heavy side effects.

### File Inventory

| File | Description |
|------|-------------|
| `chat-envelope.ts` | `stripEnvelope()` — strips `[Channel From Timestamp]` envelope headers from text |
| `config-eval.ts` | `isTruthy()`, `resolveConfigPath()`, `isConfigPathTruthyWithDefaults()` — config value evaluation |
| `device-auth.ts` | `DeviceAuthEntry`, `DeviceAuthStore` types, `normalizeDeviceAuthRole()`, `normalizeDeviceAuthScopes()` |
| `entry-metadata.ts` | `resolveEmojiAndHomepage()` — resolves emoji/homepage from metadata/frontmatter |
| `frontmatter.ts` | `normalizeStringList()`, `getFrontmatterString()`, `parseFrontmatterBool()`, `resolveOpenClawManifestBlock()` — YAML/JSON5 frontmatter parsing |
| `model-param-b.ts` | `inferParamBFromIdOrName()` — extracts parameter count (e.g. "70b") from model names |
| `node-match.ts` | `resolveNodeIdFromCandidates()`, `resolveNodeMatches()`, `normalizeNodeKey()` — fuzzy node matching by ID/name/IP |
| `requirements.ts` | `resolveMissingBins()`, `resolveMissingAnyBins()`, `resolveMissingEnv()`, `resolveMissingOs()`, `buildConfigChecks()`, `evaluateRequirementsFromMetadata()` — skill/plugin requirements checking |
| `subagents-format.ts` | `formatDurationCompact()`, `formatTokenShort()`, `truncateLine()` — display formatting for subagent info |
| `usage-aggregates.ts` | `buildUsageAggregateTail()` — aggregates usage data by channel, latency, model, daily |
| `net/ipv4.ts` | `validateIPv4AddressInput()` — IPv4 address validation |
| `text/reasoning-tags.ts` | `stripReasoningTagsFromText()` — strips `<think>`, `<thinking>`, `<thought>`, `<antthinking>`, `<final>` tags while preserving code blocks |

### Key Types

| Type | File | Description |
|------|------|-------------|
| `DeviceAuthEntry` | `device-auth.ts` | `{ token, role, scopes, updatedAtMs }` |
| `DeviceAuthStore` | `device-auth.ts` | `{ version: 1, deviceId, tokens }` |
| `Requirements` | `requirements.ts` | `{ bins, anyBins, env, config, os }` |
| `RequirementsMetadata` | `requirements.ts` | Skill manifest requirements block |
| `NodeMatchCandidate` | `node-match.ts` | `{ nodeId, displayName?, remoteIp? }` |
| `ReasoningTagMode` | `text/reasoning-tags.ts` | `"strict" \| "preserve"` |

### Internal Dependencies

- `src/compat/legacy-names.ts` — `MANIFEST_KEY`, `LEGACY_MANIFEST_KEYS`
- `src/utils/boolean.ts` — `parseBooleanValue()`

### External Dependencies

- `json5` — JSON5 parsing for frontmatter metadata

### Test Coverage

- `frontmatter.test.ts` — string list, bool, manifest block parsing
- `node-match.test.ts` — node ID matching, ambiguity, prefix matching
- `requirements.test.ts` — binary/env/OS/config requirement checking
- `text/reasoning-tags.test.ts` — tag stripping with code block preservation

---

## src/types

### Module Overview

TypeScript ambient declaration files (`.d.ts`) for untyped npm packages.

### File Inventory

| File | Declares types for |
|------|-------------------|
| `cli-highlight.d.ts` | `cli-highlight` — syntax highlighting |
| `lydell-node-pty.d.ts` | `@lydell/node-pty` — PTY spawning |
| `napi-rs-canvas.d.ts` | `@napi-rs/canvas` — canvas creation |
| `node-edge-tts.d.ts` | `node-edge-tts` — Edge TTS API |
| `node-llama-cpp.d.ts` | `node-llama-cpp` — local LLM inference |
| `osc-progress.d.ts` | `osc-progress` — terminal progress bars |
| `pdfjs-dist-legacy.d.ts` | `pdfjs-dist/legacy` — PDF parsing |
| `proper-lockfile.d.ts` | `proper-lockfile` — file locking |
| `qrcode-terminal.d.ts` | `qrcode-terminal` — QR code rendering |

No runtime code, no dependencies, no tests.

---

## src/logging

### Module Overview

Structured logging system with file + console output, subsystem filtering, redaction, and diagnostic session state tracking. Uses `tslog` for structured file logging and wraps `console.*` for formatted console output.

### File Inventory

| File | Description |
|------|-------------|
| `config.ts` | `readLoggingConfig()` — reads `logging` section from openclaw.json |
| `console.ts` | Console capture, formatting, timestamp prefixes, subsystem filtering, `enableConsoleCapture()`, `formatConsoleTimestamp()` |
| `levels.ts` | `LogLevel` type, `normalizeLogLevel()`, `levelToMinLevel()` — log level definitions |
| `logger.ts` | `getLogger()`, `getChildLogger()` — tslog-based file logger with rotation, `DEFAULT_LOG_DIR`, `DEFAULT_LOG_FILE` |
| `state.ts` | `loggingState` — mutable singleton tracking logger cache, console patches, overrides |
| `subsystem.ts` | `createSubsystemLogger()` — `SubsystemLogger` with per-subsystem filtering, console+file dual output |
| `redact.ts` | `redactSensitiveText()`, `getDefaultRedactPatterns()` — redacts API keys, tokens, PEM blocks in tool output |
| `redact-identifier.ts` | `sha256HexPrefix()`, `redactIdentifier()` — one-way hash for identifiers |
| `parse-log-line.ts` | `parseLogLine()` — parses structured JSON log lines |
| `diagnostic.ts` | `logWebhookReceived()`, `logMessageProcessed()`, `logSessionStateChange()`, `logLaneEnqueue/Dequeue()` |
| `diagnostic-session-state.ts` | `diagnosticSessionStates` Map, `getDiagnosticSessionState()`, `pruneDiagnosticSessionStates()` — bounded session state tracking |

### Key Types

| Type | File | Description |
|------|------|-------------|
| `LogLevel` | `levels.ts` | `"silent" \| "fatal" \| "error" \| "warn" \| "info" \| "debug" \| "trace"` |
| `ConsoleStyle` | `console.ts` | `"pretty" \| "compact" \| "json"` |
| `SubsystemLogger` | `subsystem.ts` | Interface: `isEnabled()`, `trace/debug/info/warn/error/fatal()`, `raw()`, `child()` |
| `ParsedLogLine` | `parse-log-line.ts` | `{ time, level, subsystem, module, message, raw }` |
| `RedactSensitiveMode` | `redact.ts` | `"off" \| "tools"` |
| `SessionState` | `diagnostic-session-state.ts` | `{ sessionId, sessionKey, lastActivity, state, queueDepth }` |

### Internal Dependencies

- `src/config` — `OpenClawConfig`, `resolveConfigPath()`
- `src/globals.ts` — `isVerbose()`
- `src/runtime.ts` — `defaultRuntime`
- `src/terminal` — `stripAnsi()`, `clearActiveProgressLine()`
- `src/channels` — `CHAT_CHANNEL_ORDER`
- `src/infra` — `diagnostic-events`, `tmp-openclaw-dir`

### External Dependencies

- `tslog` — Structured logger (file transport)
- `chalk` — Console coloring

### Configuration Keys

- `logging.level` — file log level
- `logging.consoleLevel` — console log level
- `logging.consoleStyle` — `pretty`/`compact`/`json`
- `logging.file` — custom log file path
- `logging.redact.mode` — `"off"` or `"tools"`

### Test Coverage

- `console-capture.test.ts` — console patching, stderr routing, timestamp prefix
- `console-prefix.test.ts` — redundant subsystem prefix stripping
- `console-settings.test.ts` — console settings resolution
- `console-timestamp.test.ts` — ISO timestamp formatting
- `diagnostic.test.ts` — session state pruning and eviction
- `logger.import-side-effects.test.ts` — no mkdir at import time
- `parse-log-line.test.ts` — JSON log line parsing
- `redact.test.ts` — sensitive text redaction
- `subsystem.test.ts` — subsystem logger isEnabled, filtering

---

## src/process

### Module Overview

Process execution, command queuing with lane-based concurrency, child process lifecycle management.

### File Inventory

| File | Description |
|------|-------------|
| `exec.ts` | `runCommandWithTimeout()`, `shouldSpawnWithShell()` — executes commands with timeout, Windows .cmd resolution |
| `command-queue.ts` | `enqueueCommand()`, `enqueueCommandInLane()`, `setCommandLaneConcurrency()`, `clearCommandLane()`, `waitForActiveTasks()` — lane-based in-process command serialization |
| `lanes.ts` | `CommandLane` enum: `Main`, `Cron`, `Subagent`, `Nested` |
| `child-process-bridge.ts` | `attachChildProcessBridge()` — forwards signals (SIGTERM, SIGINT, SIGHUP) to child processes |
| `spawn-utils.ts` | `spawnWithFallback()`, `resolveCommandStdio()` — spawn with EBADF retry and fallback options |
| `restart-recovery.ts` | `createRestartIterationHook()` — distinguishes first boot from restart iterations |

### Key Types

| Type | File | Description |
|------|------|-------------|
| `CommandLane` | `lanes.ts` | Enum: `Main`, `Cron`, `Subagent`, `Nested` |
| `CommandLaneClearedError` | `command-queue.ts` | Error thrown when a lane is cleared during execution |
| `SpawnWithFallbackResult` | `spawn-utils.ts` | `{ child, usedFallback, fallbackLabel }` |

### Internal Dependencies

- `src/globals.ts` — `danger`, `shouldLogVerbose`
- `src/logger.ts` — `logDebug`, `logError`
- `src/logging/diagnostic.ts` — `logLaneEnqueue`, `logLaneDequeue`

### Test Coverage

- `exec.test.ts` — shell mode, env passing, timeout
- `command-queue.test.ts` — lane concurrency, clearing, task counting
- `child-process-bridge.test.ts` — signal forwarding to child processes
- `spawn-utils.test.ts` — EBADF retry, fallback spawning
- `restart-recovery.test.ts` — first/subsequent iteration detection

---

## src/pairing

### Module Overview

Device pairing system — allows new users to pair their messaging accounts with an OpenClaw instance via one-time codes.

### File Inventory

| File | Description |
|------|-------------|
| `pairing-store.ts` | `upsertChannelPairingRequest()`, `listChannelPairingRequests()` — persistent pairing request storage with file locking, TTL, max pending limit |
| `pairing-messages.ts` | `buildPairingReply()` — generates the pairing instruction message with code and CLI command |
| `pairing-labels.ts` | `resolvePairingIdLabel()` — resolves per-channel ID label (e.g. "userId", "phone") |

### Key Types

| Type | File | Description |
|------|------|-------------|
| `PairingChannel` | `pairing-store.ts` | Alias for `ChannelId` |
| `PairingRequest` | `pairing-store.ts` | `{ id, code, channel, senderId, ... }` |

### Internal Dependencies

- `src/channels/plugins` — `ChannelId`, pairing adapters
- `src/config/paths.ts` — `resolveOAuthDir()`, `resolveStateDir()`
- `src/infra/file-lock.ts` — `withFileLock()`
- `src/cli/command-format.ts` — `formatCliCommand()`

### Test Coverage

- `pairing-store.test.ts` — store creation, upsert, listing with temp dirs
- `pairing-messages.test.ts` — reply message formatting per channel

---

## src/node-host

### Module Overview

Node host agent — runs on remote machines (Mac, Pi, etc.) and connects to the gateway via WebSocket. Handles remote command invocation, browser proxy, and skill execution.

### File Inventory

| File | Description |
|------|-------------|
| `runner.ts` | `NodeHostRunOptions`, main node host lifecycle — connects to gateway, handles invocations, skill bin caching |
| `invoke.ts` | `handleInvoke()`, `sanitizeEnv()`, `coerceNodeInvokePayload()` — dispatches incoming invoke commands (system.run, browser, etc.) with security (env sanitization, dangerous key blocking) |
| `invoke-browser.ts` | Browser proxy for node-hosted browser — creates browser control context, routes browser API calls |
| `config.ts` | `NodeHostConfig`, `ensureNodeHostConfig()`, `saveNodeHostConfig()` — node identity and gateway connection config |
| `with-timeout.ts` | `withTimeout()` — generic async timeout wrapper with AbortSignal |

### Key Types

| Type | File | Description |
|------|------|-------------|
| `NodeHostConfig` | `config.ts` | `{ version, nodeId, token, displayName, gateway }` |
| `NodeHostGatewayConfig` | `config.ts` | `{ host, port, tls, tlsFingerprint }` |

### Internal Dependencies

- `src/config` — `loadConfig()`
- `src/gateway` — `GatewayClient`
- `src/browser` — browser config, control service, routes
- `src/infra` — exec-approvals, device identity, machine name, path env
- `src/agents` — agent scope, agent config

### Test Coverage

- `runner.test.ts` — `buildNodeInvokeResultParams()` payload construction
- `invoke.sanitize-env.test.ts` — env sanitization (PATH protection, dangerous key blocking)

---

## src/macos

### Module Overview

macOS-specific integration — gateway daemon entry point and WhatsApp relay (Baileys-based).

### File Inventory

| File | Description |
|------|-------------|
| `gateway-daemon.ts` | macOS gateway daemon entry point — starts gateway with process respawn support |
| `relay.ts` | WhatsApp relay entry point — patches Bun Long for protobuf, starts relay |
| `relay-smoke.ts` | `parseRelaySmokeTest()`, `runRelaySmokeTest()` — QR code smoke test for relay |

### Internal Dependencies

- `src/infra` — gateway lock, process respawn

### Test Coverage

- `relay-smoke.test.ts` — smoke test CLI argument parsing

---

## src/compat

### Module Overview

Legacy compatibility constants for project rename/migration.

### File Inventory

| File | Description |
|------|-------------|
| `legacy-names.ts` | `PROJECT_NAME = "openclaw"`, `MANIFEST_KEY`, `LEGACY_PROJECT_NAMES`, `LEGACY_MANIFEST_KEYS`, `MACOS_APP_SOURCES_DIR` |

No runtime logic, no tests, no dependencies.

---

## src/scripts

### Module Overview

Build scripts for the canvas A2UI (AI-to-UI) feature.

### File Inventory

| File | Description |
|------|-------------|
| `canvas-a2ui-copy.test.ts` | Tests for `copyA2uiAssets()` — verifies asset copying and error handling |

(The actual `canvas-a2ui-copy.ts` implementation is in `scripts/` outside `src/`)

---

## src/docs

### Module Overview

Documentation validation tests.

### File Inventory

| File | Description |
|------|-------------|
| `slash-commands-doc.test.ts` | Validates that all built-in slash commands are documented in `docs/tools/slash-commands.md` |

---

## src/test-helpers

### Module Overview

Test setup utilities for integration tests.

### File Inventory

| File | Description |
|------|-------------|
| `state-dir-env.ts` | `snapshotStateDirEnv()`, `restoreStateDirEnv()`, `setStateDirEnv()` — manages `OPENCLAW_STATE_DIR` env for test isolation |
| `workspace.ts` | `makeTempWorkspace()`, `writeWorkspaceFile()` — creates temporary workspace directories for tests |

---

## src/test-utils

### Module Overview

Shared test utilities and mock factories.

### File Inventory

| File | Description |
|------|-------------|
| `channel-plugins.ts` | `createTestRegistry()`, `createOutboundTestPlugin()` — creates mock plugin registries for testing |
| `imessage-test-plugin.ts` | `createIMessageTestPlugin()` — iMessage channel plugin test stub |
| `ports.ts` | `isPortFree()`, `getOsFreePort()` — finds available ports for integration tests |
| `vitest-mock-fn.ts` | `MockFn<T>` — centralized Vitest mock type to avoid TS2742 inference issues |

---

## Cross-cutting Concerns

### Shared Patterns

1. **Normalize + Resolve pattern**: Almost every module follows `normalize*()` → `resolve*()` for config values (e.g., `normalizeThinkLevel()` → `resolveDefaultThinkingLevel()`)

2. **Session store**: JSON file at `~/.openclaw/state/sessions.json` (configurable) with file locking via `proper-lockfile`. Updated atomically via `updateSessionStore()` with snapshot-and-merge.

3. **Barrel re-exports**: Modules use barrel files (`reply.ts`, `commands.ts`, `directive-handling.ts`, `queue.ts`) to provide clean public APIs.

4. **Discriminated unions**: Common pattern for results: `{ kind: "reply", reply } | { kind: "result", result }` and `{ kind: "success" } | { kind: "final", payload }`.

5. **Typing controller**: Unified typing indicator system with configurable modes (always/reasoning/message) and interval-based refresh. Created once per reply and threaded through the entire pipeline.

6. **Block streaming pipeline**: Coalesces rapid LLM output chunks into platform-sized messages with idle timeout, audio buffering, and dedup.

7. **Error recovery**: Auto-recovery from context overflow, compaction failure, role ordering conflicts, session corruption, and transient HTTP errors — all with session reset and user-facing warnings.

8. **Redaction**: Sensitive data (API keys, tokens, PEM blocks) is redacted in tool output before being shown to users, controlled by `logging.redact.mode`.

9. **Config layering**: Global defaults → per-agent → per-session → per-message inline directives, with each layer overriding the previous.

### Utility Functions Used Everywhere

- `parseBooleanValue()` (src/utils/boolean) — used across config, frontmatter, commands
- `normalizeAccountId()` (src/utils/account-id) — used in session keys, delivery context, channel config
- `resolveMessageChannel()` (src/utils/message-channel) — used for formatting, capability checking
- `formatTokenCount()`, `formatUsd()` (src/utils/usage-format) — used in status, usage display
- `safeJsonStringify()` (src/utils/safe-json) — used in logging, diagnostics
- `createSubsystemLogger()` (src/logging/subsystem) — used in every major module
- `stripReasoningTagsFromText()` (src/shared/text/reasoning-tags) — used in reply normalization
- `resolveNodeIdFromCandidates()` (src/shared/node-match) — used in node tools
- `normalizeStringList()` (src/shared/frontmatter) — used in skill/plugin metadata parsing
