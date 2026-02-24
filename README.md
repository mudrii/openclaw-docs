# OpenClaw Documentation

Comprehensive codebase documentation for [OpenClaw](https://github.com/openclaw/openclaw) — the open-source AI agent platform.

**Current source version: 2026.2.23-beta.1 (git cafa8226d)**

## What's Here

This repo provides deep analysis of the OpenClaw codebase, designed for both human contributors and AI agents working on the project.

### Core Documents

| Document | Description |
|----------|-------------|
| [DEVELOPER-REFERENCE.md](DEVELOPER-REFERENCE.md) | **Start here.** Practical reference for making code changes — dependency maps, critical paths, change impact matrix, testing guide, pre-PR checklist, gotchas. |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System architecture overview — module catalog, data flow diagrams, dependency graph, security model, design patterns. |
| [CHANGELOG.md](CHANGELOG.md) | Consolidated changelog for v2026.2.14 → v2026.2.23 (`v2026.2.23` currently UNRELEASED). |
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

### PR Reviews

Example PR analyses demonstrating how to use DEVELOPER-REFERENCE.md for code review:

| PR | Analysis |
|----|----------|
| [#16960](https://github.com/openclaw/openclaw/pull/16960) — perf: hook cache-busting | [pr-16960-analysis.md](pr-reviews/pr-16960-analysis.md) |
| [#16946](https://github.com/openclaw/openclaw/pull/16946) — fix: cron timer spin loop | [Opus review](pr-reviews/pr-16946-validation-opus.md) · [MiniMax review](pr-reviews/pr-16946-validation-minimax.md) |

## For AI Agents

Load `DEVELOPER-REFERENCE.md` before making any code changes to OpenClaw. It provides:

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

Each release is tagged to match the OpenClaw version it documents:
- `v2026.2.23` (2026-02-24) — Updated for v2026.2.22 → v2026.2.23 (UNRELEASED): Vercel AI Gateway Claude shorthand normalization, session key canonicalization, Telegram reactions/polling hardening, agent reasoning fixes (thinking-block leak, error classification), context overflow detection expansion (Chinese patterns, HTTP 502/503/504), auto-reply metadata fix, Slack group policy inheritance, Anthropic OAuth token beta injection fix, OpenRouter reasoning_effort conflict fix, Gateway/WS flood protection, Config/Write immutability + path traversal hardening, exec obfuscation detection, openai-image-gen stored XSS fix, OTEL credential redaction, Python skill packaging hardening + CI linting, Kilo Gateway provider, Moonshot video provider, session maintenance hardening.
- `v2026.2.22` (2026-02-24) — Updated for v2026.2.21 → v2026.2.22: new Synology Chat channel, new Mistral provider, grounded Gemini web search, full Control UI cron edit parity, memory FTS multilingual expansion (Spanish, Portuguese, Japanese, Korean, Arabic), optional auto-updater, 4 breaking changes (Google Antigravity removed, tool-failure verbosity, dmScope default, unified streaming config + device-auth v1 removed), 30+ security fixes (exec-approval bypass, SSRF hardening, symlink escape, auth hardening across every channel).
- `v2026.2.21` (2026-02-23) — Updated for v2026.2.19 → v2026.2.21: Gemini 3.1 + Volcengine/Doubao provider expansion, per-channel model overrides (`channels.modelByChannel`), kimi-coding implicit provider fix, Discord thread-bound subagents + voice + stream preview + lifecycle reactions, Telegram streaming config simplified + reasoning/answer lane split, cron `maxConcurrentRuns` fix, compaction safeguard production fix, SHA-1→SHA-256 synthetic ID migration, 40+ security fixes (heredoc allowlist bypass, shell startup-file injection, sandbox browser hardening, TTS provider-hop injection, compaction retry amplification, prototype-chain traversal in webhook templates, WhatsApp JID auth hardening).
- `v2026.2.19` (2026-02-20) — Updated for v2026.2.17 → v2026.2.19: massive security hardening (SSRF, exec safeBins, plugin integrity, gateway auth defaults), cron/heartbeat Telegram topic delivery fix, heartbeat skip-on-empty behavior, YAML 1.2 frontmatter schema, browser relay token auth, macOS LaunchAgent `TMPDIR` fix, and DEVELOPER-REFERENCE gotchas 33–45.
- `v2026.2.17` (2026-02-18) — Updated for v2026.2.15 → v2026.2.17: subagent spawn/announce behavior hardening, cron stagger defaults + CLI controls, config `$include` confinement hardening, Telegram forum topic creation action support, Z.AI `tool_stream` defaults, and DEVELOPER-REFERENCE gotcha refresh.
- `v2026.2.15` (2026-02-16) — Updated for v2026.2.15: 7 security hardening fixes, Discord Components v2 UI tool, nested subagent orchestration (depth 2, max 5 children), plugin LLM input/output hooks, per-channel ackReaction config, major channel deduplication refactor, 10 analysis files updated
- `v2026.2.14` — Initial release, documenting OpenClaw v2026.2.14

When a new OpenClaw version is released, the documentation is re-analyzed and a new version is tagged.

## Stats

- **4,311 TypeScript files** analyzed (`src/`, `extensions/`, `test/`, `scripts/`, `packages/`; current checkout)
- **771,630 lines of TypeScript** covered (same scope as above)
- **50+ modules** documented, **40 extension directories** (**31 extension packages**), **52 bundled skills**
- **~480KB** of documentation (5 core MD files + 10 analysis files)

## Contributing

Found an inaccuracy? File paths changed? Module structure updated? PRs welcome.

When updating, please:
1. Verify all file paths against the current OpenClaw source
2. Cross-reference with at least one other analysis file
3. Note which OpenClaw version you verified against

## License

MIT — see [LICENSE](LICENSE).

## Links

- [OpenClaw](https://github.com/openclaw/openclaw) — the project this documents
- [OpenClaw Docs](https://docs.openclaw.ai) — official documentation
- [OpenClaw Dashboard](https://github.com/mudrii/openclaw-dashboard) — monitoring dashboard
