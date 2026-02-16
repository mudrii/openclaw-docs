# OpenClaw Documentation

Comprehensive codebase documentation for [OpenClaw](https://github.com/openclaw/openclaw) — the open-source AI agent platform.

**Current version: v2026.2.15**

## What's Here

This repo provides deep analysis of the OpenClaw codebase, designed for both human contributors and AI agents working on the project.

### Core Documents

| Document | Description |
|----------|-------------|
| [DEVELOPER-REFERENCE.md](DEVELOPER-REFERENCE.md) | **Start here.** Practical reference for making code changes — dependency maps, critical paths, change impact matrix, testing guide, pre-PR checklist, gotchas. |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System architecture overview — module catalog, data flow diagrams, dependency graph, security model, design patterns. |
| [CHANGELOG-v2026.2.15.md](CHANGELOG-v2026.2.15.md) | Changelog for v2026.2.14 → v2026.2.15 (880 commits, 12 security fixes, 11 features). |

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
- `v2026.2.15` (2026-02-16) — Updated for v2026.2.15: 7 security hardening fixes, Discord Components v2 UI tool, nested subagent orchestration (depth 2, max 5 children), plugin LLM input/output hooks, per-channel ackReaction config, major channel deduplication refactor, 10 analysis files updated
- `v2026.2.14` — Initial release, documenting OpenClaw v2026.2.14

When a new OpenClaw version is released, the documentation is re-analyzed and a new version is tagged.

## Stats

- **~2,980 TypeScript files** analyzed (reduced from ~3,051 via massive refactoring/consolidation)
- **~527,000+ lines of code** covered
- **50+ modules** documented
- **~420KB** of documentation

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
