# OpenClaw Channels & Messaging — Comprehensive Analysis

> Generated: 2026-02-15 | Cluster: CHANNELS & MESSAGING
> Modules analyzed: `src/telegram` (100 files), `src/discord` (77 files), `src/signal` (32 files), `src/slack` (67 files), `src/whatsapp` (3 files), `src/imessage` (20 files), `src/line` (46 files), `src/channels` (106 files)

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [src/channels — Shared Channel Infrastructure](#srcchannels)
3. [src/telegram — Telegram Bot API](#srctelegram)
4. [src/discord — Discord Bot (Carbon/Gateway)](#srcdiscord)
5. [src/signal — Signal via signal-cli JSON-RPC](#srcsignal)
6. [src/slack — Slack (Bolt Socket Mode + HTTP)](#srcslack)
7. [src/whatsapp — WhatsApp Web (Baileys)](#srcwhatsapp)
8. [src/imessage — iMessage (imsg RPC)](#srcimessage)
9. [src/line — LINE Messaging API](#srcline)
10. [Cross-Channel Data Flow](#cross-channel-data-flow)

---

## Architecture Overview

All channels follow a **plugin architecture** defined by the `ChannelPlugin` interface in `src/channels/plugins/types.plugin.ts`. Each channel implements adapters for:

- **Config** — account resolution, listing, enable/disable
- **Security** — DM policy, allowlists
- **Outbound** — sending messages, polls, media
- **Status** — probing, auditing, health snapshots
- **Gateway** — starting/stopping real-time listeners
- **Actions** — agent tool message actions (send, react, edit, etc.)
- **Onboarding** — CLI wizard for channel setup
- **Threading** — reply-to-mode, thread context for tools
- **Mentions** — mention stripping patterns
- **Groups** — requireMention, tool policy per group
- **Directory** — listing peers/groups from live APIs

**Message flow (inbound):**
1. Channel-specific monitor receives raw event (webhook, SSE, WebSocket, polling)
2. Event is normalized into an `MsgContext` (sender, text, media, chat type, thread info)
3. Allowlist/mention gating decides whether to process
4. `dispatchInboundMessage()` routes to the correct agent session
5. Agent generates reply → delivered back through channel-specific send functions

**Message flow (outbound via tools):**
1. Agent tool calls `message` action with channel/target/text
2. `ChannelMessageActionAdapter.handleAction()` dispatches to channel-specific send
3. Channel send function formats (markdown→HTML/mrkdwn/styled text), chunks, and delivers

---

## src/channels — Shared Channel Infrastructure {#srcchannels}

### Module Overview
The shared channel framework that all channel implementations depend on. Contains the plugin registry, dock system (lightweight channel metadata), and cross-cutting concerns like typing indicators, ack reactions, mention gating, and session recording.

**Architecture:** Plugin registry pattern with "docks" (lightweight proxies) for shared code paths that shouldn't import heavy channel implementations.

### File Inventory

| File | Description |
|------|-------------|
| `registry.ts` | Core channel ID registry, meta definitions, normalization, `CHAT_CHANNEL_ORDER` |
| `dock.ts` | Lightweight channel behavior proxies (capabilities, threading, mentions) for shared code |
| `session.ts` | `recordInboundSession()` — persists session metadata on inbound messages |
| `ack-reactions.ts` | Ack reaction gating (scope: all/direct/group-all/group-mentions/off) |
| `allowlist-match.ts` | `AllowlistMatch` type and `formatAllowlistMatchMeta()` |
| `channel-config.ts` | Per-channel/group config resolution with key candidate matching |
| `chat-type.ts` | `ChatType` enum (`direct`/`group`/`channel`) + normalization |
| `command-gating.ts` | Command authorization from access groups |
| `conversation-label.ts` | Human-readable conversation label builder |
| `location.ts` | `NormalizedLocation` type + `formatLocationText()` |
| `logging.ts` | Inbound drop/typing/ack failure log helpers |
| `mention-gating.ts` | Mention gating logic (requireMention + bypass) |
| `reply-prefix.ts` | Response prefix context (model name, agent name) |
| `sender-identity.ts` | Sender identity validation |
| `sender-label.ts` | `resolveSenderLabel()` from name/username/tag/e164/id |
| `targets.ts` | `MessagingTarget` type, `buildMessagingTarget()`, target parsing |
| `typing.ts` | `createTypingCallbacks()` — typing indicator lifecycle |
| `account-summary.ts` | `buildChannelAccountSnapshot()` for status display |
| `web/index.ts` | Re-exports WhatsApp Web functions from `channel-web.js` |

#### plugins/ subdirectory

| File | Description |
|------|-------------|
| `types.core.ts` | Core types: `ChannelId`, `ChannelCapabilities`, `ChannelAccountSnapshot`, `ChannelGroupContext`, etc. |
| `types.adapters.ts` | Adapter interfaces: `ChannelConfigAdapter`, `ChannelOutboundAdapter`, `ChannelGatewayAdapter`, etc. |
| `types.plugin.ts` | `ChannelPlugin` interface — the main plugin contract |
| `types.ts` | Re-exports all types from core/adapters/plugin |
| `index.ts` | `listChannelPlugins()`, `getChannelPlugin()`, `normalizeChannelId()` |
| `catalog.ts` | Plugin discovery and UI metadata |
| `load.ts` | Lazy-cached plugin loading from registry |
| `helpers.ts` | `resolveChannelDefaultAccountId()` |
| `setup-helpers.ts` | Account setup config merge helpers |
| `config-helpers.ts` | `setAccountEnabledInConfigSection()` |
| `config-schema.ts` | `buildChannelConfigSchema()` from Zod schemas |
| `config-writes.ts` | `resolveChannelConfigWrites()` — runtime config mutation gating |
| `directory-config.ts` | Config-based directory entries (peers/groups from config, not live API) |
| `group-mentions.ts` | Per-channel group `requireMention` and `toolPolicy` resolvers |
| `media-limits.ts` | `resolveChannelMediaMaxBytes()` |
| `message-action-names.ts` | `CHANNEL_MESSAGE_ACTION_NAMES` constant array |
| `message-actions.ts` | `listChannelMessageActions()`, `handleChannelMessageAction()` |
| `pairing.ts` | `listPairingChannels()`, `resolveChannelPairing()` |
| `pairing-message.ts` | `PAIRING_APPROVED_MESSAGE` constant |
| `status.ts` | `buildChannelAccountSnapshot()` via plugin hooks |
| `whatsapp-heartbeat.ts` | WhatsApp heartbeat recipient resolution |
| `channel-config.ts` | Channel-level config resolution with fallback matching |
| `allowlist-match.ts` | Plugin-level allowlist matching |
| `bluebubbles-actions.ts` | BlueBubbles (iMessage) action specs |
| `slack.actions.ts` | Slack message action adapter factory |
| `agent-tools/whatsapp-login.ts` | WhatsApp QR login agent tool |

#### plugins/normalize/ — Target normalization per channel

| File | Description |
|------|-------------|
| `telegram.ts` | `normalizeTelegramMessagingTarget()` — strips `telegram:`/`tg:` prefixes, resolves t.me links |
| `discord.ts` | `normalizeDiscordMessagingTarget()`, `looksLikeDiscordTargetId()` |
| `signal.ts` | `normalizeSignalMessagingTarget()` — strips `signal:` prefix, handles group: |
| `slack.ts` | `normalizeSlackMessagingTarget()`, `looksLikeSlackTargetId()` |
| `whatsapp.ts` | `normalizeWhatsAppMessagingTarget()`, `looksLikeWhatsAppTargetId()` |
| `imessage.ts` | `normalizeIMessageMessagingTarget()` — preserves service prefixes |

#### plugins/onboarding/ — CLI wizard adapters

| File | Description |
|------|-------------|
| `helpers.ts` | `promptAccountId()`, `addWildcardAllowFrom()` |
| `channel-access.ts` | `ChannelAccessPolicy`, `parseAllowlistEntries()` |
| `telegram.ts` | Telegram onboarding wizard (BotFather token, allowFrom) |
| `discord.ts` | Discord onboarding wizard (bot token, intents, guilds) |
| `signal.ts` | Signal onboarding wizard (signal-cli path, daemon setup) |
| `slack.ts` | Slack onboarding wizard (bot/app tokens) |
| `whatsapp.ts` | WhatsApp onboarding wizard (QR link) |
| `imessage.ts` | iMessage onboarding wizard (imsg binary, DB path) |

#### plugins/outbound/ — Channel send adapters

| File | Description |
|------|-------------|
| `load.ts` | Lazy-cached outbound adapter loading |
| `telegram.ts` | `telegramOutbound` — HTML chunks via `sendMessageTelegram()` |
| `discord.ts` | `discordOutbound` — 2000 char limit, null chunker (custom chunking) |
| `signal.ts` | `signalOutbound` — plain text chunks via `sendMessageSignal()` |
| `slack.ts` | `slackOutbound` — mrkdwn via `sendMessageSlack()` with identity |
| `whatsapp.ts` | `whatsappOutbound` — gateway delivery mode, text chunks |
| `imessage.ts` | `imessageOutbound` — plain text chunks via `sendMessageIMessage()` |

#### plugins/actions/ — Agent tool action handlers

| File | Description |
|------|-------------|
| `telegram.ts` | Telegram actions: send, react, edit, delete, forward, pin, poll, buttons |
| `discord.ts` | Discord actions adapter (delegates to `discord/handle-action.ts`) |
| `discord/handle-action.ts` | Discord action dispatcher (send, react, edit, delete, threads, etc.) |
| `discord/handle-action.guild-admin.ts` | Discord guild admin actions (ban, kick, timeout, roles, events) |
| `signal.ts` | Signal actions: send, react, removeReaction |

#### plugins/status-issues/ — Health check diagnostics

| File | Description |
|------|-------------|
| `shared.ts` | Shared status issue helpers |
| `telegram.ts` | Telegram status issues (unmentioned groups audit) |
| `discord.ts` | Discord status issues (intent checks) |
| `whatsapp.ts` | WhatsApp status issues (connection, QR) |
| `bluebubbles.ts` | BlueBubbles (iMessage) status issues |

### Key Types & Interfaces

```typescript
// The main plugin contract every channel must implement
type ChannelPlugin<ResolvedAccount, Probe, Audit> = {
  id: ChannelId;
  meta: ChannelMeta;
  capabilities: ChannelCapabilities;
  config: ChannelConfigAdapter<ResolvedAccount>;
  security?: ChannelSecurityAdapter<ResolvedAccount>;
  outbound?: ChannelOutboundAdapter;
  status?: ChannelStatusAdapter<ResolvedAccount, Probe, Audit>;
  gateway?: ChannelGatewayAdapter<ResolvedAccount>;
  actions?: ChannelMessageActionAdapter;
  onboarding?: ChannelOnboardingAdapter;
  threading?: ChannelThreadingAdapter;
  mentions?: ChannelMentionAdapter;
  groups?: ChannelGroupAdapter;
  directory?: ChannelDirectoryAdapter;
  // ... more optional adapters
};

type ChannelCapabilities = {
  chatTypes: Array<ChatType | "thread">;
  polls?: boolean; reactions?: boolean; edit?: boolean;
  unsend?: boolean; reply?: boolean; effects?: boolean;
  threads?: boolean; media?: boolean; nativeCommands?: boolean;
  blockStreaming?: boolean;
};

type ChannelDock = { /* lightweight subset of ChannelPlugin for shared code */ };

type ChatChannelId = "telegram" | "whatsapp" | "discord" | "irc" | "googlechat" | "slack" | "signal" | "imessage";
```

### Internal Dependencies
- `src/config/` — config loading, session store, group policy
- `src/auto-reply/` — reply flow, command detection, mentions, envelope formatting
- `src/routing/` — session key resolution, agent routing
- `src/infra/` — retry, backoff, errors, system events
- `src/plugins/` — plugin registry runtime
- `src/pairing/` — pairing store and messages
- Individual channel modules (telegram, discord, etc.) — for account resolution in dock.ts

### External Dependencies
- `@sinclair/typebox` — JSON Schema types for agent tools
- `@mariozechner/pi-agent-core` — `AgentTool`, `AgentToolResult`
- `zod` — config schema validation

### Test Files
- `ack-reactions.test.ts` — ack reaction scope gating
- `channel-config.test.ts` — channel config key matching
- `chat-type.test.ts` — chat type normalization
- `command-gating.test.ts` — command authorization
- `conversation-label.test.ts` — conversation label formatting
- `location.test.ts` — location text formatting
- `mention-gating.test.ts` — mention bypass logic
- `registry.test.ts` — channel ID normalization
- `sender-identity.test.ts` — sender identity validation
- `targets.test.ts` — target parsing
- `typing.test.ts` — typing callback lifecycle
- `web/index.test.ts` — web channel re-exports
- `plugins/catalog.test.ts` — plugin catalog discovery
- `plugins/config-writes.test.ts` — config write gating
- `plugins/directory-config.test.ts` — config-based directory
- `plugins/index.test.ts` — plugin listing/lookup
- `plugins/load.test.ts` — plugin lazy loading
- `plugins/actions/telegram.test.ts` — Telegram action handling
- `plugins/actions/discord.test.ts`, `plugins/actions/discord/handle-action.test.ts` — Discord actions
- `plugins/actions/signal.test.ts` — Signal actions
- `plugins/slack.actions.test.ts` — Slack actions
- `plugins/normalize/imessage.test.ts`, `plugins/normalize/signal.test.ts` — target normalization
- `plugins/onboarding/signal.test.ts` — Signal onboarding
- `plugins/outbound/telegram.test.ts`, `plugins/outbound/slack.test.ts`, `plugins/outbound/whatsapp.test.ts` — outbound delivery

---

## src/telegram — Telegram Bot API {#srctelegram}

### Module Overview
Full-featured Telegram Bot API integration using **grammY** framework. Supports long polling and webhook modes, forum topics (threads), inline buttons, sticker understanding via vision, voice messages, draft streaming (live-edit messages as AI generates), media groups, native `/commands`, and reactions.

**Entry points:** `createTelegramBot()`, `monitorTelegramProvider()`, `sendMessageTelegram()`, `startTelegramWebhook()`

### File Inventory

| File | Description |
|------|-------------|
| `index.ts` | Re-exports: `createTelegramBot`, `monitorTelegramProvider`, `sendMessageTelegram`, `reactMessageTelegram`, `sendPollTelegram`, `startTelegramWebhook` |
| `bot.ts` | `createTelegramBot()` — creates grammY Bot instance with middleware (throttler, sequentializer, dedup) |
| `bot-handlers.ts` | `registerTelegramHandlers()` — registers message/callback_query/reaction handlers on Bot |
| `bot-message.ts` | `createTelegramMessageProcessor()` — factory for message processing pipeline |
| `bot-message-context.ts` | `buildTelegramMessageContext()` — normalizes grammY context → `MsgContext` |
| `bot-message-dispatch.ts` | `dispatchTelegramMessage()` — sends to agent, delivers reply chunks |
| `bot-native-commands.ts` | `registerTelegramNativeCommands()` — registers `/command` handlers |
| `bot-native-command-menu.ts` | `buildPluginTelegramMenuCommands()` — syncs command menu with BotFather |
| `bot-updates.ts` | Update deduplication, media group aggregation (`MEDIA_GROUP_TIMEOUT_MS = 500`) |
| `bot-access.ts` | Allowlist normalization and matching for sender authorization |
| `bot/delivery.ts` | `deliverReplies()` — sends reply payloads (text/media/sticker/voice) via grammY |
| `bot/helpers.ts` | Forum thread resolution, group peer ID building, stream mode |
| `bot/types.ts` | `TelegramContext`, `TelegramStreamMode`, `StickerMetadata` |
| `monitor.ts` | `monitorTelegramProvider()` — long-polling runner with backoff/reconnect |
| `send.ts` | `sendMessageTelegram()`, `reactMessageTelegram()`, `sendPollTelegram()`, `editMessageTelegram()` — outbound API |
| `format.ts` | `renderTelegramHtmlText()`, `markdownToTelegramHtml()` — Markdown→Telegram HTML conversion |
| `accounts.ts` | `resolveTelegramAccount()`, `listTelegramAccountIds()` |
| `token.ts` | `resolveTelegramToken()` — env/tokenFile/config resolution |
| `probe.ts` | `probeTelegram()` — getMe + getWebhookInfo health check |
| `audit.ts` | `auditTelegramGroupMembership()` — verifies bot membership in configured groups |
| `targets.ts` | `parseTelegramTarget()` — chatId + optional messageThreadId |
| `caption.ts` | `splitTelegramCaption()` — 1024 char caption limit |
| `download.ts` | `getTelegramFile()`, `downloadTelegramFile()`, `downloadTelegramMedia()` |
| `voice.ts` | `resolveTelegramVoiceSend()` — voice bubble compatibility check |
| `sticker-cache.ts` | `cacheSticker()`, `describeStickerImage()` — vision-based sticker descriptions |
| `sent-message-cache.ts` | In-memory cache of bot's own message IDs (24h TTL) |
| `update-offset-store.ts` | Persistent update offset for long polling |
| `draft-stream.ts` | `createTelegramDraftStream()` — live message editing during generation |
| `draft-chunking.ts` | Draft stream chunk size resolution |
| `inline-buttons.ts` | `isTelegramInlineButtonsEnabled()` — scope-based inline button gating |
| `model-buttons.ts` | Model selection inline keyboard builder |
| `reaction-level.ts` | `resolveTelegramReactionLevel()` — off/ack/minimal/extensive |
| `outbound-params.ts` | `parseTelegramReplyToMessageId()`, `parseTelegramThreadId()` |
| `network-errors.ts` | `isRecoverableTelegramNetworkError()` — transient error detection |
| `network-config.ts` | `resolveTelegramAutoSelectFamilyDecision()` — Node.js Happy Eyeballs workaround |
| `proxy.ts` | `makeProxyFetch()` — HTTPS proxy via undici |
| `fetch.ts` | `resolveTelegramFetch()` — fetch with network workarounds |
| `api-logging.ts` | `withTelegramApiErrorLogging()` — error wrapper |
| `allowed-updates.ts` | `resolveTelegramAllowedUpdates()` — includes message_reaction |
| `webhook.ts` | `startTelegramWebhook()` — HTTP server for webhook mode |
| `webhook-set.ts` | `setTelegramWebhook()`, `deleteTelegramWebhook()` |
| `group-migration.ts` | Telegram group→supergroup migration config handling |

### Key Types

```typescript
type TelegramBotOptions = {
  token: string; accountId?: string; runtime?: RuntimeEnv;
  requireMention?: boolean; allowFrom?: Array<string | number>;
  replyToMode?: ReplyToMode; proxyFetch?: typeof fetch; config?: OpenClawConfig;
};

type ResolvedTelegramAccount = {
  accountId: string; enabled: boolean; token: string;
  tokenSource: "env" | "tokenFile" | "config" | "none";
  config: TelegramAccountConfig;
};

type TelegramContext = { message: Message; me?: UserFromGetMe; getFile: () => Promise<{file_path?: string}> };
type TelegramStreamMode = "off" | "partial" | "block";
type TelegramDraftStream = { update: (text: string) => void; flush: () => Promise<void>; stop: () => void };
type TelegramProbe = { ok: boolean; bot?: { id?; username?; canReadAllGroupMessages? }; webhook?: { url? } };
```

### Data Flow
1. **Inbound (long-polling):** grammY `run()` → `getUpdates` loop → `bot.on("message")` → `registerTelegramHandlers()` → media group aggregation → `buildTelegramMessageContext()` → allowlist check → mention gating → `dispatchInboundMessage()` → agent reply → `deliverReplies()` (with draft streaming if enabled)
2. **Inbound (webhook):** HTTP POST → `webhookCallback()` → same handler chain
3. **Outbound (tool):** `sendMessageTelegram()` → `renderTelegramHtmlText()` → `bot.api.sendMessage()` with HTML parse mode

### Configuration Keys
- `channels.telegram.accounts.<id>.{botToken, tokenFile, allowFrom, groupAllowFrom, groups, replyToMode, reactionLevel, draftChunk, mediaMaxMb, network, inlineButtonsScope}`
- `channels.telegram.{token, tokenFile, allowFrom, replyToMode, reactionLevel}`
- Env: `TELEGRAM_BOT_TOKEN`, `OPENCLAW_TELEGRAM_PROXY`, `OPENCLAW_TELEGRAM_DISABLE_AUTO_SELECT_FAMILY`

### Channel-Specific Features
- **Forum topics:** Full thread support via `message_thread_id`
- **Draft streaming:** Live-edits message as AI generates text
- **Inline buttons:** Model selection, pagination, custom callback keyboards
- **Native commands:** `/command` registration with BotFather menu sync
- **Sticker understanding:** Vision model describes stickers, cached
- **Reactions:** Emoji reactions on messages (ack + agent)
- **Media groups:** Aggregates multiple photos sent together
- **Voice messages:** Sends compatible audio as voice bubbles
- **SOCKS/HTTPS proxy:** Full proxy support via undici

### External Dependencies
- `grammy` — Telegram Bot framework
- `@grammyjs/runner` — Long-polling runner with concurrency
- `@grammyjs/transformer-throttler` — API rate limiting
- `@grammyjs/types` — Telegram API types
- `undici` — HTTP client for proxy support

### Test Coverage (28 test files)
Tests cover: bot creation, message context building, dispatch, native commands, delivery, format/HTML, draft streaming, chunking, download, send (polls, edit, video-note, caption-split, proxy), inline buttons, model buttons, sticker cache, sent message cache, accounts, token resolution, targets, reaction level, webhook, probe, audit, network config/errors, group migration, update offset store, allowlist matching, and numerous e2e integration tests.

---

## src/discord — Discord Bot (Carbon/Gateway) {#srcdiscord}

### Module Overview
Discord integration using **@buape/carbon** (REST + Gateway) with the discord.js-compatible `discord-api-types`. Supports guilds, channels, threads, DMs, group DMs, reactions, polls, voice messages, PluralKit integration, presence/status, slash commands, exec approvals via buttons, and rich permission management.

**Entry points:** `monitorDiscordProvider()`, `sendMessageDiscord()`, `sendPollDiscord()`

### File Inventory

| File | Description |
|------|-------------|
| `index.ts` | Re-exports: `monitorDiscordProvider`, `sendMessageDiscord`, `sendPollDiscord` |
| `accounts.ts` | `resolveDiscordAccount()`, `listDiscordAccountIds()` |
| `api.ts` | `fetchDiscord()` — raw REST helper with retry |
| `client.ts` | `createDiscordRestClient()`, `createDiscordClient()` — REST client factory |
| `token.ts` | `resolveDiscordToken()`, `normalizeDiscordToken()` |
| `probe.ts` | `probeDiscord()` — getMe + application info + privileged intents check |
| `audit.ts` | `auditDiscordChannelPermissions()` — verifies ViewChannel+SendMessages |
| `chunk.ts` | `chunkDiscordText()` — 2000 char + 17 line limit, fence-aware splitting |
| `guilds.ts` | `listGuilds()` |
| `targets.ts` | `parseDiscordTarget()`, `resolveDiscordTarget()` — user/channel routing |
| `resolve-channels.ts` | `resolveDiscordChannels()` — guild#channel lookup |
| `resolve-users.ts` | `resolveDiscordUsers()` — guild member search |
| `directory-live.ts` | Live directory (peers/groups) from Discord API |
| `pluralkit.ts` | `fetchPluralKitMessageInfo()` — PluralKit webhook message attribution |
| `voice-message.ts` | `sendDiscordVoiceMessage()` — OGG/Opus voice with waveform |
| `gateway-logging.ts` | Gateway event logging |
| `monitor.gateway.ts` | `DiscordGatewayHandle`, `waitForDiscordGatewayStop()` |
| `send.ts` | Re-exports all send functions from sub-modules |
| `send.shared.ts` | Shared send utilities, poll building, target resolution |
| `send.outbound.ts` | `sendMessageDiscord()` — main outbound send with chunking |
| `send.types.ts` | `DiscordSendResult`, `DiscordSendError`, action option types |
| `send.channels.ts` | Channel CRUD (create, edit, delete, move, permissions) |
| `send.guild.ts` | Guild admin (ban, kick, timeout, roles, events, voice status) |
| `send.messages.ts` | Message CRUD (read, create thread, edit, delete, pin, search) |
| `send.permissions.ts` | Permission calculation and checking |
| `send.reactions.ts` | Reaction CRUD |
| `send.emojis-stickers.ts` | Emoji/sticker upload |
| `send.voice-message.security.test.ts` | Voice message security tests |

#### monitor/ subdirectory

| File | Description |
|------|-------------|
| `provider.ts` | `monitorDiscordProvider()` — gateway startup, config resolution, listener registration |
| `message-handler.ts` | `createDiscordMessageHandler()` — debounce + dispatch |
| `message-handler.preflight.ts` | Preflight checks (allowlist, mention gating, command detection) |
| `message-handler.preflight.types.ts` | Preflight type definitions |
| `message-handler.process.ts` | `processDiscordMessage()` — context building, dispatch, reply delivery |
| `message-utils.ts` | `resolveDiscordMessageText()`, media downloading |
| `listeners.ts` | `registerDiscordListener()` — message/reaction/presence event handlers |
| `allow-list.ts` | `normalizeDiscordAllowList()`, guild entry resolution, channel config |
| `gateway-plugin.ts` | `resolveDiscordGatewayIntents()` — intent bitfield builder |
| `gateway-registry.ts` | Module-level gateway instance registry |
| `threading.ts` | `resolveDiscordReplyTarget()`, thread name sanitization |
| `presence.ts` | `resolveDiscordPresenceUpdate()` — custom status/activity |
| `presence-cache.ts` | In-memory presence cache (5000 per account cap) |
| `typing.ts` | `sendTyping()` via Carbon client |
| `reply-delivery.ts` | `deliverDiscordReply()` — chunked reply sending |
| `reply-context.ts` | `resolveReplyContext()` — referenced message resolution |
| `sender-identity.ts` | `resolveDiscordSenderIdentity()` — includes PluralKit |
| `system-events.ts` | `resolveDiscordSystemEvent()` — pin, join, boost, etc. |
| `native-command.ts` | `createDiscordNativeCommand()` — slash command builder |
| `format.ts` | `formatDiscordUserTag()`, `formatDiscordReactionEmoji()` |
| `exec-approvals.ts` | Exec approval buttons (approve/deny via Discord button interactions) |
| `agent-components.ts` | Agent interaction components (pairing buttons, select menus) |

### Key Types

```typescript
type ResolvedDiscordAccount = {
  accountId: string; enabled: boolean; token: string;
  tokenSource: "env" | "config" | "none"; config: DiscordAccountConfig;
};
type DiscordSendResult = { messageId: string; channelId: string };
type DiscordSendError = Error & { kind?: "missing-permissions" | "dm-blocked"; missingPermissions?: string[] };
type DiscordProbe = { ok: boolean; bot?: { id?; username? }; application?: DiscordApplicationSummary };
type DiscordGuildEntryResolved = { id?; slug?; requireMention?; reactionNotifications?; users?; roles? };
type DiscordSenderIdentity = { id; name?; tag?; label; isPluralKit; pluralkit?: {...} };
```

### Data Flow
1. **Inbound:** Carbon GatewayPlugin → `MessageCreateListener` → `createDiscordMessageHandler()` → preflight (allowlist, mention, command gating) → `processDiscordMessage()` → build MsgContext → `dispatchInboundMessage()` → agent → `deliverDiscordReply()`
2. **Outbound:** `sendMessageDiscord()` → resolve target (user→DM channel, channel→direct) → chunk text (2000 chars, 17 lines) → REST API calls

### Configuration Keys
- `channels.discord.accounts.<id>.{token, allowFrom, dm.allowFrom, guilds, replyToMode, activity, status, intents, reactionNotifications}`
- Env: `DISCORD_BOT_TOKEN`

### Channel-Specific Features
- **Threads:** Create/reply in threads, auto-thread creation
- **Slash commands:** Full Application Command support with autocomplete
- **Guild admin:** Ban, kick, timeout, role management, scheduled events
- **Exec approvals:** Approve/deny shell commands via button interactions
- **PluralKit:** Webhook message attribution for plural systems
- **Presence:** Custom status (playing, streaming, listening, watching, competing, custom)
- **Voice messages:** OGG/Opus with waveform visualization
- **Rich permissions:** Per-channel permission auditing
- **Polls:** Native Discord polls (up to 10 options, 32-day max)
- **Reactions:** Full reaction CRUD
- **Block streaming:** Coalesced streaming (1500 chars, 1s idle)

### External Dependencies
- `@buape/carbon` — Discord REST client + Gateway
- `@buape/carbon/gateway` — WebSocket gateway
- `discord-api-types` — Discord API type definitions
- `ws` — WebSocket (for gateway)
- `https-proxy-agent` — Proxy support

### Test Coverage (19 test files)
Tests cover: API fetch, audit, chunking, gateway logging/registry, monitor (message handling, allow-list, exec approvals, threading, presence, inbound contract), send (basic messages, thread creation, voice message security), targets, token, probe intents, PluralKit.

---

## src/signal — Signal via signal-cli JSON-RPC {#srcsignal}

### Module Overview
Signal integration via **signal-cli** JSON-RPC daemon over HTTP/SSE. Supports DMs, groups, reactions, media attachments, read receipts, typing indicators, and styled text (bold/italic/strikethrough/monospace/spoiler).

**Entry points:** `monitorSignalProvider()`, `sendMessageSignal()`, `sendReactionSignal()`, `probeSignal()`

### File Inventory

| File | Description |
|------|-------------|
| `index.ts` | Re-exports: `monitorSignalProvider`, `probeSignal`, `sendMessageSignal`, `sendReactionSignal`, `removeReactionSignal`, `resolveSignalReactionLevel` |
| `accounts.ts` | `resolveSignalAccount()`, `listSignalAccountIds()` |
| `client.ts` | `signalRpcRequest()`, `signalCheck()`, `streamSignalEvents()` — JSON-RPC + SSE client |
| `daemon.ts` | `spawnSignalDaemon()` — starts signal-cli daemon process |
| `format.ts` | `markdownToSignalText()` — Markdown→Signal styled text with style ranges |
| `identity.ts` | `resolveSignalSender()`, `isSignalSenderAllowed()` — phone/UUID parsing |
| `probe.ts` | `probeSignal()` — health check via version endpoint |
| `reaction-level.ts` | `resolveSignalReactionLevel()` |
| `rpc-context.ts` | `resolveSignalRpcContext()` — base URL + account resolution |
| `send.ts` | `sendMessageSignal()` — text + media + style ranges |
| `send-reactions.ts` | `sendReactionSignal()`, `removeReactionSignal()` |
| `sse-reconnect.ts` | `runSignalSseLoop()` — SSE reconnect with exponential backoff |
| `monitor.ts` | `monitorSignalProvider()` — daemon spawn + SSE loop + event handler |
| `monitor/event-handler.ts` | `createSignalEventHandler()` — main inbound message processing |
| `monitor/event-handler.types.ts` | `SignalEnvelope`, `SignalDataMessage`, `SignalMention`, etc. |
| `monitor/mentions.ts` | `renderSignalMentions()` — replaces Unicode object replacement chars with @names |

### Key Types

```typescript
type ResolvedSignalAccount = {
  accountId: string; enabled: boolean; baseUrl: string;
  configured: boolean; config: SignalAccountConfig;
};
type SignalSender = { kind: "phone"; raw: string; e164: string } | { kind: "uuid"; raw: string };
type SignalTextStyleRange = { start: number; length: number; style: "BOLD"|"ITALIC"|"STRIKETHROUGH"|"MONOSPACE"|"SPOILER" };
type SignalSendResult = { messageId: string; timestamp?: number };
type SignalProbe = { ok: boolean; version?: string; elapsedMs: number };
```

### Data Flow
1. **Inbound:** `monitorSignalProvider()` spawns signal-cli daemon → `runSignalSseLoop()` connects to SSE endpoint → events dispatched to `createSignalEventHandler()` → parse envelope → resolve sender (phone/UUID) → allowlist check → mention gating → `dispatchInboundMessage()` → agent → `deliverReplies()` via `sendMessageSignal()`
2. **Outbound:** `sendMessageSignal()` → `markdownToSignalText()` (produces text + style ranges) → JSON-RPC `send` call

### Configuration Keys
- `channels.signal.accounts.<id>.{baseUrl, account, allowFrom, reactionLevel}`
- `channels.signal.{cliPath, httpHost, httpPort}`
- Env: none (signal-cli configured via config)

### Channel-Specific Features
- **Styled text:** Bold, italic, strikethrough, monospace, spoiler via style ranges
- **Reactions:** Emoji reactions on specific messages
- **Read receipts:** Marks messages as read
- **UUID allowlists:** Supports both phone numbers and Signal UUIDs
- **Mention rendering:** Replaces Signal mention placeholders (U+FFFC) with @names
- **SSE reconnect:** Exponential backoff reconnection loop

### External Dependencies
- None (uses raw HTTP/SSE to talk to signal-cli REST API)

### Test Coverage (14 test files)
Tests cover: daemon log classification, format (chunking, links, visual), probe, monitor (event handler inbound contract, mention gating, sender prefix, typing/receipts, UUID allowlist), send reactions.

---

## src/slack — Slack (Bolt Socket Mode + HTTP) {#srcslack}

### Module Overview
Slack integration using **@slack/bolt** (Socket Mode or HTTP Events API). Rich feature set including slash commands, channel/thread threading, reactions, pins, file uploads, member events, channel lifecycle events, and configurable reply-to-mode per chat type.

**Entry points:** `monitorSlackProvider()`, `sendMessageSlack()`, `probeSlack()`

### File Inventory

| File | Description |
|------|-------------|
| `index.ts` | Re-exports: account functions, action functions, monitor, probe, send, token |
| `accounts.ts` | `resolveSlackAccount()`, `resolveSlackReplyToMode()`, `listEnabledSlackAccounts()` |
| `client.ts` | `createSlackWebClient()` — WebClient factory with retry config |
| `token.ts` | `resolveSlackBotToken()`, `resolveSlackAppToken()` |
| `probe.ts` | `probeSlack()` — auth.test health check |
| `send.ts` | `sendMessageSlack()` — chat.postMessage/chat.update/files.uploadV2 |
| `actions.ts` | Full Slack API wrappers: read, send, edit, delete, react, pin, member info, emojis |
| `format.ts` | `markdownToSlackMrkdwn()` — Markdown→Slack mrkdwn conversion |
| `targets.ts` | `parseSlackTarget()` — user/channel mention parsing |
| `types.ts` | `SlackMessageEvent`, `SlackAppMentionEvent`, `SlackFile` |
| `scopes.ts` | `resolveSlackScopes()` — OAuth scope discovery |
| `threading.ts` | `resolveSlackThreadContext()` — thread_ts resolution |
| `threading-tool-context.ts` | `buildSlackThreadingToolContext()` — agent tool threading |
| `channel-migration.ts` | Channel config migration |
| `directory-live.ts` | Live directory (users/channels) from Slack API |
| `message-actions.ts` | `listSlackMessageActions()`, `extractSlackToolSend()` |
| `resolve-channels.ts` | `resolveSlackChannels()` — channel name/ID lookup |
| `resolve-users.ts` | `resolveSlackUsers()` — user name/ID lookup |
| `http/registry.ts` | HTTP route registry for Slack webhook endpoints |
| `http/index.ts` | Re-exports registry |

#### monitor/ subdirectory

| File | Description |
|------|-------------|
| `provider.ts` | `monitorSlackProvider()` — Bolt app startup (socket or HTTP mode) |
| `types.ts` | `MonitorSlackOpts`, `SlackReactionEvent`, other event types |
| `context.ts` | `SlackMonitorContext` — shared runtime context for all handlers |
| `auth.ts` | `resolveSlackEffectiveAllowFrom()` — allowlist + pairing store merge |
| `allow-list.ts` | `normalizeSlackSlug()`, `allowListMatches()` |
| `channel-config.ts` | `resolveSlackChannelConfig()` — per-channel config resolution |
| `commands.ts` | `stripSlackMentionsForCommandDetection()`, slash command config |
| `policy.ts` | `isSlackChannelAllowedByPolicy()` — group policy enforcement |
| `events.ts` | `registerSlackMonitorEvents()` — registers all event handlers |
| `events/messages.ts` | Message and app_mention event handlers |
| `events/reactions.ts` | Reaction added/removed handlers |
| `events/channels.ts` | Channel created/renamed/id_changed handlers |
| `events/members.ts` | Member joined/left channel handlers |
| `events/pins.ts` | Pin added/removed handlers |
| `media.ts` | Slack file download with auth token injection |
| `message-handler.ts` | `createSlackMessageHandler()` — debounce + dispatch |
| `message-handler/prepare.ts` | `prepareSlackMessage()` — context building, mention detection |
| `message-handler/dispatch.ts` | `dispatchPreparedSlackMessage()` — agent dispatch + reply delivery |
| `message-handler/types.ts` | `PreparedSlackMessage` type |
| `replies.ts` | `deliverReplies()`, `resolveSlackThreadTs()` |
| `slash.ts` | Slash command handler with arg menus |
| `thread-resolution.ts` | Thread ts cache and resolution from history |

### Key Types

```typescript
type ResolvedSlackAccount = {
  accountId: string; enabled: boolean; botToken?; appToken?;
  botTokenSource: SlackTokenSource; appTokenSource: SlackTokenSource;
  config: SlackAccountConfig; replyToMode?; slashCommand?;
};
type SlackMessageEvent = { type: "message"; user?; text?; ts?; thread_ts?; channel; channel_type?; files? };
type SlackProbe = { ok: boolean; bot?: { id?; name? }; team?: { id?; name? } };
type SlackThreadContext = { incomingThreadTs?; messageTs?; isThreadReply; replyToId?; messageThreadId? };
```

### Data Flow
1. **Inbound (Socket Mode):** Bolt SocketModeReceiver → `event("message")` → `createSlackMessageHandler()` → `prepareSlackMessage()` (allowlist, mention, channel config) → `dispatchPreparedSlackMessage()` → agent → `deliverReplies()`
2. **Inbound (HTTP):** HTTP POST → Bolt ExpressReceiver → same handler chain
3. **Outbound:** `sendMessageSlack()` → `markdownToSlackMrkdwn()` → `chat.postMessage` with thread_ts

### Configuration Keys
- `channels.slack.accounts.<id>.{botToken, appToken, allowFrom, dm.allowFrom, channels, slashCommand, replyToMode, replyToModeByChatType, reactionNotifications, actions, mediaMaxMb, groupPolicy}`
- Env: `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN`

### Channel-Specific Features
- **Threading:** Configurable reply-to-mode (off/first/all) per chat type
- **Slash commands:** Native `/command` support with arg menus
- **Reactions:** Add/remove/list reactions, reaction notifications
- **Pins:** Pin/unpin messages, list pins
- **Channel events:** Channel create/rename/archive, member join/leave
- **File uploads:** `files.uploadV2` for media
- **Identity override:** Custom username/icon for outbound messages
- **Socket + HTTP modes:** Dual transport support

### External Dependencies
- `@slack/bolt` — Slack app framework
- `@slack/web-api` — Slack Web API client

### Test Coverage (18 test files)
Tests cover: actions read, channel migration, client, format, monitor (threading, tool results, channel config, context, media, thread-resolution, slash command), message handler (prepare inbound contract, sender prefix), resolve channels, targets, threading, threading tool context.

---

## src/whatsapp — WhatsApp Web (Baileys) {#srcwhatsapp}

### Module Overview
Minimal utility module for WhatsApp target normalization. The actual WhatsApp Web implementation (Baileys socket, QR login, message monitoring) lives in `src/channel-web.js` and `src/web/`, re-exported through `src/channels/web/index.ts`.

### File Inventory

| File | Description |
|------|-------------|
| `normalize.ts` | `normalizeWhatsAppTarget()`, `isWhatsAppGroupJid()`, `isWhatsAppUserTarget()` — JID/E.164 normalization |
| `normalize.test.ts` | Tests for target normalization |
| `resolve-outbound-target.ts` | `resolveWhatsAppOutboundTarget()` — validates and normalizes outbound targets with allowlist gating |

### Key Types

```typescript
type WhatsAppOutboundTargetResolution = { ok: true; to: string } | { ok: false; error: Error };
```

### Key Functions

```typescript
normalizeWhatsAppTarget(value: string): string | null  // "41796666864:0@s.whatsapp.net" → "+41796666864"
isWhatsAppGroupJid(value: string): boolean              // "120363401234567890@g.us" → true
isWhatsAppUserTarget(value: string): boolean            // "123@s.whatsapp.net" → true
resolveWhatsAppOutboundTarget(params): WhatsAppOutboundTargetResolution
```

### Internal Dependencies
- `src/utils.js` — `normalizeE164()`
- `src/infra/outbound/target-errors.js` — `missingTargetError()`

### Channel-Specific Features (via channel-web.js)
- **QR login:** WhatsApp Web QR code linking
- **Reactions:** Emoji reactions
- **Polls:** Native WhatsApp polls
- **Media:** Images, video, audio, documents
- **Groups:** Group messaging with JID-based targeting
- **Gateway delivery:** Messages sent via gateway (not direct from monitor)

---

## src/imessage — iMessage (imsg RPC) {#srcimessage}

### Module Overview
iMessage/SMS integration via the **imsg** CLI tool, communicating over JSON-RPC (stdin/stdout). Supports DMs, groups, media attachments, and multiple target addressing modes (chat_id, chat_guid, chat_identifier, handle with service prefix).

**Entry points:** `monitorIMessageProvider()`, `sendMessageIMessage()`, `probeIMessage()`

### File Inventory

| File | Description |
|------|-------------|
| `index.ts` | Re-exports: `monitorIMessageProvider`, `probeIMessage`, `sendMessageIMessage` |
| `accounts.ts` | `resolveIMessageAccount()`, `listIMessageAccountIds()` |
| `client.ts` | `createIMessageRpcClient()` — spawns imsg CLI, JSON-RPC over stdin/stdout |
| `constants.ts` | `DEFAULT_IMESSAGE_PROBE_TIMEOUT_MS` (10s) |
| `probe.ts` | `probeIMessage()` — checks imsg binary and RPC support |
| `send.ts` | `sendMessageIMessage()` — send text/media via RPC |
| `targets.ts` | `parseIMessageTarget()`, `formatIMessageChatTarget()`, `normalizeIMessageHandle()` |
| `target-parsing-helpers.ts` | Shared target parsing utilities (chat_id, chat_guid, service prefixes) |
| `monitor.ts` | Re-exports `monitorIMessageProvider` from monitor/ |
| `monitor/monitor-provider.ts` | `monitorIMessageProvider()` — RPC notification listener |
| `monitor/deliver.ts` | `deliverReplies()` — chunked text delivery |
| `monitor/parse-notification.ts` | `parseIMessageNotification()` — validates RPC notification payloads |
| `monitor/runtime.ts` | Runtime and allowlist normalization helpers |
| `monitor/types.ts` | `IMessagePayload`, `MonitorIMessageOpts`, `IMessageAttachment` |

### Key Types

```typescript
type ResolvedIMessageAccount = {
  accountId: string; enabled: boolean; config: IMessageAccountConfig; configured: boolean;
};
type IMessageTarget = 
  | { kind: "chat_id"; chatId: number }
  | { kind: "chat_guid"; chatGuid: string }
  | { kind: "chat_identifier"; chatIdentifier: string }
  | { kind: "handle"; to: string; service: IMessageService };
type IMessageService = "imessage" | "sms" | "auto";
type IMessagePayload = { id?; chat_id?; sender?; text?; attachments?; is_group?; participants?; ... };
```

### Data Flow
1. **Inbound:** `monitorIMessageProvider()` → `createIMessageRpcClient()` spawns imsg → listens for JSON-RPC notifications → `parseIMessageNotification()` → allowlist check → mention gating → `dispatchInboundMessage()` → agent → `deliverReplies()`
2. **Outbound:** `sendMessageIMessage()` → resolve target → RPC `send_message` call

### Configuration Keys
- `channels.imessage.accounts.<id>.{cliPath, dbPath, service, allowFrom}`
- Env: none

### Channel-Specific Features
- **Dual service:** iMessage and SMS via single interface
- **Chat addressing:** Multiple target modes (chat_id, chat_guid, handle, chat_identifier)
- **Media attachments:** File path-based attachment handling
- **Group support:** Group chat with participant list

### External Dependencies
- None (communicates with imsg CLI via child process)

### Test Coverage (5 test files)
Tests cover: client RPC, monitor (group mention skipping), probe, send, targets.

---

## src/line — LINE Messaging API {#srcline}

### Module Overview
LINE Messaging API integration with rich feature set including Flex Messages, Rich Menus, template messages, markdown→LINE conversion, webhook signature verification, and comprehensive message types. The most feature-rich of the newer channel implementations.

**Entry points:** `createLineBot()`, `monitorLineProvider()`, `sendMessageLine()`, `startLineWebhook()`, `probeLineBot()`

### File Inventory

| File | Description |
|------|-------------|
| `index.ts` | Extensive re-exports (100+ items): bot, monitor, send, webhook, accounts, probe, flex, rich menu, templates, markdown |
| `accounts.ts` | `resolveLineAccount()`, `listLineAccountIds()` |
| `types.ts` | `LineConfig`, `LineAccountConfig`, `ResolvedLineAccount`, `LineSendResult`, etc. |
| `config-schema.ts` | Zod schema for LINE config validation |
| `bot.ts` | `createLineBot()` — bot factory with webhook handler |
| `bot-handlers.ts` | `handleLineWebhookEvents()` — message/follow/unfollow/join/leave/postback handlers |
| `bot-message-context.ts` | `buildLineMessageContext()` — webhook event → MsgContext |
| `bot-access.ts` | `normalizeAllowFrom()`, `isSenderAllowed()` |
| `monitor.ts` | `monitorLineProvider()` — webhook registration + auto-reply delivery |
| `send.ts` | Full send API: text, image, location, flex, template, quick replies, push/reply modes |
| `auto-reply-delivery.ts` | `deliverLineAutoReply()` — reply with flex messages and quick replies |
| `reply-chunks.ts` | `sendLineReplyChunks()` — chunked text with reply token management |
| `actions.ts` | LINE action factories: message, URI, postback, datetime picker |
| `channel-access-token.ts` | Token resolution |
| `download.ts` | `downloadLineMedia()` — media content download |
| `probe.ts` | `probeLineBot()` — getBotInfo health check |
| `signature.ts` | `validateLineSignature()` — HMAC-SHA256 webhook signature verification |
| `webhook.ts` | `startLineWebhook()`, `createLineWebhookMiddleware()` — Express middleware |
| `webhook-node.ts` | Raw Node.js HTTP webhook handler (non-Express) |
| `webhook-utils.ts` | `parseLineWebhookBody()`, `isLineWebhookVerificationRequest()` |
| `http-registry.ts` | HTTP route registry for LINE webhooks |
| `markdown-to-line.ts` | `processLineMessage()` — extracts tables/code blocks into Flex Messages |
| `flex-templates.ts` | Re-exports from flex-templates/ subdirectory |
| `flex-templates/basic-cards.ts` | Info, list, image, action cards, carousel, notification bubble |
| `flex-templates/schedule-cards.ts` | Agenda, event, receipt cards |
| `flex-templates/media-control-cards.ts` | Media player, Apple TV remote, device control cards |
| `flex-templates/message.ts` | `toFlexMessage()` — wraps FlexContainer into FlexMessage |
| `flex-templates/common.ts` | Shared flex template utilities |
| `flex-templates/types.ts` | Flex template type definitions |
| `rich-menu.ts` | Full Rich Menu API: create, upload image, link/unlink users, grid layout |
| `template-messages.ts` | Confirm, button, carousel, image carousel templates |

### Key Types

```typescript
type ResolvedLineAccount = {
  accountId: string; enabled: boolean; channelAccessToken: string;
  channelSecret: string; tokenSource: LineTokenSource; config: LineAccountConfig;
};
type LineSendResult = { messageId?: string };
type LineProbeResult = { ok: boolean; bot?: { displayName?; userId?; basicId? } };
type ProcessedLineMessage = { text: string; flexMessages: FlexMessage[] };
type LineInboundContext = { /* message context from webhook event */ };
```

### Data Flow
1. **Inbound:** HTTP POST → `validateLineSignature()` → `handleLineWebhookEvents()` → `buildLineMessageContext()` → allowlist check → pairing → `dispatchInboundMessage()` → agent → `deliverLineAutoReply()` (reply token first, then push for remaining)
2. **Outbound:** `sendMessageLine()` / `pushMessageLine()` → LINE Messaging API

### Configuration Keys
- `channels.line.accounts.<id>.{channelAccessToken, channelSecret, tokenFile, secretFile, allowFrom, groupAllowFrom, dmPolicy, groupPolicy, groups}`
- Env: `LINE_CHANNEL_ACCESS_TOKEN`, `LINE_CHANNEL_SECRET`

### Channel-Specific Features
- **Flex Messages:** Rich card templates (info, list, image, action, carousel, receipt, media player, device control)
- **Rich Menus:** Full lifecycle management (create, image upload, user linking, grid layouts)
- **Template Messages:** Confirm, buttons, carousel, image carousel
- **Quick Replies:** Suggested action buttons below messages
- **Markdown→Flex:** Tables and code blocks extracted into Flex Messages
- **Postback actions:** Structured data callbacks
- **Datetime picker:** Native date/time selection
- **Loading animation:** Shows typing indicator
- **Webhook verification:** Signature validation + verification request handling
- **Push + Reply modes:** Reply token for first response, push API for subsequent

### External Dependencies
- `@line/bot-sdk` — LINE Messaging API SDK
- `zod` — Config schema validation
- `express` (optional) — Webhook middleware

### Test Coverage (14 test files)
Tests cover: accounts, auto-reply delivery, bot handlers, bot message context, flex templates, markdown-to-line, monitor (read body), probe, reply chunks, rich menu, send, signature, template messages, webhook, webhook-node.

---

## Cross-Channel Data Flow {#cross-channel-data-flow}

### Inbound Message Processing (all channels)

```
Channel Event (webhook/SSE/poll/socket)
  → Channel-specific normalize (raw event → MsgContext fields)
  → Allowlist check (sender authorized?)
  → Mention gating (requireMention in groups?)
  → Command detection (/command or text command?)
  → Session recording (recordInboundSession)
  → dispatchInboundMessage() (routes to agent)
  → Agent processes + generates reply
  → Channel-specific delivery (chunks, formatting, threading)
```

### Outbound Message Delivery (agent tools)

```
Agent tool call: message(action="send", target="...", message="...")
  → resolveChannelPlugin() → plugin.actions.handleAction()
  → Channel-specific send function
  → Format text (markdown → HTML/mrkdwn/styled text)
  → Chunk if needed (per-channel limit)
  → API call (REST/RPC)
  → Return result (messageId)
```

### Shared Infrastructure Used by All Channels

| Module | Purpose |
|--------|---------|
| `auto-reply/dispatch.ts` | Routes inbound messages to agent sessions |
| `auto-reply/envelope.ts` | Formats inbound context (From, To, ChatType, etc.) |
| `auto-reply/reply/mentions.ts` | Mention pattern matching |
| `auto-reply/reply/history.ts` | Conversation history management |
| `auto-reply/chunk.ts` | Text chunking with markdown awareness |
| `config/sessions.ts` | Session store (per-sender/per-chat/global) |
| `config/group-policy.ts` | Group requireMention and toolPolicy resolution |
| `routing/resolve-route.ts` | Agent routing (which agent handles which session) |
| `routing/session-key.ts` | Session key construction (agent:id:channel:type:target) |
| `infra/outbound/deliver.ts` | Outbound delivery orchestration |
| `pairing/` | Pairing request/approval flow |
| `markdown/ir.ts` | Markdown intermediate representation for format conversion |

### Capability Matrix

| Feature | Telegram | Discord | Signal | Slack | WhatsApp | iMessage | LINE |
|---------|----------|---------|--------|-------|----------|----------|------|
| DMs | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Groups | ✅ | ✅ (guilds) | ✅ | ✅ (channels) | ✅ | ✅ | ✅ |
| Threads | ✅ (forum) | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Reactions | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Polls | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ |
| Media | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Slash Commands | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Streaming | ✅ (draft edit) | ✅ (block) | ✅ (block) | ✅ (block) | ❌ | ❌ | ❌ |
| Voice Messages | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Rich Templates | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ (Flex) |
| Rich Menus | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Inline Buttons | ✅ | ✅ (components) | ❌ | ❌ | ❌ | ❌ | ✅ (quick reply) |
| Edit Messages | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Text Limit | 4000 | 2000 | 4000 | 4000 | 4000 | 4000 | 5000 |
| Format | HTML | Markdown | Styled ranges | mrkdwn | Plain | Plain | Flex/Plain |
| Transport | Poll/Webhook | WebSocket | SSE | Socket/HTTP | WebSocket | RPC | Webhook |

---

*End of analysis. Total files analyzed: ~451 across 8 modules.*
