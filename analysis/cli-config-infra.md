# OpenClaw CLI, Config & Infrastructure — Comprehensive Analysis
<!-- markdownlint-disable MD024 -->

> Updated: 2026-04-06 | Version: v2026.4.5 | Codebase: OpenClaw release tag `v2026.4.5` | Cluster: CLI, CONFIG & INFRASTRUCTURE

---

## Table of Contents

1. [src/cli — CLI Framework & Command Registration](#1-srccli)
2. [src/commands — Command Implementations](#2-srccommands)
3. [src/config — Configuration Schema & IO](#3-srcconfig)
4. [src/infra — Infrastructure Utilities](#4-srcinfra)
5. [src/daemon — System Service Management](#5-srcdaemon)
6. [src/wizard — Onboarding Wizard](#6-srcwizard)
7. [src/tui — Terminal UI](#7-srctui)
8. [src/terminal — Terminal Rendering Primitives](#8-srcterminal)

---

## 1. src/cli

### Module Overview

The CLI module is the **entry point and command registration layer** for the `openclaw` CLI tool. Built on [Commander.js](https://github.com/tj/commander.js), it uses a **lazy command registration** pattern where only the invoked subcommand's module is loaded at runtime, keeping startup fast.

**Architecture Pattern:** Lazy-loaded command tree with Commander.js. Each subcommand is registered via a `register*.ts` file that attaches commands to the program. A fast-path "route" system bypasses full Commander parsing for hot commands (status, health, sessions).

**Entry Points:**
- `run-main.ts` → `runCli()` — the main CLI entry point
- `program.ts` → re-exports `buildProgram()` from `program/build-program.ts`
- `route.ts` → `tryRouteCli()` — fast-path routing before Commander

### File Inventory (261 files)

#### Core CLI Bootstrap
| File | Description |
|------|-------------|
| `run-main.ts` | Main CLI entry: dotenv, env normalization, route-first, Commander parse |
| `program.ts` | Re-exports `buildProgram` and `forceFreePort` |
| `route.ts` | Fast-path command routing (bypasses Commander for hot paths) |
| `argv.ts` | Argv parsing utilities: `getPrimaryCommand`, `hasHelpOrVersion`, `getCommandPath`, `hasFlag`, `getFlagValue` |
| `banner.ts` | Emit startup banner with version info |
| `cli-name.ts` | Resolve CLI binary name |
| `cli-utils.ts` | Shared CLI utility functions |
| `deps.ts` | CLI dependency declarations/checks |
| `ports.ts` | Port allocation and free-port logic |
| `tagline.ts` | Random tagline generator for CLI output |
| `wait.ts` | Wait/polling utilities for CLI commands |
| `windows-argv.ts` | Windows-specific argv normalization |

#### Program Construction (`program/`)
| File | Description |
|------|-------------|
| `program/build-program.ts` | Creates Commander program, sets context, registers commands |
| `program/command-registry.ts` | Core command registry with lazy loading of 12 command groups |
| `program/context.ts` | `createProgramContext()` — version, agent channel options |
| `program/program-context.ts` | Get/set program context on Commander instance |
| `program/help.ts` | Custom help formatting for Commander |
| `program/helpers.ts` | Shared helpers for program registration |
| `program/preaction.ts` | Pre-action hooks (version check, config guard) |
| `program/action-reparse.ts` | Re-parse argv after lazy command load |
| `program/config-guard.ts` | Ensure config is valid before command execution |
| `program/routes.ts` | Route specs for fast-path commands (health, status, sessions, agents list, etc.) |
| `program/register.subclis.ts` | 28 subcli entries (gateway, daemon, nodes, browser, etc.) with lazy loading |
| `program/register.setup.ts` | `setup` command registration |
| `program/register.onboard.ts` | `onboard` command registration |
| `program/register.configure.ts` | `configure` command registration |
| `program/register.maintenance.ts` | doctor, dashboard, reset, uninstall commands |
| `program/register.message.ts` | `message` command tree registration |
| `program/register.agent.ts` | `agent` and `agents` command registration |
| `program/register.status-health-sessions.ts` | status, health, sessions commands |

#### Message Subcommands (`program/message/`)
| File | Description |
|------|-------------|
| `program/message/register.send.ts` | `message send` command |
| `program/message/register.read-edit-delete.ts` | `message read/edit/delete` commands |
| `program/message/register.broadcast.ts` | `message broadcast` command |
| `program/message/register.reactions.ts` | `message react` command |
| `program/message/register.emoji-sticker.ts` | Emoji/sticker management |
| `program/message/register.pins.ts` | Pin/unpin messages |
| `program/message/register.poll.ts` | Poll creation |
| `program/message/register.thread.ts` | Thread management |
| `program/message/register.permissions-search.ts` | Permission/search commands |
| `program/message/register.discord-admin.ts` | Discord admin commands |
| `program/message/helpers.ts` | Shared message command helpers |

#### Subcli Modules
| File | Description |
|------|-------------|
| `gateway-cli.ts` | Gateway CLI entry point |
| `gateway-cli/register.ts` | Gateway subcommand registration (start, stop, restart, status, call, dev, discover) |
| `gateway-cli/run.ts` | `gateway run` — starts gateway process |
| `gateway-cli/run-loop.ts` | Gateway run loop with restart/respawn logic |
| `gateway-cli/call.ts` | `gateway call` — invoke gateway RPC |
| `gateway-cli/dev.ts` | `gateway dev` — development mode |
| `gateway-cli/discover.ts` | `gateway discover` — mDNS/Bonjour discovery |
| `gateway-cli/shared.ts` | Shared gateway CLI utilities |
| `gateway-rpc.ts` | Gateway RPC client |
| `daemon-cli.ts` | Daemon CLI entry (legacy service management alias) |
| `daemon-cli/register.ts` | Daemon subcommand registration |
| `daemon-cli/lifecycle.ts` | start/stop/restart daemon lifecycle |
| `daemon-cli/lifecycle-core.ts` | Core lifecycle implementation |
| `daemon-cli/install.ts` | `daemon install` — system service installation |
| `daemon-cli/status.ts` | `daemon status` command |
| `daemon-cli/status.gather.ts` | Gather daemon status info |
| `daemon-cli/status.print.ts` | Format and print daemon status |
| `daemon-cli/probe.ts` | Probe daemon health |
| `daemon-cli/runners.ts` | Daemon runner detection |
| `daemon-cli/register-service-commands.ts` | Register service lifecycle commands |
| `daemon-cli/response.ts` | Format daemon command responses |
| `daemon-cli/shared.ts` | Shared daemon CLI utilities |
| `daemon-cli/types.ts` | Daemon CLI types |
| `daemon-cli-compat.ts` | Backward-compatible daemon CLI aliases |
| `browser-cli.ts` | Browser CLI entry — `openclaw browser` |
| `browser-cli-actions-input.ts` | Browser action input commands |
| `browser-cli-actions-input/register.ts` | Register browser action input commands |
| `browser-cli-actions-input/register.element.ts` | Element interaction commands (click, type, etc.) |
| `browser-cli-actions-input/register.navigation.ts` | Navigation commands (goto, back, forward) |
| `browser-cli-actions-input/register.form-wait-eval.ts` | Form, wait, evaluate commands |
| `browser-cli-actions-input/register.files-downloads.ts` | File upload/download commands |
| `browser-cli-actions-input/shared.ts` | Shared browser action utilities |
| `browser-cli-actions-observe.ts` | Browser observation commands (snapshot, screenshot) |
| `browser-cli-debug.ts` | Browser debug tools |
| `browser-cli-examples.ts` | Browser usage examples |
| `browser-cli-extension.ts` | Chrome extension relay management |
| `browser-cli-inspect.ts` | Browser page inspection |
| `browser-cli-manage.ts` | Browser profile/instance management |
| `browser-cli-shared.ts` | Shared browser CLI utilities |
| `browser-cli-state.ts` | Browser state management |
| `browser-cli-state.cookies-storage.ts` | Cookie/storage state persistence |
| `nodes-cli.ts` | Nodes CLI entry — `openclaw nodes` |
| `nodes-cli/register.ts` | Register all nodes subcommands |
| `nodes-cli/register.status.ts` | `nodes status` — list paired nodes |
| `nodes-cli/register.pairing.ts` | `nodes pair/unpair` — node pairing |
| `nodes-cli/register.camera.ts` | `nodes camera` — camera snap/clip |
| `nodes-cli/register.screen.ts` | `nodes screen` — screen recording |
| `nodes-cli/register.canvas.ts` | `nodes canvas` — canvas management |
| `nodes-cli/register.invoke.ts` | `nodes invoke/run` — remote command execution |
| `nodes-cli/register.notify.ts` | `nodes notify` — push notifications |
| `nodes-cli/register.location.ts` | `nodes location` — location queries |
| `nodes-cli/rpc.ts` | Nodes RPC client |
| `nodes-cli/cli-utils.ts` | Nodes CLI utilities |
| `nodes-cli/format.ts` | Nodes output formatting |
| `nodes-cli/types.ts` | Nodes CLI types |
| `nodes-cli/a2ui-jsonl.ts` | A2UI JSONL protocol support |
| `nodes-camera.ts` | Camera command implementation |
| `nodes-canvas.ts` | Canvas command implementation |
| `nodes-screen.ts` | Screen recording implementation |
| `node-cli.ts` | Node (single node) CLI entry |
| `node-cli/register.ts` | Node subcommand registration |
| `node-cli/daemon.ts` | Node daemon management |
| `models-cli.ts` | `openclaw models` — model configuration |
| `config-cli.ts` | `openclaw config` — config get/set/edit/unset/file/schema/validate |
| `channels-cli.ts` | `openclaw channels` — channel management |
| `logs-cli.ts` | `openclaw logs` — log viewing |
| `memory-cli.ts` | `openclaw memory` — memory management |
| `cron-cli.ts` | `openclaw cron` — cron scheduler |
| `cron-cli/register.ts` | Register cron subcommands |
| `cron-cli/register.cron-add.ts` | `cron add` command |
| `cron-cli/register.cron-edit.ts` | `cron edit` command |
| `cron-cli/register.cron-simple.ts` | Simple cron commands (list, remove, enable, disable) |
| `cron-cli/shared.ts` | Shared cron CLI utilities |
| `dns-cli.ts` | `openclaw dns` — DNS helpers |
| `docs-cli.ts` | `openclaw docs` — documentation helpers |
| `hooks-cli.ts` | `openclaw hooks` — hooks tooling |
| `webhooks-cli.ts` | `openclaw webhooks` — webhook helpers |
| `pairing-cli.ts` | `openclaw pairing` — pairing helpers |
| `plugins-cli.ts` | `openclaw plugins` — plugin management |
| `devices-cli.ts` | `openclaw devices` — device pairing/tokens |
| `directory-cli.ts` | `openclaw directory` — directory commands |
| `security-cli.ts` | `openclaw security` — security helpers |
| `skills-cli.ts` | `openclaw skills` — skills management |
| `skills-cli.format.ts` | Skills output formatting |
| `sandbox-cli.ts` | `openclaw sandbox` — sandbox tools |
| `system-cli.ts` | `openclaw system` — system events/heartbeat/presence |
| `exec-approvals-cli.ts` | `openclaw approvals` — exec approval management |
| `acp-cli.ts` | `openclaw acp` — Agent Control Protocol tools |
| `update-cli.ts` | `openclaw update` — CLI update helpers |
| `update-cli/update-command.ts` | Update command implementation |
| `update-cli/status.ts` | Update status check |
| `update-cli/progress.ts` | Update progress display |
| `update-cli/wizard.ts` | Update wizard flow |
| `update-cli/shared.ts` | Shared update CLI utilities |
| `update-cli/suppress-deprecations.ts` | Suppress Node.js deprecation warnings during update |
| `completion-cli.ts` | `openclaw completion` — shell completion generation |
| `tui-cli.ts` | `openclaw tui` — terminal UI launcher |

#### Shared Utilities
| File | Description |
|------|-------------|
| `channel-auth.ts` | Channel authentication helpers |
| `channel-options.ts` | Channel option builders for Commander |
| `command-format.ts` | Format CLI command strings for display |
| `command-options.ts` | Common command option definitions |
| `help-format.ts` | Help text formatting |
| `outbound-send-deps.ts` | Outbound message send dependency injection |
| `parse-bytes.ts` | Parse byte size strings (e.g. "10mb") |
| `parse-duration.ts` | Parse duration strings (e.g. "30m", "1h") |
| `parse-timeout.ts` | Parse timeout values |
| `plugin-registry.ts` | Plugin registry initialization for CLI |
| `profile.ts` | CLI profile management |
| `profile-utils.ts` | Profile utility functions |
| `progress.ts` | Progress display helpers |
| `prompt.ts` | Interactive prompt utilities |
| `respawn-policy.ts` | Process respawn policy (backoff, max retries) |
| `shared/parse-port.ts` | Port parsing utility |

### Key Types & Interfaces

```typescript
// program/command-registry.ts
type CommandRegistration = { id: string; register: (params: CommandRegisterParams) => void };
type CommandRegisterParams = { program: Command; ctx: ProgramContext; argv: string[] };

// program/context.ts  
type ProgramContext = {
  programVersion: string;
  agentChannelOptions: AgentChannelOptions;
};

// program/routes.ts
type RouteSpec = {
  match: (path: string[]) => boolean;
  loadPlugins?: boolean;
  run: (argv: string[]) => Promise<boolean>;
};
```

### Data Flow

1. **`runCli(argv)`** in `run-main.ts`:
   - Normalize Windows argv
   - Load `.env` files
   - Normalize environment variables
   - Ensure CLI on PATH
   - Assert supported Node.js runtime
2. **Fast-path routing** via `tryRouteCli()`:
   - Match command path against `RouteSpec[]` (health, status, sessions, agents list, etc.)
   - If matched: emit banner, ensure config ready, run handler directly → **exit early**
3. **Commander path** (if no fast route matched):
   - `buildProgram()` → create Commander program + context
   - Register pre-action hooks (version check, config guard)
   - Register core CLI commands (lazy placeholders)
   - Register subcli commands (lazy placeholders)
   - `program.parseAsync(argv)` → Commander dispatches to action handler
4. **Lazy command loading**: When a placeholder action fires, it:
   - Removes the placeholder command
   - Dynamically imports the real registrar module
   - Re-registers the real command(s)
   - Re-parses argv against the now-loaded command tree

### Command Reference (Top-Level)

| Command | Description |
|---------|-------------|
| `openclaw setup` | Setup helpers |
| `openclaw onboard` | Onboarding wizard |
| `openclaw configure` | Configuration wizard |
| `openclaw config` | Config get/set/edit/unset/file/schema/validate |
| `openclaw doctor` | Health checks + quick fixes |
| `openclaw dashboard` | Open Control UI |
| `openclaw reset` | Reset local config/state |
| `openclaw uninstall` | Uninstall gateway service + data |
| `openclaw message` | Send, read, manage messages (send/read/edit/delete/react/broadcast/poll/thread/pin) |
| `openclaw memory` | Memory management |
| `openclaw agent` | Agent commands |
| `openclaw agents` | Multi-agent management (list/add/delete/identity) |
| `openclaw status` | Gateway status overview |
| `openclaw health` | Gateway health check |
| `openclaw sessions` | Session management |
| `openclaw browser` | Browser automation tools |
| `openclaw gateway` | Gateway control (`run`/`stop`/`restart`/`status`/`call`/`discover`; supports `--dev`, `--require-rpc` on `status`; `gateway run` accepts `--cli-backend-logs` flag, `--claude-cli-logs` kept as deprecated alias) |
| `openclaw daemon` | Legacy service management alias |
| `openclaw logs` | View gateway logs |
| `openclaw system` | System events, heartbeat, presence |
| `openclaw models` | Model configuration (list/set/status/scan/auth) |
| `openclaw approvals` | Exec approval management |
| `openclaw nodes` | Node management (status/pending/approve/reject/camera/screen/canvas/invoke/notify/location) |
| `openclaw devices` | Device pairing + token management |
| `openclaw node` | Single node control |
| `openclaw sandbox` | Sandbox tools |
| `openclaw tui` | Terminal UI |
| `openclaw cron` | Cron scheduler (list/add/edit/remove/enable/disable) |
| `openclaw dns` | DNS helpers |
| `openclaw docs` | Documentation helpers |
| `openclaw hooks` | Hooks tooling |
| `openclaw webhooks` | Webhook helpers |
| `openclaw pairing` | Pairing helpers |
| `openclaw plugins` | Plugin management |
| `openclaw channels` | Channel management |
| `openclaw directory` | Directory commands |
| `openclaw security` | Security helpers |
| `openclaw skills` | Skills management |
| `openclaw update` | CLI update |
| `openclaw completion` | Shell completion script generation |
| `openclaw acp` | Agent Control Protocol tools |
| `openclaw backup` | Backup create/verify |
| `openclaw mcp` | MCP server management |
| `openclaw qr` | iOS pairing QR code generation |
| `openclaw clawbot` | Legacy aliases |
| `openclaw secrets` | Secrets reload |

### Internal Dependencies
- `src/config` — config loading, paths, types, schema
- `src/infra` — env, dotenv, errors, runtime-guard, path-env, unhandled-rejections, home-dir
- `src/commands` — actual command implementations
- `src/plugins` — plugin CLI registration
- `src/channels` — channel registry, plugins
- `src/agents` — agent scope, defaults
- `src/routing` — session key parsing
- `src/logging` — console capture, subsystem logging
- `src/runtime.ts` — default runtime
- `src/version.ts` — VERSION constant

### External Dependencies
- `commander` — CLI framework
- `json5` — config parsing (transitive)

### Test Coverage
| Test File | Coverage |
|-----------|----------|
| `argv.test.ts` | Argv parsing utilities |
| `browser-cli.test.ts` | Browser CLI registration |
| `browser-cli-extension.test.ts` | Chrome extension relay |
| `browser-cli-inspect.test.ts` | Browser inspection |
| `config-cli.test.ts` | Config CLI commands |
| `cron-cli.test.ts` | Cron CLI commands |
| `cron-cli/shared.test.ts` | Cron shared utilities |
| `daemon-cli-compat.test.ts` | Daemon CLI backward compat |
| `daemon-cli.coverage.test.ts` | Daemon CLI coverage |
| `deps.test.ts` | CLI dependency checks |
| `dns-cli.test.ts` | DNS CLI commands |
| `exec-approvals-cli.test.ts` | Exec approvals CLI |
| `gateway-cli.coverage.test.ts` | Gateway CLI coverage |
| `gateway-cli/discover.test.ts` | Gateway discovery |
| `gateway-cli/run-loop.test.ts` | Gateway run loop |
| `gateway.sigterm.test.ts` | Gateway SIGTERM handling |
| `hooks-cli.test.ts` | Hooks CLI |
| `logs-cli.test.ts` | Logs CLI |
| `memory-cli.test.ts` | Memory CLI |
| `models-cli.test.ts` | Models CLI |
| `nodes-camera.test.ts` | Nodes camera |
| `nodes-canvas.test.ts` | Nodes canvas |
| `nodes-cli.coverage.test.ts` | Nodes CLI coverage |
| `nodes-cli/register.invoke.nodes-run-approval-timeout.test.ts` | Invoke approval timeout |
| `nodes-screen.test.ts` | Nodes screen recording |
| `pairing-cli.test.ts` | Pairing CLI |
| `parse-bytes.test.ts` | Byte parsing |
| `parse-duration.test.ts` | Duration parsing |
| `profile.test.ts` | CLI profiles |
| `program.force.test.ts` | Force flag behavior |
| `program.nodes-basic.e2e.test.ts` | Nodes basic E2E |
| `program.nodes-media.e2e.test.ts` | Nodes media E2E |
| `program.smoke.test.ts` | CLI smoke tests |
| `program.test-mocks.ts` | Test mock helpers |
| `program/command-registry.test.ts` | Command registry |
| `program/config-guard.test.ts` | Config guard |
| `program/message/helpers.test.ts` | Message helpers |
| `program/register.subclis.e2e.test.ts` | Subcli registration E2E |
| `program/routes.test.ts` | Route matching |
| `progress.test.ts` | Progress display |
| `prompt.test.ts` | Interactive prompts |
| `respawn-policy.test.ts` | Respawn policy |
| `run-main.exit.test.ts` | Exit handling |
| `run-main.test.ts` | Main entry tests |
| `skills-cli.test.ts` | Skills CLI |
| `update-cli.test.ts` | Update CLI |
| `wait.test.ts` | Wait utilities |

---

## 2. src/commands

### Module Overview

The commands module contains **business logic implementations** for all CLI commands and onboarding flows. Commands are pure functions that receive options + a runtime context, perform work, and output results. They are invoked by the CLI layer but can also be called programmatically.

**Architecture Pattern:** Command functions with dependency injection via runtime context. Each command is typically a single exported async function.

**Entry Points:** Each file exports one or more command functions invoked by `src/cli` registrars.

### File Inventory (322 files)

#### Agent Commands
| File | Description |
|------|-------------|
| `agent.ts` | `agentCommand()` — run agent session |
| `agent-via-gateway.ts` | Agent command routed through gateway |
| `agent/delivery.ts` | Agent message delivery logic |
| `agent/run-context.ts` | Agent run context construction |
| `agent/session-store.ts` | Agent session persistence |
| `agent/session.ts` | Agent session management |
| `agent/types.ts` | Agent command types |
| `agents.ts` | `agentsListCommand()`, multi-agent management |
| `agents.bindings.ts` | Agent channel bindings |
| `agents.command-shared.ts` | Shared agent command utilities |
| `agents.commands.add.ts` | `agents add` implementation |
| `agents.commands.delete.ts` | `agents delete` implementation |
| `agents.commands.identity.ts` | `agents identity` implementation |
| `agents.commands.list.ts` | `agents list` implementation |
| `agents.config.ts` | Agent configuration helpers |
| `agents.providers.ts` | Agent provider resolution |

#### Auth & Onboarding
| File | Description |
|------|-------------|
| `auth-choice.ts` | Provider auth choice flow |
| `auth-choice-prompt.ts` | Auth choice interactive prompts |
| `auth-choice-options.ts` | Auth choice CLI options |
| `auth-choice-legacy.ts` | Legacy auth migration |
| `auth-choice.api-key.ts` | API key auth flow |
| `auth-choice.apply.ts` | Apply auth choice to config |
| `auth-choice.apply.anthropic.ts` | Anthropic-specific auth application |
| `auth-choice.apply.api-providers.ts` | Generic API provider auth |
| `auth-choice.apply.copilot-proxy.ts` | GitHub Copilot proxy auth |
| `auth-choice.apply.github-copilot.ts` | GitHub Copilot auth |
| `auth-choice.apply.google-gemini-cli.ts` | Gemini CLI auth |
| `auth-choice.apply.huggingface.ts` | HuggingFace auth |
| `auth-choice.apply.minimax.ts` | MiniMax auth |
| `auth-choice.apply.oauth.ts` | Generic OAuth auth |
| `auth-choice.apply.openai.ts` | OpenAI auth |
| `auth-choice.apply.openrouter.ts` | OpenRouter auth |
| `auth-choice.apply.plugin-provider.ts` | Plugin provider auth |
| ~~`auth-choice.apply.qwen-portal.ts`~~ | _Removed in v2026.3.28_ — Qwen Portal (`qwen-portal-auth`) OAuth removed; use Qwen via Model Studio (`openclaw onboard --auth-choice modelstudio-api-key`) |
| `auth-choice.apply.vllm.ts` | vLLM auth |
| `auth-choice.apply.xai.ts` | xAI auth |
| `auth-choice.apply-helpers.ts` | Auth application helpers |
| `auth-choice.default-model.ts` | Default model selection for auth |
| `auth-choice.model-check.ts` | Model availability check |
| `auth-choice.preferred-provider.ts` | Preferred provider resolution |
| `auth-token.ts` | Auth token utilities |
| `chutes-oauth.ts` | Chutes OAuth flow |
| `oauth-env.ts` | OAuth environment setup |
| `oauth-flow.ts` | OAuth flow implementation |
| `openai-codex-oauth.ts` | OpenAI Codex OAuth |
| `provider-auth-helpers.ts` | Provider auth helper utilities |

#### Onboarding
| File | Description |
|------|-------------|
| `onboard.ts` | Main onboarding entry point |
| `onboard-auth.ts` | Onboarding auth step |
| `onboard-auth.config-core.ts` | Core config for onboarding auth |
| `onboard-auth.config-gateways.ts` | Gateway config for onboarding |
| `onboard-auth.config-litellm.ts` | LiteLLM config |
| `onboard-auth.config-minimax.ts` | MiniMax config |
| `onboard-auth.config-opencode.ts` | OpenCode config |
| `onboard-auth.config-shared.ts` | Shared onboarding auth config |
| `onboard-auth.credentials.ts` | Credential storage during onboarding |
| `onboard-auth.models.ts` | Model setup during onboarding |
| `onboard-channels.ts` | Channel setup during onboarding |
| `onboard-config.ts` | Config application during onboarding |
| `onboard-custom.ts` | Custom API config prompts |
| `onboard-helpers.ts` | Onboarding utility functions |
| `onboard-hooks.ts` | Internal hooks setup during onboarding |
| `onboard-interactive.ts` | Interactive onboarding flow |
| `onboard-non-interactive.ts` | Non-interactive (CI/headless) onboarding |
| `onboard-non-interactive/api-keys.ts` | API key setup for non-interactive |
| `onboard-non-interactive/local.ts` | Local gateway setup |
| `onboard-non-interactive/local/auth-choice.ts` | Auth choice for local setup |
| `onboard-non-interactive/local/auth-choice-inference.ts` | Auth inference |
| `onboard-non-interactive/local/daemon-install.ts` | Daemon install during onboarding |
| `onboard-non-interactive/local/gateway-config.ts` | Gateway config for local |
| `onboard-non-interactive/local/output.ts` | Output formatting |
| `onboard-non-interactive/local/skills-config.ts` | Skills setup |
| `onboard-non-interactive/local/workspace.ts` | Workspace setup |
| `onboard-non-interactive/remote.ts` | Remote gateway onboarding |
| `onboard-provider-auth-flags.ts` | Provider auth CLI flags |
| `onboard-remote.ts` | Remote gateway config prompts |
| `onboard-skills.ts` | Skills setup during onboarding |
| `onboard-types.ts` | Onboarding type definitions |
| `onboarding/plugin-install.ts` | Plugin installation during onboarding |
| `onboarding/registry.ts` | Onboarding step registry |
| `onboarding/types.ts` | Onboarding registry types |

#### Configuration Commands
| File | Description |
|------|-------------|
| `configure.ts` | `configure` command entry |
| `configure.channels.ts` | Channel configuration wizard |
| `configure.commands.ts` | Command configuration |
| `configure.daemon.ts` | Daemon configuration |
| `configure.gateway.ts` | Gateway configuration |
| `configure.gateway-auth.ts` | Gateway auth configuration |
| `configure.shared.ts` | Shared configure utilities |
| `configure.wizard.ts` | Interactive configuration wizard |

#### Status & Health
| File | Description |
|------|-------------|
| `status.ts` | `statusCommand()` — gateway status |
| `status.command.ts` | Status command wrapper |
| `status.format.ts` | Status output formatting |
| `status.summary.ts` | Status summary generation |
| `status.types.ts` | Status type definitions |
| `status.scan.ts` | Status scanning |
| `status.daemon.ts` | Daemon status section |
| `status.gateway-probe.ts` | Gateway health probe |
| `status.agent-local.ts` | Agent local status |
| `status.link-channel.ts` | Channel link status |
| `status.update.ts` | Update status section |
| `status-all.ts` | Comprehensive status (`--all`) — single flat file covering agents, channels, diagnosis, formatting, gateway, and report lines |
| `health.ts` | `healthCommand()` — health check |
| `health-format.ts` | Health output formatting |
| `gateway-status.ts` | Gateway-specific status — single flat file (no subdirectory) |
| `gateway-presence.ts` | Gateway presence management |
| `sessions.ts` | `sessionsCommand()` — session management |

#### Models
| File | Description |
|------|-------------|
| `models.ts` | Models command entry |
| `models/list.ts` | `models list` — list available models |
| `models/list.list-command.ts` | List command implementation |
| `models/list.status-command.ts` | `models status` implementation |
| `models/list.auth-overview.ts` | Auth overview for models |
| `models/list.configured.ts` | Show configured models |
| `models/list.errors.ts` | Model error display |
| `models/list.format.ts` | Model list formatting |
| `models/list.probe.ts` | Model health probing |
| `models/list.registry.ts` | Model registry interaction |
| `models/list.table.ts` | Table formatting for models |
| `models/list.types.ts` | Model list types |
| `models/set.ts` | `models set` — set default model |
| `models/set-image.ts` | Set image model |
| `models/auth.ts` | Model authentication |
| `models/auth-order.ts` | Auth order management |
| `models/aliases.ts` | Model alias management |
| `models/fallbacks.ts` | Model fallback chains |
| `models/image-fallbacks.ts` | Image model fallbacks |
| `models/scan.ts` | Model scanning/discovery |
| `models/shared.ts` | Shared model utilities |
| `model-allowlist.ts` | Model allowlist management |
| `model-picker.ts` | Interactive model picker |

#### Doctor (Diagnostics)
| File | Description |
|------|-------------|
| `doctor.ts` | `doctorCommand()` — comprehensive diagnostics |
| `doctor-auth.ts` | Auth diagnostics |
| `doctor-completion.ts` | Shell completion diagnostics |
| `doctor-config-flow.ts` | Config flow diagnostics |
| `doctor-format.ts` | Doctor output formatting |
| `doctor-gateway-daemon-flow.ts` | Gateway/daemon flow diagnostics |
| `doctor-gateway-health.ts` | Gateway health diagnostics |
| `doctor-gateway-services.ts` | Gateway services diagnostics |
| `doctor-install.ts` | Installation diagnostics |
| `doctor-legacy-config.ts` | Legacy config detection |
| `doctor-memory-search.ts` | Memory search diagnostics |
| `doctor-platform-notes.ts` | Platform-specific notes |
| `doctor-prompter.ts` | Doctor interactive prompts |
| `doctor-sandbox.ts` | Sandbox diagnostics |
| `doctor-security.ts` | Security diagnostics |
| `doctor-state-integrity.ts` | State directory integrity |
| `doctor-state-migrations.ts` | State migration checks |
| `doctor-ui.ts` | UI diagnostics |
| `doctor-update.ts` | Update diagnostics |
| `doctor-workspace.ts` | Workspace diagnostics |
| `doctor-workspace-status.ts` | Workspace status |

#### Other Commands
| File | Description |
|------|-------------|
| `dashboard.ts` | Open Control UI dashboard |
| `src/commands/docs.ts` | Documentation browser |
| `message.ts` | Message command implementations |
| `message-format.ts` | Message formatting |
| `reset.ts` | Reset config/state |
| `uninstall.ts` | Uninstall gateway + data |
| `setup.ts` | Setup helpers |
| `sandbox.ts` | Sandbox management |
| `sandbox-display.ts` | Sandbox display formatting |
| `sandbox-explain.ts` | Sandbox explanation |
| `sandbox-formatters.ts` | Sandbox output formatters |
| `signal-install.ts` | Signal messenger installation |
| `systemd-linger.ts` | Systemd linger management |
| `vllm-setup.ts` | vLLM setup helpers |
| `zai-endpoint-detect.ts` | xAI endpoint detection |
| `cleanup-utils.ts` | Cleanup utility functions |
| `daemon-install-helpers.ts` | Daemon installation helpers |
| `daemon-runtime.ts` | Daemon runtime utilities |
| `node-daemon-install-helpers.ts` | Node daemon install helpers |
| `node-daemon-runtime.ts` | Node daemon runtime |
| `google-gemini-model-default.ts` | Google Gemini default model |
| `openai-model-default.ts` | OpenAI default model |
| `openai-codex-model-default.ts` | OpenAI Codex default model |
| `opencode-zen-model-default.ts` | OpenCode Zen default model |

### Internal Dependencies
- `src/config` — all config types, loading, writing, paths, schema, validation
- `src/cli` — command format utilities
- `src/infra` — env, errors, home-dir, fetch, ports, system events, heartbeat, outbound, etc.
- `src/agents` — agent scope, auth profiles, defaults, model selection
- `src/channels` — channel registry, plugins
- `src/daemon` — service management
- `src/wizard` — onboarding flows
- `src/runtime.ts` — default runtime
- `src/version.ts` — VERSION

---

## 3. src/config

### Module Overview

The config module is the **central configuration system** for OpenClaw. It defines the complete configuration schema using Zod, handles loading/saving JSON5 config files, environment variable substitution, `$include` resolution, legacy migration, and runtime defaults.

**Architecture Pattern:** Schema-first (Zod) with layered config resolution: file → includes → env substitution → validation → defaults → runtime overrides.

**Entry Points:**
- `config.ts` — barrel export: `loadConfig()`, `writeConfigFile()`, `readConfigFileSnapshot()`, etc.
- `types.ts` — barrel export for all TypeScript types
- `schema.ts` — JSON Schema generation from Zod
- `zod-schema.ts` — Zod schema definition

### File Inventory (202 files)

#### Core
| File | Description |
|------|-------------|
| `config.ts` | Barrel export for config module |
| `io.ts` | Config file I/O: load, parse JSON5, write, audit, caching |
| `validation.ts` | Zod-based config validation (with/without plugins) |
| `defaults.ts` | Apply runtime defaults (models, agents, sessions, logging, etc.) |
| `paths.ts` | Config/state directory resolution, gateway port |
| `config-paths.ts` | Additional config path utilities |
| `version.ts` | Config version comparison utilities |

#### Schema
| File | Description |
|------|-------------|
| `schema.ts` | JSON Schema generation from Zod + UI hints |
| `schema.help.ts` | Schema help text generation |
| `schema.hints.ts` | UI hints for config fields (sensitive, labels, etc.) |
| `schema.irc.ts` | IRC-specific schema additions |
| `schema.labels.ts` | Human-readable labels for config paths |
| `zod-schema.ts` | **Main Zod schema definition** (~846 lines) |
| `zod-schema.agents.ts` | Agent config Zod schemas |
| `zod-schema.agent-defaults.ts` | Agent defaults Zod schema |
| `zod-schema.agent-runtime.ts` | Agent runtime (tools) Zod schema |
| `zod-schema.allowdeny.ts` | Allow/deny list schemas |
| `zod-schema.approvals.ts` | Approvals Zod schema |
| `zod-schema.channels.ts` | Channel config Zod schemas |
| `zod-schema.core.ts` | Core schemas (hex color, model config, etc.) |
| `zod-schema.hooks.ts` | Hooks Zod schema |
| `zod-schema.providers.ts` | Provider-specific schemas (all channels) |
| `zod-schema.providers-core.ts` | Core provider schemas |
| `zod-schema.providers-whatsapp.ts` | WhatsApp provider schema |
| `zod-schema.sensitive.ts` | Sensitive field marking |
| `zod-schema.session.ts` | Session config Zod schema |

#### Types (split by domain)
| File | Description |
|------|-------------|
| `types.ts` | Barrel re-export of all type files |
| `types.openclaw.ts` | **`OpenClawConfig`** — the root config type |
| `types.base.ts` | Base types: SessionConfig, LoggingConfig, IdentityConfig, GroupPolicy, DmPolicy, etc. |
| `types.agents.ts` | AgentConfig, AgentsConfig, AgentBinding |
| `types.agent-defaults.ts` | AgentDefaultsConfig (model, heartbeat, context pruning, etc.) |
| `types.approvals.ts` | ApprovalsConfig |
| `types.auth.ts` | AuthConfig, AuthProfileConfig |
| `types.browser.ts` | BrowserConfig |
| `types.channels.ts` | ChannelsConfig |
| `types.cron.ts` | CronConfig |
| `types.discord.ts` | DiscordConfig |
| `types.gateway.ts` | GatewayConfig (port, bind, auth, TLS, remote, reload, HTTP, nodes, tools) |
| `types.googlechat.ts` | GoogleChatConfig |
| `types.hooks.ts` | HooksConfig |
| `types.imessage.ts` | IMessageConfig |
| `types.irc.ts` | IrcConfig |
| `types.memory.ts` | MemoryConfig |
| `types.messages.ts` | MessagesConfig, BroadcastConfig, AudioConfig |
| `types.models.ts` | ModelsConfig, ModelProviderConfig, ModelDefinitionConfig |
| `types.msteams.ts` | MSTeamsConfig |
| `types.node-host.ts` | NodeHostConfig |
| `types.plugins.ts` | PluginsConfig |
| `types.queue.ts` | QueueConfig |
| `types.sandbox.ts` | SandboxConfig |
| `types.signal.ts` | SignalConfig |
| `types.skills.ts` | SkillsConfig |
| `types.slack.ts` | SlackConfig |
| `types.telegram.ts` | TelegramConfig |
| `types.tools.ts` | ToolsConfig (exec, fs, web, media, message, subagents, elevated) |
| `types.tts.ts` | TTS config |
| `types.whatsapp.ts` | WhatsAppConfig |

#### Config Processing
| File | Description |
|------|-------------|
| `env-substitution.ts` | `${ENV_VAR}` substitution in config values |
| `env-preserve.ts` | Preserve env var references when writing config |
| `env-vars.ts` | Apply `env.vars` inline env vars |
| `includes.ts` | `$include` directive resolution (JSON5 file merging) |
| `includes-scan.ts` | Scan for $include paths without resolving |
| `merge-config.ts` | Config merging utilities |
| `merge-patch.ts` | JSON merge-patch implementation |
| `normalize-paths.ts` | Normalize file paths in config |
| `runtime-overrides.ts` | Apply runtime env overrides to loaded config |
| `legacy-migrate.ts` | Legacy config migration entry |
| `legacy.ts` | Legacy config detection |
| `legacy.rules.ts` | Legacy config rules |
| `legacy.shared.ts` | Shared legacy utilities |
| `legacy.migrations.ts` | Migration registry |
| `legacy.migrations.part-1.ts` | Migration batch 1 |
| `legacy.migrations.part-2.ts` | Migration batch 2 |
| `legacy.migrations.part-3.ts` | Migration batch 3 |

#### Sessions
| File | Description |
|------|-------------|
| `sessions.ts` | Session store and management |
| `sessions/store.ts` | Session file store (JSONL) |
| `sessions/paths.ts` | Session file paths |
| `sessions/types.ts` | Session types |
| `sessions/metadata.ts` | Session metadata |
| `sessions/group.ts` | Group session helpers |
| `sessions/main-session.ts` | Main session key |
| `sessions/session-key.ts` | Session key parsing |
| `sessions/delivery-info.ts` | Delivery info tracking |
| `sessions/reset.ts` | Session reset logic |
| `sessions/transcript.ts` | Session transcript management |

#### Utilities
| File | Description |
|------|-------------|
| `agent-dirs.ts` | Agent directory validation (duplicate detection) |
| `agent-limits.ts` | Agent concurrency limits (DEFAULT_AGENT_MAX_CONCURRENT=4, DEFAULT_SUBAGENT_MAX_CONCURRENT=8) |
| `backup-rotation.ts` | Config backup rotation |
| `cache-utils.ts` | Config caching utilities |
| `channel-capabilities.ts` | Channel capability resolution |
| `commands.ts` | Custom command definitions |
| `group-policy.ts` | Group policy resolution |
| `logging.ts` | Config change logging |
| `markdown-tables.ts` | Markdown table utilities |
| `plugin-auto-enable.ts` | Auto-enable plugins |
| `port-defaults.ts` | Default port assignments |
| `redact-snapshot.ts` | Redact sensitive fields in config snapshots |
| `talk.ts` | Talk/TTS config helpers |
| `telegram-custom-commands.ts` | Telegram custom command definitions |
| `test-helpers.ts` | Test helper utilities |

### Configuration Schema (Root: `OpenClawConfig`)

```
openclaw.json
├── meta                          # Config metadata
│   ├── lastTouchedVersion        # string — last OpenClaw version
│   └── lastTouchedAt             # string — ISO timestamp
├── auth                          # Authentication
│   ├── profiles                  # Record<string, AuthProfileConfig>
│   ├── order                     # Record<string, string[]> — provider auth order
│   └── cooldowns                 # Billing backoff config
├── env                           # Environment variables
│   ├── shellEnv.enabled          # boolean — import from login shell
│   ├── shellEnv.timeoutMs        # number — default 15000
│   └── vars                      # Record<string, string>
├── wizard                        # Wizard state
├── diagnostics                   # Diagnostics/OTEL config
├── logging                       # Logging config (level, file, console, redaction)
├── update                        # Update channel (stable|beta|dev), checkOnStart
├── browser                       # Browser automation config
├── ui                            # UI chrome config (seamColor, assistant name/avatar)
├── skills                        # Skills config (install, entries)
├── plugins                       # Plugin config (enabled, allow/deny, load, slots, entries, installs)
├── models                        # Model providers config
│   ├── mode                      # "merge" | "replace"
│   ├── providers                 # Record<string, ModelProviderConfig>
│   └── bedrockDiscovery          # AWS Bedrock auto-discovery
├── nodeHost                      # Node host config (browserProxy)
├── agents                        # Multi-agent config
│   ├── defaults                  # AgentDefaultsConfig (model, heartbeat, context pruning, etc.)
│   └── list                      # AgentConfig[] (id, model, workspace, skills, identity, etc.)
├── tools                         # Tool configuration
│   ├── profile                   # "minimal"|"coding"|"messaging"|"full"
│   ├── allow/deny/alsoAllow      # Tool allowlists
│   ├── web.search                # Web search config (provider, apiKey, perplexity, grok)
│   ├── web.fetch                 # Web fetch config (maxChars, maxResponseBytes, firecrawl)
│   ├── media                     # Media understanding (image/audio/video)
│   ├── links                     # Link understanding
│   ├── message                   # Message tool config
│   ├── exec                      # Exec tool config (host, security, ask, pathPrepend, etc.)
│   ├── fs                        # Filesystem guards (workspaceOnly)
│   ├── elevated                  # Elevated exec permissions
│   └── subagents/sandbox         # Sub-agent/sandbox tool policies
├── bindings                      # AgentBinding[] — channel-to-agent bindings
├── broadcast                     # BroadcastConfig
├── audio                         # AudioConfig
├── messages                      # MessagesConfig (ack reactions, markdown, human delay, etc.)
├── commands                      # CommandsConfig (custom commands)
├── approvals                     # ApprovalsConfig
├── session                       # SessionConfig (scope, reset, maintenance, sendPolicy, etc.)
├── web                           # WebConfig (WhatsApp web)
├── channels                      # ChannelsConfig
│   ├── defaults                  # Channel defaults (groupPolicy, heartbeat)
│   ├── telegram                  # TelegramConfig
│   ├── discord                   # DiscordConfig
│   ├── slack                     # SlackConfig
│   ├── whatsapp                  # WhatsAppConfig
│   ├── signal                    # SignalConfig
│   ├── imessage                  # IMessageConfig
│   ├── irc                       # IrcConfig
│   ├── googlechat                # GoogleChatConfig
│   ├── msteams                   # MSTeamsConfig
│   └── [extension]               # Dynamic extension channels
├── cron                          # CronConfig (jobs, timezone)
├── hooks                         # HooksConfig (event hooks)
├── discovery                     # DiscoveryConfig (mDNS, wide-area DNS)
├── canvasHost                    # CanvasHostConfig (enabled, root, port, liveReload)
├── talk                          # TalkConfig (provider-based voice / silence-timeout controls)
├── gateway                       # GatewayConfig
│   ├── port                      # number — default 18789
│   ├── mode                      # "local" | "remote"
│   ├── bind                      # "auto"|"lan"|"loopback"|"tailnet"|"custom"
│   ├── customBindHost            # string
│   ├── controlUi                 # enabled, basePath, allowedOrigins, auth settings
│   ├── auth                      # mode (token|password|trusted-proxy), token, password, rateLimit
│   ├── tailscale                 # mode (off|serve|funnel), resetOnExit
│   ├── remote                    # url, transport, token, password, tlsFingerprint, ssh
│   ├── reload                    # mode (off|restart|hot|hybrid), debounceMs
│   ├── tls                       # enabled, autoGenerate, certPath, keyPath, caPath
│   ├── http                      # endpoints (chatCompletions, responses)
│   ├── nodes                     # browser routing, allowCommands, denyCommands
│   ├── trustedProxies            # string[]
│   └── tools                     # deny/allow for HTTP /tools/invoke
└── memory                        # MemoryConfig (QMD, search)
```

### Key Functions

```typescript
// io.ts
loadConfig(): OpenClawConfig                          // Load, validate, apply defaults
readConfigFileSnapshot(): ConfigFileSnapshot          // Read raw + parsed + validated
readConfigFileSnapshotForWrite(): ConfigFileSnapshot  // For config modifications
writeConfigFile(config, options?): void               // Write with audit trail
parseConfigJson5(raw: string): unknown                // Parse JSON5 string

// validation.ts
validateConfigObject(parsed): { config, issues, warnings }
validateConfigObjectWithPlugins(parsed, plugins): { config, issues, warnings }

// defaults.ts
applyModelDefaults(cfg): OpenClawConfig
applyAgentDefaults(cfg): OpenClawConfig
applySessionDefaults(cfg): OpenClawConfig
applyContextPruningDefaults(cfg): OpenClawConfig
applyCompactionDefaults(cfg): OpenClawConfig
applyLoggingDefaults(cfg): OpenClawConfig

// paths.ts
resolveStateDir(env?, homedir?): string               // ~/.openclaw (or legacy)
resolveConfigPath(env?, stateDir?, homedir?): string   // openclaw.json path
resolveGatewayPort(cfg?, env?): number                 // Default: 18789
resolveOAuthDir(env?, stateDir?): string               // credentials directory

// env-substitution.ts
resolveConfigEnvVars(config): OpenClawConfig           // ${ENV_VAR} resolution

// includes.ts
resolveConfigIncludes(config, basePath): OpenClawConfig // $include merging
```

### Config Loading Pipeline

```
1. Resolve config file path (resolveConfigPath)
2. Read raw file content
3. Parse JSON5
4. Resolve $include directives (recursive, circular detection)
5. Substitute ${ENV_VAR} references
6. Validate against Zod schema
7. Detect legacy config issues
8. Apply defaults (models, agents, sessions, logging, pruning, compaction)
9. Apply runtime overrides (env vars)
10. Normalize paths
11. Cache result
```

### Internal Dependencies
- `src/channels/registry.ts` — CHANNEL_IDS
- `src/channels/chat-type.ts` — ChatType enum
- `src/agents/defaults.ts` — DEFAULT_CONTEXT_TOKENS
- `src/agents/model-selection.ts` — parseModelRef
- `src/infra/dotenv.ts` — loadDotEnv
- `src/infra/home-dir.ts` — resolveRequiredHomeDir
- `src/infra/shell-env.ts` — shell env fallback
- `src/version.ts` — VERSION
- `src/logging/subsystem.ts` — logging
- `src/utils.ts` — resolveUserPath

### External Dependencies
- `zod` — schema validation
- `json5` — config file parsing

### Test Coverage
| Test File | Covers |
|-----------|--------|
| `config.*.test.ts` (30+ files) | Broad schema validation: agents, broadcast, discord, env vars, gateway, hooks, identity, IRC, models, MS Teams, plugins, pruning, sandbox, skills, Telegram, tools, TTS, web search |
| `config-paths.test.ts` | Config path resolution |
| `agent-dirs.test.ts` | Duplicate agent directory detection |
| `channel-capabilities.test.ts` | Channel capability resolution |
| `commands.test.ts` | Custom command parsing |
| `env-preserve.test.ts` | Env var preservation |
| `env-preserve-io.test.ts` | Env var IO round-trip |
| `env-substitution.test.ts` | ${ENV} substitution |
| `includes.test.ts` | $include resolution |
| `io.compat.test.ts` | IO backward compatibility |
| `io.write-config.test.ts` | Config writing |
| `merge-patch.test.ts` | JSON merge-patch |
| `normalize-paths.test.ts` | Path normalization |
| `paths.test.ts` | State/config path resolution |
| `plugin-auto-enable.test.ts` | Plugin auto-enable |
| `redact-snapshot.test.ts` | Sensitive field redaction |
| `runtime-overrides.test.ts` | Runtime override application |
| `schema.test.ts` | JSON Schema generation |
| `schema.hints.test.ts` | UI hints |
| `sessions.test.ts` | Session management |
| `sessions.cache.test.ts` | Session caching |
| `sessions/*.test.ts` | Session store, paths, metadata, reset, transcript, locks, pruning |
| Various E2E tests | Legacy migration, Nix integration, config detection |

---

## 4. src/infra

### Module Overview

The infra module provides **cross-cutting infrastructure utilities** used throughout OpenClaw: environment management, networking, process management, file operations, service discovery, outbound message delivery, provider usage tracking, update checking, and more.

**Architecture Pattern:** Utility library with functional modules. No central entry point — each file is imported independently.

### File Inventory (327 files, grouped by domain)

#### Environment & Runtime
| File | Description |
|------|-------------|
| `env.ts` | Env var normalization, truthy checking, logging |
| `dotenv.ts` | `.env` file loading |
| `env-file.ts` | Environment file utilities |
| `shell-env.ts` | Login shell environment fallback |
| `home-dir.ts` | Home directory resolution (OPENCLAW_HOME, HOME, USERPROFILE) |
| `openclaw-root.ts` | OpenClaw installation root |
| `install-package-dir.ts` | Package installation directory |
| `install-safe-path.ts` | Safe installation path resolution |
| `machine-name.ts` | Machine hostname |
| `os-summary.ts` | OS summary string |
| `is-main.ts` | Check if current module is the entry point |
| `runtime-guard.ts` | Assert minimum Node.js version |
| `path-env.ts` | Ensure OpenClaw CLI is on PATH |
| `path-prepend.ts` | PATH prepend utility |
| `wsl.ts` | WSL (Windows Subsystem for Linux) detection |

#### Networking
| File | Description |
|------|-------------|
| `fetch.ts` | Enhanced fetch with timeout, retry, logging |
| `ws.ts` | WebSocket utilities |
| `ports.ts` | Port availability checking |
| `ports-inspect.ts` | Port inspection (what's listening) |
| `ports-lsof.ts` | lsof-based port inspection |
| `ports-format.ts` | Port info formatting |
| `ports-types.ts` | Port types |
| `tailscale.ts` | Tailscale integration |
| `tailnet.ts` | Tailnet utilities |
| `ssh-config.ts` | SSH config parsing |
| `ssh-tunnel.ts` | SSH tunnel management |
| `net/ssrf.ts` | SSRF (Server-Side Request Forgery) protection |
| `net/fetch-guard.ts` | Fetch guard with SSRF checks |
| `tls/fingerprint.ts` | TLS certificate fingerprinting |
| `tls/gateway.ts` | Gateway TLS utilities |
| `widearea-dns.ts` | Wide-area DNS-SD discovery |

#### Service Discovery
| File | Description |
|------|-------------|
| `bonjour.ts` | mDNS/Bonjour service advertisement |
| `bonjour-discovery.ts` | mDNS service discovery |
| `bonjour-ciao.ts` | Ciao mDNS library wrapper |
| `bonjour-errors.ts` | Bonjour error handling |

#### Process Management
| File | Description |
|------|-------------|
| `process-respawn.ts` | Process respawn with backoff |
| `restart.ts` | Process restart utilities |
| `restart-sentinel.ts` | Restart sentinel file watching |
| `gateway-lock.ts` | Gateway PID lock file |
| `file-lock.ts` | Generic file locking |
| `json-file.ts` | Atomic JSON file read/write |
| `jsonl-socket.ts` | JSONL-over-socket communication |
| `node-shell.ts` | Node.js shell execution |
| `tmp-openclaw-dir.ts` | Temp directory management |

#### Outbound Messaging (`outbound/`)
| File | Description |
|------|-------------|
| `outbound/message.ts` | Outbound message sending |
| `outbound/deliver.ts` | Message delivery orchestration |
| `outbound/delivery-queue.ts` | Delivery queue management |
| `outbound/envelope.ts` | Message envelope construction |
| `outbound/format.ts` | Message formatting for channels |
| `outbound/identity.ts` | Sender identity |
| `outbound/payloads.ts` | Message payloads |
| `outbound/targets.ts` | Target resolution |
| `outbound/target-resolver.ts` | Resolve target channels/users |
| `outbound/target-normalization.ts` | Target normalization |
| `outbound/target-errors.ts` | Target resolution errors |
| `outbound/tool-payload.ts` | Tool-generated message payloads |
| `outbound/channel-adapters.ts` | Channel-specific message adapters |
| `outbound/channel-selection.ts` | Channel selection logic |
| `outbound/channel-target.ts` | Channel target types |
| `outbound/agent-delivery.ts` | Agent-to-agent delivery |
| `outbound/outbound-policy.ts` | Outbound send policy enforcement |
| `outbound/outbound-session.ts` | Outbound session management |
| `outbound/outbound-send-service.ts` | Outbound send service |
| `outbound/message-action-params.ts` | Message action parameters |
| `outbound/message-action-runner.ts` | Message action execution |
| `outbound/message-action-spec.ts` | Message action specifications |
| `outbound/directory-cache.ts` | Directory cache for target resolution |
| `outbound/abort.ts` | Abort signal utilities |

#### Heartbeat & System Events
| File | Description |
|------|-------------|
| `heartbeat-runner.ts` | Heartbeat scheduler and runner |
| `heartbeat-active-hours.ts` | Active hours filtering |
| `heartbeat-events.ts` | Heartbeat event types |
| `heartbeat-events-filter.ts` | Heartbeat event filtering |
| `heartbeat-visibility.ts` | Heartbeat visibility settings |
| `heartbeat-wake.ts` | Wake-on-heartbeat logic |
| `system-events.ts` | System event emission |
| `system-presence.ts` | System presence tracking |
| `system-run-command.ts` | System command execution |
| `diagnostic-events.ts` | Diagnostic event emission |
| `diagnostic-flags.ts` | Diagnostic flag checking |

#### Exec Approvals
| File | Description |
|------|-------------|
| `exec-approvals.ts` | Exec approval logic |
| `exec-approvals-allowlist.ts` | Exec allowlist management |
| `exec-approvals-analysis.ts` | Command analysis for approvals |
| `exec-approval-forwarder.ts` | Forward approvals to channels |
| `exec-host.ts` | Exec host routing |
| `exec-safety.ts` | Exec safety checks |

#### Device & Node Pairing
| File | Description |
|------|-------------|
| `device-identity.ts` | Device identity management |
| `device-auth-store.ts` | Device auth token store |
| `device-pairing.ts` | Device pairing flow |
| `node-pairing.ts` | Node pairing management |
| `pairing-files.ts` | Pairing file management |
| `pairing-token.ts` | Pairing token generation |

#### Updates
| File | Description |
|------|-------------|
| `update-check.ts` | Check for new versions |
| `update-runner.ts` | Run update process |
| `update-startup.ts` | Startup update check |
| `update-global.ts` | Global npm update |
| `update-channels.ts` | Update channel management |

#### Provider Usage & Cost Tracking
| File | Description |
|------|-------------|
| `provider-usage.ts` | Provider usage aggregation |
| `provider-usage.auth.ts` | Provider auth normalization |
| `provider-usage.fetch.ts` | Fetch usage from providers |
| `provider-usage.fetch.claude.ts` | Anthropic usage |
| `provider-usage.fetch.gemini.ts` | Google Gemini usage |
| `provider-usage.fetch.codex.ts` | OpenAI Codex usage |
| `provider-usage.fetch.copilot.ts` | GitHub Copilot usage |
| `provider-usage.fetch.minimax.ts` | MiniMax usage |
| `provider-usage.fetch.zai.ts` | xAI usage |
| `provider-usage.fetch.shared.ts` | Shared usage fetch |
| `provider-usage.format.ts` | Usage formatting |
| `provider-usage.load.ts` | Usage data loading |
| `provider-usage.shared.ts` | Shared usage utilities |
| `provider-usage.types.ts` | Usage types |
| `session-cost-usage.ts` | Session cost tracking |
| `session-cost-usage.types.ts` | Session cost types |

#### Miscellaneous
| File | Description |
|------|-------------|
| `agent-events.ts` | Agent lifecycle events |
| `archive.ts` | Archive/zip utilities |
| `backoff.ts` | Exponential backoff |
| `binaries.ts` | Binary dependency resolution |
| `brew.ts` | Homebrew integration |
| `canvas-host-url.ts` | Canvas host URL resolution |
| `channel-activity.ts` | Channel activity tracking |
| `channel-summary.ts` | Channel summary generation |
| `channels-status-issues.ts` | Channel status issue detection |
| `clipboard.ts` | Clipboard access |
| `control-ui-assets.ts` | Control UI static asset serving |
| `dedupe.ts` | Deduplication utilities |
| `detect-package-manager.ts` | npm/pnpm/yarn/bun detection |
| `errors.ts` | Error formatting and type guards |
| `fs-safe.ts` | Safe filesystem operations |
| `git-commit.ts` | Git commit utilities |
| `http-body.ts` | HTTP body parsing |
| `npm-registry-spec.ts` | npm registry specification |
| `retry.ts` | Retry logic |
| `retry-policy.ts` | Retry policy definition |
| `session-maintenance-warning.ts` | Session maintenance warnings |
| `skills-remote.ts` | Remote skills fetching |
| `state-migrations.ts` | State directory migrations |
| `state-migrations.fs.ts` | State migration filesystem ops |
| `transport-ready.ts` | Transport readiness checks |
| `unhandled-rejections.ts` | Unhandled rejection handler |
| `voicewake.ts` | Voice wake detection |
| `warning-filter.ts` | Warning message filtering |

### Key Types

```typescript
// outbound/
type DeliveryEnvelope = { target, message, options, ... }
type OutboundPolicy = { default: "allow"|"deny", rules: PolicyRule[] }

// heartbeat
type HeartbeatEvent = { type, agent, timestamp, ... }

// provider-usage.types.ts
type ProviderUsageRecord = { provider, model, tokens, cost, ... }

// session-cost-usage.types.ts
type SessionCostEntry = { model, inputTokens, outputTokens, cost, ... }

// exec-approvals.ts
type ExecApprovalRequest = { command, args, cwd, agent, ... }
type ExecApprovalResult = "approved" | "denied" | "timeout"
```

### External Dependencies
- `ciao` — mDNS/Bonjour (optional)
- `@anthropic-ai/sdk` — Anthropic API (transitive)
- Various provider SDKs (transitive via fetch)

### Test Coverage
30+ test files covering: abort patterns, agent events, archive, bonjour, brew, control UI assets, device identity/pairing, dotenv, env, exec approvals, fetch, format-time, gateway lock, heartbeat (6 test files), home-dir, HTTP body, node pairing, outbound (8 test files), path-env, ports, process respawn, provider usage, restart sentinel, retry, runtime guard, session cost, shell-env, skills-remote, SSH config, SSRF, state migrations, system events/presence/run-command, tailscale, TLS fingerprint, tmp dir, transport-ready, unhandled rejections, update check/runner/startup, warning filter, wide-area DNS.

---

## 5. src/daemon

### Module Overview

The daemon module manages **system service installation** for the OpenClaw gateway across macOS (launchd), Linux (systemd), and Windows (schtasks). It generates platform-specific service definitions and provides install/uninstall/start/stop/restart operations.

**Architecture Pattern:** Platform abstraction via `GatewayService` interface with platform-specific implementations.

### File Inventory (42 files)

| File | Description |
|------|-------------|
| `service.ts` | **`GatewayService` interface** and platform dispatcher |
| `launchd.ts` | macOS launchd implementation (install/uninstall/start/stop/restart) |
| `launchd-plist.ts` | Generate launchd plist XML |
| `systemd.ts` | Linux systemd implementation |
| `systemd-unit.ts` | Generate systemd unit files |
| `systemd-linger.ts` | Enable systemd user linger for headless operation |
| `systemd-hints.ts` | Systemd usage hints/guidance |
| `schtasks.ts` | Windows Task Scheduler implementation |
| `schtasks-exec.ts` | schtasks execution helpers |
| `constants.ts` | Service labels, names, descriptions |
| `paths.ts` | Service-related path resolution (state dir, home dir) |
| `runtime-paths.ts` | Runtime paths for service executables |
| `runtime-format.ts` | Runtime info formatting |
| `runtime-parse.ts` | Parse service runtime output |
| `program-args.ts` | Build gateway program arguments |
| `arg-split.ts` | Argument splitting utilities |
| `node-service.ts` | Node-specific service management |
| `service-runtime.ts` | `GatewayServiceRuntime` type |
| `service-env.ts` | Service environment variable management |
| `service-audit.ts` | Service audit logging |
| `exec-file.ts` | Child process execution helper |
| `output.ts` | Output formatting (formatLine, toPosixPath) |
| `inspect.ts` | Service inspection utilities |
| `diagnostics.ts` | Service diagnostics |

### Key Types

```typescript
type GatewayService = {
  label: string;
  loadedText: string;
  notLoadedText: string;
  install(args: GatewayServiceInstallArgs): Promise<void>;
  uninstall(args): Promise<void>;
  stop(args): Promise<void>;
  restart(args): Promise<void>;
  isLoaded(args): Promise<boolean>;
  readRuntime(args): Promise<GatewayServiceRuntime | null>;
  readProgramArguments(args): Promise<string[] | null>;
};

type GatewayServiceInstallArgs = {
  env: Record<string, string | undefined>;
  stdout: NodeJS.WritableStream;
  programArguments: string[];
  workingDirectory?: string;
  environment?: Record<string, string | undefined>;
  description?: string;
};

type GatewayServiceRuntime = {
  pid?: number;
  port?: number;
  version?: string;
  uptime?: string;
  // ... platform-specific fields
};
```

### Platform Support
- **macOS**: `~/Library/LaunchAgents/com.openclaw.gateway.plist`
- **Linux**: `~/.config/systemd/user/openclaw-gateway.service`
- **Windows**: Windows Task Scheduler task

### Internal Dependencies
- `src/config/paths.ts` — state directory
- `src/infra/home-dir.ts` — home directory

### Test Coverage
| Test File | Covers |
|-----------|--------|
| `arg-split.test.ts` | Argument splitting |
| `constants.test.ts` | Service name resolution |
| `launchd.test.ts` | macOS launchd operations |
| `paths.test.ts` | Service path resolution |
| `program-args.test.ts` | Program argument building |
| `runtime-paths.test.ts` | Runtime path resolution |
| `schtasks.test.ts` | Windows schtasks |
| `service-audit.test.ts` | Audit logging |
| `service-env.test.ts` | Environment management |
| `systemd.test.ts` | Linux systemd |
| `systemd-availability.test.ts` | Systemd detection |
| `systemd-unit.test.ts` | Unit file generation |

---

## 6. src/wizard

### Module Overview

The wizard module implements the **interactive onboarding wizard** flow that guides users through initial OpenClaw setup: provider selection, auth configuration, gateway setup, and channel configuration.

**Architecture Pattern:** Step-based wizard flow with a `WizardPrompter` abstraction for testability.

### File Inventory (13 files)

| File | Description |
|------|-------------|
| `setup.ts` | Main wizard orchestration |
| `setup.types.ts` | Wizard flow types |
| `setup.finalize.ts` | Finalization step (write config, install daemon) |
| `setup.gateway-config.ts` | Gateway configuration step |
| `setup.completion.ts` | Completion message/summary |
| `prompts.ts` | `WizardPrompter` interface + `WizardCancelledError` |
| `clack-prompter.ts` | Clack-based terminal prompter implementation |
| `session.ts` | Wizard session state management |

### Key Types

```typescript
type WizardFlow = {
  mode: OnboardMode;        // "local" | "remote"
  authChoice: GatewayAuthChoice;
  config: OpenClawConfig;
  // ...
};

type WizardPrompter = {
  text(params): Promise<string>;
  confirm(params): Promise<boolean>;
  select<T>(params): Promise<T>;
  multiselect<T>(params): Promise<T[]>;
  note(title, message): void;
  // ...
};

type QuickstartGatewayDefaults = {
  port: number;
  bind: GatewayBindMode;
  // ...
};
```

### Internal Dependencies
- `src/commands/onboard-*.ts` — onboarding step implementations
- `src/commands/auth-choice*.ts` — auth selection
- `src/commands/model-picker.ts` — model selection
- `src/config` — config loading/writing
- `src/agents/auth-profiles.ts` — auth profile management

### Test Coverage
| Test File | Covers |
|-----------|--------|
| `setup.test.ts` | Main wizard flow |
| `setup.completion.test.ts` | Completion output |
| `setup.gateway-config.test.ts` | Gateway config step |
| `session.test.ts` | Wizard session state |

---

## 7. src/tui

### Module Overview

The TUI module implements a **terminal-based chat interface** for interacting with OpenClaw agents. Built on `@mariozechner/pi-tui`, it provides a rich terminal UI with message history, syntax highlighting, command handling, and real-time streaming.

**Architecture Pattern:** Component-based TUI with event handlers, a gateway WebSocket chat client, and reactive state.

### File Inventory (45 files)

| File | Description |
|------|-------------|
| `tui.ts` | Main TUI entry: `runTui(opts: TuiOptions)`, editor submit handler |
| `tui-submit.ts` | `createEditorSubmitHandler()` — editor submit handling |
| `osc8-hyperlinks.ts` | OSC-8 terminal hyperlink support |
| `tui-types.ts` | TUI types (TuiOptions, SessionInfo, AgentSummary, etc.) |
| `gateway-chat.ts` | `GatewayChatClient` — WebSocket client for gateway |
| `commands.ts` | Slash command definitions (`/help`, `/clear`, `/agent`, `/model`, etc.) |
| `tui-command-handlers.ts` | Command execution (switch agent, model, clear, etc.) |
| `tui-event-handlers.ts` | WebSocket event handling (message, tool, typing, error) |
| `tui-formatters.ts` | Message and token formatting |
| `tui-local-shell.ts` | Local shell execution from TUI |
| `tui-overlays.ts` | Overlay screens (agent picker, model picker) |
| `tui-session-actions.ts` | Session management actions |
| `tui-status-summary.ts` | Status bar summary |
| `tui-stream-assembler.ts` | Streaming response assembly |
| `tui-waiting.ts` | Waiting/thinking status messages |
| `components/assistant-message.ts` | Assistant message rendering component |
| `components/chat-log.ts` | Chat log component |
| `components/custom-editor.ts` | Input editor component |
| `components/filterable-select-list.ts` | Filterable selection list |
| `components/fuzzy-filter.ts` | Fuzzy text filter |
| `components/searchable-select-list.ts` | Searchable select list |
| `components/selectors.ts` | Selection helpers |
| `components/tool-execution.ts` | Tool execution display component |
| `components/user-message.ts` | User message rendering component |
| `components/hyperlink-markdown.ts` | Hyperlink-aware markdown rendering |
| `components/markdown-message.ts` | Markdown message component |
| `components/btw-inline-message.ts` | BTW inline message component |
| `theme/theme.ts` | TUI color theme |
| `theme/syntax-theme.ts` | Syntax highlighting theme |

### Key Types

```typescript
type TuiOptions = {
  gatewayUrl: string;
  token?: string;
  agent?: string;
  session?: string;
  model?: string;
  // ...
};

type TuiStateAccess = {
  getAgent(): string;
  setAgent(id: string): void;
  getSession(): string;
  getSessions(): SessionInfo[];
  // ...
};
```

### Internal Dependencies
- `src/config` — config loading
- `src/agents/agent-scope.ts` — agent resolution
- `src/routing/session-key.ts` — session key parsing

### External Dependencies
- `@mariozechner/pi-tui` — terminal UI framework (Container, TUI, Text, Loader, ProcessTerminal, etc.)

### Test Coverage
| Test File | Covers |
|-----------|--------|
| `tui.test.ts` | TUI initialization |
| `tui.submit-handler.test.ts` | Editor submit handler |
| `commands.test.ts` | Slash commands |
| `gateway-chat.test.ts` | Gateway chat client |
| `tui-command-handlers.test.ts` | Command handlers |
| `tui-event-handlers.test.ts` | Event handlers |
| `tui-formatters.test.ts` | Message formatting |
| `tui-input-history.test.ts` | Input history |
| `tui-local-shell.test.ts` | Local shell |
| `tui-overlays.test.ts` | Overlay screens |
| `tui-session-actions.test.ts` | Session actions |
| `tui-stream-assembler.test.ts` | Stream assembly |
| `tui-waiting.test.ts` | Waiting messages |
| `components/searchable-select-list.test.ts` | Searchable select |
| `theme/theme.test.ts` | Theme |

---

## 8. src/terminal

### Module Overview

The terminal module provides **low-level terminal rendering primitives**: ANSI handling, color palettes, table formatting, progress lines, and themed output helpers. Used by the CLI and TUI for consistent terminal output.

**Architecture Pattern:** Pure utility functions for terminal output.

### File Inventory (16 files)

| File | Description |
|------|-------------|
| `ansi.ts` | ANSI escape code stripping, visible width calculation |
| `palette.ts` | Color palette (chalk-based color definitions) |
| `theme.ts` | Themed output helpers (success, warning, error, info colors) |
| `table.ts` | Terminal table rendering (column alignment, padding) |
| `links.ts` | Terminal hyperlink (OSC-8) generation |
| `note.ts` | Boxed note rendering |
| `progress-line.ts` | Progress line with spinner |
| `prompt-style.ts` | Prompt styling helpers |
| `health-style.ts` | Health status coloring |
| `stream-writer.ts` | Buffered stream writer |
| `prompt-select-styled.ts` | Styled prompt selection helpers |
| `safe-text.ts` | Safe text rendering utilities |
| `restore.ts` | Terminal state restoration (cursor, alternate screen) |

### Key Functions

```typescript
// ansi.ts
stripAnsi(input: string): string
visibleWidth(input: string): number

// table.ts
renderTable(rows: string[][], options?): string

// links.ts
hyperlink(text: string, url: string): string

// stream-writer.ts
createStreamWriter(stream): StreamWriter
```

### Internal Dependencies
- None (leaf module)

### External Dependencies
- `chalk` — terminal colors (via palette)

### Test Coverage
| Test File | Covers |
|-----------|--------|
| `restore.test.ts` | Terminal restoration |
| `stream-writer.test.ts` | Stream writer |
| `table.test.ts` | Table rendering |

---

## Cross-Module Data Flow Summary

```
User types: openclaw <command> [args]
         │
         ▼
   src/cli/run-main.ts
   ├── Load .env, normalize env
   ├── Fast-path route? ──yes──► src/cli/program/routes.ts ──► src/commands/*.ts
   └── No → Commander path
       ├── src/cli/program/build-program.ts
       │   ├── Create Commander program
       │   ├── Register lazy command placeholders
       │   └── parseAsync(argv)
       │       └── Lazy load triggered
       │           ├── src/cli/*-cli.ts (registrar)
       │           └── src/commands/*.ts (implementation)
       │               ├── src/config/ (load/write config)
       │               ├── src/infra/ (networking, process, events)
       │               ├── src/daemon/ (service management)
       │               └── src/terminal/ (output formatting)
       └── Output to stdout/stderr
```

---

## Environment Variables Reference

| Variable | Purpose | Default |
|----------|---------|---------|
| `OPENCLAW_HOME` | Override home directory | `$HOME` |
| `OPENCLAW_STATE_DIR` | Override state directory | `~/.openclaw` |
| `OPENCLAW_CONFIG_PATH` | Override config file path | `$STATE_DIR/openclaw.json` |
| `OPENCLAW_GATEWAY_PORT` | Override gateway port | `18789` |
| `OPENCLAW_GATEWAY_TOKEN` | Gateway auth token | — |
| `OPENCLAW_GATEWAY_PASSWORD` | Gateway auth password | — |
| `OPENCLAW_PROFILE` | CLI profile name | — |
| `OPENCLAW_OAUTH_DIR` | OAuth credentials directory | `$STATE_DIR/credentials` |
| `OPENCLAW_NIX_MODE` | Nix integration mode | — |
| `OPENCLAW_SHELL` | Shell runtime marker (exec, acp, acp-client, tui-local) | — |
| `OPENCLAW_DISABLE_LAZY_SUBCOMMANDS` | Force eager command loading | — |
| `OPENCLAW_DISABLE_ROUTE_FIRST` | Disable fast-path routing | — |
| `OPENCLAW_LAUNCHD_LABEL` | Override macOS service label | `com.openclaw.gateway` |
| `OPENCLAW_SYSTEMD_UNIT` | Override Linux service name | `openclaw-gateway` |
| `OPENCLAW_LOG_PREFIX` | Log file prefix | `gateway` |
| `ANTHROPIC_API_KEY` | Anthropic API key | — |
| `OPENAI_API_KEY` | OpenAI API key | — |
| `GEMINI_API_KEY` | Google Gemini API key | — |
| `XAI_API_KEY` | xAI API key | — |
| `BRAVE_API_KEY` | Brave Search API key | — |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token | — |
| `DISCORD_BOT_TOKEN` | Discord bot token | — |
| `ELEVENLABS_API_KEY` | ElevenLabs TTS API key | — |
| `OPENCLAW_TZ` | Pin timezone inside Docker container (`docker-setup.sh`); must be a valid IANA timezone string (e.g. `Asia/Shanghai`) | — |

---

*Analysis complete. 1,228 TypeScript files across 8 modules analyzed.*

---

## v2026.2.15 Changes

### CLI
- **Account selector for pairing commands**: `nodes-cli/register.pairing.ts` — pairing commands now support selecting which account to pair with
- **Share configure section runner**: `configure.shared.ts` — extracted shared configure section runner to reduce duplication across `configure.channels.ts`, `configure.daemon.ts`, `configure.gateway.ts`
- **Dedupe daemon install**: `daemon-cli/install.ts` + `daemon-install-helpers.ts` — consolidated duplicate daemon installation logic
- **Share browser resize**: `browser-cli-shared.ts` — shared browser resize logic across browser CLI subcommands

### Config
- **Share agent model/sandbox schemas**: `zod-schema.agents.ts`, `zod-schema.agent-defaults.ts` — extracted shared Zod schemas for agent model and sandbox config to avoid duplication between agent-level and defaults-level schemas
- **Dedupe bindings migrations**: `legacy.migrations.part-3.ts` — consolidated duplicate bindings migration rules
- **Merge config.patch object arrays by id**: `merge-patch.ts` — `config.patch` now merges object arrays by `id` field instead of replacing, enabling incremental agent/binding patches

### Infra
- **Extract json file + async lock helpers**: `json-file.ts` (atomic JSON read/write), `json-files.ts` (multi-file JSON operations) — extracted from inline implementations across pairing, device-auth, and session stores
- **Share jsonl transcript reader**: Consolidated JSONL transcript reading logic (previously duplicated in session store and memory sync)
- **Dedupe device pairing token updates**: `device-pairing.ts`, `pairing-token.ts` — removed duplicate token update logic
- **Centralize openclaw root candidate scan**: `openclaw-root.ts` — unified OpenClaw installation root detection across CLI, daemon, and node-host
- **Share `isTailnetIPv4`**: `tailnet.ts` — exported shared Tailnet IPv4 detection (100.x.y.z range) used by networking, bind resolution, and node matching
- **Replace deprecated SHA-1 in sandbox config hash**: `sandbox/config-hash.ts` — replaced SHA-1 with SHA-256 for sandbox configuration change detection

## v2026.2.19 Changes

### CLI
- **Plugin uninstall** — Full plugin uninstall support added to CLI lifecycle
- **Security audit enhancements** — `openclaw security audit` now emits `gateway.http.no_auth` findings with loopback/remote severity levels

### Config
- **Gateway auth defaults** — `gateway.auth.mode` defaults to `"token"` (was implicit open); `gateway.auth.token` auto-generated and persisted on first start. See DEVELOPER-REFERENCE.md §6 for config reference
- **hooks.token ≠ gateway.auth.token** — Startup validation rejects matching tokens
- **browser.ssrfPolicy** — New config key controls SSRF behavior for browser URL navigation
- **YAML 1.2 core schema** — Frontmatter parsing uses YAML 1.2 core; no implicit `on`/`off` boolean coercion. See DEVELOPER-REFERENCE.md §9 (gotcha 37)
- **Plugin install records** — Include `name`, `version`, `spec`, integrity, shasum (`--pin` flag for npm plugins)

### Infra
- **Rate-limited control-plane RPCs** — `config.apply`, `config.patch`, `update.run` limited to 3/min per device+IP; config change audit logging (actor, device, IP, changed paths)
- **macOS LaunchAgent TMPDIR fix** — `TMPDIR` forwarded to service environment, resolving SQLite `SQLITE_CANTOPEN` failures
- **Windows daemon cmd injection** — Hardened Windows daemon service commands against command injection
- **Exec preflight guard** — Detects shell env var injection patterns in Python/Node scripts before execution. See DEVELOPER-REFERENCE.md §9 (gotcha 40)

## v2026.2.21 Changes <!-- v2026.2.21 -->

### CLI

- **`openclaw update --dry-run`** — New `--dry-run` flag for `openclaw update`. Previews the full update plan (channel, resolved tag/spec, install kind, planned actions, restart decision) without mutating config, running the package manager, syncing plugins, or restarting the gateway. Output includes a structured summary of what *would* happen; `--json` mode emits machine-readable JSON. File: `src/cli/update-cli/update-command.ts`. <!-- v2026.2.21 -->

  ```
  openclaw update --dry-run
  openclaw update --dry-run --channel beta --json
  ```

  Dry-run output fields: `root`, `installKind`, `mode`, `effectiveChannel`, `tag`, `currentVersion`, `targetVersion`, `downgradeRisk`, `actions[]`, `notes[]`.

- **`food-order` skill removed from bundle** — The bundled `food-order` skill is no longer shipped in this repository. Install and manage it from ClawHub instead. Not a breaking change for users who did not use the bundled skill. <!-- v2026.2.21 -->

### Config <!-- v2026.2.21 -->

- **Auto-updater config block (`update.auto.*`)** — New optional built-in auto-updater for package installs. Disabled by default (`update.auto.enabled = false`). When enabled: stable channel uses a rollout delay plus per-device jitter to avoid thundering-herd; beta channel checks hourly. Config keys added to the `update` section: <!-- v2026.2.21 -->

  | Key | Type | Default | Description |
  |-----|------|---------|-------------|
  | `update.auto.enabled` | `boolean` | `false` | Enable/disable background auto-updates |
  | `update.auto.channel` | `"stable" \| "beta"` | inherits `update.channel` | Channel for auto-updates |

  The existing `update.channel` and `update.checkOnStart` keys are unchanged.

- **Per-account/channel `channels.defaultTo` routing** — New `channels.defaultTo` config key. Provides a per-account/channel outbound routing fallback used by `openclaw agent --deliver` when no explicit `--reply-to` target is given. Configured in the channels config block alongside existing `defaults`. <!-- v2026.2.21 -->

- **`channels.modelByChannel`** — New config key for per-channel model overrides. Allows specifying a different model for messages arriving on specific channels without changing the global agent model. Lives in the `channels` config hierarchy. <!-- v2026.2.21 -->

- **Plugin version check prerelease handling** — `fix`: plugin version comparisons in the release-check path now correctly strip prerelease suffixes (e.g., `-beta.1`) before comparing versions. Prevents false-positive "update available" results when running a prerelease build. <!-- v2026.2.21 -->

### Infra <!-- v2026.2.21 -->

- **Update restart convergence hardened** — `fix`: the update flow's post-restart convergence step is now more robust against race conditions during gateway service restart. Stale gateway PIDs are detected and cleaned up before a second restart attempt is made. Relevant code path: `maybeRestartService()` in `src/cli/update-cli/update-command.ts`. <!-- v2026.2.21 -->

## v2026.2.22 Changes <!-- v2026.2.22 -->

### CLI <!-- v2026.2.22 -->

- **`openclaw update --dry-run` (update)** — `openclaw update --dry-run` previews channel, tag, target, and restart actions without mutating config, installing, syncing plugins, or restarting. Auto-updater config keys live under `update.auto.*` (default-off). <!-- v2026.2.22 -->

- **`openclaw doctor --fix` (config migration)** — Migrates legacy streaming config keys (`streamMode` → `channels.<channel>.streaming` with enum values). Repairs OAuth credentials directory only when affected channels are configured. <!-- v2026.2.22 -->

- **`openclaw config get` redaction** — `openclaw config get` now redacts sensitive values before printing — prevents credential leakage to terminal output and shell history. <!-- v2026.2.22 -->

### Config <!-- v2026.2.22 -->

- **`channels.modelByChannel` allowlisted** — Previously caused "unknown channel id" errors during config validation. Now allowlisted in strict validation. `bindings[].comment` is now optional in strict validation. Array-valued paths compared structurally during config diffing. <!-- v2026.2.22 -->

### Session <!-- v2026.2.22 -->

- **`session.dmScope` default changed (breaking)** — CLI local onboarding now sets `session.dmScope` to `per-channel-peer` by default. Previous behavior was `main` (shared DM history across senders). To restore old behavior, set: <!-- v2026.2.22 -->

  ```json
  { "session": { "dmScope": "main" } }
  ```

- **Session-store path template resolution** — CLI sessions resolve implicit session-store path templates using the configured default agent ID for named-agent setups. Configured sessions directory is now passed when resolving transcript paths. <!-- v2026.2.22 -->

### Doctor / Security <!-- v2026.2.22 -->

- **`approvals.exec.enabled=false` warning** — Explicit warning added: setting `approvals.exec.enabled=false` disables forwarding only — enforcement remains driven by the host-local `exec-approvals.json` file. <!-- v2026.2.22 -->

## v2026.2.23 Changes <!-- v2026.2.23 -->

### Config <!-- v2026.2.23 -->

- **`unsetPaths` immutable path-copy updates** — `unsetPaths` applied with immutable path-copy updates during config writes. Prototype-key segments are rejected in `config get/set/unset` path traversal to prevent prototype-pollution attacks. <!-- v2026.2.23 -->

## v2026.2.24 Changes <!-- v2026.2.24 -->

### Doctor <!-- v2026.2.24 -->

- **Sandbox Docker warning** (#25438): when sandbox mode is enabled but Docker is unavailable, a clear actionable warning is now surfaced (including failure impact and remediation steps) instead of a mild "skip checks" note. Contributor: @mcaxtr. <!-- v2026.2.24 -->

- **Doctor recovery hints** (#24485): stale recovery hints corrected to use valid current commands: `openclaw gateway status --deep` and `openclaw configure --section model`. Contributor: @chilu18. <!-- v2026.2.24 -->

- **Plugins auto-enable** (#25275): auto-enable resolves third-party channel plugins by manifest plugin id (not channel id), preventing invalid config writes when ids differ. Contributor: @zerone0x. <!-- v2026.2.24 -->

### Config <!-- v2026.2.24 -->

- **Meta timestamp coercion** (#25491): numeric `meta.lastTouchedAt` timestamps (e.g. `Date.now()` values) are now accepted and coerced to ISO strings, preserving compatibility with agent edits. Contributor: @mcaxtr. <!-- v2026.2.24 -->

## v2026.3.1 Changes <!-- v2026.3.1 -->

### CLI <!-- v2026.3.1 -->

- **`openclaw config file` subcommand** (#26256): new `config file` subcommand prints the active config file path using explicit precedence resolution: `OPENCLAW_CONFIG_PATH` (or legacy `CLAWDBOT_CONFIG_PATH`) first, then existing state-dir candidates, and finally the canonical `state-dir/openclaw.json` path. Implementation adds `runConfigFile()` to `config-cli.ts` and registers the subcommand under `config`. Contributor: @cyb1278588254. <!-- v2026.3.1 -->

- **Cron list: rename Agent to Agent ID, add Model column** (#26259): `cron list` output now labels the agent column as `Agent ID` (showing the actual `agentId`, not the model) and adds a separate `Model` column for isolated `agentTurn` jobs. Missing agent IDs display `-` instead of `default`. Implementation in `cron-cli/shared.ts` (`printCronList`). Contributor: @openperf. <!-- v2026.3.1 -->

- **Cron run exit code** (#31121): `cron run` now returns exit code `0` only when the gateway reports `{ ok: true, ran: true }`, and `1` for non-run/error outcomes. Allows scripting/debugging to detect actual execution status. Implementation in `cron-cli/register.cron-simple.ts`. Contributor: @Sid-Qin. <!-- v2026.3.1 -->

- **JSON preflight output** (#24368): `--json` command stdout is now kept machine-readable by suppressing doctor preflight note output while still running legacy migration/config doctor flow. Contributor: @altaywtf. <!-- v2026.3.1 -->

- **Ollama config: allow `config set` for apiKey without predeclared provider** (#29299): `config set models.providers.ollama.apiKey <key>` now auto-scaffolds the Ollama provider block (`baseUrl: "http://127.0.0.1:11434"`, `api: "ollama"`, `models: []`) when no provider config exists. Implementation adds `ensureValidOllamaProviderForApiKeySet()` with `OLLAMA_API_KEY_PATH`/`OLLAMA_PROVIDER_PATH`/`OLLAMA_DEFAULT_BASE_URL` constants. Contributor: @vincentkoc. <!-- v2026.3.1 -->

- **CLI install: npm-link fallback for Permission denied** (#17151): Docker builds use a root CLI symlink (`ln -sf /app/openclaw.mjs /usr/local/bin/openclaw`) instead of `npm link` to fix CLI startup `exit 127` failures on affected installs. Contributors: @sskyu, @vincentkoc. <!-- v2026.3.1 -->

- **Install/npm: fix global install deprecation warnings** (#28318): resolved npm global install deprecation warnings. Contributor: @vincentkoc. <!-- v2026.3.1 -->

- **Update/Global npm: fallback to `--omit=optional`** (#24896): when global `npm update` fails, the update flow now retries with `--omit=optional` so optional dependency install failures no longer abort update flows. Implementation adds `globalInstallFallbackArgs()` with `NPM_GLOBAL_INSTALL_OMIT_OPTIONAL_FLAGS` in `update-global.ts`. Contributor: @xinhuagu. <!-- v2026.3.1 -->

### Config <!-- v2026.3.1 -->

- **Gateway bind host alias normalization during migration** (#30855): legacy migration now normalizes host-style `gateway.bind` values (`0.0.0.0`, `::`, `127.0.0.1`, `localhost`) to supported bind modes (`lan`, `loopback`) so older configs recover without manual edits. Migration id: `gateway.controlUi.allowedOrigins-seed-for-non-loopback`. Also seeds `gateway.controlUi.allowedOrigins` for existing non-loopback installs that upgraded past v2026.2.26. Implementation in `legacy.migrations.part-3.ts`. Contributor: @liuxiaopai-ai. <!-- v2026.3.1 -->

- **Shell env markers: `OPENCLAW_SHELL`** (#31271): `OPENCLAW_SHELL` env var is now injected into child processes across all shell-like runtimes: `exec`, `acp`, `acp-client`, and `tui-local`. Allows shell startup/config rules (e.g. `.bashrc`, `.zshrc`) to detect OpenClaw contexts and adjust behavior. Documented in env/exec/acp/TUI docs. Contributor: @vincentkoc. <!-- v2026.3.1 -->

### Infra <!-- v2026.3.1 -->

- **Docker image health checks** (#31272): the gateway now exposes built-in HTTP liveness/readiness probe endpoints (`/health`, `/healthz`, `/ready`, `/readyz`). Probe handlers apply when no plugin has claimed those paths, so plugin routes retain precedence. `docker-compose.yml` includes a `healthcheck` directive fetching `http://127.0.0.1:18789/healthz`. Contributor: @vincentkoc. <!-- v2026.3.1 -->

- **Docker image OCI labels** (#31196): `Dockerfile` now includes `org.opencontainers.image.*` base-image annotations (base name, digest, source, URL, docs, license, title, description) for downstream image consumers. Contributor: @vincentkoc. <!-- v2026.3.1 -->

- **Docker image permissions** (#30191): `extensions/`, `.agent/`, `.agents/` directories are normalized to `755` for directories and `644` for files to harden permission consistency. Contributor: @vincentkoc. <!-- v2026.3.1 -->

- **Docker/Sandbox browser: `OPENCLAW_BROWSER_NO_SANDBOX=1`** (#29879): sandbox browser creation now sets the `--no-sandbox` Chromium flag inside Docker containers to prevent Chromium namespace clone failures in unprivileged environments. Contributor: @Lukavyi. <!-- v2026.3.1 -->

- **macOS supervised restart: `launchctl kickstart -k`** (#29078): this was the `v2026.3.1` launchd strategy for bypassing `ThrottleInterval`. In current `v2026.3.8`, supervised restart instead exits and relies on launchd `KeepAlive`, with LaunchAgent repair/restart paths `enable` before `bootstrap`. Contributor: @cathrynlavery. <!-- v2026.3.1 -->

- **macOS TLS certs: default `NODE_EXTRA_CA_CERTS`** (#27915): LaunchAgent and Node service environments now default `NODE_EXTRA_CA_CERTS` to `/etc/ssl/cert.pem` on macOS (respecting explicit overrides) so HTTPS clients no longer fail with local-issuer errors under launchd. Implementation in `service-env.ts` (`buildServiceEnvironment` and `buildNodeServiceEnvironment`). Contributor: @Lukavyi. <!-- v2026.3.1 -->

- **fs-safe: sanitize directory-read failures (EISDIR)** (#31186): `fs-safe.ts` now rejects directory paths before `open()` and defensively catches any `EISDIR` race so the raw error code never leaks to messaging or tool output. Also adds `outside-workspace` error code to `SafeOpenErrorCode`. <!-- v2026.3.1 -->

- **Sandbox mkdirp boundary checks** (#30610): sandbox `mkdirp` operations now enforce tighter directory boundary checks to prevent path escapes. Contributor: @glitch418x. <!-- v2026.3.1 -->

- **Plugins npm spec install: detect new `.tgz`** (#21039): when `npm pack` output is empty, the plugin install flow now detects newly created `.tgz` archives in the pack directory. Contributors: @graysurf, @vincentkoc. <!-- v2026.3.1 -->

- **Plugins install: clear stale install errors** (#25073): follow-up plugin install attempts now report current state correctly after clearing stale errors from previously failed npm package lookups. Contributor: @dalefrieswthat. <!-- v2026.3.1 -->

- **Windows plugin install: resolve to node + npm CLI scripts** (#31147): avoids `spawn EINVAL` on Windows npm/npx invocations by resolving to `node` + npm CLI scripts instead of spawning `.cmd` directly. Contributor: @codertony. <!-- v2026.3.1 -->

- **Onboarding custom providers: improved verification** (#27380): increased verification timeout and reduced `max_tokens` for custom provider probes during setup, improving reliability for slower local endpoints (e.g. Ollama). Contributor: @Sid-Qin. <!-- v2026.3.1 -->

### Breaking <!-- v2026.3.1 -->

- **Node exec `systemRunPlan` required**: `host=node` exec approval payloads now require `systemRunPlan`. Requests without it are rejected. This is a **breaking change** for integrations that construct Node exec approval requests manually.

## v2026.3.7 Delta Notes

- **BREAKING**: Gateway auth mode requirement — explicit `gateway.auth.mode` is now required when both token and password are configured; ambiguous dual-credential setups are rejected at startup.
- Heartbeat legacy-path auto-migration: old heartbeat config paths are automatically migrated to the new schema on startup.
- Config schema cache key stability: cache keys for config schema validation are now stable across restarts, reducing unnecessary schema re-computation.
- Provider apiKey persistence hardening: API key writes are now atomic and survive partial-write failures.
- Env substitution degraded mode: missing env vars in config substitution now enter a degraded mode with a clear warning rather than silently dropping values.
- Invalid-load fail-closed behavior: configs that fail schema validation during load now fail closed rather than proceeding with partially-loaded state.
- Docker multi-stage slim build: Dockerfile restructured to multi-stage build for smaller production images.
- Podman support: container lifecycle scripts now detect and support Podman as an alternative container runtime.
- systemd WSL2 hardening: systemd service files set correct file permissions (0600 for secrets, 0700 for dirs) and handle WSL2 cgroup constraints.
- Windows Scheduled Task management: task creation and removal are now locale-invariant, fixing failures on non-English Windows installs.

- **Node `system.run` realpath pinning**: `system.run` execution now pins path-token commands to the canonical executable path (`realpath`) in both allowlist and approval execution flows. Integrations/tests that asserted token-form argv (e.g. `tr`) must now accept canonical paths (e.g. `/usr/bin/tr`). This is a **breaking change** for approval flow consumers.

## v2026.3.8 Delta Notes

- **Backup and recovery CLI:** `openclaw backup create` and `openclaw backup verify` are now top-level commands, with config-only and no-workspace modes plus manifest validation. Destructive flows like reset/uninstall now point operators at the backup workflow first.
- **Config preflight and live-path reporting:** daemon start/restart pre-validates config before service handoff, config writes preserve/refresh runtime snapshots after disk writes, and gateway config RPC responses now report the active config path from the live IO layer.
- **launchd restart semantics changed:** supervised respawn now exits and relies on launchd `KeepAlive`, supervision detection includes `XPC_SERVICE_NAME`, LaunchAgent repair/restart paths `enable` before `bootstrap`, and restart timeout failures now exit non-zero so the supervisor retries.
- **Container/runtime ops:** Podman bind mounts now add SELinux `:Z` relabeling when needed, inaccessible-cwd setup paths fall back safely, and the runtime Docker image is smaller.
- **Version/install/release tooling:** `openclaw --version` now includes the short commit hash when available, installer/version checks accept the decorated format, and release-check validates bundled extension manifest/root-dependency mirror drift.

## v2026.3.11 Delta Notes

### CLI

- **CLI/skills JSON — strip ANSI and C1 control bytes from JSON output:** `skills list --json`, `skills info --json`, and `skills check --json` now strip ANSI escape sequences and C1 control bytes (U+0080–U+009F) from all string values before serialization. Fixes invalid JSON output when skill names or descriptions contained terminal color codes. Implementation: `src/cli/skills-cli.format.ts` — `sanitizeJsonValue()` calls `stripAnsi()` then replaces `JSON_CONTROL_CHAR_REGEX` before output. Fixes #27530.

- **CLI/tables — ASCII borders on legacy Windows consoles:** `src/terminal/table.ts` — `resolveDefaultBorder()` defaults to `"ascii"` on `win32` when none of the modern-terminal signals (`WT_SESSION`, `xterm`/`cygwin`/`msys` in `TERM`, `vscode` in `TERM_PROGRAM`) are present. Modern Windows terminals continue to receive Unicode box-drawing borders. Fixes mojibake under GBK/936 code-page legacy consoles (fixes #40853, related #41015).

- **CLI/memory teardown — close cached memory managers on CLI exit:** `src/cli/run-main.ts` now calls `closeActiveMemorySearchManagers()` from `src/plugins/memory-runtime.ts` during the one-shot CLI shutdown path. The plugin runtime then closes active plugin-backed memory search managers, so watcher-backed memory caches no longer keep the process alive after the command completes. Fixes #40389.

- **CLI/daemon scheduled restarts — consistent scheduled gateway restart handling:** gateway scheduled restart path is now handled consistently across daemon lifecycle operations, preventing cases where a scheduled restart could be missed or duplicated.

- **Exec/child commands — `OPENCLAW_CLI` env marker for subprocess detection:** `src/infra/openclaw-exec-env.ts` exports `OPENCLAW_CLI_ENV_VAR = "OPENCLAW_CLI"` and `OPENCLAW_CLI_ENV_VALUE = "1"`. Child command environments are stamped with this variable so subprocesses can detect they were launched from the CLI. Fixes #41411.

### Config/Infra

- **Gateway/config errors — surface up to three validation issues:** `config.set`, `config.patch`, and `config.apply` error messages now surface up to three validation issues at once (previously only the first). Implementation: `src/gateway/server.impl.ts` — validation issue formatting truncated at three items. Fixes #42664.

- **Gateway/Control UI config validation — surface validation issues in dashboard:** config validation issues are now surfaced in the Control UI dashboard in addition to CLI error messages, giving users a visual indication of what is wrong with the active configuration. Fixes #42664.

- **macOS/launchd restarts v2 — keep LaunchAgent registered during explicit restarts:** `src/daemon/launchd.ts` — explicit restarts now keep the LaunchAgent registered and hand off via a detached launchd helper (`src/daemon/launchd-restart-handoff.ts`). Config/hot-reload restart paths are recovered. Replaced `bootout` with `kickstart -k` for launchd restarts (falls back to `bootstrap` + `kickstart` when service was previously deregistered). Fixes #43311, #43406, #43035, #43049.

- **macOS/LaunchAgent install permissions — tighten directory and plist permissions:** `src/daemon/launchd.ts` — LaunchAgent install now sets `LAUNCH_AGENT_DIR_MODE = 0o755` and `LAUNCH_AGENT_PLIST_MODE = 0o644`, and tightens existing permissions by stripping group/world write bits (`mode & ~0o022`) during install.

- **State dir permissions — harden state dir on onboard:** state directory permissions are hardened during the onboarding flow to prevent world-writable or group-writable directories.

- **Git/runtime state — ignore `.dev-state` in repo:** `.gitignore` now includes `.dev-state` so local runtime state files (used during development) no longer appear as untracked changes. Fixes #41848.

- **Plugins/bundle repair — repair bundled plugin dirs after npm install:** bundled plugin directories are now repaired (directory structure re-validated/restored) after `npm install` runs in the plugin directory, preventing corruption from npm's behavior of overwriting or removing files.

- **Runtime version — expose runtime version in gateway status:** gateway status responses now include the Node.js runtime version alongside the OpenClaw version.

### Onboarding

- **Onboarding/Ollama — first-class Ollama setup:** `src/commands/ollama-setup.ts` — new dedicated Ollama onboarding flow with Local or Cloud + Local modes, browser-based cloud sign-in, and curated model suggestions. Cloud model handling and model defaults are improved. Fixes #41529. Thanks @BruceMacD.

- **Ollama auth flow and model defaults:** Ollama auth flow improved with cleaner browser sign-in integration and better model default selection.

- **Ollama onboarding cloud — fix cloud handling:** cloud provider handling in Ollama onboarding corrected so cloud-only and Cloud + Local modes work reliably. Fixes #41529.

- **Ollama share model context — fix context discovery:** shared model context discovery for Ollama now correctly resolves the active model list when Ollama is used alongside a cloud provider.

- **OpenCode/onboarding — new OpenCode Go provider:** `src/commands/onboard-auth.config-opencode-go.ts` — new OpenCode Go provider added. Onboarding wizard treats Zen and Go as a single OpenCode setup, stores one shared OpenCode API key, and no longer overrides the built-in `opencode-go` catalog routing. Fixes #42313.

- **macOS/onboarding remote — detect remote gateways needing shared auth token:** `src/commands/onboard-remote.ts` — remote onboarding now detects when a remote gateway requires a shared auth token, explains where to find it, and clarifies when a successful connectivity check used paired-device auth rather than the token. Fixes #43100.

## v2026.3.12 Changes <!-- v2026.3.12 -->

### CLI <!-- v2026.3.12 -->

- **Fast mode toggle — `/fast` TUI command and `params.fastMode` session field:** `src/tui/commands.ts` adds `/fast <status|on|off>` as a TUI slash command. `src/tui/tui-command-handlers.ts` handles it and emits `fast mode: on/off` status lines. `src/tui/tui-session-actions.ts` persists the toggle in session state. `src/config/sessions/types.ts` adds `fastMode?: boolean` to `SessionEntry`. `src/agents/fast-mode.ts` resolves fast mode from session entry, model config `params.fastMode` or `params.fast_mode`, and extra params.

- **Provider plugin architecture — Ollama, vLLM, SGLang:** Ollama, vLLM, and SGLang onboarding and model discovery are now handled via the plugin provider architecture (`auth-choice.apply.plugin-provider.ts`). Provider-owned onboarding, discovery, model-picker, and post-selection hooks replace older inline flows.

- **CLI/thinking help — `xhigh` level added to cron and agent commands:** `src/cli/cron-cli/register.cron-add.ts`, `register.cron-edit.ts`, and `src/cli/program/register.agent.ts` now document `xhigh` as a valid value for `--thinking <level>` (full range: `off|minimal|low|medium|high|xhigh`).

- **MiniMax onboarding — flatten to 4 direct auth choices, unify CN/Global:** MiniMax onboarding in `src/commands/auth-choice.apply.minimax.ts` routes to `minimax:cn` or `minimax:global` profiles depending on the region flag. CN and Global are unified under a single provider path (fixes #44284).

- **Commands — require owner for `/config` and `/debug`:** `src/auto-reply/commands-registry.data.ts` enforces that `/config` and `/debug` commands require owner-level authorization. The `requireOwner` flag is checked by `src/auto-reply/command-auth.ts`.

- **Terminal table rendering — grapheme display width and emoji presentation:** `src/terminal/ansi.ts` adds `graphemeWidth()` using `Intl.Segmenter` at grapheme granularity and per-codepoint Unicode East Asian width detection. `src/terminal/table.ts` tokenizes plain text by grapheme to wrap by display width. Tables no longer shrink one column due to miscounted grapheme widths. Emoji ZWJ sequences are kept as single graphemes. Emoji presentation normalization is applied in skills output.

- **Kubernetes install docs — starter K8s path with raw manifests and Kind setup:** deployment documentation for Kubernetes added (raw manifests, Kind setup).

- **Build — default to Node 24, raise Node 22 floor to 22.16:** `src/infra/runtime-guard.ts` now requires Node `>=22.16.0`. Daemon runtime locators (`src/daemon/runtime-paths.ts`, `src/daemon/program-args.ts`) prefer Node 24 and treat Node 22 LTS 22.16+ as the minimum floor.

### Config <!-- v2026.3.12 -->

- **Config/Anthropic alias normalization at startup:** inline Anthropic model alias normalization runs during config load to prevent startup crashes on stale model refs (fixes #45520). See `src/config/defaults.ts` and `src/config/model-alias-defaults.ts`.

- **Models/OpenRouter native IDs — canonicalize across config writes and runtime:** `src/agents/model-selection.ts` preserves native OpenRouter model prefixes (e.g. `openrouter/aurora-alpha`). Config writes, runtime lookups, fallback management, and `models list --plain` use canonicalized OpenRouter model keys. Legacy `openrouter/openrouter/...` entries are migrated. See `src/commands/models.set.e2e.test.ts` for migration test coverage.

- **Models/secrets — enforce source-managed SecretRef markers in `models.json`:** generated `models.json` entries now enforce `SecretRef` markers for source-managed secret fields (fixes #43759).

- **Config/validation — new accepted keys in strict validation:**
  - `agents.list[].params` per-agent parameter overrides (fixes #41171)
  - `tools.web.fetch.readability` and `tools.web.fetch.firecrawl.*` (fixes #42583)
  - `channels.signal.groups` group configuration
  - `discovery.wideArea.domain` for wide-area DNS-SD domain (fixes #35615)

- **Docker/timezone — `OPENCLAW_TZ` env var for `docker-setup.sh`:** `docker-setup.sh` reads `OPENCLAW_TZ` (must be a valid IANA timezone string) and writes it into the container `.env` file and `docker-compose.yml` `TZ` slots. Invalid or missing timezone values fail with a clear message (fixes #34119).

### Infra/Doctor <!-- v2026.3.12 -->

- **Doctor/gateway service — canonicalize entrypoint paths before comparison:** `src/commands/doctor-gateway-services.ts` resolves the installed service entrypoint via `fs.realpath()` before comparing to the current install, preventing false repair prompts caused by symlinks or relative paths (fixes #43882).

- **Doctor/cron — stop flagging canonical payload kinds as legacy:** `src/commands/doctor-cron.ts` no longer flags canonical cron payload kinds as requiring legacy repair (fixes #44012).

- **Infra/exec allowlist — tighten glob matching, preserve POSIX case sensitivity:** exec allowlist glob matching tightened; POSIX case sensitivity preserved (fixes #43798).

- **Infra/host env — block `GIT_EXEC_PATH` in sanitized host exec environments:** `src/infra/host-env-security-policy.json` includes `GIT_EXEC_PATH` in the blocked env var list for sanitized host exec contexts (fixes #43685).

- **Status resolution — resolve context window by provider-qualified key:** context window resolution prefers the provider-qualified model key; on bare-ID collision the maximum context window is used (fixes #36389).

### Windows <!-- v2026.3.12 -->

- **Windows/gateway install — bound schtasks calls, Startup-folder fallback:** `src/daemon/schtasks.ts` bounds schtasks execution with timeouts and falls back to installing a Startup-folder login item when task creation hangs or is denied.

- **Windows/gateway stop — resolve Startup-folder fallback listeners:** gateway stop resolves Startup-folder fallback listeners by inspecting the installed `gateway.cmd` port.

- **Windows/gateway auth — stop attaching device identity on local loopback shared-token calls:** device identity is no longer attached to local loopback calls that use the shared gateway token.

### macOS <!-- v2026.3.12 -->

- **macOS/launchd — `openclaw onboard --install-daemon` avoids false-fails on slower Macs and fresh VM snapshots:** daemon install path hardened against timing-sensitive launchd bootstrap races.

## v2026.3.13 Changes <!-- v2026.3.13 -->

### CLI <!-- v2026.3.13 -->

- **Gateway/status — `--require-rpc` flag:** `src/cli/daemon-cli/register-service-commands.ts` adds `--require-rpc` to the `status` command (both `openclaw gateway status` and `openclaw daemon status`). When set, the command exits non-zero if the RPC probe fails. Cannot be combined with `--no-probe`. Tested in `src/cli/daemon-cli/status.test.ts` and `register-service-commands.test.ts`.

- **Gateway/status — clearer Linux non-interactive daemon-install failure reporting:** failure messages for the Linux non-interactive daemon install path are now more explicit about the failure reason.

- **Agents/memory bootstrap — single root memory file preference:** `src/memory-host-sdk/host/internal.ts` and `extensions/memory-core/src/memory/manager-sync-ops.ts` prefer `MEMORY.md` over `memory.md` as the single root memory file when both exist (fixes #26054).

- **Build/plugin-sdk bundling — bundle plugin-sdk subpath entries in one pass:** plugin-sdk subpath entries are bundled in a single pass to prevent memory blow-up during build (fixes #45426).

### Config/Infra <!-- v2026.3.13 -->

- **Config/validation — `channels.signal.groups` and `discovery.wideArea.domain` accepted in strict validation:** both keys are now allowlisted in strict config validation (backported from v2026.3.12 scope, confirmed present in v2026.3.13).

### Windows <!-- v2026.3.13 -->

- **Windows/gateway — all three gateway fixes landed:** Windows gateway install (schtasks timeout + Startup-folder fallback), stop (fallback listener resolution), and auth (loopback device identity noise) fixes are all present in v2026.3.13.

### macOS <!-- v2026.3.13 -->

- **macOS/runtime locator — require Node >=22.16.0 during macOS runtime discovery:** `src/daemon/runtime-paths.ts` enforces the Node 22.16+ floor when scanning system Node installations on macOS.

- **macOS/onboarding — avoid self-restarting freshly bootstrapped launchd gateways:** onboarding no longer triggers a self-restart on a gateway that was just bootstrapped by the same onboarding flow.

## v2026.3.24 Delta Notes

### CLI

- **`--container` / `OPENCLAW_CONTAINER` global flag:** new global CLI flag routing CLI commands into Docker/Podman containers via `src/cli/container-target.ts`. When set, CLI commands are forwarded to a running container instead of executing locally.

- **CLI update preflight — `nodeVersionSatisfiesEngine()` check:** the update flow now checks `nodeVersionSatisfiesEngine()` before applying updates, preventing updates that target a Node.js version the current runtime does not satisfy.

- **Node 22.14 floor change:** the minimum supported Node.js version floor has been raised to 22.14.

- **CLI logging — timezone offset in timestamps:** CLI log timestamps now include the timezone offset for unambiguous time references across environments.

## v2026.3.28 Delta Notes <!-- v2026.3.28 -->

### CLI <!-- v2026.3.28 -->

- **`openclaw config schema` subcommand** (#54523): new `config schema` subcommand prints the generated JSON schema for `openclaw.json` to stdout as JSON. Implementation: `runConfigSchema()` in `src/cli/config-cli.ts` — reads the runtime config schema via `readBestEffortRuntimeConfigSchema()`, injects a `$schema` string property, and writes via `writeRuntimeJson()`. The subcommand is registered under `config` alongside `get`, `set`, `unset`, `file`, and `validate`. The `config` command description updated to `get/set/edit/unset/file/schema/validate`. <!-- v2026.3.28 -->

- **CLI/zsh completion — defer `compdef` until `compinit` available** (#56555): the generated zsh completion script now wraps `compdef` registration inside `_${rootCmd}_register_completion()`, which checks `(( ! $+functions[compdef] ))` before attempting to bind. If `compdef` is not yet available (plugin managers or manual setups that source completions before `compinit`), it queues the function via `precmd_functions` so binding happens automatically on the next prompt. This fixes blank-completion or startup-error symptoms in oh-my-zsh, zinit, and manual setups. Implementation: `generateZshCompletion()` in `src/cli/completion-cli.ts`. <!-- v2026.3.28 -->

- **CLI/onboarding — Kimi Code API key option restored in Moonshot setup menu** (#54412): the `kimi-code-api-key` auth choice (Kimi Code subscription API key) is now included as a third option in the Moonshot AI setup group alongside the existing `moonshot-api-key` (.ai) and `moonshot-api-key-cn` (.cn) choices. Registered via the `kimi-coding` bundled plugin's `openclaw.plugin.json` with `groupId: "moonshot"` so it appears under the Moonshot AI (Kimi K2.5) provider group during onboarding. <!-- v2026.3.28 -->

- **CLI/update status — explicit "up to date" when local matches npm latest** (#51409): `formatUpdateOneLiner()` in `src/commands/status.update.ts` now pushes `"up to date"` into the update one-liner when the local version (`VERSION`) equals `update.registry.latestVersion` (semver comparison == 0) and the install kind is not `"git"`. Previously, the output only showed the npm latest version string with no explicit state label. Git installs retain their own `"up to date"` label from the git sync section. <!-- v2026.3.28 -->

- **CLI/message send — write manual deliveries into resolved agent session transcript** (#54187): manual `openclaw message send` deliveries are now mirrored into the resolved agent's session transcript via `appendAssistantMessageToSessionTranscript()` when an outbound session route is available. The `mirror` field on `OutboundSendContext` carries `{ agentId, sessionKey, text, mediaUrls }` from `handleSendAction()` in `src/infra/outbound/message-action-runner.ts`. Transcript append runs in the `onHandled` callback of `tryHandleWithPluginAction()` in `src/infra/outbound/outbound-send-service.ts`. <!-- v2026.3.28 -->

- **CLI/plugins — routed commands use same bundled-channel snapshot as gateway startup** (#54809): fast-path routed commands (status, health, sessions) now apply `applyPluginAutoEnable()` on the loaded config before calling `ensurePluginRegistryLoaded()` in `src/cli/plugin-registry.ts`. This ensures bundled provider/channel plugins that are auto-enabled at gateway startup are also available for routed CLI commands without requiring manual `plugins.allow` entries. <!-- v2026.3.28 -->

### Plugins/Startup <!-- v2026.3.28 -->

- **Bundled provider and CLI-backend plugins auto-loaded from explicit config refs** (breaking for manual `plugins.allow`): bundled plugins (including the Claude CLI, Codex CLI, and Gemini CLI inference backends) are now auto-loaded when referenced via model refs or `agents.defaults.cliBackends` config keys, without needing explicit `plugins.allow` entries. The auto-enable logic in `src/config/plugin-auto-enable.ts` resolves provider plugin IDs from `BUNDLED_AUTO_ENABLE_PROVIDER_PLUGIN_IDS` (populated from `autoEnableWhenConfiguredProviders` manifest entries) and channel plugin IDs from `resolveChannelPluginIds()`. Previous setups that relied on explicit `plugins.allow` entries for bundled Claude CLI, Codex CLI, or Gemini CLI will continue to work but the entries are now redundant. <!-- v2026.3.28 -->

- **Bundled Gemini CLI backend added**: Gemini CLI is now a first-class bundled CLI-backend plugin alongside Claude CLI and Codex CLI. Plugin-surface inference defaults moved onto the plugin. <!-- v2026.3.28 -->

- **`gateway run --cli-backend-logs`** replaces `--claude-cli-logs`: the generic `--cli-backend-logs` flag (`src/cli/gateway-cli/run.ts`) filters gateway console output to the `agent/cli-backend` subsystem for all CLI backends. The old `--claude-cli-logs` flag is retained as a deprecated alias. Both flags map to the same internal `cliBackendLogs` option. <!-- v2026.3.28 -->

### Config <!-- v2026.3.28 -->

- **Config/Doctor — drop automatic config migrations older than two months (BREAKING)** (#52709): `openclaw doctor` no longer automatically rewrites config keys that were superseded more than two months ago. Very old legacy keys (older than the two-month cutoff) now fail validation instead of being silently migrated. Users on configs with such keys must apply the migration manually or with `openclaw doctor --fix` before the key reached end-of-migration-life. Runtime TTS auto-migration on normal config reads and secret resolution is retained; only the Doctor's rewrite-on-load path for pre-cutoff entries is removed. <!-- v2026.3.28 -->

- **Config/TTS — auto-migrate legacy speech config on normal reads** (#52709): legacy `tts.<provider>` API-key shapes (the old `messages.tts` / `channels.discord.voice.tts` structure) are now migrated into `messages.tts.providers` automatically on every config read and during secret resolution. Legacy-shape diagnostics remain available under `openclaw doctor` for transparency. The regular-mode runtime fallback that previously supported the old shapes at runtime has been removed; configs must pass through the migration path. Implementation: `migrateLegacyTtsConfig()` in `src/config/legacy.migrations.runtime.ts`. <!-- v2026.3.28 -->

- **Config/web fetch — `tools.web.fetch.maxResponseBytes` accepted in runtime schema validation** (#53401): the `maxResponseBytes` field under `tools.web.fetch` is now allowlisted in strict runtime schema validation. Previously set via `config set` or manually in `openclaw.json`, it was rejected by the validator. Schema definition: `src/config/zod-schema.agent-runtime.ts`. <!-- v2026.3.28 -->

### Providers <!-- v2026.3.28 -->

- **Providers/Qwen — `qwen-portal-auth` OAuth integration removed (BREAKING)** (#52709): the `qwen-portal-auth` OAuth flow for `portal.qwen.ai` has been removed. `src/commands/auth-choice.apply.qwen-portal.ts` is deleted. Users must migrate to Model Studio via `openclaw onboard --auth-choice modelstudio-api-key`. Qwen models remain available through the `modelstudio` plugin. <!-- v2026.3.28 -->

- **MiniMax — model catalog trimmed to M2.7 only**: the MiniMax bundled plugin (`extensions/minimax/`) now exposes only `MiniMax-M2.7` and `MiniMax-M2.7-highspeed`. Legacy model IDs `M2`, `M2.1`, `M2.5`, and `VL-01` are removed from `provider-models.ts` and the plugin manifest. The default model ID is `MiniMax-M2.7`. <!-- v2026.3.28 -->

### Podman/Container <!-- v2026.3.28 -->

- **Podman setup simplified around current rootless user**: `scripts/podman/setup.sh` runs as the current (non-root) user and keeps the container rootless; the release-line helper is installed under `~/.local/bin` for local container workflows. Host-CLI `openclaw --container <name> ...` forwarding routes CLI commands into the named running container without requiring `ssh` (implemented via `src/cli/container-target.ts`). <!-- v2026.3.28 -->

## v2026.3.31 Delta Notes

### CLI / Operator Workflows

- **`openclaw tasks` is now part of the released CLI surface:** `list`, `show`, and `cancel` are stable operator paths for detached work, and task lifecycle docs should be treated as first-class CLI documentation.
- **Onboarding prompt fallback is safer:** declining a discovered remote gateway resets the CLI wizard back to the safe loopback default instead of reusing the rejected URL.

### Config / Runtime Boundaries

- **Dangerous installs fail closed by default:** plugin installs and gateway-backed skill dependency installs now require an explicit `--dangerously-force-unsafe-install` override when built-in scan results are critical or errored.
- **`mcp.servers` supports remote HTTP/SSE plus `streamable-http`:** stable MCP configuration is no longer limited to local stdio transport assumptions.

## v2026.4.5 Delta Notes

### Breaking / Config

- **Legacy public config aliases are no longer the stable contract:** the release line removes documented support for old public aliases such as `talk.voiceId`, `talk.apiKey`, `agents.*.sandbox.perSession`, `browser.ssrfPolicy.allowPrivateNetwork`, `hooks.internal.handlers`, and channel/group/room `allow` toggles. Docs and examples should use canonical paths and `enabled`, with `openclaw doctor --fix` as the migration path.
- **Bundled CLI text-provider backends are removed from the public surface:** `agents.defaults.cliBackends` and the old bundled CLI text-provider path are no longer part of the release-line config contract.

### CLI / Onboarding

- **Plugin-config prompts in setup flows:** guided onboarding and setup now include plugin-config prompts, which means setup behavior is more tightly coupled to plugin manifests and runtime config schemas than in earlier releases.
- **`openclaw plugins install --force`:** existing plugin and hook-pack targets can now be replaced without the dangerous-code override flag, which changes the documented install/upgrade workflow.
- **Skills panel / ClawHub CLI parity:** the release line expects ClawHub discovery and installation to be a first-class workflow rather than an external-only/manual step.

### Config / Schema

- **`openclaw config schema` is richer:** exported JSON Schema now includes titles and descriptions, which makes schema consumers more useful but also means config-help metadata has to stay aligned with the real schema.
- **Shared model/media transport overrides:** current config docs need to treat request overrides as cross-provider surfaces used by model and media paths alike, not just classic text-generation requests.
- **`channels.<channel>.contextVisibility`:** channel context filtering is now a documented config surface, so channel docs and CLI/config references need to include it.

### Doctor / Migration

- **Doctor owns more release-line migration work:** config alias cleanup, stale Claude CLI state repair/removal, and release-line fixups are now part of the expected `openclaw doctor` path after upgrade rather than optional cleanup.
