# Documentation Integrity Audit: v2026.4.1

**Audit date:** 2026-04-02
**Source repo:** `~/src/open_claw/openclaw/` (package.json version: `2026.4.1-beta.1`)
**Docs repo:** `~/src/open_claw/openclaw-docs/`
**Docs snapshot:** ARCHITECTURE.md dated 2026-04-01, baseline `v2026.3.31`

---

## 1. Source Code Verification

### Version

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| Released baseline: `v2026.3.31` | package.json: `2026.4.1-beta.1` | STALE -- docs reference v2026.3.31 but local source has moved to 2026.4.1-beta.1. Docs explicitly note "v2026.3.31" baseline so this is expected lag, not an error. |

### Extension Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| "Extension directories: 45" (Section 15, v2026.3.23 stats) | 93 items in `extensions/` (90 excluding AGENTS.md, CLAUDE.md symlink, shared) | OUTDATED -- docs say 45 dirs at v2026.3.23. The v2026.3.31 stats section says "91 extension packages". Actual package.json count: **84 extension packages**. The "91" claim is close but slightly overstated. |
| "80+ extension packages" (v2026.3.28 stats) | 84 extensions with package.json | ACCURATE |

### Channel Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| "Channel plugins: 21" (Section 11, Extension Types table) | 22 channel-type extensions found: bluebubbles, discord, feishu, googlechat, imessage, irc, line, matrix, mattermost, msteams, nextcloud-talk, nostr, qqbot, signal, slack, synology-chat, telegram, tlon, twitch, whatsapp, zalo, zalouser | UNDERSTATED -- docs say 21 but source has **22** channel extensions. `zalouser` appears to be missing from the documented count. The docs list mentions `zalo` but not `zalouser` as a distinct channel. |

### Skill Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| "Bundled skills: 52" (v2026.3.7 stats) | 53 directories in `skills/` | SLIGHTLY UNDERSTATED -- 53 vs documented 52. |
| "65 bundled skills" (v2026.3.31 stats) | 53 currently in tree | OVERSTATED -- docs claim 65 at v2026.3.31 but only 53 present. Skills may have been removed or reorganized since the docs snapshot. |

### Source File Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| "Total source files: 2,600+ .ts (non-test)" (Section 15) | 3,141 non-test .ts files in src/ | UNDERSTATED -- actual count is ~3,141, significantly above the documented 2,600+. This is from the v2026.3.23 stats baseline; the codebase has grown. |
| "Total test files: 1,600+" (Section 15) | 2,149 test files in src/ | UNDERSTATED -- actual is 2,149, above documented 1,600+. |
| "8,607 TypeScript files" (v2026.3.31 stats, full repo) | 81,804 .ts files across entire repo | CANNOT DIRECTLY COMPARE -- the 81,804 includes node_modules and build artifacts. The 8,607 figure appears to be scoped to `src/`, `extensions/`, `ui/`, `test/`, `scripts/` excluding dependencies. Plausible but not verifiable without matching the exact scope. |
| "Total src/ .ts files: 5,290" | 5,290 .ts files (test + non-test) in src/ | ACCURATE for src/ total. |

### Module File Counts (Section 3 vs Actual)

| Module | Docs claim (total .ts) | Actual total .ts | Docs non-test estimate | Actual non-test .ts | Status |
|--------|----------------------|------------------|----------------------|-------------------|--------|
| agents/ | 1,074 | 1,074 | 370 source | 547 | TOTAL MATCHES; source-only count in table (370) is understated vs actual 547 |
| auto-reply/ | 373 | 373 | 185 source | 252 | TOTAL MATCHES; source-only understated |
| gateway/ | 441 | 441 | 200 source | 251 | TOTAL MATCHES; source-only understated |
| config/ | 285 | 285 | 125 source | -- | MATCHES |
| plugins/ | 273 | 273 | -- | -- | MATCHES |
| security/ | 29 (Section 3) | 37 | -- | -- | UNDERSTATED -- docs say 29, actual 37. |

### Top-Level src/ Directory Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| "70 directories" (Section 15) | 51 directories (excluding src/ itself) | OVERSTATED -- docs say 70 but only 51 subdirectories exist in src/. Some modules listed in docs (e.g. `providers/`, `web-search/`) are present, plus newer ones like `tasks/`, `mcp/`, `secrets/`, `flows/`, `bootstrap/`, `chat/`, `i18n/`, `interactive/`, `image-generation/`, `bindings/`, `docs/`, `scripts/`, `generated/`, `test-helpers/`, `test-utils/` that are not individually documented. Total still falls short of 70. |

### Tasks Subsystem

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| "Shared background-task control plane (src/tasks/)" described in v2026.3.31 section | 26 files in src/tasks/ including: task-executor.ts, task-registry.ts, task-registry.store.sqlite.ts, task-owner-access.ts, task-status.ts, and associated tests | CONFIRMED -- tasks subsystem exists with SQLite-backed registry, audit, maintenance, ownership, and status surfaces as documented. |

### Tool Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| "75+ tools" (Section 2, agents diagram) | Tool files found: exec, browser, web_search, web_fetch, image, memory, tts, sessions (multiple), message, nodes, cron, canvas, gateway, diffs, plus SDK tools (read, write, edit, process) and plugin tools | PLAUSIBLE -- difficult to get exact count without running the tool registry, but the combination of ~15 core OpenClaw tools + SDK tools + plugin tools + channel-specific tools could reasonably reach 75+. The "40+ tools" claim in Step 9c of the message lifecycle is also plausible for the core set. |

---

## 2. Online Verification

### GitHub Repository

| Claim | Online finding | Status |
|-------|---------------|--------|
| Repo: github.com/openclaw/openclaw | Confirmed -- active repo, MIT license | CONFIRMED |
| Description: "Multi-channel AI gateway" | package.json matches; GitHub description confirmed | CONFIRMED |
| Stars | Web search reports 345,569 stars, 68,690 forks | NOT CLAIMED IN DOCS (docs do not make star count claims) |
| Creator: Peter Steinberger (PSPDFKit founder) | Confirmed via multiple sources | NOT CLAIMED IN DOCS |

### Documentation Site

| Claim | Online finding | Status |
|-------|---------------|--------|
| docs.openclaw.ai | Confirmed active and serving documentation | CONFIRMED |
| openclaw.ai (homepage) | Confirmed active | CONFIRMED |
| openclaw-ai.com/en/docs (additional docs) | Confirmed active | NOT REFERENCED IN DOCS |

### npm Package

| Claim | Online finding | Status |
|-------|---------------|--------|
| Package name: `openclaw` | Confirmed on npmjs.com | CONFIRMED |
| Latest version: 2026.4.1 | npm shows 2026.4.1 published ~15 hours ago | CONFIRMED (matches source) |
| License: MIT | Confirmed | CONFIRMED |
| 86 dependents in npm registry | Confirmed | NOT CLAIMED IN DOCS |

### Security Incidents (not in docs, but relevant)

- Malicious npm package `@openclaw-ai/openclawai` deploying RAT was discovered March 3, 2026 and removed March 10 -- not mentioned in ARCHITECTURE.md (appropriate, as this is external).
- `@shadanai/openclaw` fork compromised via supply chain attack -- not in docs (appropriate).
- NVIDIA NemoClaw announced March 16, 2026 for running OpenClaw in NVIDIA OpenShell -- not in docs (appropriate, third-party project).

---

## 3. Cross-Reference of 5 Key Claims

### Claim 1: Channel Count

- **Docs say:** "Channel plugins: 21" with examples: telegram, discord, slack, signal, whatsapp, line, irc, matrix, msteams, nostr, twitch, zalo
- **Source has:** 22 channel-type extension directories
- **Verdict:** UNDERSTATED by 1. The `zalouser` extension is not counted. Additionally, the npm description lists more channels than the docs table (GoogleChat, Nextcloud Talk, Synology Chat, BlueBubbles, Tlon, QQ Bot, Feishu are all present in source but not all listed in the docs channel count table).

### Claim 2: Extension/Plugin Count

- **Docs say (v2026.3.31 stats):** "91 extension packages"
- **Source has:** 84 extensions with package.json
- **Verdict:** OVERSTATED by ~7. The 91 figure may have included packages that have since been consolidated or removed in the 2026.4.1-beta.1 tree.

### Claim 3: Security Model Claims

- **Docs claim:** SSRF protection (private IPs, link-local, cloud metadata), timing-safe auth, exec approval workflows, tool policies, filesystem permission hardening, sandbox Docker support
- **Source confirms:** `security/` has 37 files covering audit, external-content wrapping, dangerous-tools detection, secret-equal (timing-safe), fix (permission hardening), skill-scanner, safe-regex, channel-metadata, windows-acl. SSRF code lives in `infra/net/`. Exec approval in `infra/exec-approvals*` and `gateway/exec-approval-manager*`.
- **Verdict:** CONFIRMED -- all claimed security features have corresponding source modules.

### Claim 4: Tool Count ("75+ tools")

- **Docs claim:** 75+ tools in the tool registry, including exec, browser, web, memory, message, cron, canvas, nodes, diffs
- **Source has:** Core tools in `src/agents/tools/` (web-search, web-fetch, image-tool, message-tool, nodes-tool, cron-tool, canvas-tool, session-status-tool, sessions-send-tool, sessions-list-tool, sessions-history-tool, gateway-tool), plus exec (bash-tools.exec.ts), plus SDK tools from pi-coding-agent, plus plugin-contributed tools, plus channel-specific tools (channel-tools.ts), plus MCP bridge tools
- **Verdict:** PLAUSIBLE -- 12+ core OpenClaw tools visible in source + SDK tools (read, write, edit, exec/process, glob, grep, etc.) + plugin tools (diffs, memory-core, memory-lancedb, llm-task, etc.) could reach 75+. Cannot confirm exact number without runtime introspection.

### Claim 5: Provider Claims

- **Docs claim:** Supports Anthropic, OpenAI, Google (Gemini incl. 3.1), xAI, AWS Bedrock, Azure, Ollama, Together, Venice, HuggingFace, MiniMax, Qwen, Volcengine/BytePlus, OpenCode/Zen, GitHub Copilot, Cloudflare AI Gateway, Chutes, Mistral, SiliconFlow, and OpenAI-compatible APIs
- **Source has extension dirs for:** anthropic, anthropic-vertex, amazon-bedrock, openai, google, xai, ollama, together, venice, huggingface, minimax, modelstudio (Qwen), volcengine, byteplus, opencode, opencode-go, github-copilot, copilot-proxy, cloudflare-ai-gateway, chutes, mistral, groq, deepseek, nvidia, microsoft, microsoft-foundry, kilocode, kimi-coding, moonshot, openrouter, perplexity, litellm, vercel-ai-gateway, qianfan, sglang, vllm, fal, zai, xiaomi, lobster
- **Verdict:** CONFIRMED and UNDERSTATED -- source has significantly more providers than documented (groq, deepseek, nvidia, perplexity, openrouter, qianfan, fal, xiaomi, etc. are present in extensions but not listed in the provider summary).

---

## 4. Coverage Assessment

### Well-Documented Areas

- Security model and hardening history (very thorough, versioned)
- Module catalog with file counts and dependencies
- Data flows (message lifecycle, cron, tool execution)
- Configuration architecture and schema
- Plugin system architecture
- Channel abstraction and per-channel features

### Gaps and Drift

| Area | Issue |
|------|-------|
| **Stats are stale** | Section 15 "Cross-Module Statistics" still references v2026.3.23 counts. The v2026.3.31 stats at the bottom are closer but still diverge from the 2026.4.1-beta.1 source (e.g., 91 vs 84 extension packages, 65 vs 53 skills). |
| **Module catalog incomplete** | src/ has 51 directories but only ~30 are documented in the Module Catalog table. Missing: `bindings/`, `bootstrap/`, `chat/`, `docs/`, `flows/`, `generated/`, `i18n/`, `image-generation/`, `interactive/`, `mcp/`, `scripts/`, `secrets/`, `test-helpers/`, `test-utils/`. Some are internal/test infra, but `mcp/`, `i18n/`, `image-generation/`, `secrets/` appear substantive. |
| **Channel count stale** | Extension Types table says 21 channels; source has 22. Several channels mentioned in online npm description (GoogleChat, Zalo Personal) are not in the table. |
| **Provider list incomplete** | Many provider extensions in source (groq, deepseek, nvidia, perplexity, openrouter, qianfan, fal, xiaomi, lobster, zai) are not listed in the Supported Providers paragraph. |
| **Dependency graph file counts** | The ASCII dependency graph shows "agents/ (720 files)" and "auto-reply/ (260 files)" but the Module Catalog says 1,074 and 373 respectively. The graph uses older numbers. |
| **"70 directories" claim** | Section 15 says "70 directories" for src/ modules but only 51 exist. |

### Discrepancy Summary

| Severity | Count | Examples |
|----------|-------|---------|
| **High (materially wrong)** | 2 | src/ directory count (70 vs 51); skills count (65 vs 53) |
| **Medium (outdated numbers)** | 4 | Extension package count, channel count, source file counts, dependency graph file numbers |
| **Low (minor omissions)** | 3 | Missing module catalog entries, unlisted providers, unlisted channels |
| **Confirmed accurate** | 8+ | Module file totals (agents, auto-reply, gateway, config, plugins), security model, tasks subsystem, tool architecture, provider auth flows |

---

## 5. Recommendations

1. **Update Section 15 stats** to reflect v2026.4.1 numbers: ~3,141 non-test source files, ~2,149 test files, 51 src/ directories, 84 extension packages, 53 bundled skills.
2. **Update the Extension Types table** to show 22 channel plugins (add zalouser or consolidate with zalo).
3. **Add missing providers** to the Supported Providers paragraph: Groq, DeepSeek, NVIDIA, Perplexity, OpenRouter, Qianfan, Fal, Xiaomi, Lobster, Zai.
4. **Update the dependency graph** ASCII art file counts to match current Module Catalog numbers.
5. **Add Module Catalog entries** for `mcp/`, `i18n/`, `image-generation/`, `secrets/`, `flows/`, `chat/`, `bindings/`, `bootstrap/`.
6. **Correct the "70 directories" claim** in Section 15 to 51.
