# OpenClaw CLI, Config & Infrastructure — Comprehensive Analysis

> Updated: 2026-02-16 | Version: v2026.2.15 | Codebase: ~/src/openclaw | Cluster: CLI, CONFIG & INFRASTRUCTURE

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

### File Inventory (202 files)

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
| `program/command-registry.ts` | Core command registry with lazy loading of ~10 command groups |
| `program/context.ts` | `createProgramContext()` — version, agent channel options |
| `program/program-context.ts` | Get/set program context on Commander instance |
| `program/help.ts` | Custom help formatting for Commander |
| `program/helpers.ts` | Shared helpers for program registration |
| `program/preaction.ts` | Pre-action hooks (version check, config guard) |
| `program/action-reparse.ts` | Re-parse argv after lazy command load |
| `program/config-guard.ts` | Ensure config is valid before command execution |
| `program/routes.ts` | Route specs for fast-path commands (health, status, sessions, agents list, etc.) |
| `program/register.subclis.ts` | 25+ subcli entries (gateway, daemon, nodes, browser, etc.) with lazy loading |
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
| `nodes-run.ts` | Nodes run command implementation |
| `nodes-screen.ts` | Screen recording implementation |
| `node-cli.ts` | Node (single node) CLI entry |
| `node-cli/register.ts` | Node subcommand registration |
| `node-cli/daemon.ts` | Node daemon management |
| `models-cli.ts` | `openclaw models` — model configuration |
| `config-cli.ts` | `openclaw config` — config get/set/edit |
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
| `openclaw config` | Config get/set/edit/path |
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
| `openclaw gateway` | Gateway control (start/stop/restart/status/call/dev/discover) |
| `openclaw daemon` | Legacy service management alias |
| `openclaw logs` | View gateway logs |
| `openclaw system` | System events, heartbeat, presence |
| `openclaw models` | Model configuration (list/set/status/scan/auth) |
| `openclaw approvals` | Exec approval management |
| `openclaw nodes` | Node management (status/pair/camera/screen/canvas/invoke/notify/location) |
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

### Internal Dependencies
- `src/config` — config loading, paths, types, schema
- `src/infra` — env, dotenv, errors, runtime-guard, path-env, unhandled-rejections, home-dir
- `src/commands` — actual command implementations
- `src/plugins` — plugin CLI registration
- `src/channels` — channel registry, plugins
- `src/agents` — agent scope, defaults
- `src/routing` — session key parsing
- `src/logging` — console capture, subsystem logging
- `src/runtime` — default runtime
- `src/version` — VERSION constant

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
| `daemon-cli.coverage.e2e.test.ts` | Daemon CLI E2E |
| `deps.test.ts` | CLI dependency checks |
| `dns-cli.test.ts` | DNS CLI commands |
| `exec-approvals-cli.test.ts` | Exec approvals CLI |
| `gateway-cli.coverage.e2e.test.ts` | Gateway CLI E2E |
| `gateway-cli/discover.test.ts` | Gateway discovery |
| `gateway-cli/run-loop.test.ts` | Gateway run loop |
| `gateway.sigterm.e2e.test.ts` | Gateway SIGTERM handling |
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
| `program.smoke.e2e.test.ts` | CLI smoke tests |
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

### File Inventory (261 files)

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
| `auth-choice.apply.google-antigravity.ts` | Google Antigravity auth |
| `auth-choice.apply.google-gemini-cli.ts` | Gemini CLI auth |
| `auth-choice.apply.huggingface.ts` | HuggingFace auth |
| `auth-choice.apply.minimax.ts` | MiniMax auth |
| `auth-choice.apply.oauth.ts` | Generic OAuth auth |
| `auth-choice.apply.openai.ts` | OpenAI auth |
| `auth-choice.apply.openrouter.ts` | OpenRouter auth |
| `auth-choice.apply.plugin-provider.ts` | Plugin provider auth |
| `auth-choice.apply.qwen-portal.ts` | Qwen Portal auth |
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
| `status-all.ts` | Comprehensive status (`--all`) |
| `status-all/agents.ts` | Agent status section |
| `status-all/channels.ts` | Channel status section |
| `status-all/diagnosis.ts` | Diagnostic section |
| `status-all/format.ts` | Status-all formatting |
| `status-all/gateway.ts` | Gateway status section |
| `status-all/report-lines.ts` | Report line builders |
| `health.ts` | `healthCommand()` — health check |
| `health-format.ts` | Health output formatting |
| `gateway-status.ts` | Gateway-specific status |
| `gateway-status/helpers.ts` | Gateway status helpers |
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
| `docs.ts` | Documentation browser |
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
- `src/runtime` — default runtime
- `src/version` — VERSION

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

### File Inventory (165 files)

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
| `zod-schema.ts` | **Main Zod schema definition** (~666 lines) |
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
| `agent-limits.ts` | Agent concurrency limits (DEFAULT_AGENT_MAX_CONCURRENT=3, DEFAULT_SUBAGENT_MAX_CONCURRENT=5) |
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
│   ├── web.fetch                 # Web fetch config (maxChars, firecrawl)
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
├── talk                          # TalkConfig (ElevenLabs TTS)
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
- `src/channels/registry.js` — CHANNEL_IDS
- `src/channels/chat-type.js` — ChatType enum
- `src/agents/defaults.js` — DEFAULT_CONTEXT_TOKENS
- `src/agents/model-selection.js` — parseModelRef
- `src/infra/dotenv.js` — loadDotEnv
- `src/infra/home-dir.js` — resolveRequiredHomeDir
- `src/infra/shell-env.js` — shell env fallback
- `src/version.js` — VERSION
- `src/logging/subsystem.js` — logging
- `src/utils.js` — resolveUserPath

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

### File Inventory (223 files, grouped by domain)

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
| `provider-usage.fetch.antigravity.ts` | Google Antigravity usage |
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

### File Inventory (36 files)

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

### File Inventory (12 files)

| File | Description |
|------|-------------|
| `onboarding.ts` | Main wizard orchestration |
| `onboarding.types.ts` | Wizard flow types |
| `onboarding.finalize.ts` | Finalization step (write config, install daemon) |
| `onboarding.gateway-config.ts` | Gateway configuration step |
| `onboarding.completion.ts` | Completion message/summary |
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
| `onboarding.test.ts` | Main wizard flow |
| `onboarding.completion.test.ts` | Completion output |
| `onboarding.gateway-config.test.ts` | Gateway config step |
| `session.test.ts` | Wizard session state |

---

## 7. src/tui

### Module Overview

The TUI module implements a **terminal-based chat interface** for interacting with OpenClaw agents. Built on `@mariozechner/pi-tui`, it provides a rich terminal UI with message history, syntax highlighting, command handling, and real-time streaming.

**Architecture Pattern:** Component-based TUI with event handlers, a gateway WebSocket chat client, and reactive state.

### File Inventory (39 files)

| File | Description |
|------|-------------|
| `tui.ts` | Main TUI entry: `createTui()`, editor submit handler |
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

### File Inventory (14 files)

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

---

*Analysis complete. 952 TypeScript files across 8 modules analyzed.*

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
