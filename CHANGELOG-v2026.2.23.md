
# OpenClaw v2026.2.23 — Synthesis Review (Unreleased)

**Cycle:** v2026.2.22 → v2026.2.23 (Active Development)  
**Status:** UNRELEASED — tracking `main` branch  
**Scope:** Provider normalization, session hardening, reasoning fixes, security sweep  

---

## Executive Summary

v2026.2.23 is currently in active development. The primary feature addition is Vercel AI Gateway Claude shorthand normalization. The bulk of the release is fixes: session key canonicalization, Telegram stability (reactions, polling), agent reasoning improvements, context overflow detection expansion (including Chinese patterns), and a significant security sweep covering exec obfuscation detection, XSS prevention in skill HTML output, OTEL credential redaction, and Python skill packaging hardening.

---

## New Features

### Vercel AI Gateway Claude Shorthand Normalization
Accept Claude shorthand model refs (`vercel-ai-gateway/claude-*`) by normalizing to canonical Anthropic-routed model IDs.

**Before:** Claude shorthand refs would fail or route incorrectly through Vercel AI Gateway.  
**After:** `vercel-ai-gateway/claude-opus-4-6` normalizes automatically.

Thanks @sallyom, @markbooch, and @vincentkoc. (PR #23985)

### Google Vertex AI for Claude (v2026.2.22 backport note)
Claude models can route through Google Vertex AI — introduced in v2026.2.22 core, finalized in v2026.2.23 normalization pass.

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

Since this is unreleased, no migration steps are required yet. Changes to watch when it releases:

1. **Session keys:** If you have case-variant session keys in legacy configs, they will be automatically migrated to lowercase.
2. **Reasoning with thinking:** Models with `thinking=low` will no longer enable auto-reasoning by default — verify your reasoning-dependent workflows.
3. **Vercel AI Gateway:** Shorthand Claude refs now normalize automatically — no config change needed.
4. **OTEL users:** Review OTLP exports — credential fields are now redacted.

