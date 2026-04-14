# OpenClaw Changelog

Consolidated changelog assembled from versioned changelog files in this repository.

Release policy: this file tracks published releases only (stable tags). It does not track unreleased branch snapshots; when historically relevant, notes may mention npm republished builds tied to a stable tag. Docs-only rereleases in this repo use suffixed git tags such as `-1` while still documenting the same upstream stable release.

---

## OpenClaw v2026.4.12 — Release Summary

> **Released:** 2026-04-14 (docs release) | upstream GitHub release `v2026.4.12` published 2026-04-13 UTC | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.4.11..v2026.4.12` | 436 commits, 1200 files changed, +64,259 / -21,041 lines

### Changes

- **Feishu onboarding and auth hardening (`v2026.4.12`):** streamline QR scan-to-create onboarding; guard app registration fetches and local API barrel paths to avoid auth loops/regressions.
- **LM Studio provider behavior (`v2026.4.12`):** fix follow-up handling for header-auth flows and strengthen LM Studio provider runtime behavior in bundled/plugin runtime contexts.
- **Runtime packaging and dependencies (`v2026.4.12`):** refresh runtime dependency pinning/mirroring for Matrix and QA paths; restore 2026.4.11 appcast packaging safeguards while preparing the v2026.4.12 snapshot.
- **Security and approvals (`v2026.4.12`):** harden shell-wrapper detection, block env-argv assignment injection patterns, and prevent empty approval allowlists from enabling implicit authorization.
- **Session and control/runtime parity (`v2026.4.12`):** improve runtime session status extraction behavior, refresh Control UI slash-command discovery from live runtime command lists, and broaden QA channel attachment/media coverage.
- **Memory/utility reliability (`v2026.4.12`):** protect QMD startup and command path handling; keep qmd service startup and sync paths stable under the new release.
- **Docs release alignment (`v2026.4.12`):** synced repository metadata to upstream `v2026.4.12` and integrated freshly generated docs artifact sets (`docs/**`, i18n glossary, automation/release docs, updated provider/channel/plugin mirror files).

## OpenClaw v2026.4.12-1 — Release Summary

> **Released:** 2026-04-14 (docs correction) | upstream GitHub release `v2026.4.12` unchanged

### Corrections

- **Release metadata verification:** corrected README markdown catalog count to the validated 100-document total and aligned AGENT_README `v2026.4.12` behavioral checklist to explicitly documented 2026.4.11..2026.4.12 deltas.

## OpenClaw v2026.4.11-2 — Release Summary

> **Released:** 2026-04-13 (docs release) | upstream GitHub release `v2026.4.11` unchanged | **Policy note:** latest *documented* released section stays at top.

### Corrections

- **Changelog structure:** fixed an ordering/copy-paste issue where `v2026.4.10` and `v2026.4.9` release summaries were collapsed inside a single section.
- **Version alignment:** corrected docs snapshot/release metadata in architecture and repository overview files to match the validated `v2026.4.11` source tree (104 extension directories, 98 extension packages, 10,969 `.ts`/`.tsx` files).

## OpenClaw v2026.4.11 — Release Summary

> **Released:** 2026-04-12 (docs release) | upstream GitHub release `v2026.4.11` published 2026-04-12 UTC | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.4.10..v2026.4.11` | 276 commits, 1,717 files changed, +66,624 / -17,099 lines

### Changes

- **Dreaming/memory/wiki:** add imported-insight entrypoints for Dreaming (`Imported Insights`, `Memory Palace`) and expanded narrative/diary tooling for sourced memory workflows. (#64505)
- **Control UI/webchat:** add structured embed tags and richer media/reply/voice card rendering; gate embed URL behavior behind config surfaces. (#64104)
- **`video_generate` output and schema:** support URL-only generation responses, provider options metadata, optional audio references, per-asset role hints, and adaptive aspect-ratio inputs to improve plugin parity. (#61987, #61988)
- **Feishu:** improve document comment workflows with richer session context parsing, reaction handling, and typing feedback.
- **Microsoft Teams:** add reaction listing/actions and delegated OAuth handling improvements.
- **Plugins runtime:** plugin manifests can now declare activation/setup descriptors so setup flows can model provider/pairing/config requirements consistently.
- **Models/providers:** cache and classify OpenAI-compatible provider metadata with better debug visibility for capability and route diagnostics.
- **Gateway/QA:** add GPT-5.4 vs Opus 4.6 parity report gating and stricter scenario checks for qa verification.

### Fixes

- **OpenAI/Codex OAuth:** fix OAuth scope rewrite regressions for upstream authorize URL parameters.
- **Audio transcription:** scope DNS pinning adjustments to OpenAI-compatible multipart requests only; preserve host validation in other paths.
- **macOS Talk Mode:** after microphone permission grant, startup path continues without requiring a second UI toggle.
- **Control UI/webchat media:** persist TTS replies and tool-card pairing in webchat history so mixed output remains traceable.
- **WhatsApp routing:** preserve default account routing for active listener helpers and keep reconnect recovery for replies/media attached to expected accounts.
- **ACP/agent streaming:** stop leaking hidden tool-call payloads and progress chatter in parent ACP session streams.
- **Agent runtime:** align timeout/watchdog behavior with explicit run timeouts and avoid default timeouts ending valid long-running provider calls.
- **Config validation:** include async completion surfaces in generated schema and avoid orphaned required fields in Gemini tool unions.
- **Google Veo:** remove unsupported request fields that caused early failure before generation start.
- **Web search/web_fetch:** route QA and QA packaging artifacts from bundled release assets and avoid repo-only scenario coupling.

---

## OpenClaw v2026.4.10 — Release Summary

> **Released:** 2026-04-11 (docs release) | upstream GitHub release `v2026.4.10` published 2026-04-11 UTC | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.4.9..v2026.4.10` | 885 commits, 2,533 files changed, +99,364 / -25,126 lines

### Changes

- **Models/provider architecture:** add bundled Codex provider path and app-server auth seams so Codex-native model behavior stays in the provider plugin contract.
- **Memory:** add optional `extensions/active-memory` runtime for pre-reply recall and context-rich memory workflows with `/verbose` diagnostics and prompt tuning.
- **TTS/Talk:** add experimental local MLX provider flow for Talk mode, including local playback and interruption handling.
- **Teams and matrix QA/agent runtime:** add richer message actions and add dedicated verification lanes for Matrix/Telegram with improved coverage output and artifact handling.
- **Command surface:** introduce `commands.list` RPC and strengthen model/provider request routing, including allowPrivateNetwork controls for trusted self-hosted endpoints.
- **Gateway/exec:** add `openclaw exec-policy` for syncing and persisting execution policy settings; harden CLI exec host-policy interactions.
- **Provider transports:** route provider fallback and request transport metadata updates across runtime refresh paths and WebSocket manager caches.
- **Control UI/dreaming:** stabilize status surfaces and diary ordering for deterministic Dreaming visibility and scene/diary state transitions.

### Fixes

- **Browser security:** tighten SSRF and navigation hardening across strict profiles, redirects, CDP session recovery, and act/runtime enforcement paths.
- **Security and execution:** harden preflight reads, host allowlists, plugin install scanning, ACP tool hooks, Gmail watcher redaction, and oversized realtime frame handling.
- **Chat/communication:** preserve routing targets and threaded delivery for Slack, Telegram, WhatsApp, Teams, QQBot, and Matrix on edge cases and reconnects.
- **Model/fallback behavior:** preserve fallback state across transient model failures and classify model-provider errors with fewer false `reason=unknown` outcomes.
- **Tooling and tasks:** improve cron/task cancellation, session reset hooks, heartbeat recovery, flow ownership, and run-context cleanup to prevent stale states.
- **Config/tooling migrations:** avoid hard failures in schema migration and ensure `plugins.allow`, plugin command metadata, and auth-profile bindings resolve to current active providers.
- **UX and state handling:** stabilize `/btw` handling, ensure deterministic planning/summary ordering, and keep assistant output clean from hidden/reasoning tool payloads.

## OpenClaw v2026.4.9 — Release Summary

> **Released:** 2026-04-09 (docs release) | upstream GitHub release `v2026.4.9` published 2026-04-09 UTC | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.4.8..v2026.4.9` | 331 commits, 1,052 files changed, +40,773 / -24,675 lines

### Changes

- **Dreaming/memory:** surface the grounded scene lane in dreaming (#63395) so memory promotion can trace real-event anchors; feed grounded backfill into short-term promotion pipeline (#63370); harden grounded backfill follow-up logic and align dreaming status payloads.
- **MS Teams:** isolate channel thread sessions by `replyToId` so concurrent threads do not collide (#62713); route thread replies to the correct thread via `replyToId` (#62715); pin reply target at inbound time to prevent DM/channel leak (#62716).
- **Ollama:** enable thinking-block support for the Ollama API (#62712) so reasoning-capable local models can produce structured thinking output.
- **Slack:** deduplicate partial streaming replies (#62859); key turn-local dedupe by dispatch kind to eliminate duplicate blocks; treat ACP block text as visible output in Slack thread context (#62858).
- **Plugin setup wizard:** add explicit skip option so operators can bypass optional plugin wizard steps (#63436).
- **Providers:** add lightweight Anthropic Vertex provider discovery; keep Google provider policy lightweight for startup.
- **Auth:** filter provider auth aliases by plugin trust level to prevent cross-plugin auth leakage.
- **Plugin SDK:** split command status surface; keep command status compatibility path light.

### Fixes

- **Gateway:** suppress announce/reply skip chat leakage (#51739); drop raw gateway chat control replies that leak internal control messages; classify dream diary actions correctly.
- **Matrix/doctor:** add migration for legacy `channels.matrix.dm.policy: "trusted"` value (#62942) — run `openclaw doctor --fix` to migrate.
- **OpenRouter:** fix stale model picker refs (#63416) that caused model selection inconsistencies.
- **Browser:** surface delayed navigation blocks so callers receive explicit errors instead of silent timeouts.
- **Sessions/routing:** prevent inter-session messages from overwriting an established external `lastRoute`.
- **Windows:** repair dev-channel updater breakage.
- **Security/deps:** patch `basic-ftp` advisory.
- **Config:** stop owner-display barrel cycles; break console/logger type cycle; extract exec approvals allowlist types.
- **Commands:** split doctor prompt option types; split auth choice apply types.

---

## OpenClaw v2026.4.8 — Release Summary

> **Released:** 2026-04-08 (docs release) | upstream GitHub release `v2026.4.8` published 2026-04-08 UTC | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.4.7..v2026.4.8` | 42 commits, 269 files changed, +4,413 / -1,468 lines
> **Note:** incorporates `v2026.4.7-1` hotfix (revert bundled channel fallback masking).

### Changes

- **Slack/proxy:** honor `HTTPS_PROXY` and `HTTP_PROXY` environment variables for Socket Mode WebSocket connections (#62878); handle leading-dot `NO_PROXY` entries that match apex domains.
- **Slack/downloads:** pass resolved `botToken` through to `downloadSlackFile` so authenticated file downloads work in all routing paths (#62097).
- **Net/proxy:** skip DNS pinning before dispatching through trusted environment proxy, preventing false SSRF rejections on proxy-routed traffic.
- **Z.AI:** default to GLM-5.1 instead of the deprecated GLM-5 base model (#61998).
- **Docs:** refresh config schema baseline, slash command references, and TTS endpoint documentation.

### Fixes

- **Bundled channels:** revert bundled channel entry fallback resolution masking that broke plugin sidecar lookups (incorporated from `v2026.4.7-1`).

---

## OpenClaw v2026.4.7 — Release Summary

> **Released:** 2026-04-08 (docs release) | upstream GitHub release `v2026.4.7` published 2026-04-08 UTC | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.4.5..v2026.4.7` | 1,663 commits, 4,916 files changed, +206,144 / -104,901 lines

### Changes

- **Memory Wiki:** add belief-layer digests with compat migration, per-claim health reports, compiled digest prompts, and wiki-based memory retrieval so agents can perform structured recall against a maintained belief graph rather than raw memory search.
- **Compaction:** add pluggable compaction provider registry (#56224) so channel/plugin authors can register custom compaction strategies; add gateway compaction checkpoints (#62146) to stabilize long-session context management.
- **CLI / `openclaw infer`:** add first-class `openclaw infer` command (#62129) for direct inference workflows from the CLI without running a full gateway.
- **Arcee AI provider:** add bundled Arcee AI provider plugin with OpenRouter integration and updated onboarding docs.
- **Discord:** add cover image support to event creation (#60883).
- **Ollama:** auto-detect vision capability from `/api/show` and set image input flags accordingly (#62193).
- **Slack:** add `channels.slack.thread.requireExplicitMention` config option (#58276) so Slack thread replies only trigger the agent when explicitly mentioned.
- **iOS:** improve gateway connection error UX with clearer error messaging (#62650).
- **Context engine:** expose prompt-cache runtime context to context engines (#62179) so providers can make cache-aware prompt decisions.
- **Google:** add Gemma 4 model entries.
- **Media:** preserve media generation intent across provider fallback so partial-failure routes keep the original generation request intact.
- **GitHub:** add `gh-read` GitHub App helper for repository read operations.

### Fixes

- **Security/allowlist:** gate `/allowlist add` and `/allowlist remove` behind owner check before channel resolution (#62383).
- **Security/SSRF:** stop SSRF guard from rejecting operator-configured proxy hostnames (#62312); harden browser SSRF redirect guard against non-navigation document hops (#62355); drop request body on cross-origin unsafe-method redirects (#62357).
- **Browser:** align `browser.proxy` profile mutation guards (#60489).
- **Heartbeat:** always target main session — prevent routing to active subagent sessions (#61803).
- **Exec/env:** expand host-exec environment variable blocklist to cover Java, Rust, and Cargo toolchain variables (#62291); align inherited host-exec env filtering (#59119); expand git env denylist coverage (#62002).
- **Compaction:** fix agent infinite loop triggered after tool-use abortion (#62600).
- **Doctor:** warn when stale Codex provider overrides shadow OAuth credentials (#40143).
- **Daemon:** skip machine-scope DBus fallback on `permission-denied` errors (#62337).
- **Logging:** correct `levelToMinLevel` mapping and related filter logic for tslog v4 (#44646).
- **Sessions/routing:** prevent inter-session messages from overwriting an established external `lastRoute` (#58013).

---

## OpenClaw v2026.4.5 — Release Summary

> **Released:** 2026-04-06 (docs release) | upstream GitHub release `v2026.4.5` published 2026-04-06 UTC | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.4.2..v2026.4.5` | 2,562 commits, 5,746 files changed, +331,486 / -268,299 lines

### Breaking

- **Config/public aliases:** remove legacy public config aliases such as `talk.voiceId` / `talk.apiKey`, `agents.*.sandbox.perSession`, `browser.ssrfPolicy.allowPrivateNetwork`, `hooks.internal.handlers`, and channel/group/room `allow` toggles in favor of canonical public paths and `enabled`, while keeping load-time compatibility plus `openclaw doctor --fix` migration support for existing configs. (#60726) Thanks @vincentkoc.

### Changes

- **Media generation:** add built-in `video_generate` and `music_generate` tools, with bundled xAI, Alibaba Model Studio Wan, Runway, Google Lyria, MiniMax, and Comfy-backed provider paths for direct media generation plus async completion tracking.
- **Comfy workflows:** add the bundled `comfy` workflow media plugin for local ComfyUI and Comfy Cloud workflows, covering shared image/video generation and workflow-backed music generation with optional reference-image upload.
- **Providers:** add bundled Qwen, Fireworks AI, and StepFun providers; add Bedrock Mantle support with inference-profile discovery and automatic request-region injection for Bedrock-hosted Claude, GPT-OSS, Qwen, Kimi, GLM, and related routes.
- **Control UI:** add multilingual support across the released Control UI, add plugin-config prompts to guided onboarding, and bring ClawHub search/detail/install flows directly into the Skills panel. (#60590, #60544, #60134)
- **Exec approvals:** add generic APNs approval notifications that open an in-app modal after authenticated operator reconnect, and add Matrix-native exec approval prompts with account-scoped approvers plus room/thread-aware resolution. (#60239, #58635)
- **Channels/context routing:** add per-channel `contextVisibility` (`all`, `allowlist`, `allowlist_quote`) so supplemental context can be filtered by sender allowlists instead of always passing through unchanged.
- **Provider transport overrides:** add shared request transport overrides for model and media requests across OpenAI-, Anthropic-, Google-, and compatible provider paths, including headers, auth, proxy, and TLS controls. (#60200)
- **OpenAI/Codex runtime:** add forward-compat `openai-codex/gpt-5.4-mini`, an opt-in GPT personality, and provider-owned GPT-5 prompt contributions so Codex/GPT runs stay cache-stable even when bundled catalogs lag.
- **Claude CLI / ACPX:** expose OpenClaw tools to background Claude CLI runs through a loopback MCP bridge, embed ACPX directly in the bundled plugin, and add a generic `reply_dispatch` hook so bundled plugins can own reply interception without hardcoded core routing.
- **Progress reporting:** add experimental structured plan updates and structured execution item events so compatible UIs can render clearer step-by-step progress during long-running runs.
- **Memory / dreaming:** add weighted short-term recall promotion, `/dreaming`, Dreams UI, multilingual conceptual tagging, configurable aging controls, REM preview tooling, replay-safe promotion, and the `dreams.md` trail model for durable memory promotion.
- **Prompt caching:** stabilize cache-relevant prompt fingerprints, keep stable files above volatile `HEARTBEAT.md`, remove duplicate in-band tool inventories from system prompts, and surface cache diagnostics more clearly in `openclaw status --verbose`.
- **Config/schema:** enrich `openclaw config schema` JSON Schema output with field titles and descriptions so editors, agents, and other schema consumers get the same config help metadata. (#60067)

### Fixes

- **Security:** preserve restrictive plugin-only tool allowlists, require owner access for `/allowlist add` and `/allowlist remove`, fail closed when `before_tool_call` hooks crash, and block browser SSRF redirect bypasses earlier in the request path. (#58476, #59836, #59822, #58771, #59120)
- **OpenAI reply delivery:** make GPT-5 and Codex runs act sooner with lower-verbosity defaults, retry one-shot planning-only turns, preserve native strict schemas and `reasoning.effort: "none"` where supported, and keep commentary buffered until `final_answer` so planning text no longer leaks into user-visible replies. Fixes #59150, #59643, #61282.
- **Telegram:** fix model-picker checks, topic replies, reaction ownership persistence, media placeholder preservation, DM voice-note preflight transcription, reasoning preview leaks, and long command-menu truncation.
- **Discord / Slack / WhatsApp:** keep Discord traffic on configured proxies, preserve reply-tag threading and generated media paths, route Slack DM replies back to the concrete inbound DM channel, and restore WhatsApp streaming-block config plus reconnect watchdog resets.
- **Matrix / Teams:** harden Matrix exec approvals and DM session scoping, improve Matrix recovery when secret storage is missing, and fix Teams inline-image downloads plus proactive thread fallback.
- **Gateway restarts:** improve gateway restart reliability across macOS and Windows, default `gateway.mode` to `local` when unset, detect lockfile PID recycling, and recover installed-but-unloaded launch agents or scheduled-task restarts more accurately.
- **CLI / Cron:** route `skills --json` output to stdout again, preserve Commander-computed exit codes, replay interrupted recurring jobs on the first gateway restart, and send cron failure notifications through the job's primary delivery channel when `failureDestination` is unset.
- **Claude CLI hardening:** clear inherited Claude Code config/plugin/provider-routing env overrides, force host-managed runs to `--setting-sources user`, keep explicit `--session-id` bindings resumable, and stabilize image path hydration for local CLI runs.
- **Plugin runtime / tasks:** preserve plugin activation provenance, reuse compatible runtime registries and snapshot caches, export missing context-engine/plugin SDK types, and reconcile stale cron/chat-backed task rows against live ownership instead of treating persisted session keys as proof of liveness.

## OpenClaw v2026.4.2 — Release Summary

> **Released:** 2026-04-03 (docs release) | upstream GitHub release `v2026.4.2` published 2026-04-02 UTC | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.4.1..v2026.4.2` | 284 commits, 882 files changed, +51,562 / -9,916 lines

### Breaking

- **Plugins/xAI:** move `x_search` settings from the legacy core `tools.web.x_search.*` path to the plugin-owned `plugins.entries.xai.config.xSearch.*` path, standardize `x_search` auth on `plugins.entries.xai.config.webSearch.apiKey` / `XAI_API_KEY`, and migrate legacy config with `openclaw doctor --fix`. (#59674) Thanks @vincentkoc.
- **Plugins/web fetch:** move Firecrawl `web_fetch` config from the legacy core `tools.web.fetch.firecrawl.*` path to the plugin-owned `plugins.entries.firecrawl.config.webFetch.*` path, route `web_fetch` fallback through the new fetch-provider boundary instead of a Firecrawl-only core branch, and migrate legacy config with `openclaw doctor --fix`. (#59465) Thanks @vincentkoc.

### Changes

- **Tasks/Task Flow:** restore the core Task Flow substrate with managed-vs-mirrored sync modes, durable flow state/revision tracking, and `openclaw tasks flow` inspection/recovery primitives so background orchestration can persist and be operated separately from plugin authoring layers. (#58930) Thanks @mbelinky.
- **Tasks/Task Flow:** add managed child task spawning plus sticky cancel intent, so external orchestrators can stop scheduling immediately and let parent Task Flows settle to `cancelled` once active child tasks finish. (#59610) Thanks @mbelinky.
- **Plugins/Task Flow:** add a bound `api.runtime.taskFlow` seam so plugins and trusted authoring layers can create and drive managed Task Flows from host-resolved OpenClaw context without passing owner identifiers on each call. (#59622) Thanks @mbelinky.
- **Android/assistant:** add assistant-role entrypoints plus Google Assistant App Actions metadata so Android can launch OpenClaw from the assistant trigger and hand prompts into the chat composer. (#59596) Thanks @obviyus.
- **Exec defaults:** make gateway/node host exec default to YOLO mode by requesting `security=full` with `ask=off`, and align host approval-file fallbacks plus docs/doctor reporting with that no-prompt default.
- **Providers/runtime:** add provider-owned replay hook surfaces for transcript policy, replay cleanup, and reasoning-mode dispatch. (#59143) Thanks @jalehman.
- **Plugins/hooks:** add `before_agent_reply` so plugins can short-circuit the LLM with synthetic replies after inline actions. (#20067) Thanks @JoshuaLelon.
- **Channels/session routing:** move provider-specific session conversation grammar into plugin-owned session-key surfaces, preserving Telegram topic routing and Feishu scoped inheritance across bootstrap, model override, restart, and tool-policy paths.
- **Feishu/comments:** add a dedicated Drive comment-event flow with comment-thread context resolution, in-thread replies, and `feishu_drive` comment actions for document collaboration workflows. (#58497) Thanks @wittam-01.
- **Matrix/plugin:** emit spec-compliant `m.mentions` metadata across text sends, media captions, edits, poll fallback text, and action-driven edits so Matrix mentions notify reliably in clients like Element. (#59323) Thanks @gumadeiras.
- **Diffs:** add plugin-owned `viewerBaseUrl` so viewer links can use a stable proxy/public origin without passing `baseUrl` on every tool call. (#59341) Related #59227. Thanks @gumadeiras.
- **Agents/compaction:** resolve `agents.defaults.compaction.model` consistently for manual `/compact` and other context-engine compaction paths, so engine-owned compaction uses the configured override model across runtime entrypoints. (#56710) Thanks @oliviareid-svg.
- **Agents/compaction:** add `agents.defaults.compaction.notifyUser` so the `🧹 Compacting context...` start notice is opt-in instead of always being shown. (#54251) Thanks @oguricap0327.
- **WhatsApp/reactions:** add `reactionLevel` guidance for agent reactions. Thanks @mcaxtr.
- **Exec approvals/channels:** auto-enable DM-first native chat approvals when supported channels can infer approvers from existing owner config, while keeping channel fanout explicit and clarifying forwarding versus native approval client config.

### Fixes

- **Providers/transport policy:** centralize request auth, proxy, TLS, and header shaping across shared HTTP, stream, and websocket paths, block insecure TLS/runtime transport overrides, and keep proxy-hop TLS separate from target mTLS settings. (#59682) Thanks @vincentkoc.
- **Providers/Copilot:** classify native GitHub Copilot API hosts in the shared provider endpoint resolver and harden token-derived proxy endpoint parsing so Copilot base URL routing stays centralized and fails closed on malformed hints. (#59644) Thanks @vincentkoc.
- **Providers/streaming headers:** centralize default and attribution header merging across OpenAI websocket, embedded-runner, and proxy stream paths so provider-specific headers stay consistent and caller overrides only win where intended. (#59542) Thanks @vincentkoc.
- **Providers/media HTTP:** centralize base URL normalization, default auth/header injection, and explicit header override handling across shared OpenAI-compatible audio, Deepgram audio, Gemini media/image, and Moonshot video request paths. (#59469) Thanks @vincentkoc.
- **Providers/OpenAI-compatible routing:** centralize native-vs-proxy request policy so hidden attribution and related OpenAI-family defaults only apply on verified native endpoints across stream, websocket, and shared audio HTTP paths. (#59433) Thanks @vincentkoc.
- **Providers/Anthropic routing:** centralize native-vs-proxy endpoint classification for direct Anthropic `service_tier` handling so spoofed or proxied hosts do not inherit native Anthropic defaults. (#59608) Thanks @vincentkoc.
- **Gateway/exec loopback:** restore legacy-role fallback for empty paired-device token maps and allow silent local role upgrades so local exec and node clients stop failing with pairing-required errors after `2026.3.31`. (#59092) Thanks @openperf.
- **Agents/subagents:** pin admin-only subagent gateway calls to `operator.admin` while keeping `agent` at least privilege, so `sessions_spawn` no longer dies on loopback scope-upgrade pairing with `close(1008) "pairing required"`. (#59555) Thanks @openperf.
- **Exec approvals/config:** strip invalid `security`, `ask`, and `askFallback` values from `~/.openclaw/exec-approvals.json` during normalization so malformed policy enums fall back cleanly to the documented defaults instead of corrupting runtime policy resolution. (#59112) Thanks @openperf.
- **Exec approvals/doctor:** report host policy sources from the real approvals file path and ignore malformed host override values when attributing effective policy conflicts. (#59367) Thanks @gumadeiras.
- **Exec/runtime:** treat `tools.exec.host=auto` as routing-only, keep implicit no-config exec on sandbox when available or gateway otherwise, and reject per-call host overrides that would bypass the configured sandbox or host target. (#58897) Thanks @vincentkoc.
- **Slack/mrkdwn formatting:** add built-in Slack mrkdwn guidance in inbound context so Slack replies stop falling back to generic Markdown patterns that render poorly in Slack. (#59100) Thanks @jadewon.
- **WhatsApp/presence:** send `unavailable` presence on connect in self-chat mode so personal-phone users stop losing all push notifications while the gateway is running. (#59410) Thanks @mcaxtr.
- **WhatsApp/media:** add HTML, XML, and CSS to the MIME map and fall back gracefully for unknown media types instead of dropping the attachment. (#51562) Thanks @bobbyt74.
- **Matrix/onboarding:** restore guided setup in `openclaw channels add` and `openclaw configure --section channels`, while keeping custom plugin wizards on the shared `setupWizard` seam. (#59462) Thanks @gumadeiras.
- **Matrix/streaming:** keep live partial previews for the current assistant block while preserving completed block updates as separate messages when `channels.matrix.blockStreaming` is enabled. (#59384) Thanks @gumadeiras.
- **Feishu/comment threads:** harden document comment-thread delivery so whole-document comments fall back to `add_comment`, delayed reply lookups retry more reliably, and user-visible replies avoid reasoning/planning spillover. (#59129) Thanks @wittam-01.
- **MS Teams/streaming:** strip already-streamed text from fallback block delivery when replies exceed the 4000-character streaming limit so long responses stop duplicating content. (#59297) Thanks @bradgroux.
- **Slack/thread context:** filter thread starter and history by the effective conversation allowlist without dropping valid open-room, DM, or group DM context. (#58380) Thanks @jacobtomlinson.
- **Mattermost/probes:** route status probes through the SSRF guard and honor `allowPrivateNetwork` so connectivity checks stay safe for self-hosted Mattermost deployments. (#58529) Thanks @mappel-nv.
- **Zalo/webhook replay:** scope replay dedupe key by chat and sender so reused message IDs across different chats or senders no longer collide, and harden metadata reads for partially missing payloads. (#58444)
- **QQBot/structured payloads:** restrict local file paths to QQ Bot-owned media storage, block traversal outside that root, reduce path leakage in logs, and keep inline image data URLs working. (#58453) Thanks @jacobtomlinson.
- **Image generation/providers:** route OpenAI, MiniMax, and fal image requests through the shared provider HTTP transport path so custom base URLs, guarded private-network routing, and provider request defaults stay aligned with the rest of provider HTTP. Thanks @vincentkoc.
- **Image generation/providers:** stop inferring private-network access from configured OpenAI, MiniMax, and fal image base URLs, and cap shared HTTP error-body reads so hostile or misconfigured endpoints fail closed without relaxing SSRF policy or buffering unbounded error payloads. Thanks @vincentkoc.
- **Browser/host inspection:** keep static Chrome inspection helpers out of the activated browser runtime so `openclaw doctor browser` and related checks do not eagerly load the bundled browser plugin. (#59471) Thanks @vincentkoc.
- **Browser/CDP:** normalize trailing-dot localhost absolute-form hosts before loopback checks so remote CDP websocket URLs like `ws://localhost.:...` rewrite back to the configured remote host. (#59236) Thanks @mappel-nv.
- **Agents/output sanitization:** strip namespaced `antml:thinking` blocks from user-visible text so Anthropic-style internal monologue tags do not leak into replies. (#59550) Thanks @obviyus.
- **Kimi Coding/tools:** normalize Anthropic tool payloads into the OpenAI-compatible function shape Kimi Coding expects so tool calls stop losing required arguments. (#59440) Thanks @obviyus.
- **Image tool/paths:** resolve relative local media paths against the agent `workspaceDir` instead of `process.cwd()` so inputs like `inbox/receipt.png` pass the local-path allowlist reliably. (#57222) Thanks Priyansh Gupta.
- **Podman/launch:** remove noisy container output from `scripts/run-openclaw-podman.sh` and align the Podman install guidance with the quieter startup flow. (#59368) Thanks @sallyom.
- **Plugins/runtime:** keep LINE reply directives and browser-backed cleanup/reset flows working even when those plugins are disabled while tightening bundled plugin activation guards. (#59412) Thanks @vincentkoc.
- **ACP/gateway reconnects:** keep ACP prompts alive across transient websocket drops while still failing boundedly when reconnect recovery does not complete. (#59473) Thanks @obviyus.
- **ACP/gateway reconnects:** reject stale pre-ack ACP prompts after reconnect grace expiry so callers fail cleanly instead of hanging indefinitely when the gateway never confirms the run.
- **Gateway/session kill:** enforce HTTP operator scopes on session kill requests and gate authorization before session lookup so unauthenticated callers cannot probe session existence. (#59128) Thanks @jacobtomlinson.
- **MS Teams/logging:** format non-`Error` failures with the shared unknown-error helper so logs stop collapsing caught SDK or Axios objects into `[object Object]`. (#59321) Thanks @bradgroux.
- **Channels/setup:** ignore untrusted workspace channel plugins during setup resolution so a shadowing workspace plugin cannot override built-in channel setup/login flows unless explicitly trusted in config. (#59158) Thanks @mappel-nv.
- **Exec/Windows:** restore allowlist enforcement with quote-aware `argPattern` matching across gateway and node exec, and surface accurate dynamic pre-approved executable hints in the exec tool description. (#56285) Thanks @kpngr.
- **Gateway:** prune empty `node-pending-work` state entries after explicit acknowledgments and natural expiry so the per-node state map no longer grows indefinitely. (#58179) Thanks @gavyngong.
- **Webhooks/secret comparison:** replace ad-hoc timing-safe secret comparisons across BlueBubbles, Feishu, Mattermost, Telegram, Twilio, and Zalo webhook handlers with the shared `safeEqualSecret` helper and reject empty auth tokens in BlueBubbles. (#58432) Thanks @eleqtrizit.
- **OpenShell/mirror:** constrain `remoteWorkspaceDir` and `remoteAgentWorkspaceDir` to the managed `/sandbox` and `/agent` roots, and keep mirror sync from overwriting or removing user-added shell roots during config synchronization. (#58515) Thanks @eleqtrizit.
- **Plugins/activation:** preserve explicit, auto-enabled, and default activation provenance plus reason metadata across CLI, gateway bootstrap, and status surfaces so plugin enablement state stays accurate after auto-enable resolution. (#59641) Thanks @vincentkoc.
- **Exec/env:** block additional host environment override pivots for package roots, language runtimes, compiler include paths, and credential/config locations so request-scoped exec cannot redirect trusted toolchains or config lookups. (#59233) Thanks @drobison00.
- **Dotenv/workspace overrides:** block workspace `.env` files from overriding `OPENCLAW_PINNED_PYTHON` and `OPENCLAW_PINNED_WRITE_PYTHON` so trusted helper interpreters cannot be redirected by repo-local env injection. (#58473) Thanks @eleqtrizit.
- **Plugins/install:** accept JSON5 syntax in `openclaw.plugin.json` and bundle `plugin.json` manifests during install/validation, so third-party plugins with trailing commas, comments, or unquoted keys no longer fail to install. (#59084) Thanks @singleGanghood.
- **Telegram/exec approvals:** rewrite shared `/approve … allow-always` callback payloads to `/approve … always` before Telegram button rendering so plugin approval IDs still fit Telegram's `callback_data` limit and keep the Allow Always action visible. (#59217) Thanks @jameslcowan.
- **Cron/exec timeouts:** surface timed-out `exec` and `bash` failures in isolated cron runs even when `verbose: off`, including custom session-target cron jobs, so scheduled runs stop failing silently. (#58247) Thanks @skainguyen1412.
- **Telegram/exec approvals:** fall back to the origin session key for async approval followups and keep resume-failure status delivery sanitized so Telegram followups still land without leaking raw exec metadata. (#59351) Thanks @seonang.
- **Node-host/exec approvals:** bind `pnpm dlx` invocations through the approval planner's mutable-script path so the effective runtime command is resolved for approval instead of being left unbound. (#58374)
- **Exec/node hosts:** stop forwarding the gateway workspace cwd to remote node exec when no workdir was explicitly requested, so cross-platform node approvals fall back to the node default cwd instead of failing with `SYSTEM_RUN_DENIED`. (#58977) Thanks @Starhappysh.
- **Exec approvals/channels:** decouple initiating-surface approval availability from native delivery enablement so Telegram, Slack, and Discord still expose approvals when approvers exist and native target routing is configured separately. (#59776) Thanks @joelnishanth.
- **Plugins/runtime:** reuse compatible active registries for `web_search` and `web_fetch` provider snapshot resolution so repeated runtime reads do not re-import the same bundled plugin set on each agent message. Related #48380.

---

## OpenClaw v2026.4.1 — Release Summary

> **Released:** 2026-04-02 (docs release) | upstream GitHub release `v2026.4.1` published 2026-04-01 UTC | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.3.31..v2026.4.1` | 252 commits, 932 files changed, +27,149 / -10,402 lines

### Changes

- **Tasks/chat:** add `/tasks` as a chat-native background task board for the current session, with recent task details and agent-local fallback counts when no linked tasks are visible. Related #54226. Thanks @vincentkoc.
- **Web search/SearXNG:** add the bundled SearXNG provider plugin for `web_search` with configurable host support. (#57317) Thanks @cgdusek.
- **Amazon Bedrock/Guardrails:** add Bedrock Guardrails support to the bundled provider. (#58588) Thanks @MikeORed.
- **macOS/Voice Wake:** add the Voice Wake option to trigger Talk Mode. (#58490) Thanks @SmoothExec.
- **Feishu/comments:** add a dedicated Drive comment-event flow with comment-thread context resolution, in-thread replies, and `feishu_drive` comment actions for document collaboration workflows. (#58497) Thanks @wittam-01.
- **Gateway/webchat:** make `chat.history` text truncation configurable with `gateway.webchat.chatHistoryMaxChars` and per-request `maxChars`, while preserving silent-reply filtering and existing default payload limits. (#58900)
- **Agents/default params:** add `agents.defaults.params` for global default provider parameters. (#58548) Thanks @lpender.
- **Agents/failover:** cap prompt-side and assistant-side same-provider auth-profile retries for rate-limit failures before cross-provider model fallback, add the `auth.cooldowns.rateLimitedProfileRotations` knob, and document the new fallback behavior. (#58707) Thanks @Forgely3D.
- **Cron/tools allowlist:** add `openclaw cron --tools` for per-job tool allowlists. (#58504) Thanks @andyk-ms.
- **Channels/session routing:** move provider-specific session conversation grammar into plugin-owned session-key surfaces, preserving Telegram topic routing and Feishu scoped inheritance across bootstrap, model override, restart, and tool-policy paths.
- **WhatsApp/reactions:** add `reactionLevel` guidance for agent reactions. Thanks @mcaxtr.
- **Telegram/errors:** add configurable `errorPolicy` and `errorCooldownMs` controls so Telegram can suppress repeated delivery errors per account, chat, and topic without muting distinct failures. (#51914) Thanks @chinar-amrutkar.
- **Z.AI/models:** add `glm-5.1` and `glm-5v-turbo` to the bundled Z.AI provider catalog. (#58793) Thanks @tomsun28.
- **Agents/compaction:** resolve `agents.defaults.compaction.model` consistently for manual `/compact` and other context-engine compaction paths, so engine-owned compaction uses the configured override model across runtime entrypoints. (#56710) Thanks @oliviareid-svg.

### Fixes

- **Chat/error replies:** stop leaking raw provider/runtime failures into external chat channels, return a friendly retry message instead, and add a specific `/new` hint for Bedrock toolResult/toolUse session mismatches. (#58831) Thanks @ImLukeF.
- **Gateway/reload:** ignore startup config writes by persisted hash in the config reloader so generated auth tokens and seeded Control UI origins do not trigger a restart loop, while real `gateway.auth.*` edits still require restart. (#58678) Thanks @yelog.
- **Tasks/gateway:** keep the task registry maintenance sweep from stalling the gateway event loop under synchronous SQLite pressure, so upgraded gateways stop hanging about a minute after startup. (#58670) Thanks @openperf.
- **Tasks/status:** hide stale completed background tasks from `/status` and `session_status`, prefer live task context, and show recent failures only when no active work remains. (#58661) Thanks @vincentkoc.
- **Tasks/gateway:** re-check the current task record before maintenance marks runs lost or prunes them, so a task heartbeat or cleanup update that lands during a sweep no longer gets overwritten by stale snapshot state.
- **Exec/approvals:** honor `exec-approvals.json` security defaults when inline or configured tool policy is unset, and keep Slack and Discord native approval handling aligned with inferred approvers and real channel enablement so remote exec stops falling into false approval timeouts and disabled states. Thanks @scoootscooob and @vincentkoc.
- **Exec/approvals:** make `allow-always` persist as durable user-approved trust instead of behaving like `allow-once`, reuse exact-command trust on shell-wrapper paths that cannot safely persist an executable allowlist entry, keep static allowlist entries from silently bypassing `ask:"always"`, and require explicit approval when Windows cannot build an allowlist execution plan instead of hard-dead-ending remote exec. Thanks @scoootscooob and @vincentkoc.
- **Exec/cron:** resolve isolated cron no-route approval dead-ends from the effective host fallback policy when trusted automation is allowed, and make `openclaw doctor` warn when `tools.exec` is broader than `~/.openclaw/exec-approvals.json` so stricter host-policy conflicts are explicit. Thanks @scoootscooob and @vincentkoc.
- **Sessions/model switching:** keep `/model` changes queued behind busy runs instead of interrupting the active turn, and retarget queued followups so later work picks up the new model as soon as the current turn finishes.
- **Gateway/HTTP:** skip failing HTTP request stages so one broken facade no longer forces every HTTP endpoint to return 500. (#58746) Thanks @yelog.
- **Gateway/nodes:** stop pinning live node commands to the approved node-pair record. Node pairing remains a trust/token flow, while per-node `system.run` policy stays in that node's exec approvals config. Fixes #58824.
- **WebChat/exec approvals:** use native approval UI guidance in agent system prompts instead of telling agents to paste manual `/approve` commands in webchat sessions. Thanks @vincentkoc.
- **Web UI/OpenResponses:** preserve rewritten stream snapshots in webchat and keep OpenResponses final streamed text aligned when models rewind earlier output. (#58641) Thanks @neeravmakwana.
- **Discord/inbound media:** pass Discord attachment and sticker downloads through the shared idle-timeout and worker-abort path so slow or stuck inbound media fetches stop hanging message processing. (#58593) Thanks @aquaright1.
- **Telegram/retries:** keep non-idempotent sends on the strict safe-send path, retry wrapped pre-connect failures, and preserve `429` / `retry_after` backoff for safe delivery retries. (#51895) Thanks @chinar-amrutkar.
- **Telegram/exec approvals:** route topic-aware exec approval followups through Telegram-owned threading and approval-target parsing, so forum-topic approvals stay in the originating topic instead of falling back to the root chat. (#58783)
- **Telegram/local Bot API:** preserve media MIME types for absolute-path downloads so local audio files still trigger transcription and other MIME-based handling. (#54603) Thanks @jzakirov.
- **WhatsApp/timestamps:** pass inbound message timestamp to model context so the AI can see when WhatsApp messages were sent. (#58590) Thanks @Maninae.
- **QQ Bot/bot-logs:** keep `/bot-logs` export gated behind a truly explicit QQBot allowlist, rejecting wildcard and mixed wildcard entries while preserving the real framework command path. Thanks @vincentkoc.
- **Channels/plugins:** keep bundled channel plugins loadable from legacy `channels.<id>` config even under restrictive plugin allowlists, and make `openclaw doctor` warn only on real plugin blockers instead of misleading setup guidance. (#58873) Thanks @obviyus.
- **Plugins/bundled runtimes:** restore externalized bundled plugin runtime dependency staging across packed installs, Docker builds, and local runtime staging so bundled plugins keep their declared runtime deps after the 2026.3.31 externalization change. (#58782)
- **LINE/runtime:** resolve the packaged runtime contract from the built `dist/plugins/runtime` layout so LINE channels start correctly again after global npm installs on `2026.3.31`. (#58799) Thanks @vincentkoc.
- **MiniMax/plugins:** auto-enable the bundled MiniMax plugin for API-key auth/config so MiniMax image generation and other plugin-owned capabilities load without manual plugin allowlisting. (#57127) Thanks @tars90percent.
- **Ollama/model picker:** show only Ollama models after provider selection in the CLI picker. (#55290) Thanks @Luckymingxuan.
- **CDP/profiles:** prefer `cdpPort` over stale WebSocket URLs so browser automation reconnects cleanly. (#58499) Thanks @Mlightsnow.
- **Media/paths:** resolve relative `MEDIA` paths against the agent workspace so local attachment references keep working. (#58624) Thanks @aquaright1.
- **Memory/session indexing:** keep full reindexes from skipping session transcripts when sync is triggered by `session-start` or `watch`, so restart-driven reindexes preserve session memory. (#39732) Thanks @upupc.
- **Memory/QMD:** prefer `--mask` over `--glob` when creating QMD collections so default memory collections keep their intended patterns and stop colliding on restart. (#58643) Thanks @GitZhangChi.
- **Subagents/tasks:** keep subagent completion and cleanup from crashing when task-registry writes fail, so a corrupt or missing task row no longer takes down the gateway during lifecycle finalization. Thanks @vincentkoc.
- **Sandbox/browser:** compare browser runtime inspection against `agents.defaults.sandbox.browser.image` so `openclaw sandbox list --browser` stops reporting healthy browser containers as image mismatches. (#58759) Thanks @sandpile.
- **Plugins/install:** forward `--dangerously-force-unsafe-install` through archive and npm-spec plugin installs so the documented override reaches the security scanner on those install paths. (#58879) Thanks @ryanlee-gemini.
- **Auto-reply/commands:** strip inbound metadata before slash command detection so wrapped `/model`, `/new`, and `/status` commands are recognized. (#58725) Thanks @Mlightsnow.
- **Agents/Anthropic:** preserve thinking blocks and signatures across replay, cache-control patching, and context pruning so compacted Anthropic sessions continue working instead of failing on later turns. (#58916) Thanks @obviyus.
- **Agents/failover:** unify structured and raw provider error classification so provider-specific `400`/`422` payloads no longer get forced into generic format failures before retry, billing, or compaction logic can inspect them. (#58856) Thanks @aaron-he-zhu.
- **Auth profiles/store:** coerce misplaced SecretRef objects out of plaintext `key` and `token` fields during store load so agents without ACP runtime stop crashing on `.trim()` after upgrade. (#58923) Thanks @openperf.
- **ACPX/session recovery:** repair `queue owner unavailable` session recovery by replacing dead named sessions and resuming the backend session when ACPX exposes a stable session id, so the first ACP prompt no longer inherits a dead handle. (#58669) Thanks @neeravmakwana.
- **ACPX/session recovery:** retry dead-session queue-owner repair without `--resume-session` when the reported ACPX session id is stale, so recovery still creates a fresh named session instead of failing session init. Thanks @obviyus.
- **Auth/OpenAI Codex:** persist plugin-refreshed OAuth credentials to `auth-profiles.json` before returning them, so rotated Codex refresh tokens survive restart and stop falling into `refresh_token_reused` loops. (#53082)
- **Discord/gateway:** hand reconnect ownership back to Carbon, keep runtime status aligned with close/reconnect state, and force-stop sockets that open without reaching READY so Discord monitors recover promptly instead of waiting on stale health timeouts. (#59019) Thanks @obviyus.

---

## OpenClaw v2026.3.31 — Release Summary

> **Released:** 2026-04-01 (docs release) | upstream GitHub release `v2026.3.31` published 2026-03-31 UTC | latest prerelease `v2026.4.1-beta.1` intentionally excluded | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.3.28..v2026.3.31` | **Scan stats:** 2,829 files changed, +139,607 / -53,848 lines

### Breaking

- **Nodes/exec:** remove the duplicated `nodes.run` shell wrapper from the CLI and agent `nodes` tool. Released guidance is now `nodes invoke` plus `exec host=node` for shell work.
- **Skills/install and Plugins/install:** built-in dangerous-code `critical` findings and install-time scan failures now fail closed by default. Operators must explicitly opt in with `--dangerously-force-unsafe-install` to override.
- **Gateway/auth:** `trusted-proxy` now rejects mixed shared-token configs, and local-direct fallback requires the configured token instead of implicitly authenticating same-host callers.
- **Gateway/node trust:** node commands now stay disabled until node pairing is explicitly approved, and node-originated runs stay on a reduced trusted surface.
- **Plugin SDK:** legacy provider-compat and channel-runtime compatibility shims are now deprecated with migration warnings; the documented public path remains `openclaw/plugin-sdk/*`.

### Features

- **Background tasks:** detached work now runs on a shared SQLite-backed control plane that unifies ACP, subagent, cron, and background CLI execution. The released CLI adds `openclaw tasks list|show|cancel`, and stable-line status/tooling surfaces now expose task state more consistently.
- **Channels:** bundled QQ Bot support lands on the released line, alongside LINE outbound image/video/audio sends, LINE current-conversation ACP binding, Matrix `historyLimit` / proxy / draft streaming / per-DM `threadReplies`, Slack native exec approvals, Microsoft Teams `member-info`, and WhatsApp reaction guidance.
- **MCP/ACPX:** `mcp.servers` now supports remote HTTP/SSE endpoints with auth headers and `streamable-http`, bundled MCP tools materialize with provider-safe names, and the ACPX plugin-tools bridge ships as an explicit default-off capability with documented trust boundaries.
- **Agents/runtime:** embedded Pi runs gain native Codex web search, idle-stream timeout control, and OpenAI Responses `text.verbosity` forwarding. `/status` and related status surfaces now report task context more accurately on the stable line.
- **Memory/QMD:** per-agent `memorySearch.qmd.extraCollections` allows controlled cross-agent transcript search without flattening the whole transcript space into one namespace.
- **TTS/ops:** TTS fallback attempts now emit structured diagnostics and analytics, improving release-line observability for speech regressions.

### Security And Operations

- **Gateway hardening:** trusted-proxy browser `Origin` enforcement, immediate device-token session revocation after rotation, read-only plugin-auth runtime scoping, tighter HTTP tool-invoke authorization, and safer WebSocket rate limiting all landed on the stable line.
- **Exec hardening:** approval persistence, wrapper unwrapping, `safeBins` tightening, env override blocking, and `host=auto` fail-closed behavior materially change the released exec trust model.
- **Node safety:** node shell/workdir handling now preserves node-local context, and owner-only or elevated node commands stay off unapproved or under-scoped paths.

### Fixes (key ones)

- **Gateway/OpenAI compatibility:** `/v1/responses` now accepts flat Responses API function-tool definitions and preserves strict hosted-tool handling, while `/v1/chat/completions` regains default operator scopes when clients omit `x-openclaw-scopes`.
- **Onboarding/pairing:** QR bootstrap onboarding for iPhone pairing is repaired so the first node pairing can auto-approve cleanly and issue a reusable device token.
- **Pi/TUI:** message-boundary replies flush on `message_end`, fixing turns that previously looked stuck after the final response was already ready.
- **CLI/onboarding:** declining a discovered remote gateway now resets the wizard prompt back to the safe loopback default instead of reusing the rejected URL.
- **Reliability:** task maintenance, session-status reporting, duplicate status replies, inbound media fetches, and per-channel runtime edge cases all received released-line fixes.

---

## OpenClaw v2026.3.28 — Historical Release Summary

> **Released:** 2026-03-29 (docs release) | upstream GitHub release `v2026.3.28` published 2026-03-29 | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.3.24..v2026.3.28` | **Scan stats:** 3806 files changed, +217,667 / -92,006 lines

### Breaking

- **Providers/Qwen:** remove the deprecated `qwen-portal-auth` OAuth integration for `portal.qwen.ai`; migrate to Model Studio with `openclaw onboard --auth-choice modelstudio-api-key`. (#52709)
- **Config/Doctor:** drop automatic config migrations older than two months; very old legacy keys now fail validation instead of being rewritten on load or by `openclaw doctor`.

### Features

- **xAI/Responses API:** move the bundled xAI provider to the Responses API, add first-class `x_search` plugin tool, and auto-enable the xAI plugin from owned web-search and tool config so bundled Grok auth/configured search flows work without manual plugin toggles. (#56048) Thanks @huntharo.
- **MiniMax:** add image generation provider for `image-01` model, supporting generate and image-to-image editing with aspect ratio control; trim model catalog to M2.7 only, removing legacy M2, M2.1, M2.5, and VL-01 models. (#54487) Thanks @liyuan97.
- **Plugins/hooks:** add async `requireApproval` to `before_tool_call` hooks, letting plugins pause tool execution and prompt the user for approval via the exec approval overlay, Telegram buttons, Discord interactions, or the `/approve` command on any channel. (#55339) Thanks @vaclavbelak and @joshavant.
- **ACP/channels:** add current-conversation ACP binds for Discord, BlueBubbles, and iMessage so `/acp spawn codex --bind here` can turn the current chat into a Codex-backed workspace without creating a child thread.
- **OpenAI:** enable `apply_patch` by default for OpenAI and OpenAI Codex models, aligned with `write` sandbox permissions.
- **Plugins/CLI backends:** move bundled Claude CLI, Codex CLI, and Gemini CLI inference defaults onto the plugin surface; add bundled Gemini CLI backend support; replace `gateway run --claude-cli-logs` with generic `--cli-backend-logs` (old flag kept as compatibility alias).
- **Podman:** simplify rootless container setup, install the launch helper under `~/.local/bin`, and document the host-CLI `openclaw --container <name> ...` workflow.
- **Slack:** add an explicit `upload-file` Slack action routing file uploads through the existing Slack upload transport, with optional filename/title/comment overrides; unify file-first sends for Microsoft Teams, Google Chat, and BlueBubbles under the canonical `upload-file` action.
- **Matrix TTS:** send auto-TTS replies as native Matrix voice bubbles instead of generic audio attachments. (#37080) Thanks @Matthew19990919.
- **CLI:** add `openclaw config schema` to print the generated JSON schema for `openclaw.json`. (#54523) Thanks @kvokka.
- **Config/TTS:** auto-migrate legacy speech config on normal reads and secret resolution; remove regular-mode runtime fallback for old bundled `tts.<provider>` API-key shapes.
- **Memory/plugins:** move the pre-compaction memory flush plan behind the active memory plugin contract so `memory-core` owns flush prompts and target-path policy.
- **MCP/channels:** add a Gateway-backed channel MCP bridge with Codex/Claude-facing conversation tools, Claude channel notifications, and safer stdio bridge lifecycle handling for reconnects and routed session discovery.
- **Plugins/runtime:** expose `runHeartbeatOnce` in the plugin runtime `system` namespace so plugins can trigger a single heartbeat cycle with an explicit delivery target override. (#40299) Thanks @loveyana.

### Security

- **LINE:** make webhook signature validation run the timing-safe compare even when the supplied signature length is wrong, closing a timing side-channel. (#55663) Thanks @gavyngong.
- **Security/audit:** extend web search key audit to recognize Gemini, Grok/xAI, Kimi, Moonshot, and OpenRouter credentials via a boundary-safe bundled-web-search registry shim. (#56540)
- **ACP/ACPX agent registry:** align OpenClaw's ACPX built-in agent mirror with the latest `openclaw/acpx` command defaults, pin versioned `npx` built-ins to exact versions, and stop unknown ACP agent IDs from falling through to raw `--agent` command execution on the MCP-proxy path. (#28321) Thanks @m0nkmaster and @vincentkoc.

### Fixes (key ones)

**Agents**
- **Agents/stop reason:** recover unhandled provider stop reasons (e.g. `sensitive`) as structured assistant errors instead of crashing the agent run. (#56639)
- **Agents/cooldowns:** scope rate-limit cooldowns per model so one 429 no longer blocks every model on the same auth profile; replace exponential 1 min → 1 h escalation with a stepped 30 s / 1 min / 5 min ladder; surface a user-facing countdown when all models are rate-limited. (#49834) Thanks @kiranvk-2011.
- **Agents/compaction:** preserve the post-compaction AGENTS refresh on stale-usage preflight compaction for both immediate replies and queued followups. (#49479) Thanks @jared596. Surface safeguard-specific cancel reasons and relabel benign manual `/compact` no-ops as skipped instead of failed. (#51072) Thanks @afurm.

**Memory/QMD**
- **Memory/QMD:** weight CJK-heavy text correctly when estimating chunk sizes, preserve surrogate-pair characters during fine splits, and keep long Latin lines on old chunk boundaries for better-sized CJK memory indexing. (#40271) Thanks @AaronLuo00.
- **Memory/QMD:** resolve slugified `memory_search` file hints back to the indexed filesystem path before returning search hits so `memory_get` works again for mixed-case and spaced paths. (#50313) Thanks @erra9x.
- **Memory/QMD:** honor `memory.qmd.update.embedInterval` even when regular QMD update cadence is disabled or slower by arming a dedicated embed-cadence maintenance timer. (#37326) Thanks @barronlroth.
- **Agents/memory flush:** keep daily memory flush files append-only during embedded attempts so compaction writes do not overwrite earlier notes. (#53725) Thanks @HPluseven.

**Telegram**
- **Telegram/splitting:** replace proportional text estimate with verified HTML-length search so long messages split at word boundaries instead of mid-word. (#56595)
- **Telegram/delivery:** skip whitespace-only and hook-blanked text replies in bot delivery to prevent GrammyError 400 empty-text crashes. (#56620)
- **Telegram/send:** validate `replyToMessageId` at all four API sinks with a shared normalizer that rejects non-numeric, NaN, and mixed-content strings. (#56587)
- **Telegram/forum topics:** keep native `/new` and `/reset` routed to the active topic by preserving the topic target on forum-thread command context. (#35963)
- **Telegram:** deliver verbose tool summaries inside forum topic sessions again. (#43236) Thanks @frankbuild.

**WhatsApp**
- **WhatsApp:** fix infinite echo loop in self-chat DM mode where the bot's own outbound replies were re-processed as new inbound user messages. (#54570) Thanks @joelnishanth.

**Discord**
- **Discord/reconnect:** drain stale gateway sockets, clear cached resume state before forced fresh reconnects, and fail closed when old sockets refuse to die so Discord recovery stops looping on poisoned resume state. (#54697) Thanks @ngutman.

**Matrix**
- **Matrix/E2EE thumbnails:** encrypt E2EE image thumbnails with `thumbnail_file` while keeping unencrypted-room previews on `thumbnail_url`. (#54711) Thanks @frischeDaten.
- **Matrix/ESM crypto:** load bundled `@matrix-org/matrix-sdk-crypto-nodejs` through `createRequire(...)` so E2EE media send and receive keep package-local native binding lookup working in packaged ESM builds. (#54566) Thanks @joelnishanth.

**LINE**
- **LINE/status:** stop `openclaw status` from warning about missing credentials when sanitized LINE snapshots are already configured. (#45701) Thanks @tamaosamu.

**Microsoft Teams**
- **Teams/config:** accept the existing `welcomeCard`, `groupWelcomeCard`, `promptStarters`, and feedback/reflection keys in strict config validation so already-supported Teams runtime settings stop failing schema checks. (#54679) Thanks @gumclaw.
- **Teams/Entra JWT:** prefer the freshest personal conversation reference for `user:<aadObjectId>` sends when multiple stored references exist, so replies stop targeting stale DM threads. (#54702) Thanks @gumclaw.

**BlueBubbles**
- **BlueBubbles/debounce:** guard debounce flush against null message text by sanitizing at the enqueue boundary and adding an independent combiner guard. (#56573)
- **BlueBubbles/contacts:** optionally enrich unnamed participant lists with local macOS Contacts names after group gating passes, so group member context can show names instead of only raw phone numbers.

**iMessage**
- **iMessage:** stop leaking inline `[[reply_to:...]]` tags into delivered text by sending `reply_to` as RPC metadata and stripping stray directive tags from outbound messages. (#39512) Thanks @mvanhorn.

**CLI**
- **CLI/zsh completion:** defer `compdef` registration until `compinit` is available so zsh completion loads cleanly with plugin managers and manual setups. (#56555)
- **CLI/onboarding:** show the Kimi Code API key option again in the Moonshot setup menu so the interactive picker includes all Kimi setup paths. Fixes #54412 Thanks @sparkyrider.
- **CLI/update status:** explicitly say `up to date` when the local version already matches npm latest. (#51409) Thanks @dongzhenye.

**Control UI**
- **Control UI/config:** keep sensitive raw config hidden by default, replace the blank blocked editor with an explicit reveal-to-edit state, and restore raw JSON editing without auto-exposing secrets. Fixes #55322.

**Auto-reply**
- **Auto-reply:** suppress JSON-wrapped `{"action":"NO_REPLY"}` control envelopes before channel delivery with a strict single-key detector; preserves media when text is only a silent envelope. (#56612)

**xAI**
- **xAI/code execution config:** register Codex for media understanding and route image prompts through Codex instructions so image analysis no longer fails on missing provider registration. (#54829) Thanks @neeravmakwana.
- **xAI/auth discovery:** let the bundled Grok web-search plugin offer optional `x_search` setup during `openclaw onboard` and `openclaw configure --section web`, including an x_search model picker with the shared xAI key.

**Daemon/Linux**
- **Daemon/Linux:** stop flagging non-gateway systemd services as duplicate gateways just because their unit files mention OpenClaw, reducing false-positive doctor/log noise. (#45328) Thanks @gregretkowski.

**Feishu**
- **Feishu/WebSocket:** close WebSocket connections on monitor stop/abort so ghost connections no longer persist, preventing duplicate event processing and resource leaks. (#52844) Thanks @schumilin.
- **Feishu/timestamps:** use the original message `create_time` instead of `Date.now()` for inbound timestamps so offline-retried messages carry the correct authoring time. (#52809) Thanks @schumilin.

**Google/OpenAI**
- **Google/models:** resolve Gemini 3.1 pro, flash, and flash-lite for all Google provider aliases by passing the actual runtime provider ID and adding a template-provider fallback. (#56567)
- **OpenAI/WebSocket:** preserve reasoning replay metadata and tool-call item ids on WebSocket tool turns, and start a fresh response chain when full-context resend is required. (#53856) Thanks @xujingchen1996.

**Other significant fixes**
- **Mistral:** normalize OpenAI-compatible request flags so official Mistral API runs no longer fail with remaining `422 status code (no body)` chat errors.
- **CLI/plugins:** make routed commands use the same auto-enabled bundled-channel snapshot as gateway startup so configured bundled channels like Slack load without requiring a prior config rewrite. (#54809) Thanks @neeravmakwana.
- **Agents/embedded replies:** surface mid-turn 429 and overload failures when embedded runs end without a user-visible reply. (#50930) Thanks @infichen.
- **Claude CLI/MCP:** always pass a strict generated `--mcp-config` overlay for background Claude CLI runs so Claude does not inherit ambient user/global MCP servers. (#54961) Thanks @markojak.
- **Discord/slash-command descriptions:** trim overlong descriptions to Discord's 100-character limit and map rejected deploy indexes back to command names so deploys stop failing on long descriptions. (#54118) Thanks @huntharo.
- **Agents/openai-compatible tool calls:** deduplicate repeated tool call ids across live assistant messages and replayed history so OpenAI-compatible backends no longer reject duplicate `tool_call_id` values with HTTP 400. (#40996) Thanks @xaeon2026.
- **CLI/status:** detect node-only hosts and show the configured remote gateway target instead of a false local `ECONNREFUSED`, suppress contradictory local-gateway diagnosis output.
- **Matrix/streaming:** add `streaming: "partial"` draft replies that stay on a single editable preview message, stop preview streaming once text no longer fits one Matrix event, and clear stale previews before media-only finals. (#56387) Thanks @jrusz.
- **Plugins/SDK:** thread `moduleUrl` through plugin-sdk alias resolution so user-installed plugins outside the openclaw directory correctly resolve `openclaw/plugin-sdk/*` subpath imports. (#54283) Thanks @xieyongliang.

---

## OpenClaw v2026.3.24 — Historical Release Summary

> **Released:** 2026-03-27 (docs release) | upstream GitHub release `v2026.3.24` published 2026-03-26 | **Policy note:** historical section.

### Features

- **Gateway OpenAI compat:** `/v1/models`, `/v1/embeddings` endpoints, model override forwarding through `/v1/chat/completions` and `/v1/responses`
- **Microsoft Teams:** official Teams SDK migration with streaming 1:1 replies, welcome cards, prompt starters, feedback/reflection, typing indicators, native AI labeling (PR #51808). Message edit and delete support (PR #49925).
- **Skills:** one-click install recipes for bundled skills (coding-agent, gh-issues, openai-whisper-api, session-logs, tmux, trello, weather). `SkillInstallSpec` with kind brew/node/go/uv/download.
- **Control UI/skills:** status-filter tabs (All / Ready / Needs Setup / Disabled) with counts, click-to-detail dialog
- **Slack interactive replies:** restored rich parity, auto-render `Options:` lines as buttons/selects (PR #53389)
- **CLI `--container` / `OPENCLAW_CONTAINER`:** for Docker/Podman container commands (PR #52651)
- **Discord `autoThreadName: "generated"`:** for LLM-generated thread titles (PR #43366)
- **Plugin hook `before_dispatch`:** with canonical inbound metadata and final-delivery routing (PR #50444)
- **Control UI:** agent workspace expandable `<details>` with lazy-loaded markdown preview
- **macOS app:** collapsible tree sidebar replacing pill navigation
- **Node 22.14 minimum floor** (lowered from 22.16). CLI update preflight node engine check.
- **Agents `/tools`:** now shows currently usable tools with "Available Right Now" UI section

### Security

- **Sandbox media dispatch:** closed `mediaUrl`/`fileUrl` alias bypass (PR #54034)
- **Outbound media policy:** aligned with fs `workspaceOnly` setting

### Fixes (key ones)

- **Gateway restart sentinel:** heartbeat-wake instead of best-effort note; thread/topic routing preserved (PR #53940)
- **Gateway channels:** sequential startup with per-channel failure isolation (PR #54215)
- **Docker setup:** avoid pre-start namespace loop (PR #53385)
- **WhatsApp:** group echo ID tracking (PR #53624), botInvokeMessage unwrap for reply-to-bot detection, selfLid from creds.json
- **Telegram:** forum topic #General routing recovery (PR #53699), 403 error preservation (PR #53635), photo dimension preflight (PR #52545)
- **Discord:** gateway supervision centralization, timeout replies (PR #53823), slash-command description truncation (PR #54118)
- **ACP:** terminal result delivery when TTS has no audio (PR #53692)
- **Embedded runs:** SecretRef crash fix (Fixes #45838)
- **Compaction:** sessions.json.compactionCount reconciliation (PR #45493)

---

## OpenClaw v2026.3.23 — Historical Release Summary

> **Released:** 2026-03-24 (docs release) | upstream GitHub release `v2026.3.23` published 2026-03-23 | npm `latest` correction build `2026.3.23-2` / git tag `v2026.3.23-2` has no separate GitHub release page as of 2026-03-24 | **Policy note:** historical section.
> **Window analyzed:** `v2026.3.22..v2026.3.23` | **Scan stats:** 458 files changed, +13,208 / -3,773 lines
> **Correction delta also audited:** `v2026.3.23..v2026.3.23-2` | **Delta stats:** 119 files changed, +5,016 / -1,297 lines

## Highlights

- **Qwen / ModelStudio:** standard DashScope endpoints added for China and global API keys; provider group relabeled to `Qwen (Alibaba Cloud Model Studio)`.
- **Control UI:** inline `<script>` blocks are now covered by computed SHA-256 hashes in CSP; the UI clarity pass also shipped the updated Knot theme, icon polish, and better accessibility labels.
- **Packaging:** published npm installs once again include bundled plugin runtime sidecars such as WhatsApp `light-runtime-api.js` and Matrix `runtime-api.js`.
- **Channel auth:** `channels login` / `logout` now auto-select the single configured login-capable channel and harden channel-id handling against prototype-chain and control-character abuse.
- **ClawHub:** install/uninstall/browse flows are aligned with the active runtime version; macOS auth-path handling and gateway skill browsing were fixed across `v2026.3.23` and the shipped `v2026.3.23-2` correction build.
- **Browser:** Chrome MCP existing-session attach now waits for tabs to become usable and reuses a running loopback browser more reliably on slower hosts.
- **Agents / runtime:** active runtime `web_search` provider selection fixed; generic transient `api_error` payloads now fall back more accurately; malformed assistant replay content is canonicalized before reuse.
- **Ops:** one-shot cron jobs now honor `--tz` wall-clock time for offset-less `--at` datetimes, including DST boundaries; same-base correction-version warnings no longer complain when `2026.3.23-2` config is read by `2026.3.23`.
- **Gateway security:** canvas routes now require auth, and agent session reset now requires admin scope.
- **Models:** Mistral max-token defaults are repaired to safe output budgets and `doctor --fix` can repair persisted over-large Mistral settings.

For full detail, see the upstream release notes for `v2026.3.23` and the shipped correction-tag changelog delta for `v2026.3.23-2` in `openclaw/openclaw`.

## Breaking / Behavior Shifts

1. **No new stable-tag breaking config changes were introduced in `v2026.3.23`.** The notable operator-facing behavior shifts in the shipped correction build are same-base version warning suppression (`2026.3.23-2` vs `2026.3.23`) and the cron one-shot timezone fix.

## Major Features

- DashScope standard endpoint support for Qwen / ModelStudio
- Control UI CSP inline-script hashes
- Restored bundled plugin runtime sidecars in published npm installs
- Channel-auth auto-select for single-channel setups
- ClawHub compatibility checks tied to the active runtime version
- Chrome MCP existing-session attach readiness fix
- Loopback browser reuse after short reachability misses
- Cron one-shot `--tz` wall-clock handling for offset-less datetimes
- Gateway auth required for canvas routes; admin scope required for agent session reset

## Maintainer Upgrade Checklist (v2026.3.23)

1. **Review browser configs:** Chrome extension relay settings were already removed in `v2026.3.22`; current released guidance is `driver: "existing-session"` with optional `browser.profiles.<name>.userDataDir`.
2. **Check packaged installs:** if you distribute or validate npm builds, confirm bundled plugin runtime sidecars are present in the published package.
3. **Audit ClawHub workflows:** plugin compatibility now binds to the active runtime version, and macOS auth-path fixes should be validated on default and XDG-style setups.
4. **Re-test cron one-shots:** if you use `openclaw cron add|edit --at ... --tz <iana>`, verify local wall-clock scheduling behavior after upgrading.
5. **Re-test Control UI/operator auth:** canvas routes are auth-protected and session reset is admin-scoped; stale under-scoped operator tokens should now produce clearer fallback messaging.

---

## OpenClaw v2026.3.22 — Historical Release Summary

> **Released:** upstream tag `v2026.3.22` published 2026-03-23 | **Policy note:** historical section.
> **Window analyzed:** `v2026.3.13-1..v2026.3.22` | **Scan stats:** 5,917 files changed, +555,370 / -200,641 lines

## Highlights

- **ClawHub-first plugin installs:** bare `openclaw plugins install <package>` now prefers ClawHub before npm for npm-safe package names, and native `openclaw skills search|install|update` flows landed.
- **Browser breaking change:** the legacy Chrome extension relay path was removed; `driver: "extension"` and `browser.relayBindHost` are gone, and host-local attach now uses Chrome MCP `existing-session`.
- **Image generation:** the stock image path is standardized on the core `image_generate` tool; bundled `nano-banana-pro` was removed.
- **Plugin SDK:** the released public SDK surface is `openclaw/plugin-sdk/*`; `openclaw/extension-api` remains only as a deprecated compatibility bridge.
- **Legacy env/state migration:** `CLAWDBOT_*` and `MOLTBOT_*` compatibility env names, plus `.moltbot` state-dir fallback, were removed from the released runtime.
- **Platform and runtime:** pluggable sandbox backends landed (`OpenShell`, SSH), `browser.profiles.<name>.userDataDir` was added for existing-session attach, and bundled provider/plugin flows expanded across ClawHub, marketplaces, and model plugins.

## Breaking / Behavior Shifts

1. **ClawHub-first installs:** bare `openclaw plugins install <package>` now prefers ClawHub before npm.
2. **Legacy Chrome relay removed:** `driver: "extension"`, bundled extension assets, and `browser.relayBindHost` were removed; run `openclaw doctor --fix` to migrate to `existing-session`.
3. **Legacy env/state compatibility removed:** `CLAWDBOT_*`, `MOLTBOT_*`, `.moltbot`, and `moltbot.json` fallbacks are gone from released code.
4. **Plugin SDK migration:** plugin authors should use `openclaw/plugin-sdk/*`; legacy monolithic import guidance is stale.

## Major Features

- Native ClawHub skill and plugin flows
- Plugin marketplace/bundle installs
- OpenShell and SSH sandbox backends
- `browser.profiles.<name>.userDataDir` for existing-session attach
- OpenAI `gpt-5.4`, `gpt-5.4-mini`, and `gpt-5.4-nano` released defaults/catalog support
- Expanded provider-plugin architecture, including Chutes, Tavily, Firecrawl, and additional bundled providers

## Maintainer Upgrade Checklist (v2026.3.22)

1. **Migrate browser configs:** run `openclaw doctor --fix` if any profile still uses `driver: "extension"` or `browser.relayBindHost`.
2. **Review plugin install docs:** ClawHub is now the default first-party marketplace path for bare plugin install flows.
3. **Audit env/state references:** remove any lingering `CLAWDBOT_*`, `MOLTBOT_*`, `.moltbot`, or `moltbot.json` assumptions from tooling and docs.
4. **Update plugin guidance:** prefer `openclaw/plugin-sdk/<subpath>` imports and treat `openclaw/extension-api` as deprecated compatibility only.

---

## OpenClaw v2026.3.13 — Historical Release Summary

> **Released:** 2026-03-15 (docs release) | upstream tag `v2026.3.13-1` published 2026-03-13 | **Policy note:** historical section.
> **Window analyzed:** `v2026.3.12..v2026.3.13-1` | **Scan stats:** 1,189 files changed, +59,900 / -32,153 lines

## Highlights

- **Browser — Chrome DevTools MCP attach mode:** new `profile="user"` and `profile="chrome-relay"` built-in browser profiles enable attaching to a signed-in live Chrome session via Chrome DevTools MCP. Batched act automation is also supported in this mode.
- **Android:** redesigned chat settings sheet; Google Code Scanner integration for QR-based onboarding flows.
- **iOS:** first-run welcome pager displayed before gateway setup, improving initial onboarding UX.
- **Gateway:** RPC timeout for unanswered client requests; `--require-rpc` flag on `gateway status`; session reset now preserves `lastAccountId` and `lastThreadId`.
- **Security — 6 exec approval hardening items (v2026.3.13):** pnpm, Perl `-M`/`-I`, PowerShell `-File`/`-f`, env wrappers, macOS line continuation, and skill auto-allow are all hardened in the exec approval pipeline. Single-use bootstrap pairing codes are now enforced. External content zero-width character stripping added. Telegram webhook auth now happens before body read. iMessage attachment path rejection tightened.
- **Agents:** blank API key accepted for loopback custom providers; compaction token sanity check validates against full pre-compaction totals; compaction safeguard language continuity via `agents.defaults.compaction.customInstructions`; memory bootstrap now prefers `MEMORY.md` over `memory.md`; replayed Anthropic/Bedrock thinking blocks are dropped; tool warning distinction between gated-core and plugin-only.
- **Cron:** nested cron lane routing prevents cross-lane deadlocks.
- **Config validation:** `agents.list[].params`, `tools.web.fetch.readability`/`firecrawl`, `channels.signal.groups`, `discovery.wideArea.domain` are now accepted config keys (previously rejected by Zod validation).
- **Docker:** `OPENCLAW_TZ` environment variable for timezone pinning in container deployments.
- **Platform:** gateway install/stop/status fixes on Windows; macOS onboarding self-restart avoidance; macOS Node >=22.16.0 runtime guard.
- **Dependencies:** pi packages bumped to 0.58.0.
- **Dashboard:** fixed full reload on every live tool result; oversized plain-text replies now rendered as paragraphs; `chat-new-messages` scroll pill styling corrected.

For full detail, see the v2026.3.13 notes in the upstream release changelog (`openclaw/openclaw` tag `v2026.3.13-1`).

## Change Distribution (By Top-Level Area)

| Area | Files changed |
| --- | ---: |
| `src/` | ~850 |
| `extensions/` | ~140 |
| `apps/` | ~85 |
| `docs/` | ~60 |
| `ui/` | ~30 |
| `scripts/` | ~10 |
| `skills/` | ~8 |
| `.github/` | ~6 |

*Note: additional root/config/release files (`package.json`, `pnpm-lock.yaml`, release metadata) make up the remainder of the 1,189-file release window.*

## Breaking / Behavior Shifts

1. **One stable-tag breaking config change in v2026.3.13:** `MEMORY.md` now wins over `memory.md` during workspace bootstrap. Review any custom bootstrap workflows that relied on the lowercase file taking precedence.

## Major Features

- Chrome DevTools MCP attach mode (`profile="user"`, `profile="chrome-relay"` built-in profiles)
- Batched act automation in browser
- Android chat settings sheet redesign + Google Code Scanner for QR onboarding
- iOS first-run welcome pager before gateway setup
- Gateway RPC timeout for unanswered client requests
- `--require-rpc` on `gateway status`
- Session reset preserves `lastAccountId` / `lastThreadId`
- `agents.defaults.compaction.customInstructions` for compaction language continuity
- Memory bootstrap prefers `MEMORY.md` over `memory.md`
- `OPENCLAW_TZ` Docker timezone pinning
- pi packages bumped to 0.58.0

## Security Hardening

- Single-use bootstrap pairing codes (replaces shared credentials for `/pair` setup)
- External content zero-width character stripping (prompt injection hardening)
- Telegram webhook auth now enforced before body read
- iMessage attachment path rejection tightened
- Exec approval hardening: pnpm, Perl `-M`/`-I`, PowerShell `-File`/`-f`, env wrappers, macOS line continuation, skill auto-allow

## Maintainer Upgrade Checklist (v2026.3.13)

1. **Review exec allowlists:** exec approval hardening may block pnpm, Perl `-M`/`-I`, PowerShell `-File`/`-f`, env wrapper invocations, or macOS `\`-continued` shell commands. Audit and update allowlist entries.
2. **Pairing codes:** bootstrap pairing codes are now single-use. Update any automated pairing flows that reuse a code.
3. **Memory bootstrap:** if you rely on a lowercase `memory.md` as the primary memory bootstrap file, rename it to `MEMORY.md` or update your config.
4. **Docker timezone:** use `OPENCLAW_TZ` to pin timezone in containerized deployments rather than host mounts or environment hacks.
5. **Windows gateway:** verify install/stop/status behavior after upgrading on Windows gateway deployments.

---

## OpenClaw v2026.3.12 — Historical Release Summary

> **Released:** upstream tag `v2026.3.12` published 2026-03-12 (docs release 2026-03-15) | **Policy note:** historical section.
> **Window analyzed:** `v2026.3.11..v2026.3.12` | **Scan stats:** 830 files changed, +48,951 / -9,749 lines

## Highlights

- **Fast mode:** new `/fast` toggle enables `service_tier` fast-mode for OpenAI and Anthropic — per-model config defaults, TUI/ACP support; isolated cron runs can use fast mode.
- **Dashboard-v2:** modular overview/chat/config/agent/session views; command palette; mobile bottom tabs; slash commands; search; export; pinned messages.
- **Provider plugins:** Ollama, vLLM, and SGLang are now modularized onto the provider-plugin architecture, enabling operator-managed provider updates independent of core.
- **`sessions_yield` tool:** orchestrators can end a turn immediately, skip queued work, and carry a hidden follow-up payload — a new cooperative turn-ending primitive.
- **Context engine:** `sessionKey` is forwarded through all ContextEngine lifecycle calls.
- **Slack:** Block Kit messages supported in shared reply delivery; opt-in interactive replies via `channels.slack.capabilities.interactiveReplies`.
- **ACP:** final assistant text snapshot preserved before `end_turn`.
- **Mattermost:** `replyToMode` support (`off`/`first`/`all`).
- **Compaction:** post-compaction memory sync + transcript updates; status reaction during compaction; skip double `cache-ttl` write.
- **Security — 20+ GHSAs fixed (v2026.3.12):** GHSA-99qw (workspace plugin trust), GHSA-pcqg (exec format char escape), GHSA-9r3v (Unicode obfuscation normalization), GHSA-f8r2 (exec allowlist glob), GHSA-r7vr (`/config`/`/debug` owner-only), GHSA-rqpp (WebSocket unbound scope strip), GHSA-vmhq (browser profile create/delete block), GHSA-2rqg (spawned workspace boundary), GHSA-wcxr (`session_status` sandbox visibility), GHSA-2pwv (device token scope cap), GHSA-jv4g + GHSA-xwx2 (WebSocket preauth limits), GHSA-6rph (media store size cap), GHSA-jf5v (`GIT_EXEC_PATH` block), GHSA-57jw + GHSA-jvqh + GHSA-x7pp + GHSA-jc5j (exec approval inline loader/pnpm/npx hardening), GHSA-g353 (Feishu `encryptKey`), GHSA-m69h (Feishu reaction), GHSA-mhxh (LINE signatures), GHSA-5m9r (Zalo rate limiting).
- **Bootstrap tokens:** `/pair` setup codes switch from shared credentials to short-lived tokens.
- **Kubernetes:** starter K8s install path with raw manifests.
- **Node 24 default, Node 22.16 minimum floor.**
- **Hooks:** fail closed on unreadable loader paths; idempotency key deduplication.
- **Memory:** post-compaction session reindexing via `agents.defaults.compaction.postIndexSync` and `agents.defaults.memorySearch.sync.sessions.postCompactionForce`.
- **Zalo:** markdown-to-Zalo text style parsing; stable group ID enforcement.
- **Terminal:** grapheme display width and table rendering fixes.

For full detail, see the v2026.3.12 notes in the upstream release changelog (`openclaw/openclaw` tag `v2026.3.12`).

## Change Distribution (By Top-Level Area)

| Area | Files changed |
| --- | ---: |
| `src/` | ~520 |
| `extensions/` | ~140 |
| `apps/` | ~90 |
| `docs/` | ~45 |
| `ui/` | ~20 |
| `scripts/` | ~8 |

*Note: additional root/config/release files (`package.json`, `pnpm-lock.yaml`, release metadata) make up the remainder of the 830-file release window.*

## Breaking / Behavior Shifts

1. **No new breaking config changes in v2026.3.12.** Node 22.16 is now the minimum supported runtime; upgrades from older Node LTS releases must update runtime before upgrading OpenClaw.

## Major Features

- `/fast` mode toggle for OpenAI and Anthropic (`service_tier`) — per-model defaults, TUI/ACP support
- Isolated cron runs can use fast mode
- Dashboard-v2 modular UI: overview, chat, config, agent, session views
- Command palette, mobile bottom tabs, slash commands, search, export, pinned messages
- Ollama, vLLM, SGLang modularized as provider plugins
- `sessions_yield` tool (cooperative turn-ending primitive)
- `sessionKey` forwarded through all ContextEngine lifecycle calls
- Slack Block Kit messages in shared reply delivery
- Slack opt-in interactive replies (`channels.slack.capabilities.interactiveReplies`)
- ACP final assistant text snapshot before `end_turn`
- Mattermost `replyToMode`
- Post-compaction memory sync + transcript updates
- Status reaction during compaction
- Bootstrap `/pair` codes switched to short-lived tokens
- Starter Kubernetes install path with raw manifests
- Node 24 default; Node 22.16 minimum
- Hooks: fail-closed on unreadable loader paths; idempotency key deduplication
- Post-compaction session reindexing (`postIndexSync`, `postCompactionForce`)
- Zalo markdown-to-text parsing; stable group ID enforcement
- Terminal: grapheme display width + table rendering fixes

## Security Hardening

- **GHSA-99qw:** workspace plugin trust boundary enforcement
- **GHSA-pcqg:** exec format character escape hardening
- **GHSA-9r3v:** Unicode obfuscation normalization in exec preflight
- **GHSA-f8r2:** exec allowlist glob pattern escape hardening
- **GHSA-r7vr:** `/config` and `/debug` endpoints restricted to owner-only scope
- **GHSA-rqpp:** WebSocket unbound scope strip hardening
- **GHSA-vmhq:** browser profile create/delete restricted; block unauthorized profile ops
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

## Maintainer Upgrade Checklist (v2026.3.12)

1. **Node runtime:** upgrade to Node 22.16+ (minimum) before upgrading OpenClaw. Node 24 is the new default for fresh installs.
2. **Bootstrap pairing tokens:** `/pair` now issues short-lived tokens instead of shared credentials. Update any automated pairing integrations.
3. **Provider plugins (Ollama/vLLM/SGLang):** these are now provider plugins; ensure no config references to old inline provider paths remain.
4. **Post-compaction reindexing:** if memory freshness after compaction is important, enable `agents.defaults.compaction.postIndexSync: true` and `agents.defaults.memorySearch.sync.sessions.postCompactionForce: true`.
5. **Exec allowlists:** review and update allowlists in light of multiple exec approval hardening GHSAs (inline loader, pnpm, npx paths).
6. **Slack interactive replies:** opt in to Block Kit interactive replies by setting `channels.slack.capabilities.interactiveReplies: true`; this is opt-in and off by default.
7. **Hook loader paths:** unreadable hook loader paths now fail closed. Review hook configurations to ensure all loader paths are accessible.

---

## OpenClaw v2026.3.11 — Historical Release Summary

> **Released:** 2026-03-12 | **Policy note:** historical section.
> **Window analyzed:** `v2026.3.8..v2026.3.11` | **Scan stats:** 235 commits, 977 files changed, +48,586 / -9,950 lines

## Highlights

- **Memory/multimodal indexing:** new opt-in multimodal image and audio indexing for `memorySearch.extraPaths` using Gemini `gemini-embedding-2-preview`, with strict fallback gating and scope-based reindexing. Configurable output dimensions; reindexing triggers automatically when dimensions change.
- **iOS Home canvas overhaul:** bundled welcome screen with live agent overview (refreshes on connect, reconnect, and foreground return), replaced floating controls with a docked toolbar, and chat now opens in the resolved main session instead of a synthetic `ios` session.
- **macOS chat model picker:** new model picker in the macOS chat UI with persistent explicit thinking-level selections and hardened provider-aware session model sync.
- **Onboarding/Ollama first-class:** full Ollama setup wizard with Local or Cloud + Local modes, browser-based cloud sign-in, curated model suggestions, and cloud-model handling that skips unnecessary local pulls.
- **OpenCode Go provider:** new OpenCode Go provider in the wizard; Zen and Go share a single OpenCode setup with one shared key while runtime providers remain split.
- **Channel and platform polish:** Discord auto-created threads now support configurable `autoArchiveDuration`; Telegram HTML sends and final-preview cleanup avoid duplicate or malformed deliveries; macOS remote onboarding now explains shared-token gateways; iOS gains a local Fastlane-backed TestFlight beta flow.
- **ACP session resume + UX:** spawned ACP sessions can now resume existing ACPX/Codex conversations via `resumeSessionId`; `loadSession` replays stored user and assistant text; `tool_call` events enriched with file-location hints; `main` alias canonicalized so restarted ACP main sessions rehydrate cleanly.
- **macOS/launchd restart hardening v2:** LaunchAgent stays registered during explicit restarts; self-restarts handed off through a detached launchd helper; config/hot reload restart paths recovered without unloading the service. Fixes #43311, #43406, #43035, #43049.
- **BREAKING — Cron/doctor isolation:** cron jobs can no longer notify through ad hoc agent sends or fallback main-session summaries; `openclaw doctor --fix` migrates legacy cron storage and legacy notify/webhook delivery metadata.
- **Security — GHSA-5wcw-8jjv-m286 (Gateway/WebSocket):** browser origin validation now enforced for all browser-originated connections regardless of proxy headers, closing a cross-site WebSocket hijacking path in `trusted-proxy` mode.
- **Extensive security hardening:** symlink-safe secret file reads, TAR/bz2 extraction staging, exec SecretRef traversal rejection, fs-bridge staged-write pinning, gateway auth fail-closed, session-reset auth split, plugin HTTP route scope isolation, session_status sandbox guards, and more (see Security section).

For full detail, see the v2026.3.11 notes in the upstream release changelog (`openclaw/openclaw` tag `v2026.3.11`).

## Change Distribution (By Top-Level Area)

| Area | Files changed |
| --- | ---: |
| `src/` | 713 |
| `extensions/` | 106 |
| `docs/` | 53 |
| `apps/` | 49 |
| `ui/` | 15 |
| `skills/` | 10 |
| `.github/` | 8 |
| `scripts/` | 6 |
| `test/` | 3 |

*Note: additional root/config/release files (`AGENTS.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `SECURITY.md`, `docs.acp.md`, `package.json`, `pnpm-lock.yaml`, app assets/config files, and release metadata) make up the remainder of the 977-file release window.*

## Breaking / Behavior Shifts

1. **Cron/doctor isolation (BREAKING):** cron jobs can no longer send ad hoc agent notifications or fallback summaries to the main session. Run `openclaw doctor --fix` to migrate legacy cron state and legacy notify/webhook delivery metadata.

## Major Features

- Multimodal memory indexing (images + audio) with `gemini-embedding-2-preview`
- iOS Home canvas overhaul — live agent overview + docked toolbar
- macOS chat model picker with persistent thinking-level selections
- Ollama first-class onboarding wizard (Local and Cloud + Local modes)
- OpenCode Go provider (shared key with Zen, split runtime providers)
- Discord `autoArchiveDuration` for auto-created threads
- Telegram HTML send + final-preview delivery hardening
- macOS remote onboarding shared-token detection
- iOS local TestFlight beta flow
- ACP session resume via `resumeSessionId` in `sessions_spawn`
- ACP `loadSession` session context replay
- `tool_call` / `tool_call_update` events enriched with file-location hints
- macOS/launchd v2 restart — detached helper hand-off, no unload/reload cycle
- Node pending-work queue primitives (`node.pending.enqueue` / `node.pending.drain`)
- Gateway runtime version exposed in gateway status
- `OPENCLAW_CLI` environment marker in child command environments

## Security Hardening

- **GHSA-5wcw-8jjv-m286:** WebSocket origin enforced regardless of proxy headers (cross-site hijack fix)
- Symlink-safe `*File` secret reads (reject path-swap races and symlink-backed secret files)
- TAR / `tar.bz2` extraction staging against destination symlink escapes
- Exec SecretRef traversal id rejection across schema, runtime, and gateway (#42370)
- fs-bridge staged writes pinned to verified parent directories
- Gateway auth fails closed when local SecretRefs configured but unavailable (#42672)
- Config writes enforce both originating and targeted account scope
- `system.run` approval-backed commands fail closed without concrete single local file operand
- `/new` and `/reset` split from admin-only `sessions.reset` RPC
- Unauthenticated plugin HTTP routes blocked from inheriting synthetic admin gateway scopes
- `session_status` enforces sandbox session-tree visibility before reads/mutations
- `nodes` tool treated as owner-only fallback policy
- Whitespace-delimited `EXTERNAL UNTRUSTED CONTENT` markers treated like underscore-delimited variants
- Telegram `/approve` commands reject approvals aimed at other bots
- Subagent leaf vs orchestrator scope persisted at spawn time
- Discord reaction ingress enforces users/roles allowlist
- Gateway browser origin check enforced regardless of proxy headers
- State dir permissions hardened during onboard
- Plugin discovery env isolated from global state
- OpenAI WebSocket replay hardened

## Maintainer Upgrade Checklist (v2026.3.11)

1. **Run `openclaw doctor --fix` for cron migration:** cron jobs that previously sent ad hoc notifications or fallback main-session summaries will break without migration.
2. **Review WebSocket origin policy:** if running behind a reverse proxy with `trusted-proxy` mode, the new origin enforcement may reject same-origin WebSocket connections that previously passed. Verify gateway CORS/origin config.
3. **Review ACP cron session usage:** if ACP spawns previously relied on silent error-state completions mapped to `refusal`, those are now `end_turn`.
4. **Check `*File` secret paths:** paths pointing to symlinks will now be rejected. Update to direct regular file paths.
5. **Review plugin HTTP routes:** unauthenticated plugin HTTP routes can no longer call `runtime.subagent.*` admin methods. Add gateway auth if needed.
6. **Ollama onboarding:** if upgrading Ollama setups, re-run the wizard to pick up the new Local/Cloud + Local mode and updated model defaults.

---

## OpenClaw v2026.3.8 — Historical Release Summary

> **Released:** 2026-03-09 | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.3.7..v2026.3.8` | **Scan stats:** 260 commits, 769 files changed, +29,292 / -8,663 lines

## Highlights

- **Backup and recovery CLI:** new `openclaw backup create` and `openclaw backup verify` commands create local state archives, support `--only-config` / `--no-include-workspace`, and are now recommended before destructive flows like reset/uninstall.
- **ACP provenance:** ACP can now preserve upstream origin metadata and inject visible provenance receipts with `openclaw acp --provenance off|meta|meta+receipt`; spawned ACP child sessions also persist lineage and transcript metadata more reliably.
- **Browser/CDP remote reliability:** direct `ws://` / `wss://` CDP profiles are now first-class, wildcard debugger URLs from remote `/json/version` responses are rewritten back to the external host/port, and `browser.relayBindHost` enables WSL2/cross-namespace extension relay binding while strict redirect-hop SSRF checks fail closed.
- **Talk/runtime config:** top-level `talk.silenceTimeoutMs` now controls auto-send silence windows with platform defaults (`700 ms` on macOS/Android, `900 ms` on iOS), and Talk config resolution now exposes a normalized provider payload.
- **Plugin/channel onboarding hardening:** onboarding clears plugin discovery cache after installs, bundled channel plugins win over duplicate npm-installed copies during onboarding/update sync, and release checks validate bundled-extension manifest/root-dependency drift.
- **Daemon/restart hardening:** launchd repair/restart paths now re-enable LaunchAgents before bootstrap, launchd supervision detection includes `XPC_SERVICE_NAME`, and supervised restart flows exit non-zero / rely on the supervisor `KeepAlive` path instead of self-kickstarting in-process.
- **Web search/runtime updates:** Brave gains `tools.web.search.brave.mode: "llm-context"` with source-grounded snippets, while Perplexity now uses the native Search API for direct Perplexity auth but preserves OpenRouter/Sonar compatibility for legacy `OPENROUTER_API_KEY` or explicit model/baseUrl setups.
- **Channel/platform fixes:** Telegram DM dedupe and polling cleanup were hardened, Matrix DM routing now prefers explicit room bindings over brittle `m.direct` heuristics, Microsoft Teams sender allowlists remain enforced under route allowlists, macOS remote mode now surfaces `gateway.remote.token`, and Android Play builds remove self-update/background-location/screen-record/background-mic behavior.

For full detail, see the v2026.3.8 notes in the upstream release changelog (`openclaw/openclaw` tag `v2026.3.8`).

## Change Distribution (By Top-Level Area)

| Area | Files changed |
| --- | ---: |
| `src/` | 374 |
| `apps/` | 227 |
| `extensions/` | 89 |
| `docs/` | 35 |
| `scripts/` | 18 |

*Note: additional root/config/release files (`Dockerfile*`, `package.json`, `pnpm-lock.yaml`, `appcast.xml`, workflow/config files, and fixtures) make up the remainder of the 769-file release window.*

## Breaking / Behavior Shifts

1. **No new stable-tag breaking config was introduced in `v2026.3.8`,** but operational behavior changed in ways that matter during upgrades: launchd restart now exits and relies on supervisor restart, Android Play builds remove self-update/background-capture capabilities, and Perplexity web search routing now depends on the auth path in use.

## Major Features

- Backup and verify CLI (`openclaw backup create`, `openclaw backup verify`)
- ACP provenance metadata + receipt injection
- Brave LLM Context mode for `web_search`
- `talk.silenceTimeoutMs` runtime config
- Direct WebSocket CDP support + wildcard debugger URL rewrite
- `browser.relayBindHost` for WSL2/cross-namespace relay access
- Decorated `openclaw --version` output with short commit hash
- macOS remote gateway token UI + preservation of unsupported token shapes

## Security Hardening

- Browser strict redirect-hop SSRF validation now fails closed for remote tab-open paths when hop inspection is unavailable
- `system.run` approvals now bind `bun` / `deno run` script operands to on-disk file snapshots
- Skill download installs pin the validated tools root before writing archives
- Microsoft Teams sender allowlists remain enforced even when route allowlists match
- Control UI asset serving allows bundled/package-proven hardlinks while preserving strict checks for configured/custom roots

## Maintainer Upgrade Checklist (v2026.3.8)

1. **Adopt the backup workflow:** use `openclaw backup create` before `reset`, uninstall, or large config surgery; verify archives with `openclaw backup verify` if recovery matters.
2. **Review launchd restart assumptions:** supervised macOS restart now relies on launchd `KeepAlive`, `enable` before `bootstrap`, and `XPC_SERVICE_NAME` detection rather than in-process self-kickstart.
3. **Check browser remote profiles:** if using remote Chrome/Browserless/WSL2, validate direct WS CDP URLs, wildcard-host rewrites, and `browser.relayBindHost` as needed.
4. **Review web-search auth paths:** direct Perplexity keys use the native Search API; OpenRouter or explicit Perplexity `baseUrl` / `model` keeps the Sonar/OpenRouter path. Confirm the intended routing.
5. **Review Talk and remote-app settings:** set `talk.silenceTimeoutMs` if you need non-default pause windows, and verify whether the macOS app can consume your `gateway.remote.token` format directly.
6. **Validate bundled plugin behavior:** if you previously relied on npm-installed duplicates shadowing bundled channel plugins, that precedence is now reversed during onboarding/update sync.

---

## OpenClaw v2026.3.7 — Historical Release Summary

> **Released:** 2026-03-08 | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.3.2..v2026.3.7` | **Scan stats:** 893 commits, 2,412 files changed, +127,093 / -22,412 lines

## Highlights

- **ContextEngine plugin interface:** New `ContextEngine` plugin slot with full lifecycle hooks (`bootstrap`, `ingest`, `assemble`, `compact`, `afterTurn`, `prepareSubagentSpawn`, `onSubagentEnded`), slot-based registry with config-driven resolution, `LegacyContextEngine` wrapper preserving existing compaction behavior. Enables plugins like `lossless-claw` to provide alternative context management strategies. Zero behavior change when no context engine plugin is configured.
- **ACP persistent channel bindings:** Durable Discord channel and Telegram topic binding storage survives restarts; routing resolution and CLI/docs support added so ACP thread targets can be managed consistently.
- **Telegram topic agent routing:** Per-topic `agentId` overrides in forum groups and DM topics; topic thread binding (`--thread here|auto`); actionable approval buttons; successful bind confirmations pinned in-topic.
- **Spanish locale:** Control UI now supports Spanish (`es`) with locale detection, lazy loading, and language picker.
- **Mattermost interactive model picker:** Telegram-style interactive model selection and slash command integration.
- **Discord native slash commands:** Schema validation for `agentComponents` config with parity enforcement.
- **New model support:** Google Gemini 3.1 Flash-Lite (first-class), GPT-5.4 for OpenAI API and Codex OAuth; Venice default updated to `kimi-k2-5`; MiniMax Lightning removed.
- **Docker slim variant + Podman support:** Multi-stage Docker build with slim image; Podman as alternative runtime.
- **systemd WSL2 hardening:** File permission enforcement (0600/0700), Windows Scheduled Task management (locale-invariant detection), WSL2 systemd support.
- **Config and secret hardening:** SecretRef models.json persistence hardening prevents API keys from being written to disk when managed via SecretRef; config validation now fail-closed on errors.
- **Security:** ZIP archive path traversal hardening, password-file input hardening, auth token snippet removal from status output, plugin hook policy validation, Nodes `system.run` approval enforcement.

For full detail, see the v2026.3.7 notes in the upstream release changelog (`openclaw/openclaw` tag `v2026.3.7`).

## Change Distribution (By Top-Level Area)

| Area | Files changed |
| --- | ---: |
| `src/` | ~1,450 |
| `extensions/` | ~580 |
| `ui/` | ~95 |
| `apps/` | ~115 |
| `scripts/` | ~42 |
| `test/` | ~75 |

*Note: This distribution is a sampled breakdown; additional files in generated, tooling, or misc paths (`.github`, `.pi`, root config/workflow files, release assets) make up the remaining count to match the stated total.*

## Breaking / Behavior Shifts

1. **Gateway auth mode requirement:** When both `gateway.auth.token` and `gateway.auth.password` are configured (including via SecretRef), an explicit `gateway.auth.mode` must now be set; failing to do so causes a startup error.

## Major Features

- ContextEngine plugin interface (PR #22201)
- ACP persistent channel bindings (PR #34873)
- Telegram topic agent routing (PR #33647)
- Spanish locale in Control UI (PR #35038)
- Mattermost interactive model picker (PR #38767)
- Discord native slash commands (PR #39378)
- Gemini 3.1 Flash-Lite + GPT-5.4 model support
- Docker slim variant + Podman support
- systemd WSL2 hardening + Windows Scheduled Task management
- Compaction post-context configurability (PR #34556)
- Gateway channel-backed readiness probes (PR #18446, @vibecodooor, @mahsumaktas, @vincentkoc)

## Security Hardening

- Config validation fail-closed on errors
- ZIP archive path traversal hardening
- SecretRef models.json persistence hardening (PR #38955)
- Password-file input hardening (PR #39067)
- Control UI device auth token signing alignment
- Plugin hook policy validation
- Nodes system.run approval enforcement
- Auth token/key snippet removal from status output

## Maintainer Upgrade Checklist (v2026.3.7)

1. **Check gateway auth mode:** If you configure both `gateway.auth.token` and `gateway.auth.password` (or SecretRefs), add an explicit `gateway.auth.mode` value before upgrading.
2. **Review ContextEngine plugin config:** If you operate third-party plugins like `lossless-claw`, confirm plugin slot config is correct; no config = no behavior change.
3. **Update model catalog references:** Remove any references to `minimax-lightning`; update Venice references to use `kimi-k2-5` default.
4. **Refresh SecretRef configurations:** Verify API keys are stored in SecretRef and not in `models.json`; run `openclaw secrets audit` to confirm.
5. **Test Discord agentComponents config:** If using Discord with `agentComponents`, validate config against updated schema.
6. **Verify ZIP-based plugin assets:** Confirm no plugin asset extraction depends on paths outside the intended directory.

---

## OpenClaw v2026.3.2 — Historical Release Summary

> **Released:** 2026-03-03 | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.3.1..v2026.3.2` | **Scan stats:** 862 commits, 2,119 files changed, +121,000 / -55,228 lines

## Highlights

- **SecretRef and credential surface unification:** `openclaw secrets` runtime flow now covers 64 supported credential targets and enforces fail-fast only on active surfaces; inactive surfaces remain non-blocking. Onboarding and CLI flows receive explicit Surface planning visibility.
- **PDF and diff enhancements:** new first-class `pdf` tool plus `tools/pdf` integration for Anthropic/Google extract paths, plus diffs plugin PDF renderer and quality controls (`fileQuality`, `fileScale`, `fileMaxWidth`) for generated artifact outputs.
- **Config quality gates:** `openclaw config validate` with `--json` is added, with invalid-key path reporting and stricter runtime startup diagnostics.
- **Channel and runtime control improvements:** `channelRuntime` is exposed in plugin context, plus outbound `sendPayload` support across direct-text-media, Discord/Slack/WhatsApp/Zalo/Zalouser with chunk-aware fallback behavior.
- **Session/session-spawn evolution:** `sessions_spawn` now accepts base64/utf8 attachments with per-run redaction and lifecycle cleanup controls; delivery-mode behavior now enforces disabled messaging when `mode: "none"` is configured.
- **Telegram reliability:** default streaming for `channels.telegram.streaming` is now `partial`; DM streaming uses `sendMessageDraft` with separation of reasoning/answer preview lanes.
- **Gateway reliability and security:** ws route and auth hardening, TLS pairing/runtime pairing bypass safety, plugin webhook auth/route canonicalization fixes, and startup/reporting improvements including runtime self-version preference.
- **Feishu/LINE/Voice-call reliability:** Feishu/LINE inbound/outbound routing and metadata handling improvements, safer replay and webhook processing, Twilio verification compatibility updates, and improved voice-call/notification reliability.
- **Plugin and hooks hardening:** stricter plugin command normalization and lifecycle hooks, including `session_start`, `session_end`, `message:transcribed`, and `message:preprocessed`/`message:sent` contexts for downstream integrations.
- **Model/streaming robustness:** additional model/provider compatibility checks, failover tuning, and usage-window presentation improvements.

For full detail, see the v2026.3.2 notes in the upstream release changelog (`openclaw/openclaw` tag `v2026.3.2`).

## Change Distribution (By Top-Level Area)

| Area | Files changed |
| --- | ---: |
| `src/` | 1,371 |
| `extensions/` | 326 |
| `docs/` | 98 |
| `ui/` | 41 |
| `apps/` | 201 |
| `scripts/` | 35 |
| `test/` | 9 |

*Note: This distribution is a sampled breakdown; additional files in generated, tooling, or misc paths (`.github`, `.pi`, root config/workflow files, release assets) make up the remaining count to match the stated total.*

## Breaking / Behavior Shifts

1. **`openclaw` credential flow defaults changed for new installs:** tooling defaults now default `tools.profile` to `messaging` and ACP dispatch is enabled by default; explicit disablement is required for non-dispatch behavior.
2. **Zalo Personal plugin runtime changed to native JS path:** `@openclaw/zalouser` no longer depends on external CLI transports for login/send/listen; operators should refresh sessions with `openclaw channels login --channel zalouser`.
3. **Plugin SDK HTTP route registration API changed:** `api.registerHttpHandler` was removed in favor of explicit route registration.

## Major Features

- **SecretRef surface expansion:** 64 active credential targets covered by onboarding/secret planning and validation pipelines.
- **PDF support:** native `tools/pdf` support with provider selection and extraction fallbacks.
- **`openclaw config validate`:** command-level validation for config and config-key diagnostics.
- **Plugin runtime extensions:** session and message lifecycle hooks now expose richer contexts for event consumers.
- **Attachments in subagent sessions:** `sessions_spawn` accepts encoded attachments with retention and cleanup controls.
- **Shared send payload pipeline:** cross-channel outbound send path receives shared `sendPayload` handling, including chunked and media-first fallback behavior.

## High-Signal Runtime Fixes

- Telegram draft finalization reliability and DM topic routing defaults tightened for ambiguous multi-account setups.
- Plugin command validation hardened for malformed registrations.
- Gateway startup and restart diagnostics improved with explicit runtime version reporting and safer TLS pairing behavior.
- Feishu, LINE, and Telegram ingress/egress edge-cases hardened for session metadata consistency and webhook safety.
- Browser/CDP startup error paths expose better Chromium diagnostics and safer fallback handling.
- Health and version reporting in status output updated to prefer runtime-reported version metadata.

## Security Hardening

- **Webhook auth-before-body:** pre-auth body limits now apply consistently to BlueBubbles/GoogleChat/LINE webhook paths.
- **Gateway/Plugin route authorization:** stricter route ownership, path canonicalization, and auth checks for plugin handlers reduce alternate-path bypass opportunities.
- **Secrets + runtime surfaces:** stricter execution around secret provider retries, secret surface resolution, and session-context propagation.
- **Runtime execution boundaries:** additional sandbox/workspace boundary checks and startup guardrails for filesystem and process paths.
- **Browser security hardening:** CDP/profile and output handling protections against transient/failure-state edge cases.

## Maintainer Upgrade Checklist (v2026.3.2)

1. **Validate `openclaw secrets` migration behavior:** confirm active surfaces fail fast and inactive surfaces only emit non-blocking diagnostics.
2. **Review `tools.profile` and ACP dispatch defaults:** align with expected local/operator defaults after upgrade.
3. **Refresh Zalo Personal sessions:** run `openclaw channels login --channel zalouser` to migrate from external CLI transport dependency.
4. **Audit plugin route registrations:** update any custom plugins using removed `api.registerHttpHandler`.
5. **Run `openclaw config validate` once post-upgrade:** catch invalid-key paths and schema migrations before gateway restart.
6. **Test Telegram streaming defaults:** verify partial mode meets operator UX expectations.
7. **Verify webhook auth for BlueBubbles/GoogleChat/LINE paths:** confirm no unintended 202 responses remain on missing auth and large unauthenticated bodies are bounded.
8. **Enable plugin/session hook telemetry:** validate downstream integrations expecting `session_start`, `message:sent`, and `message:preprocessed` still receive full context.

---

## OpenClaw v2026.3.1 — Historical Release Summary

## Highlights

- **Gateway health probes:** built-in `/health`, `/healthz`, `/ready`, `/readyz` HTTP endpoints for Docker/Kubernetes liveness and readiness checks. Probe handlers run when no plugin route matches those paths, so existing plugin routes retain precedence.
- **OpenAI Responses WebSocket transport:** `openai` provider defaults to WebSocket-first (`transport: "auto"`) with SSE fallback, shared WS stream/connection runtime, per-session cleanup, and server-side compaction payload mutation on the WS path.
- **Telegram DM topics:** per-DM `direct` + topic config (allowlists, `dmPolicy`, `skills`, `systemPrompt`, `requireTopic`) with topic-aware authorization, debounce, and distinct inbound/outbound session routing.
- **Discord thread inactivity lifecycle:** `idleHours` (default 24h) and optional `maxAgeHours` replace fixed TTL, with `/session idle` + `/session max-age` commands.
- **Android nodes expansion:** camera, device health, notifications, motion sensors, voice TTS, contacts, calendar, and photos tools for Android node runtimes.
- **Feishu massive hardening:** 20+ changes covering docx tables, reactions, chat tools, reply-in-thread, multi-account routing, group session scopes, typing backoff, post markdown parsing, media attachment fixes, and prompt-spoofing prevention.
- **Subagent runtime events:** typed `task_completion` internal events replace ad-hoc system-message handoff, with gateway/CLI plumbing for structured `internalEvents`.
- **Security hardening:** prompt spoofing prevention (Feishu preview injection blocked), Feishu webhook ingress bounded rate-limit state, gateway WS flood protection for unauthorized request floods, and various auth/routing fixes.

For full detail, see the v2026.3.1 notes in the upstream release changelog (`openclaw/openclaw` tag `v2026.3.1`).

## Change Distribution (By Top-Level Area)

| Area | Files changed |
| --- | ---: |
| `src/` | 593 |
| `extensions/` | 194 |
| `docs/` | 56 |
| `ui/` | 32 |
| `apps/` | 17 |
| `scripts/` | 13 |
| `test/` | 2 |

*Note: This distribution is a sampled breakdown; additional files in generated, tooling, or misc paths make up the remaining count to match the stated total.*

## Breaking / Behavior Shifts

1. **Node exec approval payloads now require `systemRunPlan`.** `host=node` approval requests without that plan are rejected outright.
2. **Node `system.run` pins path-token commands to canonical executable paths (`realpath`)** in both allowlist and approval execution flows. Integrations/tests that asserted token-form argv (e.g. `tr`) must now accept canonical paths (e.g. `/usr/bin/tr`).

## Major Features

- **Gateway health probes** (`/health`, `/healthz`, `/ready`, `/readyz`) for Docker/K8s container orchestration.
- **OpenAI Responses WebSocket-first transport** with SSE fallback, per-session cleanup, and compaction payload mutation.
- **Telegram DM topics** with per-DM topic config, session routing, and topic-aware authorization/debounce.
- **Discord thread inactivity lifecycle** (`idleHours`/`maxAgeHours`) replacing fixed TTL.
- **Android nodes parity:** `camera.list`, `device.permissions`, `device.health`, `notifications.actions`, `system.notify`, `photos.latest`, `contacts.search`/`contacts.add`, `calendar.events`/`calendar.add`, `motion.activity`/`motion.pedometer`, plus voice TTS via ElevenLabs WebSocket.
- **Feishu docx tables + uploads:** `create_table`, `write_table_cells`, `create_table_with_values`, `upload_image`, `upload_file` actions on `feishu_doc` tool.
- **Feishu reactions:** inbound `im.message.reaction.created_v1` handling with verified reaction routing and fail-closed filtering.
- **Feishu chat tooling:** `feishu_chat` tool for chat info and member queries (`channels.feishu.tools.chat`).
- **Feishu reply-in-thread:** `replyInThread` config (`disabled|enabled`) for group replies with topic-scoped session routing.
- **Feishu group session scopes:** configurable `group`, `group_sender`, `group_topic`, `group_topic_sender` with legacy `topicSessionMode` compatibility.
- **CLI `config file`:** prints active config path resolved from `OPENCLAW_CONFIG_PATH` or the default location.
- **Diffs plugin tool:** read-only diff rendering from before/after text or unified patches, with gateway viewer URLs for canvas and PNG.
- **Memory LanceDB custom config:** support custom OpenAI `baseUrl` and embedding dimensions.
- **ACP/ACPX streaming:** pinned to `0.1.15` with configurable command/version probing and streamlined `final_only` delivery.
- **Shell env markers:** `OPENCLAW_SHELL` set across `exec`, `acp`, `acp-client`, `tui-local` runtimes.
- **Cron/heartbeat light bootstrap context:** opt-in lightweight bootstrap mode (`--light-context`, `agents.*.heartbeat.lightContext`).
- **Subagent runtime events:** typed `task_completion` events with gateway/CLI `internalEvents` plumbing.
- **Web UI cron i18n + German locale:** localized cron pages plus German (`de`) locale support.

## High-Signal Runtime Fixes

- Feishu multi-account routing with `defaultAccount` outbound routing support, cross-channel session disambiguation with inbound `account_id` metadata.
- Feishu prompt spoofing prevention: inbound message previews no longer enqueued as system events, blocking preview text from leaking into trusted `System:` context.
- Feishu typing backoff: rate-limit and quota errors now re-thrown so keepalive circuit breaker stops retries.
- Feishu post markdown parsing: shared markdown-aware parser with locale-wrapper support and code fidelity.
- Telegram reply media context: replied media files included in inbound context with DM authorization gating.
- Telegram outbound chunking: oversize splitting routed through shared outbound pipeline with retry on escaped HTML limits.
- Slack user-token resolution normalized through resolved account metadata for consistent token sourcing.
- Slack native commands: `/agentstatus` registration to avoid Slack-reserved `/status` conflict.
- LINE voice transcription: M4A classified as `audio/mp4` instead of `video/mp4`.
- Cron delivery mode `none` now correctly disables agent messaging tool.
- Sessions list transcript paths handle missing/relative/template values correctly.
- Model directives: email-based auth profile IDs now parsed correctly (split at first `@` after last slash).
- Thinking fallback: retry with `think=off` when providers reject unsupported thinking levels without alternatives.
- Failover reason classification: avoid false TPM rate-limit classification from incidental substrings.
- OpenAI Responses compaction: unified store patches, `compat.supportsStore=false` honored, auto-injected `context_management`.
- Gateway WS: repeated post-handshake unauthorized request floods per connection now closed with sampled rejection logs.
- Usage normalization: negative prompt/input token values clamped to zero.
- Auto-reply `NO_REPLY` token stripped from mixed-content messages.

## Security Hardening

- **Prompt spoofing prevention:** Feishu inbound message previews blocked from system-event injection.
- **Feishu webhook ingress:** bounded rate-limit state with stale-window pruning and hard key cap prevents unbounded pre-auth memory growth.
- **Gateway WS flood protection:** close repeated post-handshake unauthorized request floods per connection with sampled rejection logs.
- **Feishu reaction verification:** timeout + fail-closed filtering drops non-bot or unverified reactions.
- **Compaction audit:** post-compaction audit injection message removed.
- **Web tools RFC2544:** benchmark range allowed for trusted endpoints to avoid false SSRF blocks through proxy fake-IP networking.

## Maintainer Upgrade Checklist (v2026.3.1)

1. **Node exec approvals:** verify all `host=node` exec approval integrations send `systemRunPlan` in payloads; requests without it are now rejected.
2. **Node system.run canonical paths:** update any tests or integrations that assert token-form argv (e.g. `tr`) to accept canonical `realpath` form (e.g. `/usr/bin/tr`).
3. **Feishu prompt spoofing:** confirm inbound preview text is no longer appearing in agent system context after upgrade.
4. **Feishu multi-account routing:** if using multiple Feishu accounts, configure `channels.feishu.defaultAccount` for explicit outbound routing.
5. **Feishu group session scopes:** review group session isolation settings if using topic-based or sender-based session routing.
6. **Discord thread lifecycle:** audit any Discord thread TTL configurations; `idleHours` (default 24h) replaces fixed TTL.
7. **Telegram DM topics:** if using DM topics, configure per-DM topic allowlists, `dmPolicy`, and `requireTopic` settings.
8. **Gateway health probes:** verify `/health`, `/healthz`, `/ready`, `/readyz` paths are not already claimed by custom handlers.
9. **OpenAI WS transport:** test WebSocket-first transport behavior; use `transport: "sse"` to opt out if needed.
10. **Gateway WS flood protection:** verify no misbehaving clients are causing repeated unauthorized request floods; these connections are now actively closed.

---

## OpenClaw v2026.2.26 — Release Summary

> **Released:** 2026-02-27 | **Policy note:** latest *documented* released section stays at top.
> **Window analyzed:** `v2026.2.25..v2026.2.26` | **Scan stats:** 340 commits, 856 files changed, +58,881 / -7,073 lines

## Highlights

- **External secrets management:** new `openclaw secrets` workflow (audit/configure/apply/reload) with strict path validation and safer migration scrubbing.
- **ACP thread-bound agents:** ACP runtimes now own thread sessions with lifecycle controls and coalesced thread replies.
- **Agents routing CLI:** `openclaw agents bindings/bind/unbind` for account-scoped route management and binding upgrades.
- **Codex WebSocket transport:** `openai-codex` defaults to WebSocket-first with SSE fallback and transport docs.
- **Onboarding + reliability fixes:** plugin-owned onboarding hooks, Gemini CLI OAuth risk confirmation, DM allowlist inheritance enforcement, delivery-queue backoff, and typing cleanup guardrails.

For full detail, see the v2026.2.26 notes in the upstream release changelog (`openclaw/openclaw` tag `v2026.2.26`).

---

## OpenClaw v2026.2.25 — Release Summary

> **Released:** 2026-02-26

## Highlights

- **Breaking behavior shift:** heartbeat direct/DM delivery default changed back to `allow`; use `agents.defaults.heartbeat.directPolicy: "block"` to preserve v2026.2.24 behavior.
- **Security hardening wave:** broad ingress authorization enforcement for reaction/pin/interaction system events across channels, plus gateway/browser auth and workspace FS protections.
- **Delivery reliability upgrades:** subagent announce/delivery pipeline hardened with clearer queue/direct/fallback behavior and stricter Telegram delivery success criteria.
- **Webhook/session stability:** Telegram webhook and polling handling hardened; Slack parent-thread forking gained guardrails (`session.parentForkMaxTokens`).
- **Model/fallback resilience:** multiple fallback traversal and cooldown classification fixes keep failover moving under more error and profile-state combinations.

For full detail, see the v2026.2.25 notes in the upstream release changelog (`openclaw/openclaw` tag `v2026.2.25`).

---

# Synthesis Review: v2026.2.23 → v2026.2.24 (Released)

> **Window analyzed:** `v2026.2.23..v2026.2.24`
> **Release date:** 2026-02-25
> **Scan stats:** 228 commits, 458 files changed, +20,743 / -4,989 lines

## Change Distribution (By Top-Level Area)

| Area | Files changed |
| --- | ---: |
| `src/` | 224 |
| `apps/` | 34 |
| `extensions/` | 63 |
| `docs/` | 28 |
| `ui/` | 12 |
| `scripts/` | 5 |
| `test/` | 4 |

*Note: This distribution is a sampled breakdown; additional files in generated, tooling, or misc paths make up the remaining count to match the stated total.*

## Breaking / Behavior Shifts

1. Heartbeat delivery now blocks direct/DM targets; heartbeat still runs but only non-DM destinations can receive outbound delivery.
2. Sandbox Docker `network: "container:<id>"` namespace-join mode is blocked by default; break-glass enablement now requires explicit config (`dangerouslyAllowContainerNamespaceJoin: true`).
3. Heartbeat default target changed from `last` to `none`, making external heartbeat delivery explicit opt-in.

## High-Signal Runtime Fixes

- Shared-session routing hardened to fail closed for cross-channel replies and preserve originating channel metadata through followup/queue paths.
- Typing keepalive lifecycle improved across core and extension dispatch paths so indicators survive long inference runs.
- Discord voice path reliability improved (DAVE dependency/runtime handling, decrypt-failure tolerance and recovery logic, stale listener cleanup).
- Fallback traversal on model fallback chains fixed to continue through configured fallbacks when already on fallback models.
- Gateway model allowlist behavior fixed so explicit allowlisted refs remain selectable even when bundled catalog metadata is stale.

## Security Hardening Themes

- Exec approval/display binding tightened so shell-wrapper payloads cannot drift from approved command text.
- Workspace and sandbox path guards tightened (including `@` path normalization and tmp-media hardlink alias rejection).
- Ingress authorization hardened across channel-specific paths (for example Telegram DM media write authorization and Synology Chat DM fail-closed allowlists).
- Safe-bin trust defaults tightened to immutable system directories with stronger audit/doctor guidance for risky trusted-dir configurations.

## Maintainer Upgrade Checklist (v2026.2.24)

1. Verify heartbeat targets: if external delivery is expected, set explicit non-DM targets; default `none` and DM blocking are now enforced.
2. Audit sandbox Docker settings for namespace-join assumptions; enable break-glass key only where intentionally required.
3. Re-test shared-session multi-channel routes and followup behavior (especially Discord/Webchat mixed sessions).
4. Re-run security audit and review safe-bin trusted-dir findings after upgrade.
5. Validate channel-specific reliability paths you depend on (Discord voice, WhatsApp reconnect, typing keepalive).

---

# Changelog: v2026.2.14 -> v2026.2.15

> **Released:** 2026-02-16 | **Commits:** 705 | **Files changed:** 1,492 | **Lines changed:** 108,031 (+62,497 / -45,534)

---

## Security Fixes (12)

| Change | Impact |
|--------|--------|
| Harden sandbox Docker config validation | Prevents config injection in sandbox setup |
| Harden prompt path sanitization | Prevents directory traversal in prompt file paths |
| Restrict skill download target paths (`infra/install-safe-path.ts`) | Prevents writes outside allowed directories |
| Scope session tools and webhook secret fallback | Prevents cross-session tool/secret leakage |
| Preserve control-UI scopes in bypass mode | Scope enforcement even when bypass is active |
| Scope pairing stores by account (`pairing/pairing-store.ts`) | Prevents cross-account pairing data access |
| Account-scoped pairing allowlists (Telegram/WhatsApp) | Allowlists isolated per account |
| Redact Telegram bot tokens in errors | Prevents token leakage in error messages |
| Redact sensitive status details for non-admin scopes | Status API respects scope restrictions |
| Harden `chat.send` message input sanitization | Prevents injection via message tool |
| LINE webhook fail-closed when auth missing | Rejects unauthenticated LINE webhooks instead of passing through |
| Control UI XSS fix (JSON endpoint + CSP lockdown) | Prevents XSS via control UI |

## Features (11)

| Feature | Files | PR |
|---------|-------|----|
| Discord Component v2 UI tool support | `discord/components.ts`, `discord/components-registry.ts`, `discord/send.components.ts` | #17419 |
| Cross-platform skill install fallback for non-brew | `infra/install-safe-path.ts` | #17687 |
| Account selector for pairing commands (CLI) | `pairing/pairing-labels.ts`, CLI commands | — |
| Cron finished-run webhook | `cron/` service | #14535 |
| Plugin LLM input/output hook payloads | `plugins/hooks.ts`, `plugins/types.ts` | #16724 |
| Multi-image support in image tool | `agents/tools/image-tool.ts`, `agents/tools/image-tool.helpers.ts` | #17512 |
| Per-channel `ackReaction` config | `config/types.telegram.ts`, `types.discord.ts`, `types.slack.ts`, `types.whatsapp.ts` | #17092 |
| Nested subagent orchestration controls | `agents/subagent-depth.ts`, `agents/subagent-announce-queue.ts`, `agents/tools/subagents-tool.ts` | #14447 |
| `messages.suppressToolErrors` config option | `config/types.messages.ts` | #16620 |
| Gateway: preserve partial output on abort | `gateway/` | #15026 |
| Model fallback support for `sessions_spawn` | `agents/` spawn logic | #17197 |

## Memory / QMD (6)

- Isolate managed collections per agent
- Rebind drifted managed collection paths
- Harden context window cache collisions
- Support unicode tokens in FTS query builder
- Inject runtime date-time into memory flush prompt
- Verify QMD index artifact after manual reindex

## Cron (5)

- Normalize skill-filter snapshots
- Pass agent-level skill filter to isolated cron sessions
- Treat missing `enabled` as `true` in `update()`
- Infer payload kind for model-only update patches
- Preserve cron prompts for tagged interval events

## Telegram (8)

- Simplify send/dispatch/target handling (major refactor)
- Stop block streaming splitting messages when `streamMode` off
- Treat no-op `editMessage` as success
- Stream replies in-place without duplicate final sends
- Stop dropping voice messages on `getFile` network errors
- Include voice transcript in body text
- Restore `thread_id=1` handling for DMs
- Legacy `allowFrom` migration to account-scoped format

## Performance (6)

- Massive test consolidation (100+ test refactors)
- Lazy-load heavy deps in models list, onboarding, plugins, telegram
- Speed up Slack/Gateway/Telegram test suites
- Avoid async cron timer callbacks
- Skip skill scans for inline directives
- Isolate browser profile hot-reload

## Refactoring (Major)

- **300+ refactor commits** deduplicating code across all modules
- Centralized helpers: SHA-256, env snapshots, JSON file locks, etc.
- Shared test harnesses across modules
- Total file count reduced from ~3,051 to ~2,980 despite new features

## Other Notable Changes

- Preserve OpenAI reasoning replay IDs
- Fix Codex/PTY process spawning (#14257)
- Discord role-based allowlist fix (#16369)
- Session key normalization fix
- Gateway: keep boot sessions ephemeral
- Honor configured `contextWindow` overrides
- Gateway `config.patch` merge object arrays by ID

## Documentation Updates

- Updated `DEVELOPER-REFERENCE.md`: new high-blast-radius files, v2026.2.15 gotchas, new config keys
- Updated `ARCHITECTURE.md`: security hardening section, updated stats, Discord Component v2, account-scoped pairing
- Updated `README.md`: version bump, updated file counts

## Breaking Changes

None — all changes are backward compatible. Legacy `allowFrom` in Telegram is auto-migrated.

---

*Generated: 2026-02-16*

---

# Synthesis Review: v2026.2.15 → v2026.2.17

> **Window analyzed:** `v2026.2.15..v2026.2.17` (979 commits)
> **Method:** prioritize behavior and operator/developer impact; ignore low-signal test churn

---

## Executive synthesis

This release window is best understood as a **runtime-hardening and orchestration-stability cycle** with four dominant themes:

1. **Subagent execution became more deterministic** (spawn/announce semantics, routing, completion reliability).
2. **Cron scheduling became safer and more explicit** (default stagger behavior + operator controls).
3. **Security boundaries tightened** (notably config `$include` confinement and sandbox/input hardening).
4. **Tool/runtime behavior shifted toward guarded streaming + loop control** (Z.AI tool-stream default, stronger loop detection, read-context guard behavior).

The most important takeaway for maintainers is not a new feature checklist; it is that **assumptions that used to be “soft defaults” are now enforced behaviors**.

---

## What changed in practice (high signal)

### 1) Subagents: from best-effort to managed flow

Subagent workflows now lean heavily into push-based lifecycle semantics:

- `/subagents spawn` adds deterministic command-path spawning.
- `sessions_spawn` user experience now explicitly sets expectation that completion is auto-announced.
- Completion routing/retry/output selection has been hardened.

**Operational implication:**
Do not design runbooks around aggressive polling of spawned work. Treat spawned runs as async jobs with eventual callback/announce.

### 2) Cron: schedules now carry intent, not just expression

Top-of-hour recurring jobs now receive deterministic stagger by default and persist it as `schedule.staggerMs`; operators can force exact timing via `--exact` / `staggerMs: 0`.

**Operational implication:**
If your mental model is “`0 * * * *` means exact wall-clock fire”, that is no longer universally true. Exactness must be explicit.

### 3) Security posture: config/file boundaries tightened

The most consequential hardening in this window is strict confinement of config `$include` paths to config-root boundaries (with traversal/symlink guards), plus additional sandbox/env sanitization work.

**Operational implication:**
Older include layouts that reached outside config root may fail after upgrade; this is expected security behavior, not parser regression.

### 4) Tool/runtime behavior: safer defaults, stricter failure handling

- Z.AI tool-call streaming defaults to enabled.
- Loop detection now escalates/no-progress blocks on repeated poll/log patterns.
- Read-context handling emphasizes bounded incremental reads over full-dump retries.

**Operational implication:**
Agent/tool orchestration prompts and automation scripts need explicit progress checks, bounded retries, and chunked-read recovery patterns.

---

## Recommended maintainer workflow updates

1. **Before blaming runtime bugs, validate config confinement constraints first** (`$include` path containment, symlink behavior).
2. **When debugging cron timing, inspect `schedule.staggerMs`**, not only cron expression.
3. **Treat subagent completion as announce-driven**, and only poll for intervention/debug.
4. **For tool loops, require no-progress detection + backoff in agent logic docs/tests.**
5. **For large file analysis flows, enforce paged reads (`offset`/`limit`) in prompts and examples.**

---

## Docs in this repo updated for this window

- `README.md` — version/index updates for v2026.2.17 docs.
- `DEVELOPER-REFERENCE.md` — concrete gotchas + workflow guidance for this release window.
- `CHANGELOG.md` — this synthesized review (consolidated section for v2026.2.15 → v2026.2.17).

---

*Generated: 2026-02-18*
---

# Synthesis Review: v2026.2.17 → v2026.2.19

> **Window analyzed:** `v2026.2.17..v2026.2.19` (572 commits)
> **Method:** behavior and operator/developer impact focus; ignore low-signal test/style churn
> **Tag/package note:** stable release tag is `v2026.2.19`; npm also published a repackaged build `2026.2.19-2` in this release window.

---

## Executive synthesis

This release window is a **security-surge + runtime-stability cycle** defined by four major themes:

1. **Security hardening across virtually every surface** — 40+ distinct security fixes spanning SSRF, plugin integrity, exec sandboxing, gateway auth, canvas session scoping, and protocol injection vectors. This is the most security-dense release in the observed history of these docs.
2. **Gateway auth semantics shifted to secure-by-default** — token auth is now the default; explicit `"none"` is required for intentional open loopback setups, and a new security audit fires when `no_auth` is detected with remote exposure risk.
3. **Cron/heartbeat delivery fixed in multiple important ways** — explicit Telegram topic targets now work, heartbeat skips when no content exists, cron announces route correctly, and spin-loop prevention is now solid.
4. **iOS/APNs platform matured** — full Apple Watch companion, APNs wake-before-invoke, share extension, and major pairing/auth stabilization. Low impact for this setup but shows platform direction.

The most important operational takeaway: **security boundaries that were advisory in 2026.2.17 are now enforced or audited at boot**. Review gateway auth, plugin integrity, hook token uniqueness, and exec safeBins settings before upgrading.

---

## What changed in 2026.2.19 for us

### 1) Gateway auth is now token-mode by default

Gateway auth now defaults to token mode with auto-generation and persistence of `gateway.auth.token`. Explicit `gateway.auth.mode: "none"` is required for intentional open loopback setups.

Additionally, the new security audit (`Security/Audit`) emits `gateway.http.no_auth` findings when `mode="none"` leaves gateway HTTP APIs reachable, with loopback-only → warning and remote-exposure → critical severity.

**Impact for this setup:** If the current config relies on implicit open auth, expect audit warnings after upgrade. Review `openclaw security audit` output post-upgrade.

### 2) `hooks.token` must differ from `gateway.auth.token` — startup failure if equal

New startup validation rejects configs where `hooks.token` matches `gateway.auth.token`. The gateway fails to start if these values are identical.

**Impact:** If these were previously set to the same value, the gateway will refuse to start after upgrade. Separate them before upgrading.

### 3) Heartbeat behavior: skip when HEARTBEAT.md missing or empty

Interval heartbeats are now skipped automatically when `HEARTBEAT.md` is missing or empty and no tagged cron events are queued. The cron-event fallback for queued tagged reminders is preserved.

**Impact for this setup:** This is the correct behavior — less noise, fewer empty heartbeat runs. No action needed unless you expected heartbeats to fire on empty HEARTBEAT.md.

### 4) Cron/heartbeat Telegram topic delivery now works

Explicit Telegram topic targets in cron and heartbeat delivery (`<chatId>:topic:<threadId>`) now correctly route scheduled sends into the configured topic instead of defaulting to the last active thread.

**Impact:** If heartbeat/cron sends were previously landing in the wrong Telegram thread, this is the fix. No config change required — existing `<chatId>:topic:<threadId>` targets now work correctly.

### 5) YAML frontmatter uses YAML 1.2 core schema — `on`/`off` are now strings

Frontmatter YAML parsing now uses YAML 1.2 core schema to avoid implicit coercion. Previously, `on`/`off`/`yes`/`no`/`true`/`false` in quoted or unquoted frontmatter could be auto-coerced to booleans.

**Impact:** Any AGENTS.md, HEARTBEAT.md, or cron prompt frontmatter that relied on `on`/`off` being booleans must now be explicit. Use `true`/`false` as YAML boolean literals.

### 6) Browser/Relay requires gateway-token auth on both `/extension` and `/cdp`

The Chrome extension relay endpoint and the CDP endpoint now require `gateway.auth.token` authentication. The Chrome extension setup aligns to a single `gateway.auth.token` input.

**Impact for this setup:** If the browser relay was previously accessible without auth, it now requires the token. The `browser` tool in agent context handles this automatically, but any manual or external relay clients must supply the token.

### 7) `read` tool auto-pages based on model context window

The `read` tool now auto-pages across chunks when no explicit `limit` is provided, scaling its per-call output budget from the model's `contextWindow`. Larger-context models can read more before context guards kick in.

**Impact:** Less need for manual `offset`/`limit` for small-to-medium files. Context compaction (`[compacted: tool output removed to free context]`) in large reads now recovers more gracefully.

### 8) exec tool has preflight guard for shell env var injection

A new preflight guard detects likely shell env var injection patterns (e.g., `$DM_JSON`, `$TMPDIR`) in Python/Node scripts before execution, preventing recurring cron failures when models emit mixed shell+language source.

**Impact:** Cron jobs that previously silently failed from env-var-in-code patterns now get early warnings instead of wasted tokens and failed runs.

### 9) SSRF hardening: NAT64/6to4/Teredo/octal/hex IPv4 now blocked

SSRF bypass vectors via IPv6 transition addresses (NAT64 `64:ff9b::/96`, 6to4 `2002::/16`, Teredo `2001:0000::/32`) and non-standard IPv4 forms (octal, hex, short, packed — e.g., `0177.0.0.1`, `127.1`, `2130706433`) are now blocked.

**Impact:** Low operational impact for normal use, but important for security posture. Any automation that targets private addresses via these forms will be blocked.

### 10) macOS LaunchAgent SQLite fix: `TMPDIR` now forwarded to service environment

`TMPDIR` is now forwarded into installed service environments, resolving `SQLITE_CANTOPEN` failures in macOS LaunchAgent gateway runs when SQLite can't write temp/journal files.

**Impact for this setup:** If the gateway daemon was experiencing intermittent SQLite errors under macOS, this is the fix. No config change needed.

### 11) Canvas session capabilities now node-scoped (not shared-IP fallback)

The `/__openclaw__/canvas/*` and `/__openclaw__/a2ui/*` endpoints now use node-scoped session capability URLs instead of shared-IP fallback auth, failing closed when trusted-proxy requests omit forwarded client headers.

**Impact:** Canvas/A2UI automation that relied on shared-IP auth will need to use scoped session URLs. Gateway-proxied canvas calls via normal tool paths are unaffected.

### 12) Security/Exec: `safeBins` now reject PATH-hijacked binaries

`tools.exec.safeBins` binaries must now resolve from trusted bin directories (system defaults plus gateway startup `PATH`). A trojan binary in a user-added PATH directory with the same name as a safe bin will be rejected.

**Impact:** Better security posture. No impact unless you have non-standard PATH layouts during gateway startup.

### 13) Cron webhook delivery now SSRF-guarded

Cron webhook POST delivery now routes through SSRF-guarded outbound fetch (`fetchWithSsrFGuard`), blocking private/metadata destinations before dispatch.

**Impact:** Webhook targets pointing to private addresses (localhost, RFC1918) will be rejected. Use only publicly reachable webhook endpoints.

### 15) Plaintext `ws://` connections blocked to non-loopback hosts

Plaintext `ws://` WebSocket connections to non-loopback hosts are now rejected. Secure `wss://` transport is required for remote WebSocket endpoints.

**Impact:** Any automation or integration connecting to remote WebSocket endpoints via `ws://` must switch to `wss://`. Loopback (`ws://localhost`, `ws://127.0.0.1`) is still allowed.

### 16) Control-plane write RPCs are now rate-limited

`config.apply`, `config.patch`, and `update.run` RPCs are rate-limited to 3 requests per minute per `deviceId+clientIp`. Gateway restarts are coalesced with a 30-second cooldown, and config change audit details (actor, device, IP, changed paths) are now logged.

**Impact:** Automation that rapidly applies config changes (e.g., scripted `config.patch` loops) will hit 429 rate limits. Space out config writes or batch changes into single `config.apply` calls.

### 17) Discord moderation actions require trusted-sender guild permissions

Moderation actions (`timeout`, `kick`, `ban`) now enforce guild permission checks on the trusted sender and ignore untrusted `senderUserId` parameters. This prevents privilege escalation via tool-driven flows.

**Impact:** Discord operators using moderation tools must ensure the bot/sender has appropriate guild permissions. Untrusted sender IDs in tool parameters are now rejected.

### 14) Sub-agent context guard + compacted-output recovery guidance

Accumulated tool-result context is now guarded before model calls, truncating oversized outputs and compacting oldest tool-result messages to avoid context-window overflow crashes. Explicit guidance added to recover from `[compacted: ...]` / `[truncated: ...]` markers by re-reading with smaller chunks.

**Impact for this setup:** Sub-agent runs that previously crashed on large tool outputs now degrade more gracefully. Agent prompts in AGENTS.md that handle compacted output are already correct.

---

## Recommended operational checks after upgrading to 2026.2.19

1. **Run `openclaw security audit`** — check for `gateway.http.no_auth` and any new plugin/hook findings.
2. **Verify `hooks.token` ≠ `gateway.auth.token`** — gateway will refuse to start if they match.
3. **Check `gateway.auth.mode`** — if previously relying on implicit open mode, you may need to add `mode: "none"` explicitly (or configure a token).
4. **Review YAML frontmatter in agent prompts** — switch `on`/`off` to `true`/`false` where boolean semantics are intended.
5. **Test cron/heartbeat Telegram topic routing** — if using `<chatId>:topic:<threadId>` targets, verify sends now land in the correct topic.
6. **Check browser relay clients** — any non-standard browser relay clients need gateway-token auth on `/extension` and `/cdp`.
7. **Review cron webhook targets** — ensure all webhook URLs are publicly reachable (not localhost/RFC1918).
8. **Switch remote WebSocket connections to `wss://`** — plaintext `ws://` to non-loopback hosts is now blocked.
9. **Check config automation pacing** — control-plane RPCs (`config.apply`, `config.patch`, `update.run`) are rate-limited to 3/min per device+IP.
10. **Verify Discord moderation bot permissions** — moderation actions now enforce guild permission checks on the trusted sender.

---

## Docs in this repo updated for this window

- `README.md` — version bump to v2026.2.19, changelog table updated.
- `DEVELOPER-REFERENCE.md` — v2026.2.19 workflow additions, new gotchas (items 33–45).
- `CHANGELOG.md` — this synthesized review (consolidated section for v2026.2.17 → v2026.2.19).

---

*Generated: 2026-02-20*

---

# Synthesis Review: v2026.2.19 → v2026.2.21

> **Window analyzed:** `v2026.2.19..v2026.2.21` (~290 commits, 832 files changed, +45,078/-9,997 lines)
> **Method:** behavior and operator/developer impact focus; ignore low-signal test/style churn
> **Package version confirmed:** `2026.2.21` (npm)

---

## Executive synthesis

This release window is a **provider expansion + Discord maturity + security breadth cycle** defined by four major themes:

1. **New model providers and per-channel routing controls** — Gemini 3.1 and Volcengine/Byteplus (Doubao) land as first-class providers; per-channel model overrides via `channels.modelByChannel` and per-account/channel `defaultTo` outbound routing give operators fine-grained delivery control without rewriting agent configs.
2. **Discord platform matured to production-grade** — thread-bound subagents, voice channel support, stream preview mode, configurable lifecycle status reactions, ephemeral slash-command defaults, forum tag management, and channel topic metadata land as a cohesive batch. Discord operators should expect significant new surface area to configure.
3. **Security breadth across execution, sandbox, networking, and prompt surfaces** — 40+ discrete security fixes covering heredoc allowlist bypass, shell startup-file injection, WhatsApp JID auth, sandbox browser hardening, prototype-chain traversal in webhook templates, compaction retry amplification, TTS provider-hop injection, and many more. Less concentrated than v2026.2.19's security surge but broader in scope.
4. **Telegram, memory, and cron reliability** — Telegram streaming reworked (simplified config, split reasoning/answer lanes, draft cleanup), QMD memory search diversified for multi-source ranking, heartbeat behavior regression partially reversed, and cron `maxConcurrentRuns` actually honored.

The most important operational takeaway: **this release is safe to upgrade to for macOS/Telegram setups but requires attention for any deployment using Discord, WhatsApp, BlueBubbles, sandbox browser, or advanced exec/shell tooling**. Security fixes in particular affect exec allowlists, heredoc parsing, and startup-file injection — review the operational checks section before upgrading in production.

---

## What changed in 2026.2.21 for us

### 1) Telegram streaming config simplified — `channels.telegram.streaming` boolean

Telegram preview streaming is now configured as a single boolean `channels.telegram.streaming: true/false`. The legacy `streamMode` values (`partial`, `block`, etc.) are auto-mapped on load, so existing configs continue to work without changes. The internal partial/block branching has been removed.

Additionally, streaming previews are now split into separate reasoning and answer lanes, preventing cross-lane overwrites when models emit interleaved reasoning + answer blocks. The first-preview 30-char debounce is restored, and `NO_REPLY` prefix suppression is scoped to partial sentinel fragments only.

**Impact for this setup (@MudrionBot):** No config migration required — legacy `streamMode` values auto-map. The reasoning/answer lane split is a quality improvement: if you were seeing garbled streaming output when using thinking-capable models (e.g., `kimi-coding/k2p5`), this release fixes it. Draft cleanup is now guaranteed even when dispatch throws.

### 2) Per-channel model overrides via `channels.modelByChannel`

Operators can now configure different models for different channels/accounts using `channels.modelByChannel`. The active model override is surfaced in `/status` output so it is visible during diagnostic sessions.

Also added: per-account/channel `defaultTo` outbound routing fallback so `openclaw agent --deliver` can send without requiring an explicit `--reply-to` when a default target is configured for the account/channel.

**Impact for this setup:** Useful if you want to use a different model for Telegram bot responses vs. gateway-direct sessions. Add `channels.modelByChannel` entries per channel ID to override the primary model on a per-source basis.

### 3) New model providers — Gemini 3.1 and Volcengine/Byteplus (Doubao)

`google/gemini-3.1-pro-preview` is now a first-class model in the catalog. Volcengine (Doubao) and BytePlus providers are added with coding variants, interactive and non-interactive onboarding flows, and documentation aligned to `volcengine-api-key`.

The `kimi-coding` implicit provider template is also fixed — previously, `kimi-coding/k2p5` returned 403 errors due to a missing provider template with the correct `anthropic-messages` API type and base URL. This is a direct fix for the current workaround in use.

**Impact for this setup:** The Kimi-coding fix is directly relevant — `kimi-coding/k2p5` now works without the manual `models.generated.js` patch workaround documented in MEMORY.md. When upgrading to v2026.2.21, the 403 errors from Kimi should be resolved natively. Gemini 3.1 is available as a fallback option if needed.

### 4) Cron `maxConcurrentRuns` now actually honored

The cron timer loop now respects `cron.maxConcurrentRuns` so due jobs can execute up to the configured parallelism instead of always running serially. Previously, this config key was accepted but had no effect in the timer loop.

**Impact for this setup:** If you have multiple cron jobs with overlapping schedules and had set `maxConcurrentRuns` expecting parallel execution, it now works. If you prefer serial cron execution, verify your `maxConcurrentRuns` value (default is 1 if unset, which preserves serial behavior).

### 5) Heartbeat behavior regression — missing `HEARTBEAT.md` no longer suppresses runs

The v2026.2.19 behavior change that skipped heartbeats when `HEARTBEAT.md` was missing or empty has been partially reversed. Missing `HEARTBEAT.md` no longer suppresses runs; only effectively empty files now skip. Prompt-driven and tagged-cron execution paths are preserved regardless.

**Impact for this setup:** If you removed or never created `HEARTBEAT.md` expecting heartbeats to fire anyway, they now will again. If you had an empty `HEARTBEAT.md` and want heartbeats to fire, add actual content. The tagged-cron fallback is unchanged.

### 6) Compaction safeguard now loads in production builds

The embedded compaction safeguard/context-pruning extension failed to load in production builds due to runtime file-path resolution failure. It is now wired into the bundled extension factory so it activates correctly in installed (non-dev) deployments.

**Impact for this setup:** If running a production-installed `openclaw` (not `pnpm dev`), the compaction safeguard was silently absent in v2026.2.19. It is now active. This means long-running agent sessions will have proper context compaction behavior and be less likely to hit context-window exhaustion.

### 7) Security: exec and shell allowlist hardening

Multiple exec-path security fixes in this release that affect any deployment using `tools.exec.safeBins` or shell tooling:

- **Heredoc allowlist bypass blocked** — unquoted heredoc body expansion tokens are now blocked in shell allowlist analysis; unterminated heredocs are rejected; explicit approval is required for allowlisted heredoc execution on gateway hosts.
- **Shell startup-file injection blocked** — `BASH_ENV`, `ENV`, `BASH_FUNC_*`, `LD_*`, `DYLD_*` are now stripped across config env ingestion, node-host inherited environment sanitization, and macOS exec host runtime.
- **macOS `system.run` allowlists** — evaluated per shell segment in macOS node runtime and companion exec host, including chained operators (`&&`, `||`, `;`). Shell/process substitution parsing fails closed. Unsafe parse cases require explicit approval.
- **Prototype-chain traversal in webhook templates blocked** — `__proto__`, `constructor`, `prototype` keys are blocked in `getByPath` webhook template path resolution.

**Impact for this setup:** If using `tools.exec` with any shell-based tool runs or cron scripts that use heredocs, test your exec paths after upgrading. Legitimate heredoc use that was previously passing through implicitly now requires explicit gateway approval.

### 8) Security: sandbox browser defaults hardened

The sandbox browser container no longer runs with `--no-sandbox` by default. The new default is sandboxed; opt-in requires `OPENCLAW_BROWSER_NO_SANDBOX=1` / `CLAWDBOT_BROWSER_NO_SANDBOX=1` env vars. Additionally:

- noVNC observer sessions now require password auth and use one-time observer tokens.
- Sandbox browser containers default to a dedicated Docker network (`openclaw-sandbox-browser`).
- `openclaw security audit` warns when browser sandbox runs on bridge network without source-range limits.

**Impact for this setup:** No sandbox browser in use on this setup. No action needed.

### 9) Security: WhatsApp JID allowlist enforcement hardened

WhatsApp reaction actions now enforce allowlist JID authorization so authenticated callers cannot target non-allowlisted chats by forging `chatJid` + valid `messageId` pairs. Outbound auth is also centralized and returns HTTP 403 on tool auth errors.

**Impact for this setup:** No WhatsApp in use on this setup. No action needed.

### 10) Security: TTS model-driven provider switching now opt-in

Model-driven TTS provider switching is now opt-in by default (`messages.tts.modelOverrides.allowProvider=false` unless explicitly enabled). Voice/style overrides remain available. This prevents prompt-injection-driven provider hops and unexpected TTS cost escalation.

**Impact for this setup:** No TTS in use on this setup. If TTS is ever enabled, note that `allowProvider=true` must be explicitly set to permit model-driven TTS provider selection.

### 11) Security: compaction retry amplification blocked

The overflow compaction retry budgeting is now global across tool-result truncation recovery. Previously, successful truncation could reset the overflow retry counter and amplify retry/cost cycles — a potential prompt-injection vector. This is now fixed.

**Impact for this setup:** Directly relevant for production agent runs. Long-running sessions with large tool outputs will no longer have their retry counter reset, preventing unexpected cost amplification.

### 12) Security: inbound metadata stripped from chat surfaces

Untrusted inbound user context metadata blocks are now stripped at display boundaries across TUI, webchat, and macOS surfaces, preventing injected metadata from leaking into chat history. User-authored content is preserved.

**Impact for this setup:** Relevant for TUI-based interactions. If you use the TUI (`openclaw` interactive mode), injected metadata from untrusted senders no longer appears in the chat log.

### 13) Auto-updater and `openclaw update --dry-run` added

An optional built-in auto-updater (`update.auto.*`) is now available, default-off. The `openclaw update --dry-run` command lets you check what would be updated without applying changes. Prerelease suffixes are also now correctly ignored in release-check version comparisons.

**Impact for this setup:** `update.auto` is off by default — no change to current behavior. Use `openclaw update --dry-run` to check for available updates without committing to them.

### 14) BlueBubbles (iMessage) channel added

BlueBubbles is now a supported channel for iMessage routing. Authentication is required for all webhook requests (no passwordless fallback). A dedicated `xurl` skill is also added.

**Impact for this setup:** Not currently in use. If iMessage support is wanted in the future, BlueBubbles is the integration path.

### 15) Discord platform features (not in use but changes to monitor)

Discord received significant new surface area: thread-bound subagents (#21805), stream preview mode, configurable lifecycle status reactions with emoji/timing overrides, voice channel join/leave/status via `/vc`, configurable ephemeral defaults, forum `available_tags` management, and channel topics in trusted metadata. Discord message handlers now properly await processing instead of fire-and-forget.

**Impact for this setup:** Discord is not in use. However, if Discord is ever added, the v2026.2.21 config surface for Discord is substantially larger than previous versions. These items require no action now.

### 16) Memory/QMD search reliability improvements

QMD memory search received a broad reliability fix: per-agent `memorySearch.enabled=false` is now respected at gateway startup; multi-collection searches split into per-collection queries to avoid sparse-term drops; collection-hinted doc resolution preferred to avoid stale-hash collisions; boot updates retry on transient lock/timeout failures; `qmd embed` skipped in BM25-only `search` mode; embed runs are serialized globally with failure backoff to prevent CPU storms on multi-agent hosts.

Mixed-source ranking is also fixed so session transcript hits no longer crowd out durable memory-file matches in top results.

**Impact for this setup:** If using QMD memory backend (`memory.backend: qmd`), this is a significant reliability improvement. Multi-agent setups on the same host will no longer cause CPU storms from concurrent embed runs.

### 17) Gateway auth improvements for trusted-proxy and custom bind-host

Several gateway auth edge cases are fixed:

- `trusted-proxy` mode with `bind="loopback"` now requires `gateway.trustedProxies` to include a loopback address, preventing same-host proxy misconfiguration from silently blocking auth.
- `gateway.customBindHost` is now accepted in strict config validation when `bind="custom"`, fixing startup failures for valid custom bind configurations.
- `health` method is now accessible to all authenticated clients across roles/scopes while still enforcing role/scope for other methods.

**Impact for this setup (gateway on port 18789, loopback bind):** If you run a local reverse proxy in front of the gateway, the trusted-proxy loopback requirement now enforces correct config. Standard loopback-direct setups are unaffected.

### 18) CLI and config fixes

Several CLI/config improvements:

- `openclaw config unset <path>` no longer re-introduces defaulted keys (e.g. `commands.ownerDisplay`) through schema normalization after a write.
- `openclaw -v` is preserved as root-only version alias so subcommand `-v, --verbose` flags no longer intercept it globally.
- `config set --strict-json` is now canonical; `--json` is kept as a legacy alias.
- Bootstrap agent files with missing/invalid paths are skipped with a warning instead of crashing the agent session.
- `pairing list` and `pairing approve` now default to the sole available pairing channel when omitted.

**Impact for this setup:** The `config unset` fix is directly relevant — if you have previously used `openclaw config unset` and noticed keys reappearing, this is fixed. The `-v` alias restoration prevents flag parsing confusion in verbose subcommand flows.

### 19) Docker build fixed

Two Docker build issues are resolved: `ownerDisplay` is now included in `CommandsSchema` object-level defaults (fixing `TS2769` build failures), and Playwright Chromium is installed into `/home/node/.cache/ms-playwright` with correct `node:node` ownership so browser binaries are available to the runtime user.

**Impact for this setup:** Not running Docker. No action needed.

### 20) SHA-1 → SHA-256 migration for synthetic IDs

Gateway lock and tool-call synthetic IDs are migrated from SHA-1 to SHA-256 with unchanged truncation length. The owner-ID obfuscation now uses a dedicated `ownerDisplaySecret` HMAC key from configuration, decoupled from the gateway token.

**Impact for this setup:** Transparent upgrade — no config change required. If you have external tooling that parses synthetic IDs, the hash basis changes but the format/length does not. The `ownerDisplaySecret` field is new; if not configured, a derived default is used.

---

## Recommended operational checks after upgrading to 2026.2.21

1. **Verify Kimi-coding works natively** — the `kimi-coding` implicit provider template is now correct. If you applied the `models.generated.js` patch workaround from MEMORY.md, verify it is not needed after upgrade and test `kimi-coding/k2p5` responds without 403 errors.
2. **Check Telegram streaming behavior** — if you had `streamMode` configured explicitly (e.g. `streamMode: "partial"`), the auto-map should handle it, but verify streaming previews still behave as expected in @MudrionBot after upgrade.
3. **Test cron job concurrency** — if you have multiple cron jobs and rely on serial execution, confirm `cron.maxConcurrentRuns` is explicitly set to `1` in config (it was previously the effective behavior regardless of config).
4. **Verify heartbeat fires if `HEARTBEAT.md` is absent** — v2026.2.19 suppressed heartbeats on missing `HEARTBEAT.md`; v2026.2.21 restores firing behavior when the file is absent. Confirm this is your intended behavior.
5. **Test compaction in long agent sessions** — the compaction safeguard now loads correctly in production builds. After upgrading, run a long agent session and confirm context compaction activates without crashing.
6. **Review exec/shell tool runs for heredoc patterns** — if any cron scripts or agent tools use heredoc syntax, test them after upgrade. Unquoted heredoc body expansion is now blocked; legitimate use requires explicit gateway approval.
7. **Check `openclaw config unset` side effects** — if you previously worked around the key-reappearance bug by not using `config unset`, verify your current config has no unwanted defaulted keys after upgrade.
8. **Run `openclaw update --dry-run`** — use the new dry-run flag to confirm the updater sees the expected version and does not attempt unexpected upgrades.
9. **Check `ownerDisplaySecret`** — if owner-ID obfuscation is in use, a new dedicated `ownerDisplaySecret` HMAC key is now expected. If not explicitly set, a derived default is used. Set explicitly if you require stable obfuscated owner IDs across gateway restarts.
10. **Review QMD memory search if enabled** — if using `memory.backend: qmd`, the serialized embed behavior and per-collection search split may change result ordering. Verify memory recall quality after upgrade.

---

## Docs in this repo updated for this window

- `README.md` — version bump to v2026.2.21, changelog table updated, versioning section updated.
- `CHANGELOG.md` — this synthesized review (consolidated section for v2026.2.19 → v2026.2.21).

---

*Generated: 2026-02-23*

---

# OpenClaw v2026.2.22 — Synthesis Review

**Cycle:** v2026.2.21 → v2026.2.22  
**Released:** 2026-02-23  
**Scope:** Platform expansion, runtime hardening, multilingual memory, and a major security cycle  
**Breaking changes:** 4  
**Security fixes:** 30+  

---

## Executive Summary

v2026.2.22 is the largest single release in recent OpenClaw history. The release adds two new channel plugins (Synology Chat), a new AI provider (Mistral), web search grounding via Gemini, full Control UI cron edit parity, and an optional auto-updater. Simultaneously it ships over 30 security fixes — including critical exec-approval bypasses, SSRF hardening, symlink escape prevention, and auth hardening across nearly every channel. Four breaking changes affect streaming config, device auth, DM scope defaults, and tool-failure reply verbosity.

---

## Breaking Changes

### 1. Google Antigravity Provider Removed
`google-antigravity/*` model and profile configs no longer work. The `google-antigravity-auth` bundled plugin is removed.

**Migration:** Switch to `google-gemini-cli` or another supported Google provider.

```json
// Before
{ "model": "google-antigravity/gemini-2.0-flash" }
// After  
{ "model": "google-gemini-cli/gemini-2.0-flash" }
```

### 2. Tool-Failure Replies Hide Raw Errors by Default
Detailed error suffixes (provider/runtime messages, local path fragments) are now hidden by default.

**Impact:** Agents still send failure summaries; enable `/verbose on` or `/verbose full` to restore raw details.

### 3. `session.dmScope` Defaults to `per-channel-peer`
CLI local onboarding now sets `session.dmScope` to `per-channel-peer` for new/implicit DM scope configurations.

**Migration:** If you depend on shared DM continuity across senders, explicitly set:
```json
{ "session": { "dmScope": "main" } }
```

### 4. Unified Streaming Config + Device Auth v1 Removed
- Channel preview-streaming now uses `channels.<channel>.streaming` with enum values `off | partial | block | progress`
- Slack native stream toggle moved to `channels.slack.nativeStreaming`
- Legacy keys (`streamMode`, Slack boolean `streaming`) still read and migrated by `openclaw doctor --fix`
- **Device auth signature v1 removed.** Clients must sign `v2` payloads with `connect.challenge` nonce; nonce-less connects are rejected.

---

## Major New Features

### New Channel: Synology Chat
Native Synology Chat channel plugin with webhook ingress, direct-message routing, outbound send/media support, per-account config, and DM policy controls.

**Config key:** `channels.synologychat.*`

### New Provider: Mistral
Full Mistral provider support including memory embeddings and voice support.

**Usage:** Set `model: "mistral/<model-id>"` in agent config.

### Grounded Web Search via Gemini
Grounded Gemini provider support for web search with provider auto-detection.

**Config:** Enable via `tools.webSearch.provider: "gemini"` with appropriate API key.

### Full Control UI Cron Edit Parity
Complete web cron management with:
- Clone and rich validation/help text
- All-jobs run history with pagination, search, sort, and multi-filter controls
- Improved cron page layout for scheduling and failure triage

### Memory FTS Multilingual Expansion
Full-text search query expansion now supports:
- **Spanish + Portuguese** stop-word filtering
- **Japanese** tokenization with mixed-script (ASCII + katakana) support  
- **Korean** particle-aware extraction with mixed Korean/English stems
- **Arabic** stop-word filtering

### Optional Auto-Updater
Built-in auto-updater for package installs (`update.auto.*`), default-off.
- Stable rollout delay + jitter
- Beta hourly cadence
- `openclaw update --dry-run` previews actions without mutating config

### Control UI: Tools Panel Data-Driven
Tools panel now driven from runtime `tools.catalog` with per-tool provenance labels (`core` / `plugin:<id>`). Static fallback list when runtime catalog is unavailable.

### Control UI: Version Status Pill
Web header now shows a version status pill before the Health indicator.

---

## Security Fixes (30+)

This release is a major security hardening cycle. Fixes organized by subsystem:

### Exec Approval System (Critical)
- **Safe-bin PATH hijacking:** Stop trusting `PATH`-derived directories for safe-bin allowlist checks; add `tools.exec.safeBinTrustedDirs` and pin to resolved absolute paths.
- **Wrapper-path bypass:** When users choose `allow-always` for shell-wrapper commands, persist allowlist patterns for the inner executable, not the wrapper shell binary.
- **Shell line continuations:** Fail closed on `\\\n`/`\\\r\n` line continuations and treat shell-wrapper execution as approval-required in allowlist mode.
- **`env` wrapper transparency:** Treat `env` and shell-dispatch wrappers as transparent during allowlist analysis so policy checks match the effective executable.
- **Safe-bin profiles:** Require explicit safe-bin profiles for `tools.exec.safeBins` entries; add `tools.exec.safeBinProfiles` for custom binaries.
- **`sort --compress-program` bypass:** Block `sort --compress-program` in allowlist mode.
- **macOS app basename matching:** Enforce path-only allowlist matching; migrate legacy basename entries to resolved paths; harden shell-chain handling.
- **Sandbox fail-closed:** Fail closed when `tools.exec.host=sandbox` is configured but sandbox runtime is unavailable.
- **Shell exec env:** Block request-scoped `HOME`/`ZDOTDIR` overrides; block `SHELLOPTS`/`PS4`; restrict shell-wrapper env to explicit allowlist.
- **Shell startup injection:** Validate login-shell executable paths; block `SHELL`/`HOME`/`ZDOTDIR` in config env ingestion.

### SSRF Hardening
- Expand IPv4 fetch guard to RFC special-use/non-global ranges (benchmarking `198.18.0.0/15`, TEST-NET, multicast, reserved/broadcast).
- Normalize IPv6 dotted-quad transition literals (`::127.0.0.1`, `64:ff9b::8.8.8.8`).
- Enable `autoSelectFamily` on pinned undici dispatchers so IPv6-unreachable environments fall back to IPv4.
- MSTeams: enforce allowlist checks for SharePoint URLs and redirect targets across full redirect chains.

### Symlink Escape Prevention
- Browser uploads: accept in-root paths when uploads directory is a symlink alias.
- Security/Archive: block zip symlink escapes during extraction.
- Media sandbox: enforce symlink-escape checks before sandbox-validated reads.
- Control UI: block symlink-based out-of-root static file reads via realpath containment.
- Gateway avatars: block symlink traversal during local avatar resolution.
- Hooks transforms: enforce symlink-safe containment for webhook transform module paths.

### Auth and Identity
- **Elevated scope bypass:** Match `tools.elevated.allowFrom` against sender identities only (not recipient `ctx.To`).
- **Feishu display-name collision:** Enforce ID-only allowlist matching; ignore mutable display names.
- **Group policy toolsBySender:** Require explicit sender-key types (`id:`, `e164:`, `username:`, `name:`).
- **Discord allowlist:** Canonicalize resolved names to IDs; add audit warnings for name/tag-based entries.
- **Discord: block node-role connections** when device identity metadata is missing.
- **Gateway insecure-config warning:** Emit startup warning when dangerous flags are enabled.
- **Owner display secret:** Auto-generate dedicated `commands.ownerDisplaySecret`; remove gateway token fallback.
- **Prototype pollution:** Block `__proto__`, `constructor`, `prototype` traversal in config merge helpers.
- **Hook auth rate limits:** Normalize IPv4/IPv4-mapped IPv6 addresses to one throttle bucket.

### Session and Credential Security
- **`sessions_history` redaction:** Redact sensitive token patterns from output; surface `contentRedacted` metadata.
- **`openclaw config get` redaction:** Redact sensitive values before printing paths.
- **Voice call WebSocket:** Strict pre-start timeouts; pending/per-IP connection limits; total connection caps for streaming endpoints.
- **Media inbound byte limits:** Enforce limits during download/read across Discord, Telegram, Zalo, MSTeams, BlueBubbles.
- **WhatsApp `allowFrom`:** Enforce for direct-message outbound targets in all send modes.
- **Channels/Security:** Fail closed on missing provider group policy config — runtime defaults to `allowlist` when `channels.<provider>` is absent.

---

## Channels

### Telegram
- WSL2: disable `autoSelectFamily` by default; memoize WSL2 detection.
- DNS: default Node 22+ to `ipv4first` for Telegram fetch paths; add `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER` override.
- Forward bursts: coalesce forwarded text+media through a dedicated forward lane debounce window.
- Streaming: preserve archived draft preview mapping after flush; clean superseded reasoning preview bubbles.
- Replies: scope messaging-tool text/media dedupe to same-target sends only.
- Polling: persist safe update-offset watermark bounded by pending updates; force-restart stuck runner instances.
- Media: send user-facing reply when media download fails (non-size errors).
- Webhook: keep monitors alive until gateway abort signals fire; add `channels.telegram.webhookPort` config.

### Slack
- Threading: keep parent-session forking and thread-history context active beyond first turn.
- Threading: respect `replyToMode` when Slack auto-populates top-level `thread_ts`.
- Extension: forward `message read` `threadId` to `readMessages`; use delivery-context `threadId` as outbound fallback.
- Upload: resolve bare user IDs to DM channel IDs via `conversations.open` before `files.uploadV2`.
- Slash commands: preserve Bolt app receiver when registering external select options handlers.
- Queue routing: preserve string `thread_ts` values through collect-mode queue drain.

### Discord
- Allowlist: canonicalize resolved Discord allowlist names to IDs.
- Allowlist: add security audit warnings for name/tag-based entries.

### BlueBubbles
- DM history: restore DM backfill context with account-scoped rolling history.
- Private API cache: treat unknown (`null`) cache status as disabled.
- Webhooks: accept payloads when BlueBubbles omits `handle` but provides DM `chatGuid`.

### Webchat
- Apply assistant `final` payload messages directly to chat state (no history refresh wait).
- For out-of-band final events, append final assistant payloads directly.
- Preserve external session routing metadata when internal `chat.send` turns run under `webchat`.
- Preserve existing session `label` across `/new` and `/reset` rollovers.

### Browser Extension Relay
- MV3 worker: preserve debugger attachments across relay drops; auto-reconnect with bounded backoff+jitter.
- Persist and rehydrate attached tab state via `chrome.storage.session`.
- Recover from `target_closed` navigation detaches; enforce per-tab operation locks.
- Remote CDP: extend stale-target recovery to reuse the sole available tab.

---

## Agents

### Reasoning
- Avoid classifying provider reasoning-required errors as context overflows.
- OpenRouter: map `/think` levels to `reasoning.effort` in embedded runs.
- OpenRouter: default reasoning to enabled when model advertises `reasoning: true`.

### Compaction
- Count auto-compactions only after non-retry `auto_compaction_end`.
- Restore embedded compaction safeguard/context-pruning extension loading in production builds.
- Strip stale assistant usage snapshots from pre-compaction turns to prevent immediate re-triggering.
- Pass model metadata through embedded runtime for safeguard summarization.
- Cancel safeguard compaction when summary generation cannot run.

### Context Overflow
- Treat HTTP 502/503/504 as failover-eligible transient timeouts.
- Kimi: classify Moonshot `token limit` failures as context overflows for auto-compaction.

### Provider-Specific
- **Moonshot/Kimi:** Force `supportsDeveloperRole=false`; mark K2.5 as image-capable; enable consecutive-user turn merging.
- **OpenRouter:** Inject `cache_control` on system prompts for Anthropic models; preserve `openrouter/` prefix during normalization.
- **Google:** Sanitize non-base64 `thought_signature` values from assistant replay transcripts.
- **Mistral:** Sanitize tool-call IDs; generate strict provider-safe pending tool-call IDs.
- **Ollama:** Preserve unsafe integer tool-call arguments as exact strings during NDJSON parsing.
- **Anthropic:** Default missing `api` fields to `anthropic-messages` during config validation.

### Subagents
- Honor `tools.subagents.tools.alsoAllow` when resolving built-in subagent deny defaults.
- Make announce call timeouts configurable via `agents.defaults.subagents.announceTimeoutMs` (default 60s).

### Workspace/Exec
- Guard `resolveUserPath` against undefined/null input.
- Honor explicit agent context when resolving `tools.exec` defaults.
- Map sandbox container-workdir file-tool paths to host workspace roots before workspace-only validation.

### Transcripts/Tool-calls
- Validate assistant tool-call names before persistence and during replay sanitization.
- Emit default `✅ Done.` acknowledgement only for direct/private tool-only completions.

---

## Cron

### Major Fixes
- **`maxConcurrentRuns`:** Now enforced in the timer loop (was always serializing).
- **Manual run timeout:** Apply same per-job timeout guard for `cron.run` executions.
- **Startup catch-up:** Enforce per-job timeout guards for startup replay runs.
- **Isolated sessions:** Force fresh session IDs for `sessionTarget="isolated"` executions.
- **Schedule timing:** For `every` jobs, prefer `lastRunAtMs + everyMs` when in future.

### Status/Visibility
- Split execution outcome (`lastRunStatus`) from delivery outcome (`lastDeliveryStatus`) in persisted state.
- Persist `delivered` state in cron job records.
- Clean up settled per-path run-log write queue entries.

### Auth/Delivery
- Propagate auth-profile resolution to isolated cron sessions.
- Pass resolved `agentDir` through isolated cron and queued follow-up embedded runs.
- Route text-only announce jobs with explicit thread/topic targets through direct outbound delivery.
- Telegram: validate cron `delivery.to` with shared target parsing; resolve legacy `@username`/`t.me` targets.

---

## Gateway

### Auth Unification
- Unify call/probe/status/auth credential-source precedence on shared resolver helpers.
- Refactor gateway credential resolution and websocket auth handshake paths.
- Add explicit `auth.deviceToken` support in connect frames.

### Pairing
- Treat `operator.admin` as satisfying other `operator.*` scope checks.
- Auto-approve loopback `scope-upgrade` pairing requests.
- Include `operator.read` and `operator.write` in default operator connect scope bundles.
- Treat `operator.admin` pairing tokens as satisfying `operator.write` requests.

### Stability
- Fix restart-loop edge cases: explicit bootstrap detection, lock reacquisition.
- Use optional gateway-port reachability as primary stale-lock liveness signal.
- Retry short-lived missing config snapshots during reload.
- Compare array-valued config paths structurally during diffing.
- Verify gateway health after daemon restart.

---

## Memory

### QMD
- Migrate legacy unscoped collection bindings to per-agent scoped names during startup.
- Normalize Han-script BM25 search queries for mixed CJK+Latin prompts.
- Add optional `memory.qmd.mcporter` search routing through mcporter keep-alive flows.
- On Windows, resolve bare `qmd`/`mcporter` command names to npm shim executables (`.cmd`).

### Embeddings
- Centralize remote memory HTTP calls behind shared guarded helper.
- Apply configured remote-base host pinning across OpenAI/Voyage/Gemini embedding requests.
- Enforce per-input 8k safety cap before embedding batching; 2k fallback for local providers.

### Indexing
- Detect memory source-set changes and trigger full reindex automatically.

---

## Plugins/Extensions

### Install/Discovery
- Strip `workspace:*` devDependency entries from copied plugin manifests before `npm install`.
- Ignore scanned extension backup/disabled directory patterns (`.backup-*`, `.bak`, `.disabled*`).
- Move updater backup directories under `.openclaw-install-backups`.
- Make `openclaw plugins enable` update allowlists via shared plugin-enable policy.
- Auto-enable built-in channels by writing `channels.<id>.enabled=true` (not `plugins.entries.<id>`).
- When `plugins.allow` is active, also allowlist configured built-in channels automatically.

### Feishu
- Restore bundled Feishu SDK availability for global installs.
- Prefer `file_key` over `image_key` when both are present in video messages.
- In group chats, command authorization falls back to top-level `channels.feishu.allowFrom`.

### Hooks
- Run legacy `before_agent_start` once per agent turn; reuse across model-resolve and prompt-build.
- Suppress duplicate main-session events for delivered hook turns.
- Avoid redundant hook-module recompilation on gateway restart using stable file metadata keys.

---

## Logging and Delivery

- Cap single log-file size with `logging.maxFileBytes` (default 500 MB) to prevent disk exhaustion.
- Quarantine queue entries immediately on permanent delivery errors — move to `failed/` instead of retrying.
- Classify undici `TypeError: fetch failed` as transient in unhandled-rejection detection.

---

## TUI

- Enable multiline-paste burst coalescing on macOS Terminal.app and iTerm.
- Isolate right-to-left script lines (Arabic/Hebrew) with Unicode bidi isolation marks.
- Request immediate renders after setting `sending`/`waiting` activity states.
- Arm Ctrl+C exit timing when clearing non-empty composer text; add SIGINT fallback.

---

## iOS

- Prefetch TTS segments and suppress expected speech-cancellation errors for smoother talk playback.

---

## Removed

- Bundled `food-order` skill removed from core — install from ClawHub instead.
- Google Antigravity provider and `google-antigravity-auth` plugin removed.

---

## Config Changes

| Key | Change |
|-----|--------|
| `channels.<channel>.streaming` | Unified streaming config (replaces `streamMode`) |
| `channels.slack.nativeStreaming` | Replaced Slack boolean `streaming` |
| `session.dmScope` | Now defaults to `per-channel-peer` on new installs |
| `update.auto.*` | New: optional auto-updater |
| `agents.defaults.subagents.announceTimeoutMs` | New: configurable announce timeout |
| `tools.exec.safeBinTrustedDirs` | New: explicit trusted directories for safe-bin |
| `tools.exec.safeBinProfiles` | New: profiles for custom safe binaries |
| `memory.qmd.mcporter` | New: optional mcporter search routing |
| `agents.defaults.memorySearch.provider` | Now accepts `"mistral"` |
| `logging.maxFileBytes` | New: log file size cap (default 500 MB) |
| `channels.modelByChannel` | New: per-channel model overrides (allowlisted) |
| `config.bindings[].comment` | New: optional annotation field |

---

## Upgrade Notes

1. **Run `openclaw doctor --fix`** after upgrading — migrates legacy streaming config keys automatically.
2. **Review device auth:** Device-auth signature v1 is removed; all clients must sign `v2` payloads.
3. **Check DM scope:** If your setup relies on shared DM history across multiple senders, set `session.dmScope: "main"` explicitly.
4. **Google Antigravity configs:** Replace with `google-gemini-cli` or another supported Google provider.
5. **Security:** Review `tools.exec.safeBins` — safe-bin path resolution and profiles changed significantly.

---


# OpenClaw v2026.2.23 — Synthesis Review (Released)

**Cycle:** v2026.2.22 → v2026.2.23 (Release Cycle)
**Status:** RELEASED — tag `v2026.2.23` (2026-02-24)
**Scope:** Provider normalization, session hardening, reasoning fixes, security sweep  

---

## Executive Summary

v2026.2.23 shipped on 2026-02-24. The primary feature addition is Vercel AI Gateway Claude shorthand normalization. The bulk of the release is fixes: session key canonicalization, Telegram stability (reactions, polling), agent reasoning improvements, context overflow detection expansion (including Chinese patterns), and a significant security sweep covering exec obfuscation detection, XSS prevention in skill HTML output, OTEL credential redaction, and Python skill packaging hardening.

---

## New Features

### Vercel AI Gateway Claude Shorthand Normalization
Accept Claude shorthand model refs (`vercel-ai-gateway/claude-*`) by normalizing to canonical Anthropic-routed model IDs.

**Before:** Claude shorthand refs would fail or route incorrectly through Vercel AI Gateway.  
**After:** `vercel-ai-gateway/claude-opus-4-6` normalizes automatically.

Thanks @sallyom, @markbooch, and @vincentkoc. (PR #23985)

---

## Session Fixes

### Session Key Canonicalization
Canonicalize inbound mixed-case session keys for metadata and route updates. Migrate legacy case-variant entries to a single lowercase key.

**Impact:** Prevents duplicate sessions and missing TUI/WebUI history for users with legacy case-variant session keys. (PR #9561)

---

## Telegram Fixes

### Reactions: Soft-Fail + snake_case Support
- Soft-fail reaction action errors (policy/token/emoji/API)
- Accept snake_case `message_id` in reaction payloads
- Fallback to inbound message-id context when explicit `messageId` is omitted

**Impact:** DM reactions stay stable without regeneration loops. (PR #20236, #21001)

### Polling: Per-Bot Offset Scoping
Scope persisted polling offsets to bot identity and reuse a single awaited runner-stop path on abort/retry.

**Impact:** Prevents cross-token offset bleed and overlapping pollers during restart/error recovery. (PR #10850, #11347)

---

## Agent Fixes

### Reasoning: Thinking-Block Leak Prevention
When model-default thinking is active (e.g., `thinking=low`), keep auto-reasoning disabled unless explicitly enabled.

**Impact:** Prevents `Reasoning:` thinking-block leakage in channel replies. (PR #24335, #24290)

### Reasoning: Error Classification
Avoid classifying provider reasoning-required errors as context overflows.

**Impact:** These failures no longer trigger compaction-style overflow recovery. (PR #24593)

### Model Config Boundary
Codify `agents.defaults.model` / `agents.defaults.imageModel` config-boundary input as `string | {primary, fallbacks}`.

**Impact:** Fixes `models status --agent` source attribution — defaults-inherited agents labeled as `defaults`. (PR #24210)

### Compaction: Agent Directory Scoping
- Pass `agentDir` into manual `/compact` command runs for correct auth/profile resolution.
- Pass model metadata through embedded runtime so safeguard summarization can run when `ctx.model` is unavailable.
- Cancel safeguard compaction when summary generation cannot run (missing model/API key), preserving history.

(PR #24133, #3479, #10711)

### Context Overflow: Extended Detection
- Detect additional provider context-overflow error shapes (`input length` + `max_tokens` exceed-context variants).
- Add Chinese context-overflow pattern detection in `isContextOverflowError` for localized provider errors.
- Treat HTTP 502/503/504 errors as failover-eligible transient timeouts.
- Groq: avoid classifying Groq TPM limit errors as context overflow.

(PR #9951, #22855, #20999, #16176)

---

## Auto-Reply Fix

### Direct-Chat Metadata Handling
Hide direct-chat `message_id`/`message_id_full` and sender metadata only from normalized chat type (not sender-id sentinels).

**Impact:** Preserves group metadata visibility; prevents sender-id spoofed direct-mode classification. (PR #24373)

---

## Slack Fix

### Group Policy Inheritance
Move Slack account `groupPolicy` defaulting to provider-level schema defaults so multi-account configs inherit top-level `channels.slack.groupPolicy` instead of silently overriding with per-account `allowlist`.

(PR #17579)

---

## Provider Fixes

### Anthropic: Skip context-1m Beta for OAuth Tokens
Skip `context-1m-*` beta injection for OAuth/subscription tokens (`sk-ant-oat-*`).

**Impact:** Avoids Anthropic 401 auth failures when `params.context1m` is enabled with OAuth/subscription tokens. (PR #10647, #20354)

### OpenRouter: Remove Conflicting `reasoning_effort`
Remove conflicting top-level `reasoning_effort` when injecting nested `reasoning.effort`.

**Impact:** Prevents OpenRouter 400 payload-validation failures for reasoning models. (PR #24120)

---

## Gateway Fix

### WebSocket: Unauthorized Request Flood Protection
Close repeated post-handshake `unauthorized role:*` request floods per connection. Sample duplicate rejection logs.

**Impact:** Prevents a single misbehaving client from degrading gateway responsiveness. (PR #20168)

---

## Config Fix

### Config Write Immutability + Path Traversal Hardening
Apply `unsetPaths` with immutable path-copy updates so config writes never mutate caller-provided objects.

**Security:** Harden `openclaw config get/set/unset` path traversal by rejecting prototype-key segments and inherited-property traversal. (PR #24134)

---

## Security Fixes

### Exec: Obfuscated Command Detection
Detect obfuscated commands before exec allowlist decisions and require explicit approval for obfuscation patterns.

**Impact:** Closes a class of allowlist bypass via command obfuscation. (PR #8592)

### Skills/openai-image-gen: Stored XSS Prevention
Escape user-controlled prompt, filename, and output-path values in `openai-image-gen` HTML gallery generation.

**Impact:** Prevents stored XSS in generated `index.html` output when images are served locally. (PR #12538)

### Skills/skill-creator: Symlink Escape Prevention
Harden `skill-creator` packaging by skipping symlink entries and rejecting files whose resolved paths escape the selected skill root.

(PR #24260, #16959)

### OTEL: Credential Redaction
Redact sensitive values (API keys, tokens, credential fields) from diagnostics-otel log bodies, log attributes, and error/reason span fields before OTLP export.

(PR #12542)

### CI: Pre-Commit Security Coverage
Add pre-commit security hook coverage for private-key detection and production dependency auditing. Enforce in CI alongside baseline secret scanning.

---

## Python Skills Hardening

### Packaging Validation
- Skip self-including `.skill` outputs
- Handle CRLF frontmatter parsing
- Strict `--days` validation
- Safer image file loading

### CI + Pre-Commit Linting
- Add `ruff` linting for Python scripts/tests under `skills/`
- Add pytest discovery coverage including package test execution from repo root

---

## Config Changes

| Key | Change |
|-----|--------|
| `agents.defaults.model` | Type formalized as `string \| {primary, fallbacks}` |
| `agents.defaults.imageModel` | Type formalized as `string \| {primary, fallbacks}` |

---

## Upgrade Notes for v2026.2.23

Now that `v2026.2.23` is released, apply these checks after upgrade:

1. **Session keys:** If you have case-variant session keys in legacy configs, they will be automatically migrated to lowercase.
2. **Reasoning with thinking:** Models with `thinking=low` will no longer enable auto-reasoning by default — verify your reasoning-dependent workflows.
3. **Vercel AI Gateway:** Shorthand Claude refs now normalize automatically — no config change needed.
4. **OTEL users:** Review OTLP exports — credential fields are now redacted.

---

# Synthesis Review: v2026.2.22 → v2026.2.23 (Released)

> **Window analyzed:** `v2026.2.22..v2026.2.23` (release window)
> **Status:** RELEASED (2026-02-24)

---

## Executive Summary

This update cycle continues the security-hardening focus with expanded exec approval protections, channel security unification, and automated reasoning/thinking leak prevention. Key themes include:

1. **Exec approval system hardened** — Multi-layer fixes for wrapper bypasses, busybox/toybox applet handling, shell env sanitization, and two-phase approval registration.
2. **Channel security unified** — `dangerouslyAllowNameMatching` policy applied consistently across all channels with shared audit/doctor detection.
3. **Reasoning block leaks eliminated** — Automatic suppression of thinking/reasoning blocks in channel delivery paths, with provider-specific handling.
4. **Webhooks and plugins stabilized** — Synology Chat webhook deregistration, plugin config schema fallback, and Feishu media handling fixes.

---

## Critical Security Fixes

### Exec Approval System

| Fix | Impact |
|-----|--------|
| Bind `host=node` approvals to explicit `nodeId` | Prevents cross-node replay attacks |
| Two-phase approval registration + wait-decision | Eliminates orphaned `/approve` flows and immediate-return races |
| Canonical wrapper execution plans | Prevents `env -S/--split-string` interpretation-mismatch bypasses |
| Busybox/toybox applet recognition | Inner executables persisted instead of wrapper binaries |
| `autoAllowSkills` pathless invocation requirement | `./<skill-bin>`/absolute-path basename collisions blocked |
| Safe-bin long-option validation | Unknown/ambiguous GNU long-option abbreviations rejected |
| Sort filesystem-dependent flags denied | `--random-source`, `--temporary-directory`, `-T` blocked |
| Shell env fallback hardening | Only trust login shells registered in `/etc/shells` |

### Channel & Messaging Security

| Fix | Impact |
|-----|--------|
| WhatsApp JID allowlist on all sends | Tool-initiated sends now validate JID |
| Reasoning/thinking payload suppression | Non-Telegram channels no longer emit internal reasoning blocks |
| Telegram media SSRF policy | RFC2544 benchmark range blocked by default with explicit opt-in |
| `commands.allowFrom` sender-only matching | Blocks conversation-shaped `From` identities |
| Config prototype pollution prevention | Blocks `__proto__`, `constructor`, `prototype` traversal |

---

## Major Changes

### Providers

- **Kilo Gateway** — First-class `kilocode` provider support with auth, onboarding, implicit detection, and default model `kilocode/anthropic/claude-opus-4.6` (#20212)
- **Vercel AI Gateway** — Claude shorthand normalization (`vercel-ai-gateway/claude-*`) (#23985)
- **Moonshot/Kimi** — Native video provider, grounded web search support, `supportsDeveloperRole=false` enforced

### Tools & Capabilities

- **`web_search` provider: "kimi"** — Moonshot search with citation extraction (#16616, #18822)
- **Media understanding/Video** — Native Moonshot video provider with auto key detection (#12063)
- **`agents.defaults.subagents.runTimeoutSeconds`** — Configurable default timeout for `sessions_spawn`

### Sessions & Cron

- **Session maintenance hardening** — `openclaw sessions cleanup`, per-agent store targeting, disk-budget controls (#24753)
- **Isolated cron sessions** — Full prompt mode for skill/extension availability (#24944)
- **Cron webhook deregistration** — Synology Chat stale route cleanup (#24971)

### Gateway & Channels

- **HTTP security headers** — Optional `gateway.http.securityHeaders.strictTransportSecurity` (#24753)
- **Telegram reactions** — Soft-fail errors, snake_case `message_id`, context fallback (#20236, #21001)
- **Telegram polling** — Scoped offsets, single runner-stop path (#10850, #11347)
- **Slack group policy** — Inheritance fix for multi-account configs (#17579)
- **Discord threading** — Parent ID recovery, thread-bound subagents (#24897)
- **Web UI i18n** — Locale hydration on startup (#24795)

---

## Breaking Changes

1. **Control UI origin requirements** — Non-loopback Control UI requires explicit `gateway.controlUi.allowedOrigins`; fails closed without `dangerouslyAllowHostHeaderOriginFallback=true`
2. **Channel `allowFrom` ID-only default** — Mutable name/tag/email matching disabled by default; use `dangerouslyAllowNameMatching=true` for compatibility mode (#24907)

---

## Fixes (Release Selection)

### Agents & Runtime
- Bootstrap file snapshots cached per session key, cleared on reset (#22220)
- Context pruning `cache-ttl` extended to Moonshot/Kimi and ZAI/GLM (#24497)
- Kimi `token limit` failures classified as context overflows (#9562)
- Consecutive-user turn merging for non-OpenAI providers (#7693)
- OpenRouter `cache_control` injection for Anthropic models (#17473)
- Anthropic OAuth tokens skip `context-1m` beta (#10647, #20354)
- Bedrock cache retention disabled for non-Anthropic models (#20866, #22303)
- Groq TPM errors no longer classified as context overflow (#16176)

### Telegram
- Reaction soft-fail for policy/token/emoji/API errors (#20236)
- Polling offset scoped to bot identity (#10850, #11347)
- Reasoning suppression when `/reasoning off` active (#24626, #24518)

### WhatsApp
- Group policy `groupAllowFrom` fix when no explicit `groups` (#24670)
- DM routing only updates main-session when bound (#24949)
- `selfChatMode` honored in access control (#24738)
- Outbound recipient identifier redaction (#24980)

### Webchat/UI
- Locale hydration during startup (#24795)
- Assistant `final` payloads applied directly (#14928, #11139)
- External session routing metadata preserved (#23258)
- Session `label` preserved across resets (#23755)

### Gateway & Infra
- Browser control loads `server.js` reliably (#23974)
- Chrome relay debugger detach handling (#19766)
- Relay port derivation clarified for custom gateway ports (#22252)
- Pairing recovery shows explicit approval hints (#24771)
- OAuth scope failures classified as auth failures (#24761)

---

## Recommended Operational Checks (Post-Release)

1. **Review `allowFrom` configurations** — If relying on mutable name matching, migrate to stable IDs or set `dangerouslyAllowNameMatching=true`
2. **Control UI origin settings** — Non-loopback deployments need explicit `gateway.controlUi.allowedOrigins`
3. **Test reasoning suppression** — Verify thinking-capable models don't leak reasoning blocks in channel replies
4. **Review exec allowlists** — Busybox/toybox and wrapper command patterns may need re-approval
5. **Kimi provider testing** — Verify `kimi-coding/k2p5` works without 403 errors

---

*Generated: 2026-02-24*
