# OpenClaw Codebase Analysis — PART 4: CLI, TOOLS & MEDIA
<!-- markdownlint-disable MD024 -->

> Updated: 2026-03-12 | Version: v2026.3.11

## Overview

| Module | Files | Lines | Purpose |
|--------|-------|-------|---------|
| src/cli/ | 261 | 38,455 | CLI commands, argument parsing, program construction |
| src/commands/ | 322 | 59,211 | Command implementations (agent, doctor, onboard, etc.) |
| src/tui/ | 45 | 7,589 | Terminal UI, interactive chat mode |
| src/browser/ | 125 | 20,579 | Browser control, CDP, Playwright automation |
| src/media/ | 34 | 3,962 | Media handling, MIME detection, storage |
| src/media-understanding/ | 51 | 6,336 | Attachment processing (image/audio/video understanding) |
| src/link-understanding/ | 6 | 333 | URL/link extraction and processing |
| src/tts/ | 4 | 2,240 | Text-to-speech (Edge, OpenAI, ElevenLabs) |
| src/markdown/ | 14 | 2,725 | Markdown → IR → platform-specific rendering |
| src/canvas-host/ | 6 | 1,395 | Canvas/A2UI system for node displays |

---

## 1. src/cli/ — CLI Commands & Argument Parsing

### Purpose
Entry point for the `openclaw` CLI binary. Handles argument parsing, command routing, program construction via Commander.js, and all subcommand registrations.

### Key Files

| File | Role |
|------|------|
| `run-main.ts` | Main entry: `runCli()` — normalizes argv, loads dotenv, routes commands |
| `program.ts` | Re-exports `buildProgram` and `forceFreePort` |
| `program/build-program.ts` | Creates Commander `Program`, registers all commands via `registerProgramCommands` |
| `program/command-registry.ts` | Central registry mapping command names to their registration functions |
| `program/context.ts` | Creates `ProgramContext` (version, config, runtime) |
| `program/preaction.ts` | Pre-action hooks (config validation, auth checks) |
| `program/config-guard.ts` | Ensures config is ready before command execution |
| `program/routes.ts` | Fast-path routing for known commands (avoids full Commander parse) |
| `program/help.ts` | Custom help formatting |
| `program/program-context.ts` | Get/set program context on Commander instance |
| `route.ts` | `tryRouteCli()` — fast command routing before full program parse |
| `argv.ts` | Argv parsing utilities: `getCommandPath`, `getPrimaryCommand`, `hasHelpOrVersion` |
| `banner.ts` | CLI startup banner |
| `cli-name.ts` | CLI name constant |
| `cli-utils.ts` | Shared CLI utilities |
| `command-format.ts` | `formatCliCommand()` — formats command strings for display |
| `command-options.ts` | Shared Commander option definitions |
| `deps.ts` | `CliDeps` type and `createDefaultDeps()` factory |
| `browser-cli.ts` | `openclaw browser` subcommand root |
| `browser-cli-*.ts` (8 files) | Browser CLI sub-subcommands (actions, debug, extension, inspect, manage, state) |
| `browser-cli-actions-input/` | Browser action input registration (element, files, form, navigation) |
| `daemon-cli.ts` | `openclaw gateway` (daemon) subcommand |
| `daemon-cli/` (12 files) | Daemon lifecycle, install, status, probe, runners |
| `gateway-cli.ts` | Gateway CLI commands |
| `gateway-cli/` (7 files) | Gateway run, dev, discover, call |
| `nodes-cli.ts` | `openclaw nodes` subcommand |
| `nodes-cli/` (12 files) | Node pairing, camera, canvas, location, notify, screen, invoke |
| `cron-cli.ts` | `openclaw cron` subcommand |
| `cron-cli/` (5 files) | Cron add, edit, simple operations |
| `update-cli.ts` | `openclaw update` subcommand |
| `update-cli/` (5 files) | Update wizard, progress, status |
| `program/message/` (10 files) | Message subcommand registrations (send, broadcast, poll, reactions, etc.) |
| `program/register.*.ts` (7 files) | Top-level command registrations (agent, configure, maintenance, setup, etc.) |
| `channels-cli.ts` | Channel management CLI |
| `channel-auth.ts` | Channel authentication |
| `channel-options.ts` | Channel option definitions |
| `config-cli.ts` | `openclaw config` subcommand |
| `completion-cli.ts` | Shell completion generation |
| `devices-cli.ts` | Device management |
| `directory-cli.ts` | Directory/contacts CLI |
| `dns-cli.ts` | DNS management |
| `docs-cli.ts` | Documentation viewer |
| `exec-approvals-cli.ts` | Exec approval management |
| `hooks-cli.ts` | Hooks management |
| `logs-cli.ts` | Log viewer |
| `memory-cli.ts` | Memory management |
| `models-cli.ts` | Model management CLI |
| `node-cli.ts` | Node daemon CLI |
| `pairing-cli.ts` | Device pairing |
| `plugins-cli.ts` | Plugin management |
| `profile.ts` / `profile-utils.ts` | Auth profile utilities |
| `sandbox-cli.ts` | Sandbox management |
| `security-cli.ts` | Security settings |
| `skills-cli.ts` | Skills management |
| `system-cli.ts` | System info |
| `tui-cli.ts` | TUI launcher |
| `webhooks-cli.ts` | Webhook management |
| `plugin-registry.ts` | Plugin command registry |
| `gateway-rpc.ts` | Gateway RPC calls |
| `ports.ts` | `forceFreePort()` — port management |
| `progress.ts` | CLI progress indicators |
| `prompt.ts` | Interactive prompts |
| `respawn-policy.ts` | Process respawn policy |
| `tagline.ts` | Random taglines |
| `wait.ts` | Wait utilities |
| `parse-bytes.ts` / `parse-duration.ts` / `parse-timeout.ts` | Value parsers |
| `windows-argv.ts` | Windows argv normalization |
| `help-format.ts` | Help text formatting |
| `acp-cli.ts` | ACP (Agent Communication Protocol) CLI |
| `outbound-send-deps.ts` | Outbound message sending dependencies |

### Key Exports
- `runCli(argv)` — main CLI entry point
- `buildProgram()` — constructs the Commander program
- `forceFreePort(port)` — ensures a port is available

### Dependencies
- `commander` (Commander.js)
- `@clack/prompts` (interactive prompts)
- `src/config/` — config loading
- `src/infra/` — env, dotenv, runtime guard
- `src/agents/` — agent scope, model selection
- `src/runtime.ts` — runtime environment
- `src/commands/` — actual command implementations

### Dependents
- Package `bin` entry point
- `src/commands/` reference CLI utilities

---

## 2. src/commands/ — Command Implementations

### Purpose
The actual business logic for all CLI commands: `agent`, `doctor`, `onboard`, `configure`, `status`, `models`, `channels`, `sandbox`, `sessions`, `health`, `reset`, etc.

### Key Files (grouped by feature)

#### Agent
| File | Role |
|------|------|
| `agent.ts` | `agentCommand()` — runs an agent with a message (CLI mode) |
| `agent-via-gateway.ts` | Sends agent requests through gateway |
| `agent/delivery.ts` | Delivers agent command results |
| `agent/run-context.ts` | Resolves agent run context |
| `agent/session-store.ts` | Session store updates after agent run |
| `agent/session.ts` | Session resolution |
| `agent/types.ts` | `AgentCommandOpts` type |
| `agents.ts` | `openclaw agents` command (list, add, delete agents) |
| `agents.bindings.ts` | Agent bindings |
| `agents.command-shared.ts` | Shared agent command utilities |
| `agents.commands.add/delete/identity/list.ts` | CRUD for agents |
| `agents.config.ts` | Agent configuration |
| `agents.providers.ts` | Agent provider management |

#### Doctor
| File | Role |
|------|------|
| `doctor.ts` | `doctorCommand()` — interactive health check/repair wizard |
| `doctor-auth.ts` | Auth profile health checks |
| `doctor-completion.ts` | Shell completion checks |
| `doctor-config-flow.ts` | Config migration flow |
| `doctor-format.ts` | Doctor output formatting |
| `doctor-gateway-*.ts` (3 files) | Gateway health, daemon, services checks |
| `doctor-install.ts` | Install verification |
| `doctor-memory-search.ts` | Memory search health |
| `doctor-platform-notes.ts` | Platform-specific notes |
| `doctor-prompter.ts` | Interactive prompter for doctor |
| `doctor-sandbox.ts` | Sandbox health |
| `doctor-security.ts` | Security warnings |
| `doctor-state-integrity.ts` | State integrity checks |
| `doctor-state-migrations.ts` | Legacy state migrations |
| `doctor-ui.ts` | UI protocol checks |
| `doctor-update.ts` | Update checks |
| `doctor-workspace*.ts` | Workspace health |

#### Onboarding
| File | Role |
|------|------|
| `onboard.ts` | Main onboarding wizard |
| `onboard-auth.ts` | Auth onboarding |
| `onboard-channels.ts` | Channel setup |
| `onboard-config.ts` | Config creation |
| `onboard-custom.ts` | Custom setup |
| `onboard-helpers.ts` | Wizard helpers |
| `onboard-hooks.ts` | Hook setup |
| `onboard-interactive.ts` | Interactive onboarding |
| `onboard-non-interactive.ts` + subdir | Non-interactive (scripted) onboarding |
| `onboard-provider-auth-flags.ts` | Provider auth flags |
| `onboard-remote.ts` | Remote gateway onboarding |
| `onboard-skills.ts` | Skills setup |
| `onboard-types.ts` | Onboarding types |
| `onboarding/` (3 files) | Plugin install, registry, types |

#### Auth
| File | Role |
|------|------|
| `auth-choice.ts` | Main auth choice flow |
| `auth-choice.apply.ts` + provider files | Apply auth for each provider (Anthropic, OpenAI, Google, xAI, etc.) |
| `auth-choice-legacy.ts` | Legacy auth migration |
| `auth-choice-options.ts` | Auth choice options |
| `auth-choice-prompt.ts` | Auth choice prompting |
| `auth-choice.default-model.ts` | Default model selection |
| `auth-choice.model-check.ts` | Model verification |
| `auth-choice.preferred-provider.ts` | Provider preference |
| `auth-token.ts` | Token management |

#### Status/Health
| File | Role |
|------|------|
| `status.ts` | Status types and shared logic |
| `status.command.ts` | `openclaw status` command |
| `status-all.ts` + subdir | Comprehensive status report |
| `status.agent-local.ts` | Local agent status |
| `status.daemon.ts` | Daemon status |
| `status.format.ts` | Status formatting |
| `status.gateway-probe.ts` | Gateway health probe |
| `status.scan.ts` | Status scanning |
| `status.summary.ts` | Status summary |
| `health.ts` | `openclaw health` command |
| `health-format.ts` | Health output formatting |

#### Models
| File | Role |
|------|------|
| `models.ts` | `openclaw models` root |
| `models/list.ts` + related | Model listing, probing, formatting |
| `models/set.ts` | Set default model |
| `models/set-image.ts` | Set image model |
| `models/scan.ts` | Auto-scan available models |
| `models/auth.ts` | Model auth management |
| `models/aliases.ts` | Model aliases |
| `models/fallbacks.ts` | Fallback model configuration |
| `models/shared.ts` | Shared model utilities |

#### Other Commands
| File | Role |
|------|------|
| `configure.ts` + related | `openclaw configure` wizard |
| `channels.ts` + subdir | Channel management |
| `sandbox.ts` + related | Sandbox management |
| `sessions.ts` | Session management |
| `message.ts` | `openclaw message` command |
| `src/commands/docs.ts` | Documentation |
| `dashboard.ts` | Dashboard |
| `reset.ts` | Factory reset |
| `setup.ts` | Initial setup |
| `uninstall.ts` | Uninstall |
| `daemon-runtime.ts` | Daemon runtime |
| `daemon-install-helpers.ts` | Daemon install helpers |
| `gateway-presence.ts` | Gateway presence |
| `gateway-status.ts` | Gateway status |
| `oauth-flow.ts` / `oauth-env.ts` | OAuth flows |
| `signal-install.ts` | Signal messenger install |
| `model-picker.ts` | Interactive model picker |
| `model-allowlist.ts` | Model allowlist |
| Various `*-model-default.ts` | Provider-specific model defaults |

### Dependencies
- `src/cli/` — CLI utilities, formatting
- `src/config/` — config loading/writing
- `src/agents/` — agent scope, model selection, auth
- `src/gateway/` — gateway communication
- `src/daemon/` — daemon service
- `src/channels/` — channel plugins
- `src/infra/` — infrastructure utilities
- `@clack/prompts` — interactive prompts

### Dependents
- `src/cli/program/` — registers these as commands
- `src/cli/route.ts` — fast-paths to some commands

---

## 3. src/tui/ — Terminal UI

### Purpose
Interactive terminal chat interface using `@mariozechner/pi-tui`. Provides a full-screen TUI with chat log, editor, command handling, and gateway websocket communication.

### Key Files

| File | Role |
|------|------|
| `tui.ts` | Main TUI setup: `createEditorSubmitHandler()`, `shouldEnableWindowsGitBashPasteFallback()` |
| `tui-types.ts` | Types: `TuiOptions`, `SessionInfo`, `AgentSummary`, `TuiStateAccess`, `SessionScope` |
| `tui-command-handlers.ts` | Slash command handlers (`/model`, `/agent`, `/session`, etc.) |
| `tui-event-handlers.ts` | Event handlers for TUI events |
| `tui-formatters.ts` | Text/token formatting, `resolveFinalAssistantText()` |
| `tui-local-shell.ts` | `!` bang-line local shell execution |
| `tui-overlays.ts` | Overlay/modal handlers |
| `tui-session-actions.ts` | Session management actions |
| `tui-status-summary.ts` | Status bar summary |
| `tui-stream-assembler.ts` | Assembles streaming responses |
| `tui-waiting.ts` | Waiting status messages |
| `commands.ts` | `getSlashCommands()` — slash command definitions |
| `gateway-chat.ts` | `GatewayChatClient` — WebSocket client for gateway communication |
| `components/chat-log.ts` | Chat log component |
| `components/custom-editor.ts` | Custom text editor component |
| `components/assistant-message.ts` | Assistant message rendering |
| `components/user-message.ts` | User message rendering |
| `components/tool-execution.ts` | Tool execution display |
| `components/filterable-select-list.ts` | Filterable selection list |
| `components/searchable-select-list.ts` | Searchable selection list |
| `components/fuzzy-filter.ts` | Fuzzy text filtering |
| `components/selectors.ts` | UI selectors |
| `theme/theme.ts` | TUI theme (colors, styles) |
| `theme/syntax-theme.ts` | Syntax highlighting theme |

### Key Exports
- `createEditorSubmitHandler()` — creates the submit handler for the editor
- `resolveFinalAssistantText()` — extracts final assistant text from stream
- `TuiOptions` type

### Dependencies
- `@mariozechner/pi-tui` — TUI framework (Container, Loader, ProcessTerminal, Text, TUI)
- `src/agents/` — agent scope, model selection
- `src/config/` — config loading
- `src/routing/` — session key parsing
- `src/gateway/` — gateway communication (via `GatewayChatClient`)

### Dependents
- `src/cli/tui-cli.ts` — launches the TUI

---

## 4. src/browser/ — Browser Control & Automation

### Purpose
Full browser automation system. Runs an Express HTTP server for browser control, manages Chrome/Chromium profiles via CDP and Playwright, supports Chrome extension relay for attaching to existing tabs, and provides agent-facing snapshot/action APIs.

### Key Files

#### Server & Core
| File | Role |
|------|------|
| `server.ts` | `startBrowserControlServerFromConfig()` — Express server for browser control |
| `server-context.ts` | `BrowserServerState`, `createBrowserRouteContext()` |
| `server-context.types.ts` | Server context type definitions |
| `server-middleware.ts` | Auth & common middleware |
| `config.ts` | `resolveBrowserConfig()`, `resolveProfile()` |
| `constants.ts` | Browser constants |
| `paths.ts` | Browser-related file paths |

#### Chrome Management
| File | Role |
|------|------|
| `chrome.ts` | Chrome process management (launch, kill) |
| `chrome.executables.ts` | Chrome executable detection across platforms |
| `chrome.profile-decoration.ts` | Profile decoration/identification |
| `profiles.ts` | Profile type definitions |
| `profiles-service.ts` | Profile lifecycle management |

#### CDP (Chrome DevTools Protocol)
| File | Role |
|------|------|
| `cdp.ts` | CDP connection management |
| `cdp.helpers.ts` | CDP helper utilities |
| `target-id.ts` | Tab/target ID management |

#### Playwright Integration
| File | Role |
|------|------|
| `pw-session.ts` | Playwright browser session management |
| `pw-ai.ts` | Playwright AI module (snapshot → action) |
| `pw-ai-module.ts` | AI module loader |
| `pw-ai-state.ts` | AI module state tracking |
| `pw-role-snapshot.ts` | Role-based accessibility snapshot |
| `pw-tools-core.ts` | Core Playwright tools orchestration |
| `pw-tools-core.activity.ts` | Activity/navigation tracking |
| `pw-tools-core.downloads.ts` | Download management |
| `pw-tools-core.interactions.ts` | Click, type, hover, drag interactions |
| `pw-tools-core.responses.ts` | Response handling |
| `pw-tools-core.shared.ts` | Shared Playwright utilities |
| `pw-tools-core.snapshot.ts` | Page snapshot capture |
| `pw-tools-core.state.ts` | Browser state management |
| `pw-tools-core.storage.ts` | Cookie/storage management |
| `pw-tools-core.test-harness.ts` | Test harness integration |
| `pw-tools-core.trace.ts` | Tracing support |

#### Client
| File | Role |
|------|------|
| `client.ts` | Browser client types (`BrowserStatus`, `BrowserTab`, `SnapshotResult`) |
| `client-fetch.ts` | `fetchBrowserJson()` — HTTP client for browser server |
| `client-actions.ts` | Client-side action dispatching |
| `client-actions-core.ts` | Core action implementations |
| `client-actions-observe.ts` | Observation actions |
| `client-actions-state.ts` | State query actions |
| `client-actions-types.ts` | Action type definitions |

#### Routes
| File | Role |
|------|------|
| `routes/index.ts` | `registerBrowserRoutes()` — route registration |
| `routes/dispatcher.ts` | Request dispatching |
| `routes/basic.ts` | Basic routes (status, start, stop) |
| `routes/tabs.ts` | Tab management routes |
| `routes/agent.ts` | Agent-facing routes |
| `routes/agent.act.ts` | Act (perform action) route |
| `routes/agent.act.shared.ts` | Shared act utilities |
| `routes/agent.snapshot.ts` | Snapshot route |
| `routes/agent.storage.ts` | Storage routes |
| `routes/agent.debug.ts` | Debug routes |
| `routes/agent.shared.ts` | Shared agent route utilities |
| `routes/path-output.ts` | Path-based output |
| `routes/types.ts` | Route types |
| `routes/utils.ts` | Route utilities |

#### Auth & Security
| File | Role |
|------|------|
| `control-auth.ts` | Browser control authentication |
| `control-service.ts` | Control service management |
| `http-auth.ts` | HTTP auth middleware |
| `csrf.ts` | CSRF protection |
| `bridge-auth-registry.ts` | Bridge auth token registry |

#### Extension & Relay
| File | Role |
|------|------|
| `extension-relay.ts` | Chrome extension WebSocket relay server |
| `bridge-server.ts` | Bridge server for extension communication |

#### Other
| File | Role |
|------|------|
| `screenshot.ts` | Screenshot capture |
| `proxy-files.ts` | File proxy for browser access |
| `trash.ts` | Profile cleanup |
| `resolved-config-refresh.ts` | Config refresh handling |

### Key Exports
- `startBrowserControlServerFromConfig()` — starts browser control server
- Client types: `BrowserStatus`, `BrowserTab`, `SnapshotResult`
- `fetchBrowserJson()` — HTTP client for browser API

### Dependencies
- `express` — HTTP server
- `playwright` / `playwright-core` — browser automation
- `ws` — WebSocket (extension relay)
- `src/config/` — config loading
- `src/media/mime.ts` — MIME detection
- `src/logging/` — subsystem logging

### Dependents
- `src/cli/browser-cli*.ts` — CLI browser commands
- `src/agents/tools/` — agent browser tool
- Gateway daemon — starts browser server

---

## 5. src/media/ — Media Handling

### Purpose
Low-level media utilities: MIME detection, file storage with TTL, base64 encoding, image operations, audio metadata, HTTP media serving, and SSRF-guarded fetching.

### Key Files

| File | Role |
|------|------|
| `constants.ts` | `MediaKind` type, `mediaKindFromMime()`, `maxBytesForKind()`, size limits (6MB image, 16MB audio/video, 100MB document) |
| `mime.ts` | `detectMime()` (from buffer via `file-type`), `extensionForMime()`, MIME↔extension maps |
| `store.ts` | `saveMediaSource()` — downloads URL to `~/.openclaw/media/` with TTL, SSRF guard, 5MB limit |
| `fetch.ts` | `fetchMedia()` — SSRF-guarded HTTP fetch with size limits, `MediaFetchError` |
| `host.ts` | `ensureMediaHosted()` — starts media server, returns hosted URL |
| `server.ts` | `attachMediaRoutes()` — Express routes for serving stored media |
| `base64.ts` | Base64 encode/decode utilities |
| `sniff-mime-from-base64.ts` | Detect MIME from base64 data |
| `parse.ts` | Media URL/path parsing |
| `image-ops.ts` | Image resize/convert operations |
| `audio.ts` | Audio format detection, voice compatibility check |
| `audio-tags.ts` | Audio metadata/tag reading |
| `input-files.ts` | Input file extraction (PDF text, document content) |
| `outbound-attachment.ts` | Outbound attachment handling |
| `png-encode.ts` | PNG encoding |
| `load-options.ts` | `OutboundMediaLoadParams`, `resolveOutboundMediaLocalRoots()` — outbound media load options and local root resolution |
| `read-response-with-limit.ts` | Read HTTP response with byte limit |

### Key Exports
- `MediaKind`, `mediaKindFromMime()`, `maxBytesForKind()`
- `detectMime()`, `extensionForMime()`
- `saveMediaSource()`, `ensureMediaHosted()`
- `fetchMedia()`, `MediaFetchError`
- `attachMediaRoutes()`

### Dependencies
- `file-type` — magic byte MIME detection
- `src/infra/net/ssrf.ts` — SSRF hostname pinning
- `src/infra/net/fetch-guard.ts` — SSRF-guarded fetch
- `src/infra/fs-safe.ts` — safe file access

### Dependents
- `src/media-understanding/` — attachment processing
- `src/browser/` — MIME detection
- `src/canvas-host/` — MIME detection
- `src/tts/` — audio compatibility checks
- `src/channels/` — media sending
- `src/auto-reply/` — media in replies

---

## 6. src/media-understanding/ — Attachment Processing

### Purpose
Processes inbound media attachments (images, audio, video) through AI providers for transcription/description. Supports multiple providers: Anthropic, Google, OpenAI, Groq, Deepgram, MiniMax, xAI. Applies scope rules to decide whether to process.

### Key Files

| File | Role |
|------|------|
| `index.ts` | Barrel exports: `applyMediaUnderstanding`, `formatMediaUnderstandingBody`, `resolveMediaUnderstandingScope` |
| `types.ts` | Core types: `MediaAttachment`, `MediaUnderstandingOutput`, `MediaUnderstandingProvider`, `MediaUnderstandingKind` |
| `runner.ts` | Main runner: `runCapability()`, `buildProviderRegistry()`, provider/CLI entry execution |
| `runner.entries.ts` | `runProviderEntry()`, `runCliEntry()`, `buildModelDecision()` |
| `apply.ts` | `applyMediaUnderstanding()` — orchestrates full media understanding pipeline |
| `resolve.ts` | Resolves model entries, scope decisions, timeouts |
| `scope.ts` | `resolveMediaUnderstandingScope()` — allow/deny based on channel, chat type, session |
| `attachments.ts` | `normalizeAttachments()`, `selectAttachments()`, `MediaAttachmentCache` |
| `concurrency.ts` | `runWithConcurrency()` — concurrent processing with limits |
| `format.ts` | `formatMediaUnderstandingBody()` — formats outputs for system prompt injection |
| `output-extract.ts` | Extracts responses from provider outputs |
| `defaults.ts` | Default models, providers, timeouts |
| `errors.ts` | `isMediaUnderstandingSkipError()` |
| `audio-preflight.ts` | Audio pre-processing checks |
| `video.ts` | Video processing utilities |

#### Providers
| File | Role |
|------|------|
| `providers/index.ts` | Provider registry, `buildMediaUnderstandingRegistry()` |
| `providers/shared.ts` | Shared provider utilities |
| `providers/image.ts` | Generic image description provider |
| `providers/anthropic/index.ts` | Anthropic vision provider |
| `providers/google/index.ts` | Google (Gemini) provider |
| `providers/google/audio.ts` | Google audio transcription |
| `providers/google/video.ts` | Google video description |
| `providers/google/inline-data.ts` | Google inline data handling |
| `providers/openai/index.ts` | OpenAI provider |
| `providers/openai/audio.ts` | OpenAI Whisper transcription |
| `providers/deepgram/index.ts` | Deepgram provider |
| `providers/deepgram/audio.ts` | Deepgram audio transcription |
| `providers/groq/index.ts` | Groq (Whisper) provider |
| `providers/minimax/index.ts` | MiniMax provider |
| `providers/zai/index.ts` | xAI provider |

### Data Flow
1. Inbound message with attachments arrives
2. `resolveMediaUnderstandingScope()` checks if processing is allowed
3. `normalizeAttachments()` prepares attachment metadata
4. `runCapability()` dispatches to appropriate provider(s) based on media type
5. Provider calls AI API (transcription/description)
6. `formatMediaUnderstandingBody()` injects results into system prompt context

### Dependencies
- `src/agents/model-auth.ts` — API key resolution
- `src/agents/model-catalog.ts` — model capability checks
- `src/media/input-files.ts` — file content extraction
- `src/process/exec.ts` — CLI tool execution
- `src/config/` — config types

### Dependents
- `src/auto-reply/` — applies media understanding before generating replies

---

## 7. src/link-understanding/ — URL/Link Processing

### Purpose
Extracts URLs from messages and processes them through configured tools (CLI commands or future provider-based) to fetch/summarize link content.

### Key Files

| File | Role |
|------|------|
| `index.ts` | Barrel: `applyLinkUnderstanding`, `extractLinksFromMessage`, `formatLinkUnderstandingBody`, `runLinkUnderstanding` |
| `detect.ts` | `extractLinksFromMessage()` — URL extraction from text |
| `runner.ts` | `runLinkUnderstanding()` — runs configured CLI commands per URL |
| `apply.ts` | `applyLinkUnderstanding()` — orchestrates link processing pipeline |
| `format.ts` | `formatLinkUnderstandingBody()` — formats results for context injection |
| `defaults.ts` | `DEFAULT_LINK_TIMEOUT_SECONDS` |

### Data Flow
1. `extractLinksFromMessage()` finds URLs in message text
2. `runLinkUnderstanding()` runs CLI tools (e.g., `web_fetch`) per URL with templating
3. `formatLinkUnderstandingBody()` formats output for system prompt
4. `applyLinkUnderstanding()` orchestrates the full flow

### Dependencies
- `src/media-understanding/scope.ts` — reuses scope resolution
- `src/media-understanding/resolve.ts` — reuses timeout resolution
- `src/process/exec.ts` — CLI execution
- `src/auto-reply/templating.ts` — template variable substitution

### Dependents
- `src/auto-reply/` — applies link understanding before generating replies

---

## 8. src/tts/ — Text-to-Speech

### Purpose
Multi-provider TTS system supporting Edge TTS (free, default), OpenAI TTS, and ElevenLabs. Handles auto-mode (off/always/inbound/tagged), text summarization for long content, per-user preferences, directive parsing (`[[tts:...]]`), and Telegram voice note optimization.

### Key Files

| File | Role |
|------|------|
| `tts.ts` | **Main module** (~950 lines): `resolveTtsConfig()`, `textToSpeech()`, `textToSpeechTelephony()`, `maybeApplyTtsToPayload()`, user prefs, provider fallback chain, directive parsing |
| `tts-core.ts` | Core implementations: `edgeTTS()`, `openaiTTS()`, `elevenLabsTTS()`, `summarizeText()`, `parseTtsDirectives()`, `scheduleCleanup()` |

### Key Types & Exports
- `ResolvedTtsConfig` — full resolved TTS configuration
- `TtsResult` / `TtsTelephonyResult` — TTS operation results
- `TtsDirectiveOverrides` — parsed `[[tts:...]]` directive overrides
- `textToSpeech(params)` — main TTS function with provider fallback
- `maybeApplyTtsToPayload(params)` — auto-applies TTS to reply payloads
- `buildTtsSystemPromptHint(cfg)` — generates system prompt hint about TTS capability
- `resolveTtsConfig(cfg)` — resolves full TTS config from OpenClawConfig
- Provider preference: tries configured provider first, then falls back through openai → elevenlabs → edge

### Provider Details
- **Edge TTS**: Free, uses `node-edge-tts`, multiple output formats, subtitle support
- **OpenAI TTS**: Models `gpt-4o-mini-tts`, voices `alloy`/etc, opus/mp3 output
- **ElevenLabs**: Voice IDs, model IDs, voice settings (stability, similarity, style, speed)
- **Telegram**: Auto-selects opus format for voice notes
- **Telephony**: PCM output for phone calls

### Dependencies
- `node-edge-tts` — Edge TTS
- `@mariozechner/pi-ai` — summarization via `completeSimple`
- `src/agents/model-auth.ts` — API key resolution
- `src/agents/model-selection.ts` — model resolution for summarization
- `src/media/audio.ts` — voice compatibility check
- `src/config/` — TTS config types
- `src/line/markdown-to-line.ts` — `stripMarkdown()` for TTS text

### Dependents
- `src/auto-reply/` — `maybeApplyTtsToPayload()` in reply pipeline
- `src/agents/tools/tts-tool.ts` — agent TTS tool
- `src/cli/` — TTS CLI commands

---

## 9. src/markdown/ — Markdown Processing & IR

### Purpose
Converts Markdown to an intermediate representation (IR) with style spans and link spans, then renders to platform-specific formats (Telegram, WhatsApp, etc.). Handles tables, code blocks, frontmatter, spoilers.

### Key Files

| File | Role |
|------|------|
| `ir.ts` | **Core** (~950 lines): `markdownToIR()`, `markdownToIRWithMeta()`, `chunkMarkdownIR()` — parses Markdown via `markdown-it` into `MarkdownIR` (text + style spans + link spans) |
| `render.ts` | `renderMarkdownWithMarkers()` — renders IR back to text with configurable style markers |
| `frontmatter.ts` | `parseFrontmatterBlock()` — YAML/line-based frontmatter parsing |
| `whatsapp.ts` | `markdownToWhatsApp()` — converts standard Markdown to WhatsApp formatting (`**bold**` → `*bold*`, `~~strike~~` → `~strike~`) |
| `tables.ts` | `convertMarkdownTables()` — converts Markdown tables to bullets or code format |
| `fences.ts` | `parseFenceSpans()` — detects fenced code block ranges |
| `code-spans.ts` | `buildCodeSpanIndex()` — detects inline code and fence spans for "is inside code" queries |

### Core Types
```typescript
type MarkdownIR = {
  text: string;                    // Plain text content
  styles: MarkdownStyleSpan[];     // Style annotations (bold, italic, code, etc.)
  links: MarkdownLinkSpan[];       // Link annotations
};

type MarkdownStyle = "bold" | "italic" | "strikethrough" | "code" | "code_block" | "spoiler" | "blockquote";
```

### Data Flow
1. Markdown string → `markdown-it` tokenizer
2. Tokens → recursive `renderTokens()` → builds plain text + span arrays
3. Tables rendered as bullets or code blocks (configurable)
4. IR can be chunked via `chunkMarkdownIR()` (preserves span offsets)
5. IR rendered to platform format via `renderMarkdownWithMarkers()` with custom markers

### Dependencies
- `markdown-it` — Markdown parsing
- `yaml` — frontmatter parsing
- `src/auto-reply/chunk.ts` — text chunking
- `src/config/types.base.ts` — `MarkdownTableMode` type

### Dependents
- `src/channels/plugins/outbound/telegram.ts` — Telegram message formatting
- `src/channels/plugins/outbound/whatsapp.ts` — WhatsApp formatting
- `src/channels/plugins/outbound/discord.ts` — Discord formatting
- `src/auto-reply/` — message formatting
- `src/line/` — line-based text conversion

---

## 10. src/canvas-host/ — Canvas/A2UI System

### Purpose
Hosts a local web server for the Canvas/A2UI (Agent-to-UI) system. Serves static files, handles WebSocket live-reload, and provides the A2UI runtime for node-connected devices (iOS/Android) to display agent-generated UIs.

### Key Files

| File | Role |
|------|------|
| `server.ts` | `CanvasHostServer` — HTTP server with WebSocket upgrade, file serving with `chokidar` live-reload, default index.html |
| `a2ui.ts` | `handleA2uiHttpRequest()` — serves A2UI assets (index.html, a2ui.bundle.js), `injectCanvasLiveReload()` — injects live-reload + native bridge script |
| `file-resolver.ts` | `normalizeUrlPath()`, `resolveFileWithinRoot()` — safe file path resolution |

### Key Types
```typescript
type CanvasHostServer = {
  port: number;
  rootDir: string;
  close: () => Promise<void>;
};

type CanvasHostHandler = {
  rootDir: string;
  basePath: string;
  handleHttpRequest: (req, res) => Promise<boolean>;
  handleUpgrade: (req, socket, head) => boolean;
  close: () => Promise<void>;
};
```

### A2UI System
- Serves from `src/canvas-host/a2ui/` directory (index.html + a2ui.bundle.js)
- Provides native bridges for iOS (`webkit.messageHandlers`) and Android (`window.openclawCanvasA2UIAction`)
- WebSocket-based live-reload during development
- File watching via `chokidar`
- Default canvas shows OpenClaw Canvas landing page

### Routes
- `/__openclaw__/a2ui/*` — A2UI assets
- `/__openclaw__/canvas/*` — Canvas host path
- `/__openclaw__/ws` — WebSocket for live-reload

### Dependencies
- `ws` — WebSocket server
- `chokidar` — file watching
- `src/media/mime.ts` — MIME detection for static files
- `src/config/paths.ts` — state directory
- `src/utils.ts` — path resolution

### Dependents
- `src/cli/nodes-canvas.ts` / `src/cli/nodes-cli/register.canvas.ts` — canvas CLI commands
- Gateway daemon — starts canvas server
- Node devices — render A2UI content

---

## Cross-Module Dependency Map

```
src/cli/ ──────────► src/commands/ ──────► src/agents/
    │                     │                 src/config/
    │                     │                 src/gateway/
    │                     ▼
    ├──────────────► src/tui/ ────────────► src/routing/
    │                                       src/gateway/ (WebSocket)
    │
    ├──────────────► src/browser/ ────────► src/config/
    │                                       src/media/mime
    │                                       src/logging/
    │
    └──────────────► src/cli/nodes-cli/ ──► src/canvas-host/

src/auto-reply/ ──► src/media-understanding/ ──► src/media/
                ──► src/link-understanding/  ──► src/media-understanding/scope
                ──► src/tts/                 ──► src/media/audio
                ──► src/markdown/            ──► markdown-it

src/channels/ ────► src/markdown/ir + render (per-platform formatting)
              ────► src/media/ (attachment handling)
              ────► src/tts/ (voice notes)
```

---

## v2026.2.15 Changes

### Tools
- **Multi-image in image tool** (#17512): `agents/tools/image-tool.ts` + `image-tool.helpers.ts` — accepts array of up to 20 images per call. See agent-system.md for full details.
- **Normalize structured write/edit text params**: Tool definitions for write/edit now normalize `oldText`/`newText` vs `old_string`/`new_string` parameter variants consistently
- **Harden transcript tool-call block sanitization**: `utils/transcript-tools.ts` — improved sanitization of tool-call blocks in transcripts to prevent injection or malformed content

### Media
- **Share outbound attachment resolver**: `media/outbound-attachment.ts` — extracted shared logic for resolving outbound attachments (file path → hosted URL) previously duplicated across channel plugins
- **Base64 mime sniff helper**: `media/sniff-mime-from-base64.ts` — new utility to detect MIME type from base64-encoded data by inspecting magic bytes after decoding
- **Binary application mimes as non-text**: `media/mime.ts` — `application/octet-stream` and similar binary MIME types now correctly classified as non-text, preventing binary content from being treated as readable text in tool results

## v2026.2.19 Changes

### Tools
- **Read tool auto-paging** — `read` tool auto-pages based on model `contextWindow`; no manual `offset`/`limit` needed for small-to-medium files. See DEVELOPER-REFERENCE.md §9 (gotcha 38)
- **Exec preflight guard** — Detects shell env var injection patterns in scripts before execution. See DEVELOPER-REFERENCE.md §9 (gotcha 40)
- **safeBins trusted dirs** — `tools.exec.safeBins` binaries must resolve from trusted bin directories; PATH-hijacked binaries rejected

### CLI
- **Plugin uninstall** — Full plugin uninstall support in CLI
- **macOS LaunchAgent TMPDIR fix** — `TMPDIR` forwarded to service environment, resolving SQLite failures in daemon mode

## v2026.2.21 Changes <!-- v2026.2.21 -->

### Tools <!-- v2026.2.21 -->

- **Compound command display fix** — `fix(tool-display)`: compound commands such as `cd ~/dir && npm install` previously showed only the first stage (`cd`) in the tool call display. The full compound command is now shown. Affects all tool call monitoring surfaces in the TUI and Control UI. File: `src/agents/tool-display-common.ts`. <!-- v2026.2.21 -->

  Technical detail: `resolveExecDetail()` now calls `splitTopLevelStages()` after `stripShellPreamble()` to collect all stages, then joins them with ` → ` rather than stopping at the preamble boundary. The `summarizePipeline()` call is applied per-stage and concatenated.

- **`senderIsOwner` forwarded to embedded runner** — `fix(tools)`: `senderIsOwner` is now forwarded through to the embedded runner params so that owner-only tool access checks work correctly when tools are invoked from embedded/subagent runners (issue #22296). Previously, the owner context was not propagated into the embedded run, causing owner-only tool calls to be denied. File: `src/agents/pi-embedded-runner/run.ts` (field `senderIsOwner` threaded through `buildEmbeddedRunnerParams` and passed into `attempt.ts`). <!-- v2026.2.21 -->

- **Image resize filename logging** — `fix(agents)`: when the agent resizes images, the filename is now included in the log output. Improves debuggability when multiple images are processed in one session. <!-- v2026.2.21 -->

## v2026.2.22 Changes <!-- v2026.2.22 -->

### Media <!-- v2026.2.22 -->

- **Inbound media byte limits** — Byte limits enforced during download/read across Discord, Telegram, Zalo, Microsoft Teams, and BlueBubbles — prevents oversized payload memory spikes before rejection. <!-- v2026.2.22 -->

- **`application/pdf` MIME classification preserved** — `application/pdf` MIME type is preserved during text-like file heuristics — PDFs use PDF extraction paths instead of being inlined as raw text. <!-- v2026.2.22 -->

- **Sandbox/Media container workspace path mapping** — Container workspace paths (`/workspace/...`, `file:///workspace/...`) mapped back to host sandbox root for outbound media validation. <!-- v2026.2.22 -->

### Browser <!-- v2026.2.22 -->

- **Upload validation with symlinked uploads directory** — Browser upload validation accepts in-root upload paths when the uploads directory is a symlink alias (e.g., `/tmp` → `/private/tmp` on macOS). <!-- v2026.2.22 -->

- **ACP resource link metadata hardening** — `fix(acp)`: ACP resource link metadata is now more strictly validated and formatted before being emitted. Prevents malformed metadata from silently passing through the resource-link pipeline. <!-- v2026.2.21 -->

## v2026.2.23 Changes <!-- v2026.2.23 -->

### Tools <!-- v2026.2.23 -->

- **`web_search` provider "kimi"** — New Moonshot/Kimi web search provider with key/config schema wiring and corrected two-step `$web_search` tool flow with citation extraction (#16616, #18822). Adds `kimi` as a selectable `web_search` provider alongside existing providers. <!-- v2026.2.23 -->

### Media <!-- v2026.2.23 -->

- **Video/Moonshot provider** — Native Moonshot video provider added to `src/media-understanding/providers/`. Refactors video execution to honor `entry/config/provider` `baseUrl` + header precedence so per-entry overrides are applied correctly throughout the video processing pipeline (#12063). <!-- v2026.2.23 -->

## v2026.2.24 Changes <!-- v2026.2.24 -->

### Security <!-- v2026.2.24 -->

- **Native images workspaceOnly** (@tdjackey): `tools.fs.workspaceOnly` is now enforced for native prompt image auto-load (including history refs), preventing out-of-workspace sandbox mounts from being implicitly ingested as vision input. Released in `v2026.2.24`. <!-- v2026.2.24 -->

### Usage accounting <!-- v2026.2.24 -->

- **Moonshot-Kimi cache metrics** (#25436): `cached_tokens` and `prompt_tokens_details.cached_tokens` fields from Moonshot/Kimi API responses are parsed into normalized cache-read usage metrics. Contributor: @Elarwei001. <!-- v2026.2.24 -->

## v2026.3.1 Changes <!-- v2026.3.1 -->

### Tools <!-- v2026.3.1 -->

- **Diffs plugin tool** — New `extensions/diffs/` plugin (26 files, ~4,900 lines) providing a read-only diff viewer and PNG renderer for agents. Accepts before/after text or unified patches and renders them via a Playwright-based screenshotter. Supports `view`, `image`, or `both` output modes with configurable themes (dark/light), layouts (unified/split), and artifact TTL. Gateway viewer URLs are generated for canvas output. Registered as a standard plugin tool with HTTP handler and `before_prompt_build` guidance injection. Files: `extensions/diffs/index.ts`, `src/tool.ts`, `src/render.ts`, `src/browser.ts`, `src/store.ts`, `src/http.ts`, `src/config.ts`, `src/prompt-guidance.ts`. <!-- v2026.3.1 -->

- **Edit workspace boundary errors** — `fs-safe.ts` now uses a new `"outside-workspace"` error code (distinct from `"invalid-path"`) with the message "file is outside workspace root" when a path escapes the workspace root. Previously the generic `"invalid-path"` code was used, which consumers often mapped to a misleading "not found" message. Downstream consumers (`pi-tools.read.ts`, `browser/paths.ts`, `media/server.ts`, `media/store.ts`) now surface the accurate "outside workspace" message. Contributor: @YuzuruS (#29715). Files: `src/infra/fs-safe.ts`, `src/media/server.ts`, `src/media/store.ts`. <!-- v2026.3.1 -->

### Browser <!-- v2026.3.1 -->

- **Browser navigate targetId resolution** (#25326) — After a navigation that triggers a Chrome renderer swap (the old CDP target disappears), `resolveTargetIdAfterNavigate()` in `routes/agent.snapshot.ts` now refreshes the tab list, matches the navigated URL to the replacement target, and returns the correct new `targetId`. Includes a retry with 800ms delay if the replacement tab has not yet appeared. <!-- v2026.3.1 -->

- **Browser open & navigate `url` alias** (#29260) — The `open` and `navigate` browser tool actions now accept `url` as an alias parameter for `targetUrl`, via a new `readTargetUrlParam()` helper. Contributor: @vincentkoc. Files: `src/agents/tools/browser-tool.ts`, `src/agents/tools/browser-tool.schema.ts`. <!-- v2026.3.1 -->

- **Browser auth fail-closed** — If browser control auth bootstrap fails and no explicit auth (token or password) is configured, the browser control HTTP server now refuses to start rather than running unauthenticated. File: `src/browser/server.ts`. <!-- v2026.3.1 -->

- **Browser harden writable output paths** — Stricter validation of writable output paths in the browser module. File: `src/browser/output-atomic.ts`, `src/browser/paths.ts`. <!-- v2026.3.1 -->

- **Sandbox browser docker no-sandbox** (#29879) — Sandbox browser creation applies `--no-sandbox` Chrome flag for Docker environments. Contributor: @Lukavyi. File: `src/agents/sandbox/browser.ts`. <!-- v2026.3.1 -->

### Media <!-- v2026.3.1 -->

- **TTS voice bubbles for Feishu and WhatsApp** (#27366) — Previously only Telegram received opus audio output and had `shouldVoice=true` for voice-bubble playback. A new `VOICE_BUBBLE_CHANNELS` set (`telegram`, `feishu`, `whatsapp`) is used by `resolveOutputFormat()` and `maybeApplyTtsToPayload()` so all three channels get opus output and voice-bubble delivery. Contributors: @smthfoxy, @Takhoffman. File: `src/tts/tts.ts`. <!-- v2026.3.1 -->

- **LINE voice transcription M4A detection** (#31151) — `detectContentType()` in `src/line/download.ts` now inspects the MPEG-4 ftyp box major brand (bytes 8-11) to distinguish M4A audio (`audio/mp4`, brands `M4A ` / `M4B `) from MP4 video. Previously all ftyp containers were classified as `video/mp4`, preventing voice messages from being transcribed. Contributor: @scoootscooob. <!-- v2026.3.1 -->

- **Feishu opus media send type** (#28269) — When sending opus audio files to Feishu, the outbound `msg_type` is now `"audio"` instead of `"media"`. The Feishu API requires the `audio` message type for opus files to render as playable voice messages. Contributor: @Glucksberg. File: `extensions/feishu/src/media.ts`. <!-- v2026.3.1 -->

- **Telegram reply media context** (#28488) — When a user replies to a message containing media (photos, stickers, documents), the replied-to media files are now included in the inbound message context. Media fetch is deferred to the debounce flush for performance. Cached sticker reply media is preserved. Contributor: @obviyus. Files: `src/telegram/bot-handlers.ts`, `src/telegram/bot-message-context.ts`, `src/telegram/bot-message-dispatch.ts`. <!-- v2026.3.1 -->

- **Feishu post embedded media** (#21786) — `parsePostContent()` in `extensions/feishu/src/post.ts` now extracts embedded video and audio (`media` tags with `file_key`) from rich-text post messages, in addition to existing image extraction. `resolveFeishuMediaList()` downloads the embedded media via the messageResource API. Contributor: @laopuhuluwa. <!-- v2026.3.1 -->

### Web Tools <!-- v2026.3.1 -->

- **RFC 2544 fake-IP compatibility** (#31176) — `WEB_TOOLS_TRUSTED_NETWORK_SSRF_POLICY` in `src/agents/tools/web-guarded-fetch.ts` now sets `allowRfc2544BenchmarkRange: true`, permitting the 198.18.0.0/15 address range for trusted fetch endpoints. This range is used by some corporate proxies and benchmarking infrastructure. Contributor: @sunkinux. <!-- v2026.3.1 -->

### Usage <!-- v2026.3.1 -->

- **Clamp negative prompt/input token values** (#30765) — `normalizeUsage()` in `src/agents/usage.ts` now clamps `rawInput` to zero. Some OpenAI-format providers pre-subtract `cached_tokens` from `prompt_tokens` upstream; when `cached_tokens` exceeds `prompt_tokens` the subtraction produces negative values that flowed through to the TUI status bar and `/usage` dashboard. <!-- v2026.3.1 -->

## v2026.3.7 Delta Notes

- Web search: provider selection added to onboarding wizard (PR #34009); Perplexity backend migrated to Search API with filter support.
- OpenAI-compatible TTS endpoint: TTS module now accepts any OpenAI-compatible TTS endpoint in addition to the native OpenAI endpoint.
- Diffs tool: system-prompt hints added so the agent better understands when to invoke the diffs tool.
- Markdown image hardening: markdown image tag handling hardened against malformed URLs and path traversal in rendered output.
- Tool-capability probe budget: increased from 32 to 256 tokens, allowing richer capability probing without exhausting the budget on complex tool schemas.

- **Codex weekly usage window label** (#26267) — The secondary window label in `src/infra/provider-usage.fetch.codex.ts` now labels windows >= 168 hours as "Week" instead of "Day". Codex plans can have a weekly (604800s) quota window; the label previously always showed "Day" for any window >= 24h. Contributor: @Sid-Qin. <!-- v2026.3.1 -->

## v2026.3.8 Delta Notes

- **CLI/backups:** the CLI now has first-class backup creation and verification commands, and recovery-oriented flows (`reset`, uninstall) point users at them before destructive actions.
- **Web search routing:** Brave can now run in `llm-context` mode with source-grounded snippets, while Perplexity uses the native Search API for direct Perplexity auth but preserves the Sonar/OpenRouter path for legacy OpenRouter or explicit `baseUrl` / `model` setups.
- **Browser/CDP remote tooling:** direct `ws://` / `wss://` CDP endpoints are supported as primary profiles, wildcard debugger URLs are rewritten back to the external host/port, and `browser.relayBindHost` supports WSL2/cross-namespace relay exposure.
- **TUI/runtime UX:** the TUI can infer the active agent from the current workspace, and the light-terminal palette now auto-detects `COLORFGBG` with an `OPENCLAW_THEME=light|dark` override.
- **Talk/platform behavior:** top-level `talk.silenceTimeoutMs` now controls pause-to-send thresholds, and Android Play-distributed builds drop self-update, background-location, screen-record, and background-mic capture behavior.

## v2026.3.11 Delta Notes

### Tools

- **Tools/web search Brave — `llm-context` grounding snippets as plain strings:** `src/agents/tools/web-search.ts` — `BraveLlmContextResult.snippets` entries are now filtered with `typeof s === "string"` before use, treating grounding snippet fields as plain strings rather than assuming they are always strings. Prevents `web_search` from returning empty snippet arrays in LLM Context mode when the API returns non-string snippet values. Fixes #41387.

- **Tools/web search OpenRouter Perplexity — recover citation extraction from `message.annotations`:** `src/agents/tools/web-search.ts` — OpenRouter Perplexity citation extraction now reads from `choice.message.annotations` in addition to the top-level response, recovering citations that were silently dropped when Perplexity routed through OpenRouter. Fixes #40881.

- **Resolve web tool SecretRefs atomically at runtime:** `src/cli/command-secret-gateway.ts` + `src/secrets/runtime-web-tools.ts` — web tool `SecretRef` values (`apiKey` and similar fields) are now resolved atomically at runtime and swapped into the gateway snapshot in a single operation, preventing partial-resolution races. Fixes #41599.

- **Models/Kimi Coding — send tools in native Anthropic format:** Kimi Coding models (`kimi-coding/k2p5` and related) now receive tool definitions in native `anthropic-messages` format again. A regression had caused tool calls to degrade to XML/plain-text encoding, reducing tool call reliability. Fixes #38669, #39907, #40552.

- **Models/Alibaba Cloud Model Studio — `MODELSTUDIO_API_KEY` env auth:** `src/agents/model-auth-env-vars.ts` — `MODELSTUDIO_API_KEY` is now wired through shared env auth, implicit provider discovery, and shell-env fallback. Alibaba Cloud Model Studio can now authenticate via environment variable without manual config. Fixes #40634.

- **Models guard — guard optional `model.input` capability checks:** capability checks on `model.input` are now guarded for optional/undefined values, preventing runtime errors when a model entry omits the `input` field. Fixes #42096.

### Memory

- **Memory/multimodal — new opt-in multimodal image and audio indexing:** `src/memory/` — new opt-in multimodal memory indexing mode using `gemini-embedding-2-preview` for supported image and audio files discovered through `memorySearch.extraPaths`. Default memory roots remain Markdown-only unless extra paths are configured. Fixes #43460.

- **Memory/Gemini embedding-2-preview — configurable output dimensions and automatic reindexing:** `src/memory/embeddings-gemini.ts` + `src/memory/manager-sync-ops.ts` — `gemini-embedding-2-preview` embeddings now support configurable `outputDimensionality`. Changing the dimensionality setting triggers automatic reindexing of the memory store. Fixes #42501.

### Models/Catalog

- **OpenRouter stealth models — Hunter Alpha and Healer Alpha in built-in catalog:** `src/agents/models-config.providers.static.ts` — temporary entries for `openrouter/hunter-alpha` (Hunter Alpha) and `openrouter/healer-alpha` (Healer Alpha) added to the built-in model catalog. Fixes #43642.

### Agents

- **Agents/context pruning — prune image-only tool results during soft-trim:** `src/agents/pi-extensions/context-pruning/` — image-only tool results are now pruned during soft-trim (replaced with placeholders), and historical image cleanup is extended to cover more cases where replayed image bytes are not needed. Reduces token usage and context pressure for vision-heavy sessions. Fixes #43045.
