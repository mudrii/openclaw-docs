# OpenClaw Channels & Messaging — Comprehensive Analysis
<!-- markdownlint-disable MD024 MD028 -->

> Updated: 2026-04-02 | Version: v2026.4.1 | Codebase: OpenClaw release tag `v2026.4.1`
> Modules analyzed: `extensions/telegram`, `extensions/discord`, `extensions/signal`, `extensions/slack`, `extensions/whatsapp`, `extensions/imessage`, `extensions/line`, `extensions/feishu`, `extensions/matrix`, `extensions/msteams`, `extensions/bluebubbles`, plus shared `src/channels` and `src/routing`

> **Release boundary note:** current released implementations for Telegram, Discord, Slack, Signal, WhatsApp, iMessage, Feishu, Matrix, and QQ Bot live under `extensions/*`. Shared channel infrastructure remains in `src/channels`, `src/routing`, `src/line`, and adjacent core modules.

> **v2026.2.22 Breaking:** Unified streaming config — most channels now use enum `off | partial | block | progress` in `channels.<channel>.streaming`. Telegram additionally accepts legacy boolean `streaming` and legacy `streamMode` values, mapping them to the enum (`true`→`partial`, `false`→`off`). Run `openclaw doctor --fix` to migrate legacy `streamMode` keys. Slack native streaming moved to `channels.slack.nativeStreaming`.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [src/channels — Shared Channel Infrastructure](#srcchannels)
3. [extensions/telegram — Telegram Bot API](#extensionstelegram)
4. [extensions/discord — Discord Bot (Carbon/Gateway)](#extensionsdiscord)
5. [extensions/signal — Signal via signal-cli JSON-RPC](#extensionssignal)
6. [extensions/slack — Slack (Bolt Socket Mode + HTTP)](#extensionsslack)
7. [extensions/whatsapp — WhatsApp Web (Baileys)](#extensionswhatsapp)
8. [extensions/imessage — iMessage (imsg RPC)](#extensionsimessage)
9. [extensions/line — LINE Messaging API](#extensionsline)
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
| `session.ts` | `recordInboundSession()` — persists session metadata on inbound messages |
| `ack-reactions.ts` | Ack reaction gating (scope: all/direct/group-all/group-mentions/off) |
| `allowlist-match.ts` | `AllowlistMatch` type and `formatAllowlistMatchMeta()` |
| `channel-config.ts` | Per-channel/group config resolution with key candidate matching |
| `command-gating.ts` | Command authorization from access groups |
| `conversation-label.ts` | Human-readable conversation label builder |
| `mention-gating.ts` | Mention gating logic (requireMention + bypass) |
| `reply-prefix.ts` | Response prefix context (model name, agent name) |
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

type ChatChannelId = "telegram" | "whatsapp" | "discord" | "irc" | "googlechat" | "slack" | "signal" | "imessage" | "line";
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

## extensions/telegram — Telegram Bot API {#extensionstelegram}

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

### Test Coverage (67 test files)
Tests cover: bot creation, message context building, dispatch, native commands, delivery, format/HTML, draft streaming, chunking, download, send (polls, edit, video-note, caption-split, proxy), inline buttons, model buttons, sticker cache, sent message cache, accounts, token resolution, targets, reaction level, webhook, probe, audit, network config/errors, group migration, update offset store, allowlist matching, and numerous e2e integration tests.

---

## extensions/discord — Discord Bot (Carbon/Gateway) {#extensionsdiscord}

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

### Test Coverage (74 test files)
Tests cover: API fetch, audit, chunking, gateway logging/registry, monitor (message handling, allow-list, exec approvals, threading, presence, inbound contract), send (basic messages, thread creation, voice message security), targets, token, probe intents, PluralKit.

---

## extensions/signal — Signal via signal-cli JSON-RPC {#extensionssignal}

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

## extensions/slack — Slack (Bolt Socket Mode + HTTP) {#extensionsslack}

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

## extensions/whatsapp — WhatsApp Web (Baileys) {#extensionswhatsapp}

### Module Overview
Full WhatsApp Web integration using Baileys. The implementation has moved to `extensions/whatsapp/src/` with 50+ files covering Baileys socket management, QR login, message monitoring, media handling, group support, and gateway delivery. `src/channels/web/index.ts` is a legacy re-export shim.

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

## extensions/imessage — iMessage (imsg RPC) {#extensionsimessage}

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

## extensions/line — LINE Messaging API {#extensionsline}

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

- **Docx tables + uploads** (#20304): New `feishu_doc` tool actions: `create_table`, `write_table_cells`, `upload_image`, `upload_file`. Markdown tables and positional insert are also supported. New file: `docx-batch-insert.ts`.
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

## v2026.3.8 Delta Notes

- **Telegram:** polling now runs through an explicit polling-session wrapper, long-poll shutdown aborts are forwarded correctly, DM preview lanes stay on message edits to avoid duplicate flashes, and inbound DM dedupe keys are agent-scoped rather than session-key-scoped.
- **Matrix:** DM fallback detection now requires a two-member room with no room name, explicit room bindings outrank DM heuristics, and room-bound agent selection is preserved for Matrix DM rooms.
- **Microsoft Teams:** route allowlists no longer widen sender access when `groupPolicy: "allowlist"` is configured; sender allowlists remain enforced after route resolution.
- **ACP / cross-channel lineage:** ACP-origin context can now surface provenance receipts and carry `originSessionId`, improving traceability when sessions cross editor/channel boundaries.
- **Bundled channel plugins:** onboarding clears short-lived discovery cache after plugin installs, and duplicate npm-installed channel plugins no longer shadow bundled channel plugins during onboarding/update sync.

---

## v2026.3.11 Delta Notes

### Discord

- **`autoArchiveDuration` for auto-created threads** (#35065): New `autoArchiveDuration` field on guild-channel config controls the archive timeout for threads created by the `autoThread` feature. Accepted values (as string or number): `"60"` (1h), `"1440"` (1d), `"4320"` (3d), `"10080"` (1w). Defined on `DiscordGuildChannelConfig` and `DiscordChannelConfigResolved` in `src/discord/monitor/allow-list.ts`; consumed in `src/discord/monitor/threading.ts`. The previous implicit behavior was equivalent to 1h.

- **Reaction ingress allowlist enforcement** (#43476 area): The `authorizeDiscordReactionIngress()` path in `src/discord/monitor/listeners.ts` now enforces the per-guild-channel users/roles allowlist, consistent with message ingress authorization. Previously the allowlist was checked on message events but could be bypassed on reaction events.

- **Reply chunking: `maxLinesPerMessage` and `chunkMode` propagated** (#40133): `resolveDiscordMaxLinesPerMessage()` is now called on all live reply paths (`message-handler.process.ts`, `native-command.ts`) and the resolved `chunkMode` is propagated into the fast-send path in `reply-delivery.ts`. This ensures long Discord replies respect the configured `channels.discord.maxLinesPerMessage` instead of splitting at the default 17-line limit.

- **`autoThread` on canonical guild-channel config type** (#35608): The `autoThread` boolean is now exposed on the canonical guild-channel config type in `src/discord/monitor/allow-list.ts`, making it accessible to downstream type consumers without casting.

- **Runtime config SecretRef threading** (#42352): Runtime-resolved config (including `SecretRef`-backed credentials) is now threaded through Discord send paths so credentials remain resolved during message delivery.

### Telegram

- **Outbound HTML chunking** (#42240): Long HTML-mode outbound messages are now chunked rather than sent as a single oversized payload. The plain-text fallback and silent-delivery (`disable_notification`) params are preserved across retry attempts. When HTML chunk planning cannot safely preserve the full message, delivery cuts over to plain text.

- **Final preview delivery: split active lifecycle from cleanup retention** (#41662): The active preview lifecycle is now tracked separately from cleanup-retain state, preventing missing archived preview edits from triggering a duplicate fallback send.

- **Final preview delivery: ambiguous `message_id` handling** (#41932): Ambiguous missing-`message_id` finals are only retained when the preview was already visible; this prevents ghost deliveries when a preview was never sent.

- **Final preview cleanup: clear stale cleanup-retain state** (#41763): Cleanup-retain state is cleared only for transient preview finals, not for persistent delivery state, preventing stale retain flags from accumulating across sessions.

- **Polling: clear cleanup timeout handles after stop** (#43188): Bounded cleanup timeout handles are cleared after `runner.stop()` and `bot.stop()` settle, so stall recovery no longer leaves stray 15-second timers behind on clean shutdown.

- **Polling: hang fix after stall detection** (#43424 area): A hang condition that could occur when restarting the poller after stall detection is resolved.

- **Config schema: `editMessage` and `createForumTopic` accepted** (#35498): Strict config validation now accepts `editMessage` and `createForumTopic` as recognized keys, eliminating spurious unknown-key warnings for configurations that use these options.

- **Exec approvals: reject `/approve` commands aimed at other bots** (#37233): `/approve` callback queries originating from interactions targeted at other bots are now rejected. Deterministic approval prompt visibility is preserved when tool-result delivery fails so the user always sees the approval prompt.

- **Direct delivery hooks** (#40185): Direct delivery sends (bypassing the normal reply pipeline) are now bridged to the internal `message:sent` hook, ensuring downstream hook listeners (e.g. logging, session recording) receive direct-delivery events.

- **Network env-proxy: apply transport policy to proxied dispatchers** (#40740): The configured transport policy (IP family preference, Happy Eyeballs settings) is now applied when creating the env-proxy HTTPS dispatcher in `src/telegram/fetch.ts`, fixing cases where proxy fetches bypassed the network policy.

- **Docs clarification** (#42451): `channels.telegram.groups` allowlists which chats the bot participates in, while `groupAllowFrom` allowlists which users inside those chats may interact with the bot. These are separate controls and not interchangeable.

- **Runtime config SecretRef threading** (#42352): Runtime-resolved config (including `SecretRef`-backed credentials) is now threaded through Telegram send paths so credentials remain resolved during message delivery.

### Signal

- **Config schema: `channels.signal.accountUuid` accepted** (#35578): Strict config validation now accepts `channels.signal.accountUuid`. This field (`src/config/zod-schema.providers-core.ts`, type `src/config/types.signal.ts`) identifies the bot's own Signal UUID for self-message filtering and was already functional but previously triggered unknown-key warnings under strict validation.

### Feishu (Lark)

- **Local image auto-convert: `mediaLocalRoots` passed through `sendText`** (#40623): The `mediaLocalRoots` allowlist is now forwarded into the `sendText` local-image shim in `extensions/feishu/src/outbound.ts`, so text payloads whose content resolves to a local image path correctly upload as Feishu images. Previously `mediaLocalRoots` was absent from the shim call and local image auto-conversion silently failed.

### Microsoft Teams

- **Allowlist resolution: General channel conversation ID used as team key** (#41838): When resolving a team allowlist entry, the General channel's conversation ID (as returned by the Graph API) is now used as the team key. This matches the `channelData.team.id` value that the Bot Framework runtime sends at message delivery time, fixing allowlist mismatches against the Graph API group GUID. Implemented in `extensions/msteams/src/resolve-allowlist.ts`.

### Mattermost

- **Markdown: first-line indentation preserved when stripping bot mentions** (#18655): Stripping bot mention prefixes no longer drops leading whitespace from the first line of the message, preserving indented list items and code blocks. Mattermost tables are now rendered natively by default rather than falling back to a fenced-code block.

- **Plugin send actions: `replyTo` fallback normalization** (#41176): Threaded plugin sends now trim blank `replyTo` IDs and reuse the correct reply target when the direct `replyTo` fallback path is taken, preventing malformed reply threads from blank-ID payloads.

### WhatsApp

- **Outbound whitespace: trim leading whitespace** (#43539): Leading whitespace in direct outbound sends is trimmed before delivery (`src/web/outbound.ts`), matching the existing behavior of other outbound paths.

### BlueBubbles

- **Self-chat echo dedupe: match on `fromMe` + same chat/body/timestamp** (#38442): Reflected duplicate webhook copies are now dropped only when a matching `fromMe` event was recently seen for the same chat, message body, and timestamp. This is the BlueBubbles-specific counterpart to the classic iMessage `is_from_me` fix, and it avoids suppressing legitimate self-chat traffic through the BlueBubbles bridge.

### Channels (shared infrastructure)

- **Stale matcher caching removed**: The allowlist matcher cache that was keyed on the array reference has been removed. Same-array allowlist edits and wildcard-entry replacements now take effect immediately on the next match evaluation rather than returning a stale compiled matcher.

*End of analysis. Total files analyzed: 823 across 9 modules (including extensions/feishu).*

---

## v2026.3.12 Delta Notes (2026-03-13)

### Slack

- **Agent reply Block Kit support** (#44592): `deliverReplies()` in `extensions/slack/src/monitor/replies.ts` now reads `payload.channelData?.slack?.blocks` via `readSlackReplyBlocks()` and forwards the parsed block array to `sendMessageSlack()`. Agents can deliver rich Block Kit layouts in shared reply delivery by supplying `channelData.slack.blocks` in the reply payload.

- **Interactive replies opt-in** (#44607): Slack button and select-menu reply directives are gated behind `channels.slack.capabilities.interactiveReplies` (disabled by default). The helper `isSlackInteractiveRepliesEnabled()` in `extensions/slack/src/interactive-replies.ts` reads the flag from per-account config (`config.capabilities.interactiveReplies`) or from the top-level `channels.slack.capabilities` object when a single account is configured. The flag is disabled by default; operators must explicitly set it to `true` to enable interactive reply directives.

  **New config key:** `channels.slack.capabilities.interactiveReplies` (boolean, default `false`)

- **Allowlist routing requires stable IDs**: Channel and team allowlist routing now requires stable Slack channel and team IDs. Mutable name-based matching is available only when `channels.slack.dangerouslyAllowNameMatching: true` is set.

  **Config key:** `channels.slack.dangerouslyAllowNameMatching` (boolean, default `false`)

- **Probe bot/team metadata mapping** (#44775, v2026.3.13): `probeSlack()` in `extensions/slack/src/probe.ts` now maps `auth.test()` response fields to the `bot` (`user_id`, `user`) and `team` (`team_id`, `team`) sub-objects on `SlackProbe`, stabilizing metadata availability for status and health checks.

### Telegram

- **Model picker session persistence** (#40105): Inline model button selections in `extensions/telegram/src/bot-handlers.ts` now persist the chosen model to the session store via `applyModelOverrideToSessionEntry()`. Selecting the default model clears any override rather than storing a redundant entry. The `/models` command validates fallback models against the configured provider catalog before offering them as selections.

- **Native command sync: suppress `BOT_COMMANDS_TOO_MUCH` noise**: Errors of type `BOT_COMMANDS_TOO_MUCH` from `setMyCommands` are now suppressed during native command menu sync; a fallback summary log is emitted instead to reduce noise in deployments with many registered commands.

- **Media download transport policy** (#44639, v2026.3.13): Direct and proxy transport policy is now threaded into SSRF-guarded file fetches in `extensions/telegram/src/bot/delivery.resolve-media.ts`, ensuring media downloads obey the configured network policy including SSRF pinning.

- **Inbound media IPv4 fallback** (v2026.3.13): SSRF-guarded downloads now retry with an explicit IPv4 fallback policy when the initial connection attempt fails, complementing the existing network-level IPv4 preference.

- **File URL redaction in error logs** (v2026.3.13): Telegram file URLs (which embed the bot token in the path) are now redacted in error log output via `redactSensitiveText()` to prevent bot token leaks through application logs.

- **Webhook auth before body parsing** (v2026.3.13): `extensions/telegram/src/webhook.ts` validates the webhook secret token before reading or parsing the request body. Unauthenticated requests are rejected immediately without consuming the body, as confirmed by the test `"rejects unauthenticated requests before reading the request body"`.

### Mattermost

- **Reply media delivery: `mediaLocalRoots` passthrough** (#44021): The agent-scoped `mediaLocalRoots` is now retrieved by `getAgentScopedMediaLocalRoots()` inside `extensions/mattermost/src/mattermost/reply-delivery.ts` and passed through the shared reply so locally allowed files are uploaded correctly.

- **Block streaming + threading: duplicate message fix** (#41362): Fixed a duplicate-message condition that occurred when block streaming was active together with thread-reply mode in Mattermost.

- **`replyToMode` support** (#29587): `extensions/mattermost/src/config-schema.ts` now accepts `replyToMode: z.enum(["off", "first", "all"])`. The value is surfaced in `MattermostAccountConfig` in `types.ts` and consumed in the channel plugin.

  **New config key:** `channels.mattermost.accounts.<id>.replyToMode` (`"off" | "first" | "all"`)

### BlueBubbles

- **Self-chat echo dedupe: `fromMe` matching** (#38442): Reflected duplicate webhook copies are dropped only when a matching `fromMe` event was recently seen for the same chat, message body, and timestamp. The logic lives in `extensions/bluebubbles/src/monitor-self-chat-cache.ts` and mirrors the iMessage fix below.

### iMessage

- **Self-chat echo dedupe: `is_from_me` matching** (#38440): Reflected duplicate RPC notifications are dropped only when a matching `is_from_me: true` event was seen for the same chat, text, and `created_at` timestamp. Implemented via `extensions/imessage/src/monitor/self-chat-cache.ts`, verified by tests in `inbound-processing.test.ts`.

- **Security: reject unsafe remote attachment paths** (v2026.3.13): Remote attachment path validation is applied in `extensions/imessage/src/monitor/monitor-provider.ts` before spawning the SCP copy, rejecting paths that could escape the expected remote root.

### Feishu (Lark)

- **Event dedupe: align with message-id contract** (#43762): The early duplicate-suppression check in `extensions/feishu/src/dedup.ts` now correctly aligns with Feishu's message-id contract. Pre-queue dedupe markers are released (via `processingClaims` TTL expiry) after a failed dispatch to allow reprocessing rather than permanently suppressing the event.

- **File uploads: preserve literal UTF-8 filenames** (#34262): `extensions/feishu/src/media.ts` now passes filenames through a sanitizer that preserves the original UTF-8 display name (Chinese characters, emoji, etc.) when calling `im.file.create`. Previous versions percent-encoded non-ASCII characters, which the Feishu API treated as literal display text rather than decoding, causing garbled filenames.

- **Security: require `encryptKey` alongside `verificationToken`** (`GHSA-g353-mgv3-8pcj`, #44087): Webhook mode now requires both `verificationToken` and `encryptKey` to be configured. Startup throws an explicit error if either is missing when `connectionMode` is `"webhook"`. Validation is enforced in both `extensions/feishu/src/config-schema.ts` (Zod schema refinement) and `extensions/feishu/src/monitor.account.ts` (runtime check).

- **Security: preserve group chat typing and fail closed on ambiguous reaction context** (`GHSA-m69h-jm2f-2pv8`, #44088): Looked-up group chat typing is preserved and ambiguous reaction contexts now fail closed to prevent information leakage.

### LINE

- **Security: require signatures for empty-event POST probes** (`GHSA-mhxh-9pjm-w7q5`, #44090): LINE webhook handlers now require a valid HMAC-SHA256 signature on empty-event POST requests (used by LINE as connectivity probes). Previously, empty-event payloads could be delivered without signature validation.

### Zalouser

- **Security: rate limit invalid secret guesses** (`GHSA-5m9r-p9g7-679c`, #44173): Invalid secret guesses on the Zalouser webhook endpoint are now rate-limited before authentication proceeds.

- **Security: require stable group IDs for allowlist auth**: Group allowlist authorization now requires stable Zalo group IDs. Mutable group-name matching is available only when `channels.zalouser.dangerouslyAllowNameMatching: true` is explicitly set in config.

  **Config key:** `channels.zalouser.dangerouslyAllowNameMatching` (boolean, optional) — defined in `extensions/zalouser/src/config-schema.ts` and `extensions/zalouser/src/types.ts`.

- **Markdown-to-Zalo text style parsing** (#43324): `extensions/zalouser/src/text-styles.ts` provides a full markdown-to-Zalo text style converter. Inline markers (bold, italic, code, color tags, underline, strikethrough) and block-level patterns are mapped to `TextStyle` values from `zca-client.js`.

- **Outbound chunker**: An outbound markdown-aware chunker is now wired in `extensions/zalouser/src/channel.ts` via the `chunker`/`chunkerMode: "markdown"` fields on the channel plugin descriptor.

### Discord

- **`autoArchiveDuration` for auto-created threads** (#35065): Confirmed from source — `autoArchiveDuration` is read from `DiscordGuildChannelConfig` in `extensions/discord/src/monitor/allow-list.ts` and applied in `extensions/discord/src/monitor/threading.ts` when creating auto-threads. Values: `"60"` (1h, default), `"1440"` (1d), `"4320"` (3d), `"10080"` (1w).

- **Gateway startup: treat `/gateway/bot` failures as transient** (#44397, v2026.3.13): `extensions/discord/src/monitor/gateway-plugin.ts` now classifies HTTP and JSON parse errors from the `/gateway/bot` metadata endpoint as transient (`createGatewayMetadataError({ transient: true })`). The gateway startup no longer propagates these as unhandled rejections; instead the error wraps with a recoverable cause so reconnect logic can handle it.

- **Allowlists: honor raw `guild_id` when hydrated guild object is missing** (v2026.3.13): `extensions/discord/src/monitor/message-handler.preflight.ts` uses `params.data.guild_id` (raw snowflake from the gateway event) as the authoritative guild identifier. When the hydrated `guild` object is absent (e.g. during cache miss), the raw `guild_id` is still used for allowlist lookups, preventing allowlisted channels and threads from being false-dropped.

### Signal

- **`channels.signal.groups` schema support** (#27199, v2026.3.13): Config validation now accepts `channels.signal.groups`, enabling per-group configuration entries for Signal group chats.

---

## v2026.3.22-v2026.3.23 Delta Notes (2026-03-24 docs snapshot)

Changes in the current released line are recorded inline above under each channel section where they first appear. Summary of what landed across `v2026.3.22`, `v2026.3.23`, and the shipped `v2026.3.23-2` correction build:

- **Discord**: privileged native commands now reply explicitly on auth failure; `message` tool schema again permits optional `components`; packaged install/runtime surfaces were stabilized.
- **Telegram**: per-account `apiRoot`, DM-topic auto-labeling, topic-edit, and `silentErrorReplies` landed in `v2026.3.22`; `v2026.3.23-2` now backfills `currentThreadTs` for DM topics and exposes `asDocument` as alias for `forceDocument`.
- **Feishu**: structured approval/launcher cards, current-conversation ACP binding, reasoning-stream cards, and fixed outbound `message(..., media=...)` routing all landed on the released line.
- **Matrix**: released plugin migration to `matrix-js-sdk`, `allowBots`, `allowPrivateNetwork`, and packaged-install runtime-api crash fixes.
- **LINE**: runtime-api export ordering fixed in the correction build to avoid packaged-install startup crashes.
- **Signal**: `channels.signal.groups` schema support remains in place on the current released line.

---

## v2026.3.24 Delta Notes

### Microsoft Teams

- **SDK migration**: migrated to official Teams SDK (`@microsoft/teams.apps`/`@microsoft/teams.api`). New files in `extensions/msteams/src/`: `streaming-message.ts`, `welcome-card.ts`, `feedback-reflection.ts`, `monitor-handler.ts`. Features: streaming 1:1 replies, welcome cards, prompt starters, feedback/reflection, typing indicators, AI labeling. Message edit/delete support via `graph-messages.ts`, `graph-thread.ts`.

### Discord

- **`autoThreadName: "generated"`**: optional config for LLM-generated thread titles in `extensions/discord/src/monitor/threading.ts`.
- **Gateway supervision**: new `gateway-supervisor.ts` with `DiscordGatewaySupervisor`, event classification, buffering/drain phases.
- **Timeout replies**: visible timeout when inbound worker times out before final reply.

### WhatsApp

- **Group echo tracking**: tracks gateway-sent message IDs, suppresses only matching echoes (preserves owner commands).
- **Reply-to-bot**: restored via `botInvokeMessage` unwrap + `selfLid` from `creds.json` in `extensions/whatsapp/src/identity.ts`.

### Telegram

- **Forum topics**: `#General` topic 1 routing recovery when Telegram omits forum metadata.
- **Outbound errors**: 403 details preserved, `bot not a member` as permanent failure.
- **Photos**: dimension/aspect-ratio preflight, document-send fallback.

### Slack

- **Interactive replies**: rich parity restored, `Options:` auto-render as buttons/selects.

### Gateway / Shared

- **Channel startup**: sequential with per-channel failure isolation.

### ACP

- **Direct chats**: terminal result delivery when TTS produces no audio.

### Feishu (Lark)

- **WebSocket**: connection cleanup on monitor stop.

---

## v2026.3.28 Delta Notes

### Telegram

- **Forum topic routing for /new and /reset** (#56654): The `/new` and `/reset` native commands now preserve the active forum topic (`message_thread_id`) when creating or resetting sessions. Previously these commands dropped the thread ID, causing subsequent bot replies to land in the General topic instead of the originating topic. Verified in `extensions/telegram/src/bot-handlers.runtime.ts`.

- **HTML-length search for long message splits** (#56595): Long outbound message splitting now measures chunk boundaries against verified HTML byte-length rather than a proportional plain-text estimate. The splitter uses `measureRendered: (html) => html.length` in `extensions/telegram/src/format.ts` so split points are validated against the actual serialized HTML, preventing oversized chunks from triggering Telegram 400 errors on messages with heavy markup.

- **Skip whitespace-only and hook-blanked text replies** (#56620): `deliverReplies()` in `extensions/telegram/src/bot/delivery.replies.ts` now drops text payloads that are whitespace-only or were blanked out by a delivery hook before calling Telegram. This prevents `GrammyError 400` errors caused by attempting to send empty or whitespace-only messages.

- **Shared `replyToMessageId` normalizer at all four API sinks** (#56587): A shared `readTelegramReplyToMessageId()` helper in `extensions/telegram/src/action-runtime.ts` is now used at all four outbound action sinks (send, edit, forward, and the draft-stream path). This ensures consistent numeric validation and coercion of `replyToMessageId` across all send paths.

- **Verbose tool summaries in forum topic sessions** (#43236): Verbose tool-result summaries are now routed correctly to the active forum topic session. The topic thread ID is carried through the tool-result delivery path so verbose output lands inside the correct topic thread rather than the General topic.

- **Ignore self-authored DM message updates** (#54530): Inbound DM message updates authored by the bot itself (e.g. bot-pinned cards, service messages) are now silently ignored rather than triggering a bogus pairing challenge. Verified by the test `"ignores private self-authored message updates instead of issuing a pairing challenge"` in `extensions/telegram/src/bot.create-telegram-bot.test.ts`.

### WhatsApp

- **Fix infinite echo loop in self-chat DM mode** (#54570): `extensions/whatsapp/src/inbound/monitor.ts` now tracks recently sent outbound message IDs via `isRecentOutboundMessage()` and drops `fromMe` echoes that match a recent outbound ID for the same `remoteJid`. This prevents the bot's own replies from being re-processed as new inbound user messages in self-chat DM mode.

- **Specific allowFrom policy error for valid blocked targets**: `extensions/whatsapp/src/resolve-outbound-target.ts` now surfaces a targeted error message when a normalized outbound target is valid but blocked by the configured `allowFrom` policy: `Target "${target}" is not listed in the configured WhatsApp allowFrom policy.` The normalized form of the target (not the raw input) is included in the message.

### Discord

- **Drain stale gateway sockets before forced fresh reconnects** (#54697): `extensions/discord/src/monitor/provider.lifecycle.reconnect.ts` introduces `disconnectGatewaySocketWithoutAutoReconnect()`, which strips the existing socket's `close`/`error` listeners and waits up to `DISCORD_GATEWAY_DISCONNECT_DRAIN_TIMEOUT_MS` (5 s) for a clean close before reconnecting. If the socket does not close in time, `socket.terminate()` is called; if terminate fails or the socket still refuses to close after a further 1 s, the reconnect fails closed rather than opening a parallel socket. `clearResumeState()` wipes `sessionId`, `resumeGatewayUrl`, and `sequence` from cached gateway state before fresh reconnects so Carbon does not attempt a RESUME on the new connection.

### Matrix

- **Encrypt thumbnails in E2EE rooms using `thumbnail_file`** (#54711): `extensions/matrix/src/matrix/send/media.ts` now sets `imageInfo.thumbnail_file` (the encrypted file descriptor) instead of `thumbnail_url` when the room is encrypted, ensuring thumbnail content is encrypted end-to-end in E2EE rooms. Unencrypted rooms continue to use `thumbnail_url`.

- **Load crypto-nodejs via `createRequire` to fix `__dirname` in ESM** (#54566): `extensions/matrix/src/matrix/sdk/crypto-node.runtime.ts` loads `@matrix-org/matrix-sdk-crypto-nodejs` with `const require = createRequire(import.meta.url)`, giving the CJS package access to `__dirname`. The bundled runtime test (`crypto-node.runtime.test.ts`) verifies the output contains `createRequire(import.meta.url)` and a `require(...)` call rather than a static ESM import.

- **TTS auto-replies as native Matrix voice bubbles** (#37080): Auto-TTS replies are now sent with `"org.matrix.msc3245.voice": {}` in the event content (`extensions/matrix/src/matrix/send/media.ts`), rendering them as native voice bubbles in Matrix clients that support MSC3245 instead of generic audio attachments.

### LINE

- **Timing-safe HMAC signature validation** (#55663): `extensions/line/src/signature.ts` now calls `crypto.timingSafeEqual()` unconditionally — padding both the computed hash and the provided signature to the same length before comparison. This closes the timing side-channel that previously allowed signature length to leak through early-exit comparison.

- **`collectStatusIssues` uses configured field instead of raw token** (#45701): `extensions/line/src/status.ts` now wires `collectLineStatusIssues` through `createDependentCredentialStatusIssueCollector` with `dependencySourceKey: "tokenSource"`. Status issue collection reads the resolved `tokenSource` field from the account snapshot rather than probing the raw token, aligning LINE with the shared status-issue pattern used by other channels.

- **Webhook mode carried into health monitor snapshots** (#47488): `resolveAccountSnapshot` in `extensions/line/src/status.ts` now includes `extra: { tokenSource: account.tokenSource, mode: "webhook" }` in every snapshot. The `mode` field makes the LINE connection mode visible in health monitor output.

### Microsoft Teams

- **Accept strict Bot Framework and Entra service tokens** (#56631): `extensions/msteams/src/sdk.ts` validates inbound JWT tokens against both the Bot Framework validator and an Entra `JwtValidator`. Both validators are tried in sequence; a token is accepted if either passes.

- **Accept `welcomeCard`, `groupWelcomeCard`, `promptStarters`, and `feedbackReflection` keys in strict config validation** (#54679): `extensions/msteams/src/monitor-handler.ts` reads `msteamsCfg.welcomeCard`, `msteamsCfg.groupWelcomeCard`, `msteamsCfg.promptStarters`, and `msteamsCfg.feedbackReflection` from the resolved account config. These keys are now recognized in runtime config resolution and no longer trigger unknown-key warnings under strict validation.

- **Preserve Entra JWT fallback on legacy validator errors**: When the Bot Framework JWT validator encounters an error (e.g. a legacy token format), the Entra validator path is still attempted before the request is rejected, preserving Entra-issued token acceptance.

### BlueBubbles

- **Guard debounce flush against null message text** (#56573): `extensions/bluebubbles/src/monitor-debounce.ts` now guards the debounce flush path against null or undefined message text, preventing flush failures when a message arrives with no text body.

- **Restore inbound prompt image refs for CLI-routed turns; reapply embedded runner image size guardrails** (#51373): Inbound prompt image references are restored for turns routed via the CLI path. Embedded runner image size guardrails are re-applied after the restore to prevent oversized payloads from reaching the inference loop.

- **Optional contacts enrichment with local macOS Contacts names after group gating** : `extensions/bluebubbles/src/monitor-processing.ts` calls `enrichBlueBubblesParticipantsWithContactNames()` after mention/group gating passes. Unnamed phone-number participants in group conversations are enriched with display names from the local macOS Contacts database. Gating is checked before the Contacts lookup so the lookup does not run for messages that will be dropped. Controlled by `config.enrichContactNames` (default `true`) in `extensions/bluebubbles/src/types.ts`.

- **ACP current-conversation bind**: Discord, BlueBubbles, and iMessage now support `/acp spawn codex --bind here` to bind the current conversation to a new ACP session. The BlueBubbles bind is implemented via `createBlueBubblesConversationBindingManager()` in `extensions/bluebubbles/src/conversation-bindings.ts` and wired into `extensions/bluebubbles/src/channel.ts`.

### iMessage

- **Stop leaking inline `[[reply_to:...]]` tags into delivered text** (#39512): `extensions/imessage/src/channel.outbound.test.ts` confirms that inline `[[reply_to:...]]` tags are stripped from outbound text and the `replyToId` is passed as the `reply_to` RPC metadata parameter instead. Stray `[[reply_to:...]]` tags without a matching `replyToId` option are also stripped.

- **ACP current-conversation bind**: iMessage now supports `/acp spawn codex --bind here` via `createIMessageConversationBindingManager()` in `extensions/imessage/src/conversation-bindings.ts`, wired into `extensions/imessage/src/channel.ts`.

### Slack

- **New explicit `upload-file` action routing file uploads through Slack upload transport**: `extensions/slack/src/message-action-dispatch.ts` maps the `upload-file` action name to the internal `uploadFile` handler. `extensions/slack/src/message-actions.ts` registers `"upload-file"` in the actions set alongside `"read"`, `"edit"`, `"delete"`, and `"download-file"`. The handler in `extensions/slack/src/action-runtime.ts` routes through `sendSlackMessage` with upload metadata (`uploadFileName`, `uploadTitle`).

### ACP / Channels

- **ACP current-conversation bind for Discord, BlueBubbles, and iMessage**: All three channels now expose `/acp spawn codex --bind here` to bind the active conversation to a new ACP session. The Discord implementation is tested end-to-end in `extensions/discord/src/monitor/acp-bind-here.integration.test.ts`; BlueBubbles and iMessage are implemented via dedicated `conversation-bindings.ts` modules in each extension.

### Feishu

- **Close WebSocket connections on monitor stop/abort** (#52844): `extensions/feishu/src/monitor.cleanup.test.ts` verifies that the WebSocket client is closed when the monitor aborts, when targeted stop cleanup runs, and during global stop cleanup. The monitor correctly tears down all open WebSocket connections on shutdown.

- **Use original message `create_time` instead of `Date.now()` for inbound timestamps** (#52809): `extensions/feishu/src/bot.ts` now parses `event.message.create_time` early in the inbound handler and uses it as the authoritative timestamp for the session and inbound payload. `Date.now()` is used only as a fallback when `create_time` is absent. This ensures pending-history entries, inbound payloads, and downstream consumers all use the original authoring timestamp.

## v2026.3.31 Delta Notes

### New Released Channel Surface

- **QQ Bot is part of the stable bundled channel set:** `extensions/qqbot/` joins the released tree with private chat, group chat, reminder, and media support.
- **LINE gains broader delivery and ACP coverage:** stable `v2026.3.31` adds LINE outbound image/video/audio sends and current-conversation ACP binding parity.

### Matrix / Slack / Teams / WhatsApp

- **Matrix adds stable room-history and threading controls:** `historyLimit`, proxy support, draft streaming, and per-DM `threadReplies` are all part of the released line.
- **Slack native exec approvals are released:** approval requests can stay in Slack instead of falling back to the Web UI or terminal.
- **Microsoft Teams ships `member-info`:** the released Teams action surface includes Graph-backed member lookup.
- **WhatsApp reactions are documented behavior now:** the stable line allows agents to react with emoji on inbound WhatsApp messages.

---

## v2026.4.1 Delta Notes

### Feishu (Lark)

- **Drive comment-event flow** (#58497): Dedicated Drive comment-event flow added with comment-thread context resolution, in-thread replies, and `feishu_drive` comment actions for document collaboration workflows. New files: `comment-dispatcher.ts` (routes comment events to the correct session), `comment-handler.ts` (processes comment content and dispatches to agent), `comment-target.ts` (resolves comment thread context for reply routing), `monitor.comment.ts` (monitors Feishu webhook events for Drive comment activity), `drive.ts` and `drive-schema.ts` (Drive API integration and config schema). This enables agents to participate in Feishu document comment threads as a collaboration channel alongside the existing chat flow.

### QQ Bot

- **Allowlist hardening:** `/bot-logs` export gated behind a truly explicit QQ Bot allowlist, rejecting wildcard and mixed wildcard entries while preserving the real framework command path.
- **Bundled plugin loading fix:** bundled channel plugins remain loadable from legacy `channels.<id>` config even under restrictive plugin allowlists; `openclaw doctor` warns only on real plugin blockers.

### WhatsApp

- **`reactionLevel` guidance:** agent reaction behavior configurable via `reactionLevel` guidance so agents can react with emoji on incoming messages with appropriate frequency/context.
- **Inbound message timestamp:** WhatsApp inbound message timestamps now passed to model context so the AI can see when messages were sent.

### Telegram

- **Configurable `errorPolicy` and `errorCooldownMs`:** per-account/chat/topic delivery error suppression controls so repeated errors can be silenced without muting distinct failures.
- **Topic-aware exec approvals:** forum-topic exec approval followups route through Telegram-owned threading and approval-target parsing, staying in the originating topic instead of falling back to the root chat.

### Slack

- **Exec approval alignment:** native Slack approval handling aligned with inferred approvers and real channel enablement so remote exec stops falling into false approval timeouts.

### Session Routing

- **Provider-specific session grammar moved to plugins:** provider-specific session conversation grammar moved into plugin-owned session-key surfaces, preserving Telegram topic routing and Feishu scoped inheritance across bootstrap, model override, restart, and tool-policy paths.
