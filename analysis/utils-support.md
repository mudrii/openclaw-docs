# Utilities & Support Modules — Comprehensive Analysis
<!-- markdownlint-disable MD024 -->

**Updated:** 2026-04-23 | **Version:** v2026.4.21 | **Codebase:** OpenClaw release tag `v2026.4.21`
**Cluster:** Utilities & Support Modules  
**Total files analyzed:** stable release-line snapshot across 14 support modules plus `apps/macos`

---

## Table of Contents

1. [src/auto-reply (287 files)](#srcauto-reply)
2. [src/utils (29 files)](#srcutils)
3. [src/shared (51 files)](#srcshared)
4. [src/types (9 files)](#srctypes)
5. [src/logging (29 files)](#srclogging)
6. [src/process (28 files)](#srcprocess)
7. [src/pairing (9 files)](#srcpairing)
8. [src/node-host (16 files)](#srcnode-host)
9. [apps/macos (353 Swift files)](#appsmacos)
10. [src/compat (1 file)](#srccompat)
11. [src/scripts (2 files)](#srcscripts)
12. [src/docs (1 file)](#srcdocs)
13. [src/test-helpers (4 files)](#srctest-helpers)
14. [src/test-utils (35 files)](#srctest-utils)

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

### Selected File Inventory

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
| `ThinkLevel` | `thinking.ts` | `"off" \| "minimal" \| "low" \| "medium" \| "high" \| "xhigh" \| "adaptive"` |
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

### Test Coverage (69 test files)

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
| `chunk-items.ts` | `chunkItems()` — splits arrays into fixed-size chunks |
| `with-timeout.ts` | `withTimeout()` — generic async timeout wrapper |
| `run-with-concurrency.ts` | `runWithConcurrency()` — concurrent task execution with limits |
| `mask-api-key.ts` | `maskApiKey()` — masks API keys for safe display |
| `parse-json-compat.ts` | `parseJsonCompat()` — lenient JSON parsing with trailing comma / comment tolerance |

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

Shared utilities used across both gateway and CLI/node-host contexts. Designed to be import-safe without heavy side effects. The module has expanded to 80+ files.

### File Inventory (selected key files)

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
| `device-auth-store.ts` | Device auth token store with atomic JSON read/write |
| `assistant-error-format.ts` | Assistant error formatting utilities |
| `text/join-segments.ts` | Text segment joining utilities |
| `text/code-regions.ts` | Code region detection and extraction |
| `text/auto-linked-file-ref.ts` | Auto-linked file reference resolution |
| `text/strip-markdown.ts` | Markdown stripping for plain-text contexts |

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
| `windows-command.ts` | Windows command resolution and `.cmd` shim handling |
| `kill-tree.ts` | Process tree termination |
| `supervisor/` | Process supervisor subsystem (new subdirectory) |

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

## apps/macos

### Module Overview

macOS app codebase for the OpenClaw menubar application, including gateway lifecycle, onboarding/settings UI, voice wake controls, canvas windows, and local IPC.

### File Inventory

| File | Description |
|------|-------------|
| `apps/macos/Sources/OpenClaw/AppState.swift` | Top-level app state, lifecycle wiring, and shared stores |
| `apps/macos/Sources/OpenClaw/GatewayProcessManager.swift` | Gateway process start/stop/restart, status tracking, and health integration |
| `apps/macos/Sources/OpenClaw/LaunchdManager.swift` | LaunchAgent install/remove/status integration for persistent gateway startup |
| `apps/macos/Sources/OpenClaw/VoiceWakeForwarder.swift` | Voice wake command forwarding into the local OpenClaw runtime |

### Internal Dependencies

- `src/gateway` — runtime process/API counterpart used by the macOS app
- `src/config` — shared config model consumed by app-driven configuration flows
- `apps/macos/Sources/OpenClawIPC` — IPC transport and request/response models

### Test Coverage

- `apps/macos/Tests/OpenClawIPCTests/GatewayProcessManagerTests.swift` — gateway lifecycle behavior
- `apps/macos/Tests/OpenClawIPCTests/GatewayLaunchAgentManagerTests.swift` — LaunchAgent management
- `apps/macos/Tests/OpenClawIPCTests/VoiceWakeForwarderTests.swift` — voice wake forwarding behavior

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

---

## v2026.2.15 Changes

### Auto-Reply
- **Expose inbound message identifiers in trusted metadata**: `reply/inbound-meta.ts` — inbound message IDs (platform-specific) are now included in trusted metadata, enabling reply-to-specific-message workflows
- **Share directive handling**: `reply/directive-handling.shared.ts` — extracted shared directive formatting/acknowledgment logic (`formatDirectiveAck()`) used across multiple directive handlers
- **Dedupe on/off/full normalization**: Consolidated boolean/tri-state normalization (`"on"/"off"/"full"`) that was duplicated across thinking, verbose, elevated, and reasoning directive handlers
- **Preserve queued items on drain retries**: `reply/queue/drain.ts` — when a followup queue drain fails, queued items are now preserved for retry instead of being lost. Related commit: `2a8360928`
- **Share abort persistence**: `reply/abort.ts` — extracted shared abort signal persistence logic used by both reply abort and queue cleanup

### Logging
- **Split diagnostic session state module**: `logging/diagnostic-session-state.ts` — extracted `diagnosticSessionStates` Map, `getDiagnosticSessionState()`, and `pruneDiagnosticSessionStates()` from `diagnostic.ts` into a dedicated module for cleaner separation
- **Skip eager debug formatting**: `logging/subsystem.ts` — debug/trace log messages now skip expensive string formatting when the log level is not enabled, reducing overhead in production

### Pairing
- **Account-scoped stores**: `pairing/pairing-store.ts` — pairing request stores are now scoped per account, supporting multi-account setups where different bot accounts have independent pairing flows
- **`infra/pairing-files.ts`** (new): Extracted pairing file path resolution from inline logic in pairing-store and device-pairing
- **`infra/device-auth-store.ts`** (new): Extracted device auth token store with atomic JSON read/write via `json-file.ts`
- **Legacy telegram `allowFrom` migration**: Migrates legacy `channels.telegram.allowFrom` config to the new account-scoped pairing store format

## v2026.2.19 Changes

### YAML Parsing
- **YAML 1.2 core schema** — Frontmatter parsing uses YAML 1.2 core schema; `on`/`off`/`yes`/`no` no longer auto-coerced to booleans. See DEVELOPER-REFERENCE.md §9 (gotcha 37)

### Logging
- **Config change audit logging** — Actor, device, IP, and changed paths logged on control-plane config mutations

### Networking
- **SSRF guard hardening** — NAT64/6to4/Teredo IPv6 transition addresses and octal/hex/short/packed IPv4 blocked
- **Plaintext ws:// blocked** — Non-loopback plaintext WebSocket connections rejected; `wss://` required for remote hosts

<!-- v2026.2.21 -->
## v2026.2.21 Changes

### Channels (shared infrastructure)

<!-- v2026.2.21 -->
- **Shared status-reactions controller** (new file: `src/channels/status-reactions.ts`, ~390 lines) — Unified, channel-agnostic lifecycle reaction system shared between Telegram and Discord. Provides:
  - `StatusReactionAdapter` — minimal interface (`setReaction`, optional `removeReaction`) that each channel implements to perform the actual API call.
  - `StatusReactionEmojis` — per-phase emoji config (all optional, with defaults): `queued`, `thinking`, `tool`, `coding`, `web`, `done`, `error`, `stallSoft`, `stallHard`.
  - `StatusReactionTiming` — debounce and stall timer thresholds: `debounceMs` (700ms), `stallSoftMs` (10s), `stallHardMs` (30s), `doneHoldMs` (1500ms), `errorHoldMs` (2500ms).
  - `StatusReactionController` — public API: `setQueued()`, `setThinking()`, `setTool(toolName?)`, `setDone()`, `setError()`, `clear()`, `restoreInitial()`.
  - `createStatusReactionController()` — factory that handles promise-chain serialization (prevents concurrent API calls), debouncing of intermediate states (terminal states are immediate), and stall timers that escalate `stallSoft → stallHard` on inactivity.
  - `resolveToolEmoji()` — selects `coding` emoji for exec/read/write/bash-family tools, `web` emoji for web_search/web_fetch/browser-family tools, generic `tool` otherwise.
  - `DEFAULT_EMOJIS` and `DEFAULT_TIMING` exported constants.
  - Telegram's `status-reaction-variants.ts` builds on this base, adding Telegram-specific emoji preselection against the platform's fixed supported-reaction set and per-chat `available_reactions` allowlist.

### Auto-Reply

<!-- v2026.2.21 -->
- **TTS model-provider switching is now opt-in** — Previously, when TTS was active and a TTS-capable model was resolved, the system could automatically switch the LLM provider to one paired with the TTS engine. This automatic switching is now opt-in and must be explicitly enabled in config. Prevents unexpected provider changes during TTS sessions.

### Gateway

<!-- v2026.2.21 -->

## v2026.2.22 Changes <!-- v2026.2.22 -->

### Auto-Reply <!-- v2026.2.22 -->

- **Default completion acknowledgement suppressed** — The default completion acknowledgement `✅ Done.` is suppressed for channel/group sessions and runs that already delivered output via messaging tools. <!-- v2026.2.22 -->

### Logging <!-- v2026.2.22 -->

- **`logging.maxFileBytes` cap** — New `logging.maxFileBytes` config key (default 500 MB) caps single log file size and suppresses additional writes after the cap is hit — prevents disk exhaustion from error storms. <!-- v2026.2.22 -->

### Pairing <!-- v2026.2.22 -->

- **Loopback scope-upgrade auto-approve** — Auto-approve loopback `scope-upgrade` pairing requests (including device-token reconnects) — local clients no longer disconnect on pairing-required scope elevation. `operator.admin` satisfies other `operator.*` scope checks. `operator.read`/`operator.write` included in default operator connect scope bundles. `operator.admin` pairing tokens satisfy `operator.write` requests. <!-- v2026.2.22 -->

### Delivery <!-- v2026.2.22 -->

- **Queue entry quarantine on permanent errors** — Queue entries quarantined immediately on permanent delivery errors (invalid recipients, missing conversation references) — moved to `failed/` instead of retrying on every restart. <!-- v2026.2.22 -->

### Network <!-- v2026.2.22 -->

- **undici `TypeError: fetch failed` classified as transient** — Classifies undici `TypeError: fetch failed` as transient in unhandled-rejection detection even when nested causes are unclassified — prevents gateway crash loops on flaky networks. <!-- v2026.2.22 -->

- **Auth-profile cooldown immutability** — `cooldownUntil`/`disabledUntil` windows kept immutable across retries — only recompute backoff after previous deadline expires. <!-- v2026.2.22 -->

### TUI <!-- v2026.2.22 -->

- **Multiline-paste burst coalescing** — Multiline-paste burst coalescing on macOS Terminal.app and iTerm. RTL script lines (Arabic/Hebrew) isolated with Unicode bidi marks. Immediate renders after `sending`/`waiting` activity states. Ctrl+C exit timing fixed; SIGINT fallback path for active runs. <!-- v2026.2.22 -->

## v2026.2.24 Changes <!-- v2026.2.24 -->

### Auto-Reply <!-- v2026.2.24 -->

- **Multilingual stop phrases** (#25103) — Stop phrase list broadened (`stop openclaw`, `stop action`, `stop run`, `stop agent`, `please stop` + variants); trailing punctuation accepted (e.g. `STOP OPENCLAW!!!`); multilingual stop keywords added (ES/FR/ZH/HI/AR/JP/DE/PT/RU forms) for reliable emergency stops. Contributors: @steipete, @vincentkoc.
- **Reset hooks** (#25459) — `/new` and `/reset` hook emission guaranteed on early-return paths; dedupe guard prevents double emission. Contributor: @chilu18.
- **Typing keepalive** — Typing keepalive indicator is now lifecycle-managed across longer runs to avoid stale "typing…" states. (See DEVELOPER-REFERENCE.md §9 gotcha #79.)

## v2026.2.23 Changes <!-- v2026.2.23 -->

### Auto-Reply <!-- v2026.2.23 -->

- **Direct-chat `message_id` and sender metadata hidden from normalized chat type** — `message_id`/`message_id_full` and sender metadata hidden from normalized chat type only — preserves group metadata visibility; prevents sender-id spoofed direct-mode classification. <!-- v2026.2.23 -->
- **Inbound metadata stripping** (`src/gateway/chat-sanitize.ts`, backed by `src/auto-reply/reply/strip-inbound-meta.ts`) — The WS connection message handler now strips internal metadata blocks from inbound messages before routing them to channel surfaces or agent sessions. `stripEnvelopeFromMessage()` applies `stripInboundMetadata()` to all text content (both string and array-of-blocks forms), then strips `[Channel From Timestamp]` envelope headers and message-id hints from user-role messages. This prevents internal marker blocks (injected by the auto-reply pipeline for message correlation) from leaking into chat surfaces or being re-injected into subsequent agent turns. The function handles `content: string`, `content: [{type:"text", text:...}]`, and `text: string` message shapes, and is applied to every inbound message array via `stripEnvelopeFromMessages()`.

## v2026.3.1 Changes <!-- v2026.3.1 -->

### Auto-Reply <!-- v2026.3.1 -->

- **NO_REPLY token stripping from mixed-content messages** (#30916, #30955) — `normalizeReplyPayload()` in `reply/normalize-reply.ts` now strips the `NO_REPLY` silent token from mixed-content messages (e.g. text containing both emoji and `NO_REPLY`) via the new `stripSilentToken()` helper in `tokens.ts`. Previously, only exact-match `NO_REPLY` was handled; mixed-content messages leaked the raw control text to end users. If stripping leaves nothing and there is no media/channelData, the message is treated as silent. The `stripSilentToken()` function uses a regex to remove a trailing silent token separated by whitespace.
- **Block reply timeout path normalized through Promise.resolve** (#31200) — `createBlockReplyPipeline()` in `reply/block-reply-pipeline.ts` now wraps the `onBlockReply()` callback return value in `Promise.resolve()` before passing it to `withTimeout()`. This ensures deterministic timeout behavior even when the callback returns `undefined` instead of a promise. Previously, a non-promise return could bypass the timeout race.
- **`ThinkLevel` gains `"adaptive"` variant** (#31227) — `thinking.ts` adds `"adaptive"` to the `ThinkLevel` union, enabling per-model thinking defaults that adapt based on provider capabilities (initially for Anthropic models). `normalizeThinkLevel()` now accepts `"adaptive"` as valid input. Per-model thinking defaults are resolved via the new `directive-handling.levels.ts` module.
- **Inbound metadata includes `account_id`** (#30984) — `reply/inbound-meta.ts` now includes `account_id` (from `ctx.AccountId`) in the `openclaw.inbound_meta.v1` trusted metadata payload, and adds `topic_id` (from `ctx.MessageThreadId`) to the user context prefix. Enables multi-account-aware agent behaviors.
- **Session routing preserves external `lastTo`/`lastChannel` for internal turns** — `reply/session.ts` refactors channel and destination resolution with a new `isExternalRoutingChannel()` guard and `resolveLastToRaw()` helper. Internal/non-deliverable sources (webchat, system) no longer overwrite previously known external delivery routes. This prevents internal session turns from corrupting the reply destination for subsequent external messages.

### Logging <!-- v2026.3.1 -->

- **Feishu runtime-gated logging** (#18841) — `extensions/feishu/src/bot.ts` replaces bare `console.log`/`console.error` calls with runtime-gated `log()`/`error()` obtained from `getFeishuRuntime()`, so typing indicator and message processing errors respect the configured log level instead of always writing to stderr.
- **Ollama empty-discovery demoted from warn to debug** (#26379) — When Ollama is unreachable and not explicitly configured, the autodiscovery log is now emitted at debug level instead of warn. Explicitly configured Ollama instances still warn on unreachable.
- **Discord allowlist diagnostics** — `src/discord/monitor/provider.allowlist.ts` and `src/discord/resolve-channels.ts` emit debug-level logs for guild/channel drops during allowlist resolution, including guild names and channel names in the log output for easier troubleshooting of permission misconfigurations.
- **Configurable stuck-session warning threshold** — `logging/diagnostic.ts` adds `resolveStuckSessionWarnMs()` that reads `diagnostics.stuckSessionWarnMs` from config (default 120s). The diagnostic heartbeat now hot-reloads this threshold from config on each tick instead of using a hardcoded value.

### Pairing <!-- v2026.3.1 -->

- **AllowFrom account fallback for omitted `accountId`** (#31369) — `pairing/pairing-store.ts` `resolveAllowFromPath()` now treats an omitted or empty `accountId` as the default account, falling back to the channel-level `<channel>-allowFrom.json` path. Previously, callers that omitted `accountId` could silently read from the wrong store or create orphaned files.
- **Device-auth v2 migration diagnostics** (#28305) — `src/gateway/protocol/connect-error-details.ts` (new) adds a device-auth detail code resolver that emits specific error codes for nonce mismatch, signature verification failure, and version downgrade attempts during device-auth handshake. Gateway server auth now includes these detail codes in WebSocket close frames for easier client-side diagnostics.

### Delivery <!-- v2026.3.1 -->

- **Subagent delivery params rejection** (#31000, #31110) — `src/agents/tools/sessions-spawn-tool.ts` now rejects unsupported channel-targeting parameters (`target`, `transport`, `channel`, `to`, `threadId`, `thread_id`, `replyTo`, `reply_to`) with a `ToolInputError` directing callers to use `message` or `sessions_send` for channel delivery. Prevents agents from passing through delivery parameters that `sessions_spawn` cannot honor.
- **Subagent Slack announce thread fix** (#31105) — `src/agents/subagent-announce.ts` no longer blindly passes `conversationId` as `threadId` for bound completion announces. Only explicit requester thread hints are preserved, preventing invalid `thread_ts` errors on Slack DM/top-level delivery.
- **Telegram multi-account fallback isolation** (#30673) — `src/telegram/bot-message-context.ts` and `mergeTelegramAccountConfig()` now exclude channel-level `groups` from the base config spread in multi-account setups. Channel-level groups are only inherited as fallback in single-account configurations. Multi-account setups must use account-level groups config. Prevents secondary bots from attempting to handle messages for groups they are not members of (fail-closed).
- **Inbound metadata `account_id` in trusted context** — (See Auto-Reply section above.)

### Usage <!-- v2026.3.1 -->

- **Clamp negative prompt/input token values to zero** — `src/agents/usage.ts` `normalizeUsage()` now clamps negative `input` token values to 0. Some providers (pi-ai OpenAI-format) pre-subtract `cached_tokens` from `prompt_tokens` upstream; when cache exceeds prompt tokens the result is negative. `deriveSessionTotalTokens()` is also simplified to use `derivePromptTokens()` directly and no longer includes completion/output tokens in the session snapshot.
- **Codex weekly usage window label** — `src/infra/provider-usage.fetch.codex.ts` now correctly labels secondary rate-limit windows >= 168 hours as "Week" (previously all windows >= 24h were labeled "Day").

### Infra/Support <!-- v2026.3.1 -->

- **Daemon systemd checks in containers** (#26089, #26699) — `src/daemon/systemd.ts` handles missing `systemctl` in containers by catching spawn failures and treating the service as unavailable rather than crashing. Unknown `is-enabled` errors fail closed.
- **macOS supervised restart via supervisor handoff** — current `v2026.3.8` supervised restart exits and relies on launchd `KeepAlive` instead of self-kickstart, while LaunchAgent repair/restart paths `enable` before `bootstrap` and detect supervision through `XPC_SERVICE_NAME`.
- **macOS TLS certs: `NODE_EXTRA_CA_CERTS` default** (#22856) — `src/daemon/service-env.ts` sets `NODE_EXTRA_CA_CERTS` to `/etc/ssl/cert.pem` on macOS in both `buildServiceEnvironment` and `buildNodeServiceEnvironment` when not already set. Fixes TLS verification failures for HTTPS requests (Telegram, webhooks) when running as a LaunchAgent, since launchd does not inherit the shell environment.
- **npm install fallback and global update** (#28318, #21039) — `src/process/exec.ts` adds `resolveNpmArgvForWindows()` to resolve npm/npx to `node + cli-script` instead of spawning `.cmd` directly, avoiding EINVAL on Windows (CVE-2024-27980). npm/npx are removed from the `.cmd` resolution list. Plugin install spawn now falls back correctly when `npm pack` output is empty.
- **Gateway healthz/readyz probe endpoints** (#31272) — New HTTP probe endpoints for container health checks.

## v2026.3.7 Delta Notes

- Models: Gemini 3.1 Flash-Lite and GPT-5.4 added to the supported model list; Venice kimi-k2-5 set as the default Venice model; MiniMax Lightning removed.
- Ollama remote provider auth fallback: Ollama remote provider now falls back to unauthenticated mode when no API key is configured, rather than rejecting the connection.
- xAI web-search collision guard: xAI provider no longer collides with web-search tool parameter names when both are active in the same session.
- Venice completion-token limit alignment: Venice provider completion token limits are aligned with the model's declared context window to prevent over-requesting.
- OpenAI completions streaming compatibility: OpenAI completions endpoint streaming now handles provider-specific delta formats that differ from the chat completions streaming format.
- Cache-trace circular-reference stability: cache-trace diagnostic output no longer crashes on circular object references in provider response metadata.

## v2026.3.8 Delta Notes

- Talk config is now provider-based rather than ElevenLabs-only in practice: top-level `talk.silenceTimeoutMs` landed with defaults of `700 ms` on macOS/Android and `900 ms` on iOS, Talk config responses expose a normalized `resolved` payload, and SecretRef-shaped Talk API keys are accepted.
- Config writes now preserve/refresh runtime snapshots after disk writes so secret-resolved follow-up reads do not silently regress to stale state.
- macOS remote mode now exposes `gateway.remote.token`, preserves unsupported non-plaintext token shapes until explicitly replaced, and warns when the app cannot consume the loaded token directly.
- iOS/macOS node and canvas flows were hardened: queued foreground actions replay safely after resume, and scoped gateway canvas loading/capability refresh has safer fallback behavior.
- Android Play-distributed builds are now foreground-only for Voice-tab capture and no longer ship self-update, background-location “always”, screen-record, or background-mic capture behavior.

## v2026.3.11 Delta Notes

### Logging/Observability

- **Agents/embedded lifecycle and failover observation** (#41336) — `src/agents/pi-embedded-runner/run/failover-observation.ts` emits structured, sanitized `embedded_run_failover_decision` events (via `createFailoverDecisionLogger()`) on the `agent/embedded` subsystem. Fields include `runId`, `stage`, `decision`, `failoverReason`, `profileFailureReason`, `provider`, `model`, `profileId`, `fallbackConfigured`, and sanitized error observation fields from `buildApiErrorObservationFields()`. The `consoleMessage` field renders a single-line summary for tail-filtering. This makes overload and provider failures easier to observe and correlate in structured log output.

- **Agents/embedded overload logs include failing model and provider** (#41236) — The `consoleMessage` rendered for embedded-run failover decisions (`embedded_run_failover_decision`) now includes the `provider/model` pair and a shortened profile ID, so the error-path console output directly identifies the failing model without requiring a structured log query.

- **Agents/fallback observability — structured model-fallback decision events** (#41337) — `src/agents/model-fallback-observation.ts` emits structured `model_fallback_decision` events (via `logModelFallbackDecision()`) on the `model-fallback/decision` subsystem. Fields include `runId`, `decision` (`skip_candidate`, `probe_cooldown_candidate`, `candidate_failed`, `candidate_succeeded`), `requestedProvider`, `requestedModel`, `candidateProvider`, `candidateModel`, `reason`, `status`, and sanitized error observation fields. Auth-profile failure-state changes are separately logged via `logAuthProfileFailureStateChange()` in `src/agents/auth-profiles/state-observation.ts` (`auth_profile_failure_state_updated` event, `agent/embedded` subsystem) with correlated `runId` for cross-event tracing.

- **Logging/probe observations — suppress probe warnings on console** (#41338) — `src/logging/subsystem.ts` adds `shouldSuppressProbeConsoleLine()`: warn-level (and below) messages from the `agent/embedded` and `model-fallback` subsystem families are silently suppressed on the console when their `runId` (or `sessionId`) matches the `probe-` prefix. Error and fatal messages are never suppressed. This prevents health-probe noise from appearing in console output while preserving full structured log output to file. Verbose mode (`isVerbose()`) bypasses suppression.

### Auth/Cooldowns

- **Auth/cooldowns — reset expired error counters before backoff recompute** (#41028) — `src/agents/auth-profiles/usage.ts` `computeNextProfileUsageStats()` now checks whether the existing `cooldownUntil`/`disabledUntil` has already elapsed before computing the next backoff. If the previous cooldown is expired, `errorCount` and `failureCounts` are reset to zero so the subsequent failure uses a fresh first-offense backoff window rather than escalating from stale counters. The same reset is applied by `clearExpiredCooldowns()` in-memory during profile ordering, but the lock-based updater path now also applies it when reading a fresh store from disk.

- **Agents/cooldowns — default unknown failure history to `”unknown”` not `”rate_limit”`** (#42911) — `src/agents/auth-profiles/usage.ts` `resolveProfilesUnavailableReason()` now scores profiles with an active cooldown window but no recorded `failureCounts` entries as `”unknown”` instead of `”rate_limit”`. Previously the implicit default caused misleading “rate limit reached” console warnings for profiles that entered cooldown due to unclassified transient errors.

### General Utilities

- **Channels/allowlists — remove stale matcher caching** — `src/channels/allowlist-match.ts` functions (`resolveAllowlistMatchSimple()`, `resolveAllowlistMatchByCandidates()`) recompile the allowlist set from the input array on every call. No per-call caching is performed, so in-place mutations to the same array reference (element replacement, wildcard substitution) take effect immediately. Regression tests in `allowlist-match.test.ts` cover same-length in-place edits and wildcard-to-literal replacement.

- **Exec/child commands — mark child command environments with `OPENCLAW_CLI`** (#41411) — `src/process/exec.ts` `resolveCommandEnv()` calls `markOpenClawExecEnv()` from the new `src/infra/openclaw-exec-env.ts` module, injecting `OPENCLAW_CLI=1` into the environment of every child command spawned via `runCommandWithTimeout()`. Child processes can use this marker to detect that they are running inside an OpenClaw-managed invocation.

- **Git/runtime state — ignore `.dev-state` in gateway working tree** (#41848) — `.gitignore` now lists `.dev-state` so gateway-generated development state files in the working tree are not picked up by git status or committed accidentally.

- **Dependencies refresh** — Workspace dependencies refreshed (excluding the pinned Carbon package). ACP session-config writes are hardened against non-string SDK values to prevent type errors during session initialization.

### Auto-reply/Pairing

- **Telegram/direct delivery hooks — bridge to internal `message:sent`** (#40185) — `src/telegram/bot/delivery.replies.ts` `fireSentHooks()` now calls `triggerInternalHook()` with a `message:sent` event when a `sessionKeyForInternalHooks` is present in the delivery context. This bridges successful Telegram direct-delivery sends to the internal hook bus so internal subscribers (e.g. session event listeners) observe Telegram deliveries on the same `message:sent` path used by other channels.

### Control UI

- **Control UI/Sessions table — restore narrow-viewport single-column collapse** (#12175) — The responsive table override in the gateway control UI is now placed adjacent to the base grid rule, and inline-size container queries are enabled. This restores correct single-column session-table layout on narrow viewports that was broken by rule ordering.

- **Gateway/Control UI auth tokens — session-scoped browser storage, gateway-URL scoping** (#40892) — Dashboard auth tokens are now stored in session-scoped browser storage (cleared on tab close) instead of persistent local storage. Tokens are scoped to the selected gateway URL, and the bootstrap flow uses fragment-only navigation to avoid leaking tokens in the URL history.

### macOS/iOS App

- **macOS/chat UI — chat model picker and persistent thinking-level selections** (#42314) — The macOS chat UI adds a model picker for selecting the active chat model. Explicit thinking-level selections are persisted across relaunches. Provider-aware session model synchronization is hardened to prevent stale model state after provider switches.

- **macOS/onboarding remote — detect shared auth token requirement** (#43100) — The macOS remote-gateway onboarding flow now detects when the selected remote gateway requires a shared auth token and explains where to find it, reducing setup confusion for remote-mode users.

- **iOS/Home canvas — bundled welcome screen with live agent overview** (#42456) — The iOS home canvas adds a bundled welcome screen that includes a live agent overview. Floating controls are replaced with a docked toolbar. The layout adapts to smaller phone screen sizes.

- **iOS/gateway foreground recovery — reconnect immediately on foreground return** (#41384) — The iOS app now tears down stale background sockets and reconnects to the gateway immediately when the app returns to the foreground, rather than waiting for the next scheduled reconnect attempt.

- **iOS/TestFlight — local beta release flow with Fastlane** (#42991) — A local beta release flow backed by Fastlane is added for TestFlight distribution, enabling automated build and upload without requiring manual Xcode Organizer steps.

### Browser/Browserbase

- **Browser/Browserbase 429 — stable no-retry rate-limit guidance** (#40491) — When Browserbase returns HTTP 429 (rate limit), the error surface now provides stable no-retry guidance without buffering or discarding the response body. Previously the 429 body was consumed and discarded before the error was surfaced, making the guidance message less informative.

### Build/CI

- **CI/CodeQL Swift — select Xcode 26.1 before Swift build tools** (#41787) — The CodeQL Swift CI job now selects Xcode 26.1 before installing Swift build tools, ensuring the job uses Swift tools 6.2 rather than the runner's default Xcode version.

## v2026.3.12 Changes <!-- v2026.3.12 -->

### Auto-Reply <!-- v2026.3.12 -->

- **Auto-reply/fast mode — `params.fastMode` resolution:** `src/agents/fast-mode.ts` introduces `resolveFastMode()`, which checks `extraParams.fastMode` / `extraParams.fast_mode`, then `modelConfig.params.fastMode` / `modelConfig.params.fast_mode`, then the `SessionEntry.fastMode` field. The embedded runner (`src/agents/pi-embedded-runner/openai-stream-wrappers.ts`) reads `extraParams.fastMode` the same way. This makes per-model config defaults and per-session overrides consistent across the agent pipeline.

- **Auto-reply/commands — require owner for `/config` and `/debug`:** `src/auto-reply/commands-registry.data.ts` marks `/config` and `/debug` as requiring owner-level sender authorization. `src/auto-reply/command-auth.ts` enforces the `requireOwner` flag during command dispatch. These commands no longer appear in `/status` output for non-owner senders (fixes #44305).

- **Auto-reply/status resolution — context window by provider-qualified key:** `src/agents/context-window-guard.ts` (`resolveContextWindowInfo`) and related compaction paths now resolve context window tokens by provider-qualified model key first, falling back to the maximum among bare-id matches on collision (fixes #36389).

### General Utilities <!-- v2026.3.12 -->

- **Infra/host env security — block `GIT_EXEC_PATH` in sanitized exec environments:** `src/infra/host-env-security-policy.json` adds `GIT_EXEC_PATH` to the blocked env var list for sanitized host exec contexts. Prevents PATH/executable injection via the Git exec helper path (fixes #43685).

- **Infra/device scope — cap device tokens to approved scope baseline:** device token scopes are capped to an approved baseline during pairing and auth, limiting the authority of self-issued tokens (fixes #43686).

- **Infra/exec allowlist — tighten glob matching, preserve POSIX case sensitivity:** exec allowlist glob pattern matching is tightened; POSIX case sensitivity is preserved so case-sensitive platforms are not incorrectly permissive (fixes #43798).

### Shared <!-- v2026.3.12 -->

- **Config/Anthropic alias normalization at load time:** `src/config/defaults.ts` normalizes Anthropic model aliases inline during config load. Stale model refs that previously caused a startup crash-loop are now recovered at load time (fixes #45520).

- **Config/models SecretRef enforcement:** generated `models.json` entries enforce `SecretRef` markers for source-managed secrets (fixes #43759).

### macOS/iOS <!-- v2026.3.12 -->

- **macOS/launchd — daemon install avoids false-fails on slower Macs and fresh VMs:** the `openclaw onboard --install-daemon` path includes hardening against launchd bootstrap race conditions on slower hardware and fresh VM snapshots.

## v2026.3.13 Changes <!-- v2026.3.13 -->

### Auto-Reply <!-- v2026.3.13 -->

- **Agents/memory bootstrap — `MEMORY.md` preferred over `memory.md`:** `src/memory-host-sdk/host/internal.ts` tries `MEMORY.md` first when scanning for the root memory file, falling back to `memory.md` only when the uppercase variant is absent. `extensions/memory-core/src/memory/manager-sync-ops.ts` follows the same preference order during sync/index operations (fixes #26054).

### General Utilities <!-- v2026.3.13 -->

- **Build/plugin-sdk bundling — single-pass bundling prevents memory blow-up:** the build pipeline bundles all plugin-sdk subpath entries in a single pass rather than sequentially, avoiding OOM conditions in large plugin graphs (fixes #45426).

- **Config/validation — `channels.signal.groups` and `discovery.wideArea.domain`:** both keys are confirmed accepted in strict config validation as of v2026.3.13. `channels.signal.groups` configures Signal group memberships. `discovery.wideArea.domain` sets the DNS-SD wide-area discovery domain.

### macOS/iOS <!-- v2026.3.13 -->

- **macOS/onboarding — avoid self-restarting freshly bootstrapped launchd gateways:** the onboarding flow no longer triggers a gateway self-restart immediately after a fresh launchd bootstrap, preventing a double-restart loop on new installs.

- **macOS/runtime locator — Node >=22.16.0 enforced during system discovery:** `src/daemon/runtime-paths.ts` rejects system Node versions below 22.16.0 during macOS runtime locator scans.

## v2026.3.24 Delta Notes

### General Utilities

- **Cooldown per-model scoping:** `src/agents/auth-profiles/usage.ts` now scopes cooldown tracking per-model rather than per-profile, preventing a rate limit on one model from unnecessarily cooling down other models on the same auth profile.

- **Embedded transport error classification improvements:** embedded transport error classification is improved to better distinguish transient from permanent errors, reducing false-positive cooldown escalation.

- **`src/process/supervisor/` new subsystem:** a new process supervisor subsystem has been added under `src/process/supervisor/`, providing structured process lifecycle management for child processes.

---

## v2026.3.28 Delta Notes <!-- v2026.3.28 -->

### Auto-Reply <!-- v2026.3.28 -->

- **Suppress JSON-wrapped `NO_REPLY` control envelopes before channel delivery** (PR #56612) — `src/auto-reply/tokens.ts` adds `isSilentReplyEnvelopeText()` which uses a strict single-key detector to recognize JSON objects of the form `{"action":"NO_REPLY"}` as silent control envelopes and suppress them before they reach any channel. Only exact single-key envelopes with `action` equal to the silent token are matched; objects with extra keys are passed through unchanged. `isSilentReplyPayloadText()` now delegates to both `isSilentReplyText()` and `isSilentReplyEnvelopeText()`. When the text is only a silent envelope but the payload carries media, the media is preserved and an empty text is used instead.

### CLI/Update Status <!-- v2026.3.28 -->

- **Explicit "up to date" when local version matches npm latest** (#51409) — `src/commands/status.update.ts` `formatUpdateOneLiner()` now explicitly appends `"up to date"` to the update summary line when the local version matches npm latest (comparison result `=== 0`) and the install kind is not `"git"`. Previously no affirmative confirmation was printed in the already-current state; the install is now labeled clearly as up to date alongside the matching npm version.

### Daemon/Linux <!-- v2026.3.28 -->

- **Stop flagging non-gateway systemd services as duplicate gateways** (PR #45328) — `src/daemon/inspect.ts` `detectMarkerLineWithGateway()` only flags a systemd unit file as a duplicate gateway when a line in that file contains both the word `"gateway"` AND one of the known OpenClaw marker strings (`openclaw`, `clawdbot`, `moltbot`). Unit files that mention an OpenClaw marker elsewhere but lack any gateway reference line are no longer surfaced as duplicate gateway warnings in `openclaw doctor` output or diagnostic logs.

### MCP/Channels Bridge <!-- v2026.3.28 -->

- **Gateway-backed channel MCP bridge** — `src/mcp/` adds a new `OpenClawChannelBridge` class (`channel-bridge.ts`) backed by a `GatewayClient` connection, exposing conversation tools to Codex/Claude-facing MCP consumers as a new support surface. Tools registered via `channel-tools.ts` include `conversations_list`, `conversation_get`, `messages_read`, `attachments_fetch`, and send/wait primitives. `channel-server.ts` (`createOpenClawChannelMcpServer()`, `serveOpenClawChannelMcp()`) wires the bridge to a stdio `StdioServerTransport` with safe lifecycle handling for SIGINT/SIGTERM, stdin end/close, and transport `onclose` events — preventing ghost connections on reconnects and ensuring routed session discovery is cleaned up on shutdown.

### BlueBubbles/Groups <!-- v2026.3.28 -->

- **Optionally enrich unnamed participants with local macOS Contacts names** — `extensions/bluebubbles/src/participant-contact-names.ts` adds `enrichBlueBubblesParticipantsWithContactNames()`, which queries the local macOS Contacts SQLite databases (across multiple database paths under `~/Library/Application Support/AddressBook/`) to resolve display names for participants whose `name` field is blank. Lookups are cached with a 1-hour positive TTL and 5-minute negative TTL. Name enrichment only runs after group gating passes, so unnamed participant lists in group context can show resolved contact names rather than only raw phone numbers. The feature is macOS-only and the enrichment is opt-in.

### Agents/Errors <!-- v2026.3.28 -->

- **Surface provider quota/reset details when available** (PR #54512) — `src/agents/pi-embedded-helpers.ts` `formatAssistantErrorText()` now extracts and surfaces provider-specific quota exhaustion messages (including reset-time hints) when the error body is non-HTML plain text. HTML responses (e.g. Cloudflare or CDN rate-limit pages) continue to fall back to the generic `"API rate limit reached. Please try again later."` copy rather than leaking raw HTML to users.

- **Distinguish common network failures from true timeouts** (PR #51419) — `src/agents/failover-error.ts` `resolveFailoverReasonFromError()` maps `ECONNREFUSED`, `ENOTFOUND`, `ECONNRESET`, `EPIPE`, and related POSIX network codes to the `"timeout"` failover reason, producing the user-facing message `"LLM request failed: connection refused by the provider endpoint."` (or equivalent) for connection-refused and DNS-lookup failures rather than a generic timeout message. This improves lifecycle diagnostic clarity — `embedded_run_agent_end` events distinguish common network unreachability from true per-request timeout exhaustion.

### Feishu <!-- v2026.3.28 -->

- **Close WebSocket connections on monitor stop/abort** (#52844) — `extensions/feishu/src/monitor.transport.ts` `monitorWebSocket()` calls `wsClient.close()` inside its `cleanup()` function when the abort signal fires or the monitor is stopped, then removes the client from the `wsClients` map. `monitor.state.ts` `stopFeishuMonitorState()` also calls `closeWsClient()` on the stored client for a given account ID before deleting it. This prevents ghost WebSocket connections and eliminates duplicate event processing from stale open sockets after monitor stop/restart.

- **Use original message `create_time` instead of `Date.now()` for inbound timestamps** (#52809) — `extensions/feishu/src/bot.ts` `handleFeishuMessage()` now parses `event.message.create_time` (a millisecond epoch string) into `messageCreateTimeMs` early in message handling, falling back to `Date.now()` only when the field is absent. All downstream consumers (pending history entries, inbound payload construction, envelope timestamps) use this original authoring timestamp instead of the delivery/processing time, ensuring Feishu message timestamps reflect when the message was authored rather than when OpenClaw received it.

## v2026.3.31 Delta Notes

### Support Surfaces

- **Task summaries and maintenance are now foundational support utilities:** the stable line adds durable task-registry support code for status summaries, audit, maintenance, and detached-run cleanup.
- **TTS diagnostics are materially richer:** fallback attempts now emit structured diagnostics and analytics, which changes how operators and docs should reason about speech regressions.

### Operator Reliability

- **Wizard safety improved on remote-gateway decline:** onboarding support paths now reset back to the safe loopback default after rejecting a discovered remote URL.

## v2026.4.9 Delta Notes

### Auto-reply / Reply Delivery

- **Commentary is buffered until `final_answer` on OpenAI/GPT paths:** utility/support code that formats previews, block replies, session summaries, or channel partials can no longer assume commentary text is immediately user-visible.
- **Planning-only retries are part of the runtime contract:** support modules around reply orchestration and run accounting now have to tolerate one-shot retries when the model narrates a plan without acting.

### Subagent / Completion Delivery

- **Completion delivery context is merged more defensively:** current release-line fixes merge completion announce delivery context with requester-session fallback so missing `to` still resolves to the original channel more reliably.

### Plugin / Runtime Support

- **Activation provenance and registry reuse remain core support behavior:** activation-source metadata and compatible-registry reuse are still important, but `v2026.4.9` extends the support surface with embedded ACPX runtime ownership and generic reply-dispatch interception.

## Changes in v2026.4.15–v2026.4.21

### Plugins / Packaging (v2026.4.15)

- **Bundled plugin runtime deps localized to owning extensions** (#67099) — Plugin bundling now places runtime dependencies adjacent to the extension that declares them rather than hoisting them to a shared location. Install guardrails are tightened so plugins cannot accidentally consume peer dependencies from unrelated extensions.

### Plugins / Tests (v2026.4.20)

- **Plugin loader alias and Jiti config reused across repeated same-context loads** (#69316) — The plugin loader now reuses the alias map and Jiti resolution config when the same plugin context is loaded more than once within a session, rather than recreating them on every call. Reduces overhead for hot-reload and repeated plugin instantiation during tests.

### Plugins / Tasks (v2026.4.20)

- **Detached runtime registration contract for plugin executors** (#68915) — The plugin task runtime introduces a formal registration contract that allows plugin executors to own the full lifecycle and cancellation of detached tasks. Previously, detached task management was handled centrally; executors can now register, monitor, and cancel their own detached runs independently.

### Auto-Reply (v2026.4.20)

- **BlueBubbles per-group `systemPrompt` forwarding** — `extensions/bluebubbles/src/monitor.ts` threads per-group `systemPrompt` values from the BlueBubbles channel config into the inbound message context (`ctx`) for group messages. A wildcard `"*"` entry serves as the fallback when no exact group handle matches. Verified by `monitor.test.ts`: `"threads per-group systemPrompt into ctx for group messages"` and `"falls back to the '*' wildcard systemPrompt when no exact group match"`.

### Plugins / Skill Workshop (v2026.4.21)

- **Skill Workshop plugin added** (`extensions/skill-workshop/`) — New plugin providing threshold-based skill capture from agent sessions. Key files: `index.ts`, `api.ts`, `src/`. Features: configurable capture thresholds to decide which agent outputs qualify as skill candidates; reviewer passes to evaluate and refine proposals; quarantine of unsafe or low-quality proposals before they are promoted to the skill registry. Registered via `openclaw.plugin.json`.
