# OpenClaw Channels & Messaging — Comprehensive Analysis
<!-- markdownlint-disable MD024 MD028 -->

> Updated: 2026-03-08 | Version: v2026.3.7 | Cluster: CHANNELS & MESSAGING
> Modules analyzed: `src/telegram` (108 files), `src/discord` (136 files), `src/signal` (32 files), `src/slack` (103 files), `src/whatsapp` (4 files), `src/imessage` (25 files), `src/line` (46 files), `src/channels` (147 files), `extensions/feishu` (77 files)

> **v2026.2.22 Breaking:** Unified streaming config — most channels now use enum `off | partial | block | progress` in `channels.<channel>.streaming`. Telegram additionally accepts legacy boolean `streaming` and legacy `streamMode` values, mapping them to the enum (`true`→`partial`, `false`→`off`). Run `openclaw doctor --fix` to migrate legacy `streamMode` keys. Slack native streaming moved to `channels.slack.nativeStreaming`.

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
10. [Synology Chat Extension](#synology-chat-extension)
11. [Cross-Channel Data Flow](#cross-channel-data-flow)

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
| `web/index.ts` | Re-exports WhatsApp Web functions from `channel-web.ts` |

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
- `channels.telegram.{token, tokenFile, allowFrom, replyToMode, reactionLevel, webhookPort}`
- Env: `TELEGRAM_BOT_TOKEN`, `OPENCLAW_TELEGRAM_PROXY`, `OPENCLAW_TELEGRAM_DISABLE_AUTO_SELECT_FAMILY`, `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER`

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
Minimal utility module for WhatsApp target normalization. The actual WhatsApp Web implementation (Baileys socket, QR login, message monitoring) lives in `src/channel-web.ts` and `src/web/`, re-exported through `src/channels/web/index.ts`.

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
- `src/utils.ts` — `normalizeE164()`
- `src/infra/outbound/target-errors.ts` — `missingTargetError()`

### Channel-Specific Features (via channel-web.ts)
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

## Synology Chat Extension {#synology-chat-extension}

### Synology Chat (`extensions/synology-chat/`)

**Status:** v2026.2.22 — New channel plugin

**Key capabilities:**
- Webhook ingress for inbound messages
- Direct-message routing with DM policy controls
- Outbound send/media support
- Per-account config under `channels.synologychat.*`

**Config:** `channels.synologychat.enabled: true`, `channels.synologychat.webhookSecret`, `channels.synologychat.botToken`

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
| Voice Messages | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ |
| Rich Templates | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ (Flex) |
| Rich Menus | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Inline Buttons | ✅ | ✅ (components) | ❌ | ❌ | ❌ | ❌ | ✅ (quick reply) |
| Edit Messages | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Text Limit | 4000 | 2000 | 4000 | 4000 | 4000 | 4000 | 5000 |
| Format | HTML | Markdown | Styled ranges | mrkdwn | Plain | Plain | Flex/Plain |
| Transport | Poll/Webhook | WebSocket | SSE | Socket/HTTP | WebSocket | RPC | Webhook |

---

## v2026.2.15 Changes (2026-02-16)

### Telegram
- **Per-channel ackReaction config** (`#17092`, thanks @zerone0x) — ackReaction scope can now be set per channel/group, not just globally
- **Simplified send/dispatch/target handling** (`#17819`) — refactored outbound send path
- **Block streaming fix** (`#17704`) — stop block streaming from splitting messages when `streamMode` is off
- **No-op editMessage fix** — treat no-op editMessage as success instead of error
- **Account-scoped pairing allowlists** — pairing allowlists now correctly scoped per account
- **Shared allowFrom normalization** — deduplicated with other channels

### Discord
- **Components v2 UI tool** (`#16364`, `#17419`) — full Discord Components v2 support: containers, sections, galleries, separators, text displays, thumbnails, media galleries. Two PRs: initial CV2 implementation and follow-up UI tool support
- **Component parsing & modal field typing fix** — fixed component parsing and modal field typing issues
- **Message action send parameter alignment** — aligned send parameters across action handlers
- **File/buffer attachments** — file and buffer-based attachment support in send actions
- **Channel session key fix** (`#17622`) — preserve channel session keys via `channel_id` fallbacks

### Channels (shared infrastructure)
- **Plugin LLM input/output hooks** (`#16724`, thanks @SecondThread) — channel plugins can now hook into LLM input/output payloads for preprocessing and postprocessing
- **Lazy jiti loader for plugins** — `jiti` loader is now lazily created, improving startup performance
- **chat.send input sanitization** — hardened `chat.send` message input sanitization in the gateway
- **Shared allowlist helpers** — deduplicated allowlist resolution, user entry collection, and config patching across all channels
- **Shared threading tool context** — deduplicated threading context across channels
- **Shared directory allowFrom parsing** — consolidated directory allowFrom parsing
- **Shared outbound hook handling** (Slack) — deduplicated outbound hook handling

### Signal
- **Outbound media limit deduplication** — shared media limit resolution with other channels

### LINE
- **Webhook auth hardening** — fail closed when webhook auth is missing (security fix)
- **Test pruning** — extensive removal of redundant/low-signal test cases (~50 commits)

### Slack
- **Outbound send deduplication** — shared outbound send flow
- **Onboarding token prompt deduplication** — shared with other channels
- **Test performance** — sped up slash handler tests

### Cross-Channel
- **Probe/token base type deduplication** (`#16986`, thanks @iyoda)
- **Consolidated test suites** — channel action, plugin, and misc test suites consolidated for performance
- **Onboarding config patching** — shared across Discord and Slack

## v2026.2.19 Changes (2026-02-19)

### Telegram
- **Cron/heartbeat topic delivery** — Explicit `<chatId>:topic:<threadId>` targets now correctly route scheduled sends into the configured topic instead of defaulting to last active thread. See DEVELOPER-REFERENCE.md §9 (gotcha 43)

### Discord
- **Moderation permission checks** — Moderation actions (`timeout`, `kick`, `ban`) now enforce guild permission checks on trusted sender; untrusted `senderUserId` parameters rejected. See DEVELOPER-REFERENCE.md §9 (gotcha 44)

### Browser/Relay
- **Chrome extension relay auth** — Both `/extension` and `/cdp` endpoints now require `gateway.auth.token` authentication

---

<!-- v2026.2.21 -->
## v2026.2.21 Changes (2026-02-21)

### Telegram

<!-- v2026.2.21 -->
- **Streaming config simplified** (`src/telegram/bot-message-dispatch.ts`) — `channels.telegram.streaming` is now a plain boolean (`true`/`false`). Legacy `streamMode` string values (`"off"`, `"partial"`, `"block"`) are auto-mapped on load. Internally the dispatch logic continues to use the same two-lane (answer + reasoning) draft-stream architecture; the simplified boolean controls whether answer-lane draft streaming is active. Reasoning-lane streaming is governed separately by the per-session `reasoningLevel` setting.

<!-- v2026.2.21 -->
- **Status reactions** (new file: `src/telegram/status-reaction-variants.ts`) — Lifecycle reaction phases now signal agent progress directly on the inbound message: `queued` → `thinking` → `tool`/`coding`/`web` (tool-type aware) → `done` or `error`. The new module provides:
  - `TELEGRAM_STATUS_REACTION_VARIANTS` — per-phase fallback emoji lists restricted to Telegram's supported reaction set.
  - `resolveTelegramStatusReactionEmojis()` — merges per-channel config overrides (`channels.telegram.accounts.<id>.statusReactions.*`) against defaults from `src/channels/status-reactions.ts`.
  - `buildTelegramStatusReactionVariants()` — builds the variant map used for chat-allowed emoji fallback resolution.
  - `resolveTelegramReactionVariant()` — picks the first available emoji from the variant list that is both in Telegram's global supported set and allowed by the specific chat's `available_reactions`.
  - The `statusReactionController` field on `TelegramMessageContext` carries a `StatusReactionController` instance; `dispatchTelegramMessage` calls `setThinking()` at start, `setTool(name)` per tool call, and `setDone()` or `setError()` on completion.

### Discord

<!-- v2026.2.21 -->
- **Model picker UI** (new file: `src/discord/monitor/model-picker.ts`, ~937 lines) — Native Discord Components v2 UI for switching models mid-conversation, restoring the model picker UX broken in a prior release (issue #21458). Key details:
  - Uses Carbon's `Container`, `Row`, `Button`, `StringSelectMenu`, `TextDisplay`, and `Separator` components.
  - State encoded in the `custom_id` field under the key `mdlpk` (max 100 chars); carries `command`, `action`, `view`, `userId`, `provider`, `page`, `providerPage`, `modelIndex`, `recentSlot`.
  - Views: `providers` (paginated buttons, up to 20 providers per page), `models` (select menu, up to 25 per page), `recents` (recently used models).
  - Actions: `open`, `provider`, `model`, `submit`, `quick`, `back`, `reset`, `cancel`, `recents`.
  - Provider buttons paginated at 20 per page (4 rows × 5 buttons, reserving 1 row for nav); single-page layout uses all 5 rows (25 max).
  - Triggered by `/model` and `/models` slash commands.

<!-- v2026.2.21 -->
- **Thread-bound subagents** (`src/discord/monitor/provider.ts`, `native-command.ts`) — New per-thread subagent session system. Each Discord thread can be bound to a dedicated agent session via a `ThreadBindingManager` (`src/discord/monitor/thread-bindings.ts`). New slash commands:
  - `/focus` — bind the current thread to a named subagent session.
  - `/agents` — list active thread bindings (`/subagents list` also exposes run-level listing).
  - Thread-bound sessions route continuation messages back to the bound subagent automatically. TTL defaults to 24 hours; configurable via `channels.discord.accounts.<id>.threadBindings.ttlHours` and per-session overrides.

<!-- v2026.2.21 -->
- **Voice channel support** (new files: `src/discord/voice/manager.ts` ~676 lines, `src/discord/voice/command.ts` ~341 lines) — Bot can join and participate in Discord voice channels for realtime voice conversations:
  - `DiscordVoiceManager` (`manager.ts`) — manages active voice sessions keyed by `guildId:channelId`. Handles audio capture (48kHz/stereo/16-bit PCM via `@discordjs/voice`), silence detection (1s gap), TTS playback queue, and per-guild session lifecycle. Integrates with the TTS pipeline (`src/tts/tts.ts`) and media understanding runner for speech-to-text. Exposes `join()`, `leave()`, `getStatus()`.
  - `createDiscordVoiceCommand()` (`command.ts`) — builds the `/vc` slash command group with subcommands `join`, `leave`, and `status`. Each subcommand is scoped to guild channels only and respects the same allowlist/group-policy authorization as other Discord commands. `ephemeralDefault` is forwarded from the account slash-command config.
  - Auto-join: if `channels.discord.accounts.<id>.voice.autoJoin` is set, the bot joins the configured voice channel on startup.
  - Requires `VoicePlugin` from `@buape/carbon/voice` registered on the Carbon client.

<!-- v2026.2.21 -->
- **Provider allowlist canonicalization** (new file: `src/discord/monitor/provider.allowlist.ts`, ~207 lines) — On startup, `resolveDiscordAllowlistConfig()` resolves all human-readable guild/channel/user names in the config to Discord snowflake IDs by querying the Discord REST API. This eliminates runtime mismatches caused by display-name changes and ensures allowlist entries remain stable. Calls `resolveDiscordUserAllowlist()` and `resolveDiscordChannelAllowlist()` in parallel per guild, then patches the live config entries via `patchAllowlistUsersInConfigEntries()`.

<!-- v2026.2.21 -->
- **Ephemeral defaults for slash commands** — `channels.discord.accounts.<id>.slashCommand.ephemeral` (boolean) controls whether slash-command responses are ephemeral by default. Resolved as `ephemeralDefault` in `provider.ts` and forwarded to every native command and the `/vc` command group. Previously responses were always non-ephemeral.

<!-- v2026.2.21 -->
- **Forum tag management** (`src/discord/send.channels.ts`) — Channel-edit actions can now supply an `availableTags` array to update a forum channel's tag list. Each entry accepts `{ id?, name, emoji?, moderated? }`. Passed through as `available_tags` in the Discord channel PATCH body.

### Channels (shared infrastructure)

<!-- v2026.2.21 -->
- **Volcengine / BytePlus provider** (`src/agents/models-config.providers.volcengine-byteplus.test.ts`, `auth-choice.apply.volcengine.ts`) — New provider integrated: Volcengine (BytePlus / Doubao family). Doubao models including coding variants are now available in the model catalog. Onboarding credentials handled via `auth-choice.apply.volcengine.ts`.

<!-- v2026.2.21 -->
- **BlueBubbles (iMessage via BlueBubbles server)** — A new channel platform entry has been added (distinct from the existing `imsg`-based iMessage channel). Status-issue diagnostics live in `src/channels/plugins/status-issues/bluebubbles.ts`; action specs in `src/channels/plugins/bluebubbles-actions.ts`. Configuration and onboarding are separate from the classic iMessage path.

<!-- v2026.2.21 -->
- **Per-channel model override** (`src/channels/model-overrides.ts`) — New `channels.modelByChannel` config section allows routing specific channels or groups to a dedicated model. Config structure:

  ```json
  {
    "channels": {
      "modelByChannel": {
        "telegram": { "-100123456789": "openai/gpt-4.1" },
        "discord": { "guild-name/channel-name": "anthropic/claude-opus-4-6", "*": "google/gemini-3-flash-preview" }
      }
    }
  }
  ```

  Keys under each channel entry are matched against the inbound `groupId`/`groupChannel`/`groupSubject` using the same candidate-matching logic as `channel-config.ts` (slug normalization, wildcard `*` fallback). Thread-suffixed group IDs (`groupId:thread:xxx`) also check the parent ID. The override is applied before inline `/model` directives but after agent-level defaults.

---

## v2026.2.22 Changes (2026-02-23)

### Telegram

- **WSL2 network hardening** — Disabled `autoSelectFamily` by default under WSL2; WSL2 detection is now memoized to avoid repeated filesystem probes at runtime.
- **DNS `ipv4first` default for Telegram fetch paths** — Telegram outbound HTTP fetches now default to `ipv4first` DNS result order. Override with `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER` env var.
- **Forward burst lane** — Forward operations now use a dedicated forward lane with per-chat debounce to prevent burst flooding when forwarding multiple messages quickly.
- **Streaming: archived draft preview mapping preserved after flush** — Fixed a regression where archived draft preview state was cleared prematurely during a flush, causing reply continuation to lose context.
- **Polling: safe update-offset watermark persisted** — The long-polling runner now persists a safe watermark offset so restarts never re-process already-handled updates.
- **Polling: stuck runner force-restart** — Stuck polling runners are detected via an inactivity deadline and force-restarted with exponential backoff.
- **Media: user-facing reply on download failure** — When a Telegram media download fails the bot now sends a user-visible error reply rather than silently dropping the message.
- **Webhook: keep monitors alive until gateway abort** — Webhook monitors are kept alive until the gateway signals an abort, preventing premature teardown under load.
- **Webhook port config** — New `channels.telegram.webhookPort` config key for explicitly binding the webhook HTTP listener to a specific port.

### Discord

- **Allowlist: canonicalize resolved names to IDs at runtime** — Resolved guild member and channel names are now immediately canonicalized to Discord snowflake IDs after lookup, eliminating stale-name mismatches.
- **Security audit warnings for name/tag-based allowlist entries** — Startup emits a warning when allowlist entries are specified as display names or tags rather than IDs, flagging slug-collision risk.

### Slack

- **Threading: parent-session forking active beyond first turn** — Thread forking from a parent session is now applied across all turns, not just the first message in a thread.
- **`replyToMode` respected when Slack auto-populates `thread_ts`** — Previously, Slack's implicit `thread_ts` injection bypassed `replyToMode`; now the configured mode is enforced consistently.
- **Extension: forward `threadId` to `readMessages`** — The `readMessages` extension action now receives the resolved `threadId` so downstream handlers can correctly scope thread reads.
- **Upload: resolve bare user IDs via `conversations.open`** — File upload targets that are bare Slack user IDs are now resolved to a DM channel ID via `conversations.open` before upload.
- **Slash: preserve Bolt app receiver for external options handlers** — The Bolt `ExpressReceiver` is preserved and re-used for external-select options endpoints rather than being recreated per request.
- **Queue routing: preserve string `thread_ts` through collect-mode drain** — String `thread_ts` values are no longer coerced to numbers during collect-mode queue drain, preventing thread routing failures in channels with high-precision timestamps.

### BlueBubbles / iMessage

- **DM history: account-scoped rolling history with bounded backfill retries** — Per-account rolling DM history is maintained with a configurable backfill retry cap to prevent unbounded API calls on reconnect.
- **Private API cache: treat unknown status as disabled** — Unrecognized Private API feature status values are now treated as disabled rather than propagating an ambiguous state.
- **Webhooks: accept payloads when `handle` is omitted but `chatGuid` is present** — Inbound webhook payloads that omit the `handle` field but supply a valid `chatGuid` are now accepted and routed correctly.

### Webchat

- **Apply `final` payload messages directly to chat state** — Messages arriving in a `final` payload are now applied directly to the in-memory chat state without an extra round-trip.
- **Preserve external session routing metadata** — Routing metadata attached to external sessions is preserved across rehydration so sessions resume on the correct agent.
- **Preserve session `label` across `/new`/`reset`** — The human-readable session label is now preserved when a session is reset or a new session is started via the `/new` or `/reset` endpoints.

### Synology Chat

- **New channel plugin** (`extensions/synology-chat/`) — Webhook ingress, DM routing with DM policy controls, outbound send/media. Config: `channels.synologychat.enabled`, `channels.synologychat.webhookSecret`, `channels.synologychat.botToken`.

---

## v2026.2.23 Changes (2026-02-24)

### Telegram

- **Reactions: soft-fail on policy/token/emoji/API errors** — Reaction delivery no longer propagates errors to the caller when the failure is due to policy restrictions, missing bot token, unsupported emoji, or transient API errors; instead it logs and continues.
- **Reactions: accept snake_case `message_id`** — Reaction action payloads now accept `message_id` (snake_case) in addition to the previously required camelCase form.
- **Reactions: fallback to inbound message-id** — When no explicit message ID is supplied for a reaction, the action falls back to the ID of the triggering inbound message.
- **Polling: scope persisted polling offsets to bot identity** — Persisted update-offset watermarks are now keyed by bot identity (token hash) rather than account ID alone, so token rotations do not cause offset collisions.
- **Polling: single awaited runner-stop path** — The polling runner shutdown sequence now uses a single awaited stop path, eliminating a race condition where concurrent stop calls could leave orphaned timers.

### Slack

- **Group policy: `groupPolicy` defaulting moved to provider-level schema defaults** — `groupPolicy` is now defaulted at the provider config schema level, enabling correct multi-account inheritance without requiring explicit per-account declarations.

---

## v2026.2.24 Changes (2026-02-25)

### Telegram

- **Media fetch IPv4 priority** (#24295, #23975): SSRF-pinned DNS address ordering now prefers IPv4 before IPv6, fixing media downloads on hosts with broken IPv6 routing. Contributor: @Glucksberg.

### Slack

- **DM routing** (#25479): `D*` channel IDs are now always treated as direct messages even when Slack sends an incorrect `channel_type` value, preventing DM traffic from being misclassified as channel/group chats. Contributor: @mcaxtr.

---

## v2026.3.1 Changes (2026-03-02)

### Telegram

- **DM topics** (#30579, thanks @kesor): Per-DM topic configuration with allowlists, `dmPolicy`, skills, `systemPrompt`, and `requireTopic`. Topic-aware session routing scopes sessions by `chatId:threadId` for full isolation. New test file `bot-message-context.dm-topic-threadid.test.ts` validates `updateLastRoute` carries `threadId` for DM topics.
- **DM topic session isolation** (#31064): Session keys for DM topic threads are now scoped by `chatId:threadId`, preventing cross-topic session bleed in private chats with multiple topics.
- **Reply media context** (#28488): Replied-to media files are now included in the inbound message context, giving the agent access to images/documents from the message being replied to.
- **Voice fallback reply chunking** (#31067, #31077, thanks @xdanger): When voice send fails and falls back to text, the reply-to reference is applied only to the first chunk. Prevents duplicate reply-to references on subsequent chunks.
- **Outbound chunking**: Oversize outbound messages now route through the shared chunking pipeline. Escaped HTML that exceeds limits triggers a retry with re-chunked payloads.
- **Empty final replies** (#30969, thanks @haosenwang1018): Null/undefined final text payloads are now skipped rather than sent as empty messages.
- **Proxy dispatcher preservation** (#29940): The HTTP proxy-aware global dispatcher is preserved across fetch paths, fixing proxy bypass regressions introduced by Node.js Happy Eyeballs workarounds.
- **Media fetch IPv4 fallback** (#30554): Media fetches now retry with explicit IPv4 when initial connect errors occur, fixing downloads on hosts with broken IPv6 routing.
- **Group allowlist ordering**: Chat-level allowlist is now evaluated before sender-level allowlist in group policy checks, ensuring group-scoped rules take precedence. New test file `group-access.policy-access.test.ts`.
- **Multi-account group isolation**: Channel-level `groups` config no longer leaks across accounts in multi-account setups. Each account's group config is now strictly scoped.
- **sendVoice caption overflow**: When `sendVoice` fails due to caption exceeding limits, the bot resends without caption instead of failing.
- **sendChatAction 401 backoff** (#27415, thanks @widingmarcus-cyber): Typing indicator sends now back off after 401 errors instead of retrying indefinitely.
- **Inline button callbacks in groups** (#27309): Inline button callback queries in groups are now processed when the command was previously authorized, rather than being silently dropped.

### Discord

- **Thread bindings lifecycle** (#27845, thanks @osolmaz): Thread bindings now use an inactivity-based lifecycle (`idleHours`/`maxAgeHours`) instead of fixed TTL. Config at `channels.discord.accounts.<id>.threadBindings.{idleHours, maxAgeHours}` with session-level fallback from `session.threadBindings.*`. New files: `thread-bindings.config.ts`, `thread-bindings.lifecycle.test.ts`, `thread-bindings.persona.ts`.
- **Components wildcard handlers** (#29459, thanks @Sid-Qin): Wildcard component registration now uses distinct sentinel IDs per component type (select/user/role/channel/modal) to prevent registration collisions.
- **Forum thread tags**: `applied_tags` parameter is now supported when creating forum threads via channel-edit actions.
- **Application ID fallback**: When the application info endpoint is unavailable, the application ID is parsed from the bot token prefix. New test file `probe.parse-token.test.ts`.
- **EventQueue timeout config** (#24458): The `listenerTimeout` on the gateway EventQueue is now exposed as a configurable option.
- **Allowlist diagnostics**: Debug-level logs now emit when messages are dropped due to guild/channel allowlist filtering, aiding troubleshooting of silent message drops.
- **Ack reactions scope override** (#28268): `ackReactionScope` can now be overridden at the account level, and explicit `off`/`none` values are supported to disable ack reactions per account.
- **Inbound media fallback** (#28906, thanks @Sid-Qin): When CDN media fetch fails, attachment metadata (filename, content type, size) is preserved in the message context rather than being dropped entirely.
- **Agent component interactions** (#29013, thanks @Jacky1n7): Components v2 `cid` payloads are now accepted and decoded in agent interaction handlers. URI decoding is guarded against malformed payloads.
- **Reconnect watchdog** (#31025, #30530): Gateway reconnect uses a unified watchdog that prevents duplicate reconnect attempts and preserves message delivery during recovery.

### Feishu (Lark)

- **Docx tables + uploads** (#20304): New `feishu_doc` tool actions: `create_table`, `write_table_cells`, `upload_image`, `upload_file`. Markdown tables and positional insert are also supported. New files: `docx-table-ops.ts`, `docx-batch-insert.ts`, `docx-color-text.ts`.
- **Reactions** (#16716, thanks @schumilin): Inbound `im.message.reaction.created_v1` events are now handled. New file: `monitor.reaction.test.ts`. Reaction notifications configurable via `reactionNotifications` (`off`/`own`/`all`).
- **Chat tooling** (#14674): New `feishu_chat` tool provides chat info and member listing.
- **Doc permissions** (#28295, thanks @zhoulongchao77): Document creation can optionally grant owner-level permission to the requesting user. Permission grants are now bound to trusted requester context (#31184).
- **Reply-in-thread routing** (#27325, thanks @kcinzgg): New `replyInThread` config controls whether replies target the parent thread. Streaming cards pass `rootId` for topic groups (#28346, thanks @Sid-Qin).
- **Group session scopes** (#17798, thanks @yfge): Four session scope modes: `group`, `group_sender`, `group_topic`, `group_topic_sender`. Honors `dm:`/`group:` prefixes in outbound session routing.
- **Multi-account + reply reliability**: `defaultAccount` support for outbound routing. Reply dispatcher has hardened fallback for target routing and dedup.
- **Post markdown parsing** (#12755): Rich text posts are now rendered as markdown via a shared markdown-aware parser. Code blocks (`code`/`code_block`/`pre` tags) are properly handled (#28591, thanks @kevinWangSheng).
- **Probe status caching** (#28907, thanks @Glucksberg): `probeFeishu()` results are cached with a 10-minute TTL to reduce API calls.
- **Opus media send type** (#28269, thanks @Glucksberg): Opus audio files are sent with `msg_type` `"audio"` instead of `"media"`, enabling voice bubble playback.
- **Mobile video media type** (#25502, thanks @4ier): `message_type` `"media"` is now handled for video downloads from mobile clients.
- **Inbound sender fallback** (#26703, thanks @NewdlDewdl): Falls back to `user_id` for sender identity when open_id is unavailable.
- **Reply context metadata** (#18529): `parent_id` and `root_id` from quoted messages are now included in inbound context for quote support.
- **Post embedded media** (#21786): Video and audio elements are extracted from post (rich text) messages.
- **Local media sends** (#27884, #27928, thanks @joelnishanth): `mediaLocalRoots` is propagated for local file path sends.
- **Group wildcard policy fallback** (#29456): Wildcard group config is honored for reply policy when no explicit group match exists.
- **Inbound media regression coverage** (#16311, thanks @Yaxuan42): Test regression for audio download with `resource type=file`.
- **Typing backoff** (#28494, thanks @guoqunabc): Rate-limit and quota errors in typing indicator sends now re-throw instead of retrying infinitely.
- **Group sender allowlist fallback** (#29174, thanks @1MoreBuild): Global `groupSenderAllowFrom` provides sender-level group access control.
- **Docx append/write ordering** (#26022, #26172, thanks @echoVic): Document blocks are inserted sequentially to preserve intended order.
- **Docx convert fallback chunking** (#14402): Large documents are chunked for write/append to avoid API 400 errors.
- **API quota controls** (#10513, thanks @BigUncle): `typingIndicator` and `resolveSenderNames` flags allow disabling API-heavy operations to stay within quota.
- **System preview prompt leakage fix** (#31209, thanks @stakeswky): System preview text no longer leaks into agent-visible prompt context.
- **Typing replay suppression** (#30709, thanks @arkyu2077; #30418): Typing indicators are suppressed for stale replay messages and old messages after context compaction.
- **Merge forward parsing** (#28707, thanks @tsu-builds): `merge_forward` message type is now parsed for inbound context.
- **Card action callbacks** (#17863): `card.action.trigger` callbacks are handled.
- **Streaming card headers** (#22826): Optional header support in streaming cards.

### Slack

- **User-token resolution normalization** (#28103, thanks @Glucksberg): `SLACK_USER_TOKEN` is now used when connecting to Slack, normalizing user-token resolution across bot and user token sources.
- **Native commands** (#29032, thanks @maloqab): `/agentstatus` is now registered as a native Slack slash command alias.
- **Announce target account routing** (#31105): Subagent completion announces to Slack no longer fail with invalid `thread_ts` on bound targets.
- **Per-thread session isolation** (#26849): DM auto-threading now isolates sessions per thread.
- **Thread participation tracking** (#29165): Auto-reply without @mention when bot has participated in the thread.
- **Stale socket detection** (#30153): Stale Slack sockets are detected and auto-restarted.
- **Socket Mode reconnect** (#27232): Socket Mode connections reconnect after disconnect events.
- **Download-file action** (#24723): New `download-file` action for on-demand file attachment access.
- **3-step upload flow** (#17558): `files.uploadV2` replaced with 3-step upload flow to fix `missing_scope` errors.
- **replyToMode per-message** (#24717): `replyToMode` is resolved per-message using chat type context.
- **replyToModeByChatType with ThreadLabel** (#26251): Configured `replyToModeByChatType` is now honored when `ThreadLabel` exists.
- **Agent identity in reply path** (#27134, thanks @hou-rong): Agent identity is threaded through the channel reply path.
- **Disabled account guard** (#30592, thanks @liuxiaopai-ai): Monitor startup is skipped for disabled accounts.
- **Slash/interactions ingress hardening** (#29091, thanks @Solvely-Colin): Slash command and interactions ingress checks are hardened.
- **Socket Mode startup guard** (#28702, thanks @Glucksberg): Socket Mode listeners access is guarded during startup.
- **Legacy streaming=false mapping** (#26020, thanks @chilu18): Legacy `streaming=false` is mapped to `off`.
- **Bot message attachment text** (#27616, #27642): Attachment text is extracted for bot messages with empty text.

### LINE

- **Voice transcription M4A detection** (#31151, thanks @scoootscooob): M4A files (MPEG-4 Audio with `ftyp`/`M4A ` brand) are now classified as `audio/mp4` with `.m4a` extension, enabling correct voice transcription pipeline routing.

### Signal

- **Sync message null-handling** (#31138, #31093, thanks @Sid-Qin): `syncMessage` presence is treated as a sync envelope even when its value is `null`. The check uses property existence (`"syncMessage" in envelope`) rather than truthiness, preventing the bot from replaying its own sent messages on daemon restart.

### Google Chat

- **Thread replies** (#30965, thanks @novan): Thread sends now include `messageReplyOption=REPLY_MESSAGE_FALLBACK_TO_NEW_THREAD`, allowing replies to existing threads to fall back to a new thread if the original thread is no longer available.

### Mattermost

- **Private channel policy routing** (#30891, thanks @BlueBirdBack): Mattermost channel type `P` (private) is now mapped to group chat policy, ensuring private channels receive group-level authorization and tool policy rather than being treated as public channels.

### Matrix

- **Directory room IDs** (#31201, thanks @williamos-dev): Room IDs from the Matrix directory now preserve original casing, fixing lookup failures on case-sensitive homeservers.

### BlueBubbles

- **Message metadata** (#23970): Send response ID extraction is hardened — message GUIDs are reliably extracted from API responses for reply threading and dedup.

### WhatsApp

- **TTS voice bubbles** (#27366): WhatsApp now receives opus-encoded TTS output with `audioAsVoice=true`, enabling voice bubble playback. WhatsApp is added to the `VOICE_BUBBLE_CHANNELS` set alongside Telegram and Feishu.

### Cross-Channel

- **Multi-account default routing**: `channels.<channel>.defaultAccount` is now supported across all channels for outbound routing when no explicit account is specified. Verified for Mattermost, Google Chat, and others.
- **TTS voice bubbles**: Voice bubble support expanded to Feishu and WhatsApp (previously Telegram only). Opus output format is used for all voice-bubble channels.
- **Unified webhook security guardrails**: Webhook memory guards and rate-limit state are hardened across all webhook-based channels.
- **Unified typing lifecycle**: Typing dispatch lifecycle and policy boundaries are unified across channels, with stale typing suppression.
- **Account-scoped pairing APIs**: Pairing request/approval APIs are enforced at the account scope, preventing DM pairing allowlists from leaking into group auth.

---

## v2026.3.7 Delta Notes

- **Telegram**: persistent topic bindings; per-topic agentId routing; DM draft streaming restored; polling offset safety hardened; webhook-mode health probes survive restarts.
- **Discord**: native slash commands added; agentComponents schema validation enforced; model picker selection now persisted across sessions.
- **Mattermost**: interactive model picker (PR #38767, ~572 LOC slash commands, ~641 LOC interaction handlers).
- **Slack**: reaction thread context routing; `app_mention` race condition deduplicated.
- **LINE**: comprehensive auth boundary synthesis; media download fixes.
- **Feishu**: video media send support; group mention detection added.
- **iMessage/BlueBubbles**: echo loop hardening; reply routing fixes; monitoring pipeline refactored.
- **WhatsApp**: media upload caps enforced.
- **Google Chat**: multi-account webhook authentication.
- **ACP**: persistent channel bindings added as new extension (~639 LOC).

*End of analysis. Total files analyzed: ~528 across 9 modules (including extensions/feishu).*
