# Repository Guidelines

- Repo: https://github.com/openclaw/openclaw
- GitHub issues/comments/PR comments: use literal multiline strings or `-F - <<'EOF'` (or $'...') for real newlines; never embed "\\n".
- GitHub comment footgun: never use `gh issue/pr comment -b "..."` when body contains backticks or shell chars. Always use single-quoted heredoc (`-F - <<'EOF'`) so no command substitution/escaping corruption.
- GitHub linking footgun: don’t wrap issue/PR refs like `#24643` in backticks when you want auto-linking. Use plain `#24643` (optionally add full URL).
- Security advisory analysis: before triage/severity decisions, read `SECURITY.md` to align with OpenClaw's trust model and design boundaries.

## Project Structure & Module Organization

- Source code: `src/` (CLI wiring in `src/cli`, commands in `src/commands`, shared web channel pieces in `src/channels/web/index.ts`, infra in `src/infra`, gateway in `src/gateway`, context engine plugin slot in `src/context-engine/`, MCP bridge in `src/mcp/`, released task control plane in `src/tasks/`, released web search runtime in `src/web-search/`, released web fetch runtime in `src/web-fetch/`, released memory host/runtime helpers in `src/memory-host-sdk/`, and released media-generation surfaces in `src/image-generation/`, `src/music-generation/`, `src/video-generation/`, and `src/media-generation/`). Memory search/QMD/sync surfaces for the current stable line live in `extensions/memory-core/src/memory/`. Browser automation for the current stable line lives in `extensions/browser/`. Notable v2026.3.12+ additions: `src/agents/tools/sessions-yield-tool.ts` (cooperative turn-ending primitive) and `src/agents/pi-extensions/compaction-instructions.ts` (per-agent compaction language continuity). v2026.3.28+: `src/mcp/` (channel-server, channel-bridge, channel-tools — gateway-backed MCP bridge for Codex/Claude channel tool access). v2026.4.1+: bundled SearXNG web search provider plugin in `extensions/searxng/`; Bedrock Guardrails in `extensions/amazon-bedrock/`; `/tasks` chat-native task board; Feishu Drive comment-event flow with `feishu_drive` actions. v2026.4.2+: Task Flow Registry (`src/tasks/task-flow-registry.*` — managed orchestration with durable flow state, revision tracking, and `openclaw tasks flow` inspection/recovery); Web Fetch Runtime (`src/web-fetch/runtime.ts` — fetch-provider boundary for `web_fetch` fallback routing); `src/agents/exec-approval-result.ts` (typed approval outcomes); `src/agents/internal-runtime-context.ts` (internal agent runtime context). v2026.4.9+: built-in `music_generate` and `video_generate`; bundled Comfy media workflows in `extensions/comfy/`; dreaming/embeddings under `src/memory-host-sdk/` plus QMD/search orchestration in `extensions/memory-core/src/memory/`; structured plan updates in the agent runtime; and embedded ACPX runtime ownership plus `reply_dispatch` hook support. v2026.4.10+: Active Memory plugin, codex provider-owned model routing, `openclaw exec-policy`, and `commands.list` RPC. v2026.4.11+: Active Media Wiki/dreaming UI imports, QA lane additions (`openclaw qa matrix`/`telegram`), and webchat embedding/media formatting updates. v2026.4.26+: bundled Cerebras provider in `extensions/cerebras/`; Tencent Yuanbao external channel as community plugin (`openclaw-plugin-yuanbao`); `extensions/coven/` removed (replaced by opt-in default-off ACP runtime bridge); plugin manifests now own provider routing tables, model-id normalization, endpoint host metadata, and catalog alias/suppression rules (no more bundled-provider routing tables in core); `openclaw migrate` CLI subcommand; `openclaw matrix encryption setup` command; raw config diff panel in Control UI.
- Tests: colocated `*.test.ts`.
- Docs: `docs/` (images, queue, Pi config). Built output lives in `dist/`.
- Plugins/extensions: live under `extensions/*` (workspace packages). Keep plugin-only deps in the extension `package.json`; do not add them to the root `package.json` unless core uses them.
- Compatibility shims: `packages/` workspace contains legacy package name shims (clawdbot, moltbot) for downstream consumers that still reference old package names.
- Plugins: install runs `npm install --omit=dev` in plugin dir; runtime deps must live in `dependencies`. Avoid `workspace:*` in `dependencies` (npm install breaks); put `openclaw` in `devDependencies` or `peerDependencies` instead (runtime resolves `openclaw/plugin-sdk` via jiti alias).
- Installers served from `https://openclaw.ai/*`: live in the sibling repo `../openclaw.ai` (`public/install.sh`, `public/install-cli.sh`, `public/install.ps1`).
- Messaging channels: always consider **all** built-in + extension channels when refactoring shared logic (routing, allowlists, pairing, command gating, onboarding, docs).
  - Core channel docs: `docs/channels/`
  - Shared channel core: `src/channels`, `src/routing`, `src/web`, `src/line`
  - Released channel/plugin implementations: `extensions/*` (for example `extensions/telegram`, `extensions/discord`, `extensions/slack`, `extensions/signal`, `extensions/whatsapp`, `extensions/imessage`, `extensions/feishu`, `extensions/matrix`, `extensions/msteams`, `extensions/zalo`, `extensions/zalouser`, `extensions/voice-call`)
- When adding channels/extensions/apps/docs, update `.github/labeler.yml` and create matching GitHub labels (use existing channel/extension label colors).

## Docs Linking (Mintlify)

- Docs are hosted on Mintlify (docs.openclaw.ai).
- Internal doc links in `docs/**/*.md`: root-relative, no `.md`/`.mdx` (example: `[Config](/configuration)`).
- When working with documentation, read the mintlify skill.
- Section cross-references: use anchors on root-relative paths (example: `[Hooks](/configuration#hooks)`).
- Doc headings and anchors: avoid em dashes and apostrophes in headings because they break Mintlify anchor links.
- When Peter asks for links, reply with full `https://docs.openclaw.ai/...` URLs (not root-relative).
- When you touch docs, end the reply with the `https://docs.openclaw.ai/...` URLs you referenced.
- README (GitHub): keep absolute docs URLs (`https://docs.openclaw.ai/...`) so links work on GitHub.
- Docs content must be generic: no personal device names/hostnames/paths; use placeholders like `user@gateway-host` and “gateway host”.

## Docs i18n (zh-CN)

- `docs/zh-CN/**` is generated; do not edit unless the user explicitly asks.
- Pipeline: update English docs → adjust glossary (`docs/.i18n/glossary.zh-CN.json`) → run `scripts/docs-i18n` → apply targeted fixes only if instructed.
- Translation memory: `docs/.i18n/zh-CN.tm.jsonl` (generated).
- See `docs/.i18n/README.md`.
- The pipeline can be slow/inefficient; if it’s dragging, ping @jospalmbier on Discord instead of hacking around it.

## exe.dev VM ops (general)

- Access: stable path is `ssh exe.dev` then `ssh vm-name` (assume SSH key already set).
- SSH flaky: use exe.dev web terminal or Shelley (web agent); keep a tmux session for long ops.
- Update: `sudo npm i -g openclaw@latest` (global install needs root on `/usr/lib/node_modules`).
- Config: use `openclaw config set ...`; ensure `gateway.mode=local` is set.
- Discord: store raw token only (no `DISCORD_BOT_TOKEN=` prefix).
- Restart: stop old gateway and run:
  `pkill -9 -f openclaw-gateway || true; nohup openclaw gateway run --bind loopback --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &`
- Verify: `openclaw channels status --probe`, `ss -ltnp | rg 18789`, `tail -n 120 /tmp/openclaw-gateway.log`.

## Build, Test, and Development Commands

- Runtime baseline: Node **22.14+** (keep Node + Bun paths working; Node 24 is the default for fresh installs; minimum floor is Node 22.14.0).
- Install deps: `pnpm install`
- If deps are missing (for example `node_modules` missing, `vitest not found`, or `command not found`), run the repo’s package-manager install command (prefer lockfile/README-defined PM), then rerun the exact requested command once. Apply this to test/build/lint/typecheck/dev commands; if retry still fails, report the command and first actionable error.
- Pre-commit hooks: `prek install` (runs same checks as CI)
- Also supported: `bun install` (keep `pnpm-lock.yaml` + Bun patching in sync when touching deps/patches).
- Prefer Bun for TypeScript execution (scripts, dev, tests): `bun <file.ts>` / `bunx <tool>`.
- Run CLI in dev: `pnpm openclaw ...` (bun) or `pnpm dev`.
- Node remains supported for running built output (`dist/*`) and production installs.
- Mac packaging (dev): `scripts/package-mac-app.sh` defaults to current arch. Release checklist: `docs/platforms/mac/release.md`.
- Type-check/build: `pnpm build`
- TypeScript checks: `pnpm tsgo` (workspace binary command; run after `pnpm install`)
- Lint/format: `pnpm check`
- Format check: `pnpm format:check` (oxfmt --check)
- Format fix: `pnpm format:fix` (oxfmt --write)
- Tests: `pnpm test` (vitest); coverage: `pnpm test:coverage`

## Coding Style & Naming Conventions

- Language: TypeScript (ESM). Prefer strict typing; avoid `any`.
- Formatting/linting via Oxlint and Oxfmt; run `pnpm check` before commits.
- Never add `@ts-nocheck` and do not disable `no-explicit-any`; fix root causes and update Oxlint/Oxfmt config only when required.
- Never share class behavior via prototype mutation (`applyPrototypeMixins`, `Object.defineProperty` on `.prototype`, or exporting `Class.prototype` for merges). Use explicit inheritance/composition (`A extends B extends C`) or helper composition so TypeScript can typecheck.
- If this pattern is needed, stop and get explicit approval before shipping; default behavior is to split/refactor into an explicit class hierarchy and keep members strongly typed.
- In tests, prefer per-instance stubs over prototype mutation (`SomeClass.prototype.method = ...`) unless a test explicitly documents why prototype-level patching is required.
- Add brief code comments for tricky or non-obvious logic.
- Keep files concise; extract helpers instead of “V2” copies. Use existing patterns for CLI options and dependency injection via `createDefaultDeps`.
- Aim to keep files under ~700 LOC; guideline only (not a hard guardrail). Split/refactor when it improves clarity or testability.
- Naming: use **OpenClaw** for product/app/docs headings; use `openclaw` for CLI command, package/binary, paths, and config keys.

## Release Channels (Naming)

- stable: tagged releases only (e.g. `vYYYY.M.D`), npm dist-tag `latest`.
- beta: prerelease tags `vYYYY.M.D-beta.N`, npm dist-tag `beta` (may ship without macOS app).
- beta naming: prefer `-beta.N`; do not mint new `-1/-2` betas. Legacy `vYYYY.M.D-<patch>` and `vYYYY.M.D.beta.N` remain recognized.
- dev: moving head on `main` (no tag; git checkout main).

## Testing Guidelines

- Framework: Vitest with V8 coverage thresholds (70% lines/branches/functions/statements).
- Naming: match source names with `*.test.ts`; e2e in `*.e2e.test.ts`.
- Run `pnpm test` (or `pnpm test:coverage`) before pushing when you touch logic.
- Do not set test workers above 16; tried already.
- If local Vitest runs cause memory pressure (common on non-Mac-Studio hosts), use `OPENCLAW_TEST_PROFILE=serial OPENCLAW_TEST_SERIAL_GATEWAY=1 pnpm test` for land/gate runs.
- Live tests (real keys): `OPENCLAW_LIVE_TEST=1 pnpm test:live` (OpenClaw-only) or `LIVE=1 pnpm test:live` (includes provider live tests). Docker: `pnpm test:docker:live-models`, `pnpm test:docker:live-gateway`. Onboarding Docker E2E: `pnpm test:docker:onboard`.
- Full kit + what’s covered: `docs/help/testing.md`.
- Changelog: in maintainer workflow, changelog updates are required for every PR (including internal/test-only changes). Use `(#<PR>)` and `thanks @<author>` when author metadata is available.
- Mobile: before using a simulator, check for connected real devices (iOS + Android) and prefer them when available.

## Commit & Pull Request Guidelines

**Maintainer PR workflow (required):** Use `https://github.com/openclaw/maintainers/blob/main/.agents/skills/PR_WORKFLOW.md` as the source of truth. Execute `review-pr` -> `prepare-pr` -> `merge-pr` in order, with a maintainer checkpoint between stages.

- Script-first contract: use wrapper scripts (`scripts/pr-review`, `scripts/pr-prepare`, `scripts/pr-merge`) in normal maintainer flow; manual low-level commands are for debugging only.
- Required maintainer artifacts: `.local/pr-meta.json`, `.local/pr-meta.env`, `.local/review.md`, `.local/review.json`, `.local/prep-context.env`, `.local/prep.md`, and `.local/prep.env`.
- Create commits with `scripts/committer "<msg>" <file...>`; avoid manual `git add`/`git commit` so staging stays scoped.
- Follow concise, action-oriented commit messages (e.g., `CLI: add verbose flag to send`).
- Group related changes; avoid bundling unrelated refactors.
- Before substantive review or prep, rebase the PR branch onto current `main` and resolve conflicts.
- Run gates before merge: `pnpm build`, `pnpm check`, and `pnpm test` unless docs-only criteria explicitly allow skipping heavy lanes.
- Resolve all `BLOCKER` and `IMPORTANT` review findings before merge.
- Merge only when required checks are green and the branch is up to date with `main`.
- Do not continue workflow if the problem cannot be validated or no meaningful verification path exists.
- Keep PRs focused, describe what/why, and for AI-assisted PRs include AI marker, testing depth, prompts/session logs when possible, and explicit confirmation you understand the code.
- For new feature/architecture work, start with GitHub Discussion before implementation.
- PR submission template (canonical): `.github/pull_request_template.md`
- Issue submission templates (canonical): `.github/ISSUE_TEMPLATE/`

## Shorthand Commands

- `sync`: if working tree is dirty, commit all changes (pick a sensible Conventional Commit message), then `git pull --rebase`; if rebase conflicts and cannot resolve, stop; otherwise `git push`.

## Git Notes

- If `git branch -d/-D <branch>` is policy-blocked, delete the local ref directly: `git update-ref -d refs/heads/<branch>`.
- Bulk PR close/reopen safety: if a close action would affect more than 5 PRs, first ask for explicit user confirmation with the exact PR count and target scope/query.

## GitHub Search (`gh`)

- Prefer targeted keyword search before proposing new work or duplicating fixes.
- Use `--repo openclaw/openclaw` + `--match title,body` first; add `--match comments` when triaging follow-up threads.
- PRs: `gh search prs --repo openclaw/openclaw --match title,body --limit 50 -- "auto-update"`
- Issues: `gh search issues --repo openclaw/openclaw --match title,body --limit 50 -- "auto-update"`
- Structured output example:
  `gh search issues --repo openclaw/openclaw --match title,body --limit 50 --json number,title,state,url,updatedAt -- "auto update" --jq '.[] | "\(.number) | \(.state) | \(.title) | \(.url)"'`

## Security & Configuration Tips

- **`requireApproval` in `before_tool_call` hooks (v2026.3.28+):** plugins can now gate tool execution via an async `requireApproval` contract. Approval requests route through exec overlay, Telegram, Discord, and the `/approve` command.
- **LINE timing-safe HMAC (v2026.3.28+):** LINE webhook signature verification uses constant-time comparison. Do not roll custom HMAC comparison for LINE — use the shared `security/secret-equal.ts` helper.
- **Extended web search key audit (v2026.3.28+):** `openclaw security audit` now covers Gemini, xAI, Kimi, Moonshot, and OpenRouter API key exposure, in addition to existing providers. Run after any provider config change.
- **`gateway.auth.mode` is now required** when both `gateway.auth.token` and `gateway.auth.password` are configured. Omitting it causes a startup error (introduced v2026.3.7). Run `openclaw doctor --fix` to auto-migrate.
- **Exec YOLO defaults (v2026.4.2+):** gateway/node host exec now defaults to YOLO mode (`security=full`, `ask=off`). Host approval-file fallbacks and docs/doctor reporting align with that no-prompt default. Review `exec-approvals.json` and `openclaw doctor` output after upgrading if you rely on approval prompts.
- **xAI `x_search` config migration (v2026.4.2):** `tools.web.x_search.*` moved to `plugins.entries.xai.config.xSearch.*`; auth standardized on `plugins.entries.xai.config.webSearch.apiKey` / `XAI_API_KEY`. Run `openclaw doctor --fix` to migrate.
- **Firecrawl `web_fetch` config migration (v2026.4.2):** `tools.web.fetch.firecrawl.*` moved to `plugins.entries.firecrawl.config.webFetch.*`; `web_fetch` fallback now routes through the fetch-provider boundary (`src/web-fetch/runtime.ts`). Run `openclaw doctor --fix` to migrate.
- **Task Flow Registry (v2026.4.2+):** managed orchestration substrate with durable flow state/revision tracking and `openclaw tasks flow` inspection/recovery. Task Flow has its own test surface (`task-flow-registry.test.ts`, `task-flow-registry.maintenance.test.ts`, `task-flow-registry.audit.test.ts`, `task-flow-owner-access.test.ts`). Plugins can drive managed Task Flows via the bound `api.runtime.taskFlow` seam.
- Web provider stores creds at `~/.openclaw/credentials/`; rerun `openclaw channels login --channel web` if logged out.
- Pi sessions live under `~/.openclaw/sessions/` by default; the base directory is not configurable.
- Environment variables: see `~/.profile`. Container runtime: set `OPENCLAW_CONTAINER` or use CLI `--container` flag for Docker/Podman container commands.
- Never commit or publish real phone numbers, videos, or live configuration values. Use obviously fake placeholders in docs, tests, and examples.
- Release flow: always read `docs/reference/RELEASING.md` and `docs/platforms/mac/release.md` before any release work; do not ask routine questions once those docs answer them.

## Behavioral Changes in v2026.4.22–v2026.4.26

### v2026.4.26
- **`extensions/coven/` removed (BREAKING):** the bundled Coven extension is deleted. Coven now lives behind an opt-in default-off ACP runtime bridge — operators must remove all `extensions/coven/` references from their configs and explicitly opt in via the new ACP runtime bridge to keep Coven workflows.
- **Plugin manifest is now the source of truth for provider routing:** model-id normalization, endpoint host metadata, OpenAI-compatible request-family hints, model-catalog aliases/suppressions, OpenAI stale Spark suppression, and reusable startup metadata snapshots moved into plugin manifests. Adding a new provider is a manifest-only change; core no longer carries bundled-provider routing tables.
- **Plugin config write API deprecated:** direct plugin config load/write helpers are deprecated in favor of passed runtime snapshots plus transactional mutation helpers with explicit restart-follow-up policy. Scanner guardrails, runtime warnings, and revision-based cache invalidation now apply.
- **`agents.subagents.allowAgents` enforced for explicit `sessions_spawn(agentId=...)`:** the previous auto-allow behavior on requester self-targets is gone (#72827). Explicit cross-agent spawns now respect the same allowlist as channel-routed calls. Failures fall closed instead of guessing a child binding.
- **`acp.dispatch.enabled=false` no longer blocks explicit ACP bootstraps (#63591):** explicit `sessions_spawn(runtime="acp")` turns are allowed even when automatic ACP thread dispatch is disabled. Code paths that gated all ACP execution on `acp.dispatch.enabled` need to distinguish explicit-runtime spawn from auto-dispatch.
- **Local OpenAI-compatible providers default to Chat Completions and trust loopback (#40024):** custom providers configured with only `baseUrl` now default to the Chat Completions adapter and automatically trust loopback model requests, fixing local proxies that were timing out without `/v1/chat/completions`.
- **Memory/OpenAI-compatible asymmetric embedding controls:** `memorySearch.inputType`, `queryInputType`, and `documentInputType` config now route to provider-specific embedding endpoints, including direct query embeddings and provider batch indexing (carries forward #63313, #60727).
- **Ollama retrieval-query prefixes:** model-specific retrieval-query prefixes for `nomic-embed-text`, `qwen3-embedding`, and `mxbai-embed-large` apply to memory-search queries while leaving document batches unchanged (carries forward #45013).
- **`agents.defaults.compaction.maxActiveTranscriptBytes` (opt-in):** new preflight trigger runs normal local compaction when the active JSONL grows too large, requiring transcript rotation so successful compaction moves future turns onto a smaller successor file. There is no default value — operators opt in explicitly.
- **`OPENCLAW_PLUGIN_STAGE_DIR` layered roots (#72396):** can now contain layered runtime-dependency roots so read-only preinstalled deps resolve before installing missing deps into the final writable root.
- **Cerebras as bundled first-party provider:** new `extensions/cerebras/` provider plugin with onboarding, static model catalog, manifest-owned `cerebras-native` endpoint class (host `api.cerebras.ai`, base URL `https://api.cerebras.ai/v1`, `enabledByDefault: true`). API key env: `CEREBRAS_API_KEY`. First concrete consumer of the manifest provider-catalog architecture.
- **`openclaw migrate` CLI subcommand:** new top-level migration command with plan, dry-run, JSON output, pre-migration backup, onboarding detection, archive-only reports, and importers for Claude Code/Desktop and Hermes (config, memory/plugin hints, model providers, MCP servers, skills, commands, supported credentials).
- **`openclaw matrix encryption setup`:** new single-shot E2EE bootstrap command that enables encryption, bootstraps recovery, and prints verification status.
- **Channel ergonomics:** Discord/Slack/Mattermost `user:`/`channel:` target syntax surfaced in the shared message target schema and Discord ambiguity errors (#72401). `openclaw nodes remove --node <id|name|ip>` and `node.pair.remove` for cleaning stale gateway-owned node pairing records.
- **`gateway.health` runtime-backed state:** snapshots and cached refreshes now preserve live runtime-backed channel/account state. Raw probe payloads are kept on sensitive/admin paths only.
- **CLI/update install path:** `openclaw update` now installs npm globals into a verified temporary prefix and atomically swaps the package tree, preventing mixed old/new installs. `OPENCLAW_NO_AUTO_UPDATE=1` is now an explicit gateway startup kill-switch (#72715). Windows-startup-launched gateway restarts after `openclaw update`.
- **Validated `--wrapper`/`OPENCLAW_WRAPPER` install path (#69400):** `openclaw gateway install` accepts a wrapper that persists executable LaunchAgent/systemd wrappers across forced reinstalls, updates, and doctor repairs.
- **Logging/OTEL trace propagation (#40353):** internal request trace scopes propagate through Gateway HTTP requests and WebSocket frames so file logs, diagnostic events, agent run traces, model-call traces, OTEL spans, and trusted provider `traceparent` headers all share a correlatable `traceId`. Top-level `hostname`, flattened `message`, `agent_id`, `session_id`, and `channel` fields land on file-log JSONL records (#51075). Configured redaction patterns now apply to persisted session transcript text (#42982).
- **Diagnostics/OTEL model-call payload metrics (#33832):** privacy-safe model-call request payload bytes, streamed response bytes, first-response latency, and total duration are captured in diagnostic events, plugin hooks, stability snapshots, and OTEL model-call spans/metrics — without logging raw model content.
- **Memory-core lifetime change (#59101):** one-shot memory CLI commands (`memory index`, `memory status --index`, `memory search`) now run through transient builtin and QMD managers instead of starting long-lived file watchers. This avoids hitting macOS `EMFILE` limits on large repos.
- **WebChat/`/new` and `/reset` (#72369):** bare `/new` and `/reset` startup instructions are kept out of visible chat history; `/reset <note>` remains user-visible.
- **Browser/CDP per-profile circuit breaker (#64271):** repeated managed Chrome launch failures per profile now circuit-break so browser requests stop spawning Chromium indefinitely when CDP cannot start.
- **Provider-native reasoning effort (#32638):** Groq and LM Studio declare provider-native reasoning effort values so Qwen thinking models receive `none`/`default` or `off`/`on` instead of OpenAI-only `low`/`medium`. Anthropic Opus 4.7/Sonnet 4.6 requests now strip trailing assistant prefill payloads when extended thinking is enabled (#72739).
- **Tasks/memory WAL bounded (#72774):** SQLite WAL sidecars on task, Task Flow, proxy capture, and builtin memory databases checkpoint and truncate on a timer and before close, bounding long-running gateway `*.sqlite-wal` growth.

### v2026.4.25
- **Cold plugin registry (breaking for break-glass users):** `OPENCLAW_DISABLE_PERSISTED_PLUGIN_REGISTRY` is deprecated. The cold registry (`plugins/installs.json`) is now the default startup path. Use `openclaw plugins registry --refresh` if the registry is stale.
- **`plugins.installs` no longer authored config:** managed install metadata is in the state-managed plugin index (`plugins/installs.json`), not in `openclaw.json`/`openclaw.yaml`.
- **GenAI OTel semantics:** `gen_ai.system` is the default attribute; `gen_ai.provider.name` requires `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`.

### v2026.4.23
- **Dreaming decoupled from heartbeat:** dreaming runs as an isolated lightweight agent turn regardless of heartbeat config. Run `openclaw doctor --fix` to migrate stale dreaming cron jobs to the new shape.
- **ACPX no longer writes `auth.json` bridge files** for Codex ACP, app-server, or CLI runs. Codex-owned runtimes use `CODEX_HOME`/`~/.codex` directly.

### v2026.4.22
- **OpenAI Codex onboarding:** `~/.codex` OAuth material is no longer imported from onboarding. Use browser login or device pairing instead.
- **`config set` mode flags:** `config set --merge` for additive updates, `config set --replace` for intentional clobbers. Previously, the default could clobber model maps accidentally.
- **Gateway config restore:** invalid/truncated config now triggers restore from last-known-good backup before crash-looping.

## v2026.4.21 Behavioral Changes

- **Default models updated (`v2026.4.15–v2026.4.21`):** Anthropic default is now `claude-opus-4-7`; Moonshot default is `kimi-k2.6`; OpenAI image generation default is `gpt-image-2`. Config that hardcodes the old defaults will continue to work but diverges from bundled behavior.
- **Cron state split (`v2026.4.20`):** mutable runtime state (last run, retry counts) moved to `jobs-state.json`; `jobs.json` is definitions-only. Code reading execution state from `jobs.json` will miss it.
- **Dreaming storage is now `"separate"` by default (`v2026.4.15`):** dreaming output goes to `memory/dreaming/{phase}/YYYY-MM-DD.md`. Set `plugins.entries.memory-core.config.dreaming.storage.mode: "inline"` to restore old behavior.
- **Unknown-tool stream guard on by default (`v2026.4.15`):** agents referencing unregistered tools will stop immediately. Plugins that register tools lazily must complete registration before the first agent turn.
- **`memory_get` restricted to canonical paths (`v2026.4.15`):** only reads `MEMORY.md`, `memory/**`, and active QMD workspace docs. Use `read` tool with tool-policy config for other workspace files.
- **SSRF hardening broadened (`v2026.4.21`):** Synology Chat file URLs, LINE media URLs, and Google Chat auth endpoints are now SSRF-validated. External content strips LLM special-token literals (Qwen/ChatML, Llama, Gemma, Mistral, Phi, GPT-OSS) to prevent role-boundary spoofing.
- **Skill Workshop plugin (`v2026.4.21`):** new `extensions/skill-workshop` plugin captures workflow corrections as workspace skills. Review config before enabling in production workspaces.
- **QQBot self-contained engine (`v2026.4.20`):** full QR-onboarding, native approval, per-account resource stacks, and credential backup now in `extensions/qqbot/`.
- **Channel startup performance (`v2026.4.21`):** Matrix saves ~1.8s, Telegram saves ~14s, Discord cuts ~98% of registration time via narrow sidecar and lazy-load patterns. Changes to these startup paths require regression testing on cold registration.

## v2026.4.14 Behavioral Changes

- **Codex/provider routing and compatibility (`v2026.4.14`):** add `gpt-5.4-pro` forward compatibility, stabilize Codex alias handling, and tighten model/provider follow-up behavior across OpenAI/Google-compatible paths.
- **Session/task reliability and QA coverage (`v2026.4.14`):** improve session-runtime status extraction, context-engine compaction sequencing, and qa-lab failure-recovery scenarios for broken turns and scenario catalog stability.
- **Media and provider transports (`v2026.4.14`):** refine media-understanding trusted-proxy behavior, fix Telegram media DNS/proxy edge paths, and keep media generation request/type handling stable for OCR/audio/image/video toolchains.
- **Browser and security hardening (`v2026.4.14`):** preserve strict SSRF compatibility while tightening loopback/CDP readiness, strict hostname handling, and outbound/proxy follow-through in hard security edge cases.
- **Security and execution controls (`v2026.4.14`):** harden shell-wrapper detection/approval timing behavior, tighten approval timeout handling, and prevent allowlist edge-cases from expanding execution policy.
- **Release operations and release integrity (`v2026.4.14`):** synchronized docs snapshot to upstream `v2026.4.14` and refreshed baselines for release verification, runtime artifact hashes, and high-signal release metadata.
- **Approval and routing behavior continue to evolve:** keep APNs/Matrix-native approval flows and `reply_dispatch` behavior in regression scope when changing callback, routing, or delivery logic.

## v2026.3.31 Breaking Changes

- **`nodes run` shell wrapper removed:** released node shell execution now goes through `exec host=node` and the `nodes invoke` surface. Do not document or reintroduce the old duplicated wrapper path.
- **Plugin/skill install scans fail closed:** dangerous-code `critical` findings and install-time scan failures now block installs by default. The only documented override is `--dangerously-force-unsafe-install`.
- **`trusted-proxy` auth tightened:** mixed shared-token configs are rejected, and local-direct fallback now requires the configured token instead of implicitly trusting same-host callers.
- **Node trust reduced:** declared node commands stay disabled until pairing approval, and node-originated runs stay on the reduced trusted surface even after pairing.
- **Plugin SDK compat shims deprecated:** current released guidance remains `openclaw/plugin-sdk/*`; older provider-compat and channel-runtime shims should be treated as migration-only.

## v2026.3.28 Breaking Changes (Historical)

- **Qwen:** `qwen-portal-auth` plugin removed. Migrate to `openclaw onboard --auth-choice modelstudio-api-key` (Model Studio API key flow).
- **Config/Doctor:** migrations older than 2 months are dropped; old keys that required those migrations now fail validation. Run `openclaw doctor --fix` before upgrading if configs have not been migrated recently.
- **`--claude-cli-logs` deprecated:** use `--cli-backend-logs` instead. The old flag will be removed in a future release.
- **MiniMax catalog reduced to M2.7 only:** M2, M2.1, M2.5, and VL-01 are no longer available. Update any agent or model configs that reference these model IDs.

## GHSA (Repo Advisory) Patch/Publish

- **v2026.3.12–v2026.3.13 context:** 20+ GHSAs were fixed across these two releases (v2026.3.12: GHSA-99qw, GHSA-pcqg, GHSA-9r3v, GHSA-f8r2, GHSA-r7vr, GHSA-rqpp, GHSA-vmhq, GHSA-2rqg, GHSA-wcxr, GHSA-2pwv, GHSA-jv4g, GHSA-xwx2, GHSA-6rph, GHSA-jf5v, GHSA-57jw, GHSA-jvqh, GHSA-x7pp, GHSA-jc5j, GHSA-g353, GHSA-m69h, GHSA-mhxh, GHSA-5m9r; v2026.3.13: exec approval hardening for pnpm/Perl/PowerShell/env-wrappers/macOS line continuation, single-use bootstrap pairing codes, external content ZWS stripping, Telegram webhook auth before body read, iMessage attachment path rejection). Before working on any exec/auth/provider/channel security issue, check if it overlaps with these advisory scopes.
- Before reviewing security advisories, read `SECURITY.md`.
- Fetch: `gh api /repos/openclaw/openclaw/security-advisories/<GHSA>`
- Latest npm: `npm view openclaw version --userconfig "$(mktemp)"`
- Private fork PRs must be closed:
  `fork=$(gh api /repos/openclaw/openclaw/security-advisories/<GHSA> | jq -r .private_fork.full_name)`
  `gh pr list -R "$fork" --state open` (must be empty)
- Description newline footgun: write Markdown via heredoc to `/tmp/ghsa.desc.md` (no `"\\n"` strings)
- Build patch JSON via jq: `jq -n --rawfile desc /tmp/ghsa.desc.md '{summary,severity,description:$desc,vulnerabilities:[...]}' > /tmp/ghsa.patch.json`
- GHSA API footgun: cannot set `severity` and `cvss_vector_string` in the same PATCH; do separate calls.
- Patch + publish: `gh api -X PATCH /repos/openclaw/openclaw/security-advisories/<GHSA> --input /tmp/ghsa.patch.json` (publish = include `"state":"published"`; no `/publish` endpoint)
- If publish fails (HTTP 422): missing `severity`/`description`/`vulnerabilities[]`, or private fork has open PRs
- Verify: re-fetch; ensure `state=published`, `published_at` set; `jq -r .description | rg '\\\\n'` returns nothing

## Troubleshooting

- Rebrand/migration issues or legacy config/service warnings: run `openclaw doctor` (see `docs/gateway/doctor.md`).

## Agent-Specific Notes

- Vocabulary: "makeup" = "mac app".
- Never edit `node_modules` (global/Homebrew/npm/git installs too). Updates overwrite. Skill notes go in `tools.md` or `AGENTS.md`.
- When adding a new `AGENTS.md` anywhere in the repo, also add a `CLAUDE.md` symlink pointing to it (example: `ln -s AGENTS.md CLAUDE.md`).
- Signal: "update fly" => `fly ssh console -a flawd-bot -C "bash -lc 'cd /data/clawd/openclaw && git pull --rebase origin main'"` then `fly machines restart e825232f34d058 -a flawd-bot`.
- When working on a GitHub Issue or PR, print the full URL at the end of the task.
- When answering questions, respond with high-confidence answers only: verify in code; do not guess.
- Never update the Carbon dependency.
- Any dependency with `pnpm.patchedDependencies` must use an exact version (no `^`/`~`).
- Patching dependencies (pnpm patches, overrides, or vendored changes) requires explicit approval; do not do this by default.
- CLI progress: use `src/cli/progress.ts` (`osc-progress` + `@clack/prompts` spinner); don’t hand-roll spinners/bars.
- Status output: keep tables + ANSI-safe wrapping (`src/terminal/table.ts`); `status --all` = read-only/pasteable, `status --deep` = probes.
- Gateway currently runs only as the menubar app; there is no separate LaunchAgent/helper label installed. Restart via the OpenClaw Mac app or `scripts/restart-mac.sh`; to verify/kill use `launchctl print gui/$UID | grep openclaw` rather than assuming a fixed label. **When debugging on macOS, start/stop the gateway via the app, not ad-hoc tmux sessions; kill any temporary tunnels before handoff.**
- macOS logs: use `./scripts/clawlog.sh` to query unified logs for the OpenClaw subsystem; it supports follow/tail/category filters and expects passwordless sudo for `/usr/bin/log`.
- If shared guardrails are available locally, review them; otherwise follow this repo's guidance.
- SwiftUI state management (iOS/macOS): prefer the `Observation` framework (`@Observable`, `@Bindable`) over `ObservableObject`/`@StateObject`; don’t introduce new `ObservableObject` unless required for compatibility, and migrate existing usages when touching related code.
- Connection providers: when adding a new connection, update every UI surface and docs (macOS app, web UI, mobile if applicable, onboarding/overview docs) and add matching status + configuration forms so provider lists and settings stay in sync.
- Version locations: `package.json` (CLI), `apps/android/app/build.gradle.kts` (versionName/versionCode), `apps/ios/Sources/Info.plist` + `apps/ios/Tests/Info.plist` (CFBundleShortVersionString/CFBundleVersion), `apps/macos/Sources/OpenClaw/Resources/Info.plist` (CFBundleShortVersionString/CFBundleVersion), `docs/install/updating.md` (pinned npm version), `docs/platforms/mac/release.md` (APP_VERSION/APP_BUILD examples), Peekaboo Xcode projects/Info.plists (MARKETING_VERSION/CURRENT_PROJECT_VERSION).
- "Bump version everywhere" means all version locations above **except** `appcast.xml` (only touch appcast when cutting a new macOS Sparkle release).
- **Restart apps:** “restart iOS/Android apps” means rebuild (recompile/install) and relaunch, not just kill/launch.
- **Device checks:** before testing, verify connected real devices (iOS/Android) before reaching for simulators/emulators.
- iOS Team ID lookup: `security find-identity -p codesigning -v` → use Apple Development (…) TEAMID. Fallback: `defaults read com.apple.dt.Xcode IDEProvisioningTeamIdentifiers`.
- A2UI bundle hash: `.bundle.hash` under `src/canvas-host/a2ui/` is auto-generated; ignore unexpected changes, and only regenerate via `pnpm canvas:a2ui:bundle` (or `scripts/bundle-a2ui.sh`) when needed. Commit the hash as a separate commit.
- Release signing/notary keys are managed outside the repo; follow internal release docs.
- Notary auth env vars (`APP_STORE_CONNECT_ISSUER_ID`, `APP_STORE_CONNECT_KEY_ID`, `APP_STORE_CONNECT_API_KEY_P8`) are expected in your environment (per internal release docs).
- **Multi-agent safety:** do **not** create/apply/drop `git stash` entries unless explicitly requested (this includes `git pull --rebase --autostash`). Assume other agents may be working; keep unrelated WIP untouched and avoid cross-cutting state changes.
- **Multi-agent safety:** when the user says "push", you may `git pull --rebase` to integrate latest changes (never discard other agents' work). When the user says "commit", scope to your changes only. When the user says "commit all", commit everything in grouped chunks.
- **Multi-agent safety:** do **not** create/remove/modify `git worktree` checkouts (or edit `.worktrees/*`) unless explicitly requested.
- **Multi-agent safety:** do **not** switch branches / check out a different branch unless explicitly requested.
- **Multi-agent safety:** running multiple agents is OK as long as each agent has its own session.
- **Multi-agent safety:** when you see unrecognized files, keep going; focus on your changes and commit only those.
- Lint/format churn:
  - If staged+unstaged diffs are formatting-only, auto-resolve without asking.
  - If commit/push already requested, auto-stage and include formatting-only follow-ups in the same commit (or a tiny follow-up commit if needed), no extra confirmation.
  - Only ask when changes are semantic (logic/data/behavior).
- Lobster seam: use the shared CLI palette in `src/terminal/palette.ts` (no hardcoded colors); apply palette to onboarding/config prompts and other TTY UI output as needed.
- **Multi-agent safety:** focus reports on your edits; avoid guard-rail disclaimers unless truly blocked; when multiple agents touch the same file, continue if safe; end with a brief “other files present” note only if relevant.
- Bug investigations: read source code of relevant npm dependencies and all related local code before concluding; aim for high-confidence root cause.
- Code style: add brief comments for tricky logic; keep files under ~500 LOC when feasible (split/refactor as needed).
- Tool schema guardrails (google-antigravity): avoid `Type.Union` in tool input schemas; no `anyOf`/`oneOf`/`allOf`. Use `stringEnum`/`optionalStringEnum` (Type.Unsafe enum) for string lists, and `Type.Optional(...)` instead of `... | null`. Keep top-level tool schema as `type: "object"` with `properties`.
- Tool schema guardrails: avoid raw `format` property names in tool schemas; some validators treat `format` as a reserved keyword and reject the schema.
- When asked to open a “session” file, open the Pi session logs under `~/.openclaw/agents/<agentId>/sessions/*.jsonl` (use the `agent=<id>` value in the Runtime line of the system prompt; newest unless a specific ID is given), not the default `sessions.json`. If logs are needed from another machine, SSH via Tailscale and read the same path there.
- Do not rebuild the macOS app over SSH; rebuilds must be run directly on the Mac.
- Never send streaming/partial replies to external messaging surfaces (WhatsApp, Telegram); only final replies should be delivered there. Streaming/tool events may still go to internal UIs/control channel.
- Voice wake forwarding tips:
  - Command template should stay `openclaw-mac agent --message "${text}" --thinking low`; `VoiceWakeForwarder` already shell-escapes `${text}`. Don’t add extra quotes.
  - launchd PATH is minimal; ensure the app’s launch agent PATH includes standard system paths plus your pnpm bin (typically `$HOME/Library/pnpm`) so `pnpm`/`openclaw` binaries resolve when invoked via `openclaw-mac`.
- For manual `openclaw message send` messages that include `!`, use the heredoc pattern noted below to avoid the Bash tool’s escaping.
- Release guardrails: do not change version numbers without operator’s explicit consent; always ask permission before running any npm publish/release step.

## NPM + 1Password (publish/verify)

- Use the 1password skill; all `op` commands must run inside a fresh tmux session.
- Sign in: `eval "$(op signin --account my.1password.com)"` (app unlocked + integration on).
- OTP: `op read 'op://Private/Npmjs/one-time password?attribute=otp'`.
- Publish: `npm publish --access public --otp="<otp>"` (run from the package dir).
- Verify without local npmrc side effects: `npm view <pkg> version --userconfig "$(mktemp)"`.
- Kill the tmux session after publish.

## Plugin Release Fast Path (no core `openclaw` publish)

- Release only already-on-npm plugins. Source list is in `docs/reference/RELEASING.md` under "Current npm plugin list".
- Run all CLI `op` calls and `npm publish` inside tmux to avoid hangs/interruption:
  - `tmux new -d -s release-plugins-$(date +%Y%m%d-%H%M%S)`
  - `eval "$(op signin --account my.1password.com)"`
- 1Password helpers:
  - password used by `npm login`:
    `op item get Npmjs --format=json | jq -r '.fields[] | select(.id=="password").value'`
  - OTP:
    `op read 'op://Private/Npmjs/one-time password?attribute=otp'`
- Fast publish loop (local helper script in `/tmp` is fine; keep repo clean):
  - compare local plugin `version` to `npm view <name> version`
  - only run `npm publish --access public --otp="<otp>"` when versions differ
  - skip if package is missing on npm or version already matches.
- Keep `openclaw` untouched: never run publish from repo root unless explicitly requested.
- Post-check for each release:
  - per-plugin: `npm view @openclaw/<name> version --userconfig "$(mktemp)"` should match the current stable release line (currently `2026.4.2`)
  - core guard: `npm view openclaw version --userconfig "$(mktemp)"` should stay at previous version unless explicitly requested.

## Changelog Release Notes

- When cutting a mac release with beta GitHub prerelease:
  - Tag `vYYYY.M.D-beta.N` from the release commit (example: `v2026.2.15-beta.1`).
  - Create prerelease with title `openclaw YYYY.M.D-beta.N`.
  - Use release notes from `CHANGELOG.md` version section (`Changes` + `Fixes`, no title duplicate).
  - Attach at least `OpenClaw-YYYY.M.D.zip` and `OpenClaw-YYYY.M.D.dSYM.zip`; include `.dmg` if available.

- Keep top version entries in `CHANGELOG.md` sorted by impact:
  - `### Changes` first.
  - `### Fixes` deduped and ranked with user-facing fixes first.
- Before tagging/publishing, run:
  - `node --import tsx scripts/release-check.ts`
  - `pnpm release:check`
  - `pnpm test:install:smoke` or `OPENCLAW_INSTALL_SMOKE_SKIP_NONROOT=1 pnpm test:install:smoke` for non-root smoke path.
