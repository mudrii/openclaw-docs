# Synthesis Review: v2026.2.17 → v2026.2.19

> **Window analyzed:** `v2026.2.17..v2026.2.19` (~900+ commits)
> **Method:** behavior and operator/developer impact focus; ignore low-signal test/style churn
> **Package version confirmed:** `2026.2.19-2` (npm)

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

---

## Docs in this repo updated for this window

- `README.md` — version bump to v2026.2.19, changelog table updated.
- `DEVELOPER-REFERENCE.md` — v2026.2.19 workflow additions, new gotchas (items 33–45).
- `CHANGELOG-v2026.2.19.md` — this synthesized review.

---

*Generated: 2026-02-20*
