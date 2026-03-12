# OpenClaw Documentation

Comprehensive codebase documentation for [OpenClaw](https://github.com/openclaw/openclaw) — the open-source AI agent platform.

**Latest upstream stable release: v2026.3.11 (published 2026-03-12 UTC).**
**Current validated docs snapshot: v2026.3.11-1 (docs rerelease for upstream tag `v2026.3.11`).**

**Scope policy:** this repository documents published releases only. It does not document unreleased `main` branch changes, betas, or speculative future behavior.

## What's Here

This repo provides deep analysis of the OpenClaw codebase, designed for both human contributors and AI agents working on the project.

### Core Documents

| Document | Description |
|----------|-------------|
| [AGENT_README.md](AGENT_README.md) | **Start here.** Practical reference for making code changes — dependency maps, critical paths, change impact matrix, testing guide, pre-PR checklist, gotchas. |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System architecture overview — module catalog, data flow diagrams, dependency graph, security model, design patterns. |
| [CHANGELOG.md](CHANGELOG.md) | Consolidated changelog for documented release windows (v2026.2.14 -> v2026.3.11); tracks released upstream tags and synthesis status. |
| [AGENTS.md](AGENTS.md) | AI agent guidelines for working with the OpenClaw codebase — repo structure, build commands, testing, PR workflow, security practices. |

### Detailed Analysis

| Module Cluster | Document |
|----------------|----------|
| Core Engine (agents, gateway, sessions, routing, hooks) | [analysis/core-engine.md](analysis/core-engine.md) |
| Agent System (PI runner, auto-reply, memory, cron) | [analysis/agent-system.md](analysis/agent-system.md) |
| Channels (Telegram, Discord, Signal, Slack, LINE, etc.) | [analysis/channels.md](analysis/channels.md) |
| CLI, Config & Infrastructure | [analysis/cli-config-infra.md](analysis/cli-config-infra.md) |
| CLI, Tools & Media | [analysis/cli-tools-media.md](analysis/cli-tools-media.md) |
| Gateway, Config & Infra (deep dive) | [analysis/gateway-config-infra.md](analysis/gateway-config-infra.md) |
| Memory, Cron & Media | [analysis/memory-cron-media.md](analysis/memory-cron-media.md) |
| Security, Web & Browser | [analysis/security-web-browser.md](analysis/security-web-browser.md) |
| Security & Plugins | [analysis/security-plugins.md](analysis/security-plugins.md) |
| Utilities & Support (auto-reply, logging, pairing) | [analysis/utils-support.md](analysis/utils-support.md) |

## For AI Agents

Load `AGENT_README.md` before making any code changes to OpenClaw. It provides:

- **Module dependency map** with risk levels (high/medium/low blast radius)
- **Critical paths** — exact file chains for message lifecycle, tool execution, cron, config loading
- **Change impact matrix** — "if you change X, you must test Y"
- **Pre-PR checklist** with common pitfalls
- **Gotchas & landmines** — things that look simple but aren't

### Effectiveness

Tested with dual-model validation (Opus + MiniMax M2.5):
- **Opus rated: 8/10** — estimated ~40% faster code review
- **MiniMax rated: 9/10** — estimated ~40% faster code review

Both models independently confirmed the reference doc significantly reduced time to understand module relationships and identify what to test.

## Versioning

Each documented release tracks an OpenClaw stable tag. Docs-only rereleases append a suffix like `-1`, `-2`, etc. while keeping the same validated upstream tag:
- `v2026.3.11` (2026-03-12, released) — Updated for v2026.3.8 → v2026.3.11 (235 commits, 977 files): multimodal memory indexing (images + audio) with `gemini-embedding-2-preview`, iOS Home canvas overhaul with live agent overview + docked toolbar, macOS chat model picker, first-class Ollama onboarding wizard (Local and Cloud + Local modes), OpenCode Go provider, Discord `autoArchiveDuration` for auto-created threads, Telegram HTML/final-preview delivery hardening, macOS remote onboarding shared-token detection, iOS TestFlight beta flow, ACP session resume via `resumeSessionId`, ACP `loadSession` session context replay, macOS/launchd v2 restart hardening (detached helper hand-off), node pending-work queue primitives, gateway runtime version in status, and 20+ security hardening fixes including GHSA-5wcw-8jjv-m286 (WebSocket cross-site hijack), symlink-safe secret reads, TAR extraction staging, exec SecretRef traversal rejection, fs-bridge write pinning, gateway auth fail-closed, session-reset auth split, plugin HTTP scope isolation, and session_status sandbox guards. **Breaking:** cron/doctor isolation — cron jobs can no longer send ad hoc notifications or fallback main-session summaries; run `openclaw doctor --fix` to migrate.
- `v2026.3.8` (2026-03-09, released) — Updated for v2026.3.7 → v2026.3.8 (260 commits, 769 files): `openclaw backup create|verify` with manifest validation and recovery-focused flags, ACP provenance metadata/receipts, top-level `talk.silenceTimeoutMs`, direct WebSocket CDP support with WSL2/remote relay fixes, Brave `llm-context` web search mode, Perplexity native-search vs OpenRouter compatibility split, bundled-channel plugin precedence during onboarding/update sync, launchd restart/repair hardening, and Android Play policy cutbacks (no self-update/background location/screen recording background capture). No new stable-tag breaking config was introduced, but restart semantics and Android/macOS operator workflows changed materially.
- `v2026.3.7` (2026-03-08, released) — Updated for v2026.3.2 → v2026.3.7 (893 commits, 2,412 files): ContextEngine plugin interface with full lifecycle hooks enabling alternative context management strategies (zero behavior change without config), durable ACP channel bindings surviving restarts, Telegram per-topic agent routing, Spanish locale in Control UI, Mattermost interactive model picker, Discord native slash commands, Gemini 3.1 Flash-Lite + GPT-5.4 support, Venice default `kimi-k2-5`, Docker slim/Podman support, systemd WSL2 hardening, config validation fail-closed, ZIP path traversal hardening, and SecretRef models.json persistence hardening. **Breaking:** explicit `gateway.auth.mode` required when both `gateway.auth.token` and `gateway.auth.password` are set.
- `v2026.3.2` (2026-03-03, released) — Updated for v2026.3.1 → v2026.3.2 (862 commits, 2,119 files): SecretRef and `openclaw secrets` planning/apply/audit expansion (64 runtime targets), first-class `pdf` tool + diffs PDF rendering support, `openclaw config validate` + invalid-key reporting, sessions/attachments support in `sessions_spawn`, Telegram streaming default `partial` + DM draft reliability fixes, plugin runtime/runtime-events/context enrichments (`session_start`, `session_end`, message hook phases), and high-signal security/reliability fixes across Feishu, LINE, browser, gateway, voice-call, and cron/message paths.
- `v2026.2.26` (2026-02-27, released) — Updated for v2026.2.25 → v2026.2.26 (340 commits, 856 files): external secrets management workflow (`openclaw secrets`), ACP thread-bound agents, new agent binding CLI, Codex WebSocket-first transport defaults, plugin-owned onboarding hooks, and multiple reliability fixes (DM allowlist inheritance, delivery-queue backoff, temp-dir safety, and Google Chat lifecycle stability).
- `v2026.2.25` (2026-02-26, released) — Updated for v2026.2.24 → v2026.2.25 (159 commits, 376 files): heartbeat direct-delivery policy is now explicit via `agents.defaults.heartbeat.directPolicy` (default `allow`), major security hardening across channel non-message ingress/auth paths and gateway/browser auth boundaries, improved subagent announce/delivery reliability, webhook/polling robustness improvements (especially Telegram), and model/fallback reliability fixes across OpenRouter and other providers.
- `v2026.2.24` (2026-02-25, released) — Updated for v2026.2.23 → v2026.2.24 (228 commits, 458 files): major heartbeat safety changes (DM target blocking + default target `none`), cross-channel shared-session routing fail-closed hardening, expanded multilingual stop/abort phrase matching, sandbox security hardening (container namespace join blocked by default, tmp/media/path guard tightening, hardlink protections), channel reliability fixes (typing keepalive, Discord voice DAVE recovery, WhatsApp 440 reconnect handling), provider/model fixes (OpenRouter cooldown bypass, allowlisted models despite stale catalog), and extensive security patch set across exec approvals, workspace boundaries, hooks normalization, and ingress authorization.
- `v2026.2.23` (2026-02-24, released) — Updated for v2026.2.22 → v2026.2.23: Vercel AI Gateway Claude shorthand normalization, session key canonicalization, Telegram reactions/polling hardening, agent reasoning fixes (thinking-block leak, error classification), context overflow detection expansion (Chinese patterns, HTTP 502/503/504), auto-reply metadata fix, Slack group policy inheritance, Anthropic OAuth token beta injection fix, OpenRouter reasoning_effort conflict fix, Gateway/WS flood protection, Config/Write immutability + path traversal hardening, exec obfuscation detection, openai-image-gen stored XSS fix, OTEL credential redaction, Python skill packaging hardening + CI linting, Kilo Gateway provider, Moonshot video provider, session maintenance hardening.
- `v2026.2.22` (2026-02-23) — Updated for v2026.2.21 → v2026.2.22: new Synology Chat channel, new Mistral provider, grounded Gemini web search, full Control UI cron edit parity, memory FTS multilingual expansion (Spanish, Portuguese, Japanese, Korean, Arabic), optional auto-updater, 4 breaking changes (Google Antigravity removed, tool-failure verbosity, dmScope default, unified streaming config + device-auth v1 removed), 30+ security fixes (exec-approval bypass, SSRF hardening, symlink escape, auth hardening across every channel).
- `v2026.2.21` (2026-02-21) — Updated for v2026.2.19 → v2026.2.21: Gemini 3.1 + Volcengine/Doubao provider expansion, per-channel model overrides (`channels.modelByChannel`), kimi-coding implicit provider fix, Discord thread-bound subagents + voice + stream preview + lifecycle reactions, Telegram streaming config simplified + reasoning/answer lane split, cron `maxConcurrentRuns` fix, compaction safeguard production fix, SHA-1→SHA-256 synthetic ID migration, 40+ security fixes (heredoc allowlist bypass, shell startup-file injection, sandbox browser hardening, TTS provider-hop injection, compaction retry amplification, prototype-chain traversal in webhook templates, WhatsApp JID auth hardening).
- `v2026.2.19` (2026-02-19) — Updated for v2026.2.17 → v2026.2.19: massive security hardening (SSRF, exec safeBins, plugin integrity, gateway auth defaults), cron/heartbeat Telegram topic delivery fix, heartbeat skip-on-empty behavior, YAML 1.2 frontmatter schema, browser relay token auth, macOS LaunchAgent `TMPDIR` fix, and DEVELOPER-REFERENCE gotchas 33–45.
- `v2026.2.17` (2026-02-18) — Updated for v2026.2.15 → v2026.2.17: subagent spawn/announce behavior hardening, cron stagger defaults + CLI controls, config `$include` confinement hardening, Telegram forum topic creation action support, Z.AI `tool_stream` defaults, and DEVELOPER-REFERENCE gotcha refresh.
- `v2026.2.15` (2026-02-16) — Updated for v2026.2.15: 7 security hardening fixes, Discord Components v2 UI tool, nested subagent orchestration (depth 2, max 5 children), plugin LLM input/output hooks, per-channel ackReaction config, major channel deduplication refactor, 10 analysis files updated
- `v2026.2.14` — Initial release, documenting OpenClaw v2026.2.14

When a new OpenClaw version is released, the documentation is re-analyzed and a new version is tagged. If the docs need factual corrections without a new upstream stable release, the docs repo publishes a suffixed rerelease tag for the same validated upstream tag.

## Stats

- **5,879 TypeScript files** analyzed (`src/`, `extensions/`, `ui/`, `test/`, `scripts/` — `.ts` + `.tsx`; `v2026.3.11` tag)
- **167,779 lines of TypeScript** covered (same scope as above)
- **49 modules** documented, **40 extension directories** (**33 extension packages**), **52 bundled skills**
- **~1.0MB** of documentation (5 core MD files + 10 analysis files)

## Contributing

Found an inaccuracy? File paths changed? Module structure updated? PRs welcome.

This docs repo follows OpenClaw contributor policy from:
- https://github.com/openclaw/openclaw/blob/main/CONTRIBUTING.md
- https://github.com/openclaw/maintainers/blob/main/.agents/skills/PR_WORKFLOW.md

When updating, please:
1. Verify all file paths against the current OpenClaw source
2. Cross-reference with at least one other analysis file
3. Note which OpenClaw version you verified against
4. Keep PR scope focused to one logical change
5. For maintainer-driven PRs, follow `review-pr` -> `prepare-pr` -> `merge-pr` in order; do not skip stages
6. Use script-first wrappers (`scripts/pr-review`, `scripts/pr-prepare`, `scripts/pr-merge`) and keep required `.local/*` workflow artifacts
7. rebase the PR branch onto current `main` before substantive review/prep work
8. Run `pnpm build && pnpm check && pnpm test` before PR (docs-only PRs may use docs-only CI criteria)
9. Ensure required CI checks are green, the branch is not behind `main`, and all `BLOCKER`/`IMPORTANT` findings are resolved before merge
10. Include a changelog update for maintainer workflow PRs (including internal/test-only changes), with `(#<PR>)` and `thanks @<author>` when available
11. Mark AI-assisted PRs clearly, describe testing depth, include prompts/session logs when possible, and confirm understanding of the code

For new features/architecture proposals in OpenClaw itself, start with a GitHub Discussion before implementation.

For security issues, report to the relevant repository security process, or email `security@openclaw.ai` if uncertain.

## License

MIT — see [LICENSE](LICENSE).

## Links

- [OpenClaw](https://github.com/openclaw/openclaw) — the project this documents
- [OpenClaw Docs](https://docs.openclaw.ai) — official documentation
- [OpenClaw Dashboard](https://github.com/mudrii/openclaw-dashboard) — monitoring dashboard
