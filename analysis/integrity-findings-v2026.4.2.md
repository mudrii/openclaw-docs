# Documentation Integrity Audit: v2026.4.2

**Audit date:** 2026-04-03
**Source repo:** `~/src/open_claw/openclaw/` (tag: `v2026.4.2`)
**Docs repo:** `~/src/open_claw/openclaw-docs/`
**Docs snapshot:** All documents updated to `v2026.4.2` baseline

---

## 1. Source Code Verification

### Version

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| Released baseline: `v2026.4.2` | git tag `v2026.4.2` confirmed, published 2026-04-02 UTC | CONFIRMED |
| Window: `v2026.4.1..v2026.4.2` | 284 commits, 882 files changed, +51,562 / -9,916 lines | CONFIRMED via `git log --oneline \| wc -l` and `git diff --shortstat` |

### TypeScript File Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| "8,658 TypeScript files" | 8,658 .ts + .tsx files in `src/`, `extensions/`, `ui/`, `test/`, `scripts/` (excluding node_modules) at tag `v2026.4.2` | CONFIRMED |
| "1,710,796 lines of TypeScript" | 1,710,796 total lines via `wc -l` across same scope at tag `v2026.4.2` | CONFIRMED |

### Extension Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| "84 extension packages" | 84 package.json files in `extensions/` (excluding node_modules) at tag `v2026.4.2` | CONFIRMED |

### Skill Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| "190 bundled skills" | 190 directories matching `*/skills/*/SKILL.md` pattern (excluding node_modules) at tag `v2026.4.2` | CONFIRMED |

### Task Flow Registry (new in v2026.4.2)

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| Task Flow substrate in `src/tasks/` with managed-vs-mirrored sync, durable state, `openclaw tasks flow` CLI | Files confirmed: task-flow-registry.ts, task-flow-registry.types.ts, task-flow-registry.store.ts, task-flow-registry.store.sqlite.ts, task-flow-registry.paths.ts, task-flow-registry.audit.ts, task-flow-registry.maintenance.ts, task-flow-runtime-internal.ts | CONFIRMED |

### Web Fetch Runtime Boundary (new in v2026.4.2)

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| New fetch-provider boundary at `src/web-fetch/runtime.ts` | File confirmed present at tag `v2026.4.2` | CONFIRMED |

### New Agent Modules (new in v2026.4.2)

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| `src/agents/exec-approval-result.ts` | File confirmed (93 lines) | CONFIRMED |
| `src/agents/internal-runtime-context.ts` | File confirmed (177 lines) | CONFIRMED |

### Breaking Changes

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| xAI x_search moved to `plugins.entries.xai.config.xSearch.*` | CHANGELOG.md confirms (#59674) | CONFIRMED |
| Firecrawl web_fetch moved to `plugins.entries.firecrawl.config.webFetch.*` | CHANGELOG.md confirms (#59465) | CONFIRMED |

---

## 2. Online Verification

### GitHub Repository

| Claim | Online finding | Status |
|-------|---------------|--------|
| Repo: github.com/openclaw/openclaw | Active repo, MIT license | CONFIRMED |
| Tag v2026.4.2 published | Available on GitHub | CONFIRMED |

### npm Package

| Claim | Online finding | Status |
|-------|---------------|--------|
| Package: `openclaw` | Available on npmjs.com | CONFIRMED |
| Latest: 2026.4.2 | Published 2026-04-02 | CONFIRMED |

### Documentation Site

| Claim | Online finding | Status |
|-------|---------------|--------|
| docs.openclaw.ai | Active and serving | CONFIRMED |

---

## 3. Key Changes Verified Against Source

### Provider Transport Centralization

- **Claim:** Centralized request auth, proxy, TLS, header shaping across HTTP/stream/websocket
- **Evidence:** 6 provider fix entries in CHANGELOG.md (#59682, #59644, #59542, #59469, #59433, #59608) covering Copilot, Anthropic, OpenAI-compatible, streaming headers, media HTTP, and transport policy
- **Verdict:** CONFIRMED — major provider infrastructure refactoring

### Exec Defaults Change

- **Claim:** Gateway/node host exec default to YOLO mode (security=full, ask=off)
- **Evidence:** CHANGELOG.md entry under Changes confirms the default shift
- **Verdict:** CONFIRMED — significant security posture change for gateway/node exec

### Plugin Hooks

- **Claim:** `before_agent_reply` hook added
- **Evidence:** CHANGELOG.md (#20067)
- **Verdict:** CONFIRMED

### Compaction Opt-in Notice

- **Claim:** `agents.defaults.compaction.notifyUser` added
- **Evidence:** CHANGELOG.md (#54251)
- **Verdict:** CONFIRMED

---

## 4. Discrepancy Resolution from v2026.4.1 Audit

| Previous Issue | Resolution in v2026.4.2 Docs |
|----------------|------------------------------|
| Extension packages: 91 vs actual 84 | Updated to 84 — RESOLVED |
| Skills: 65 vs actual | Updated to 190 — RESOLVED (major growth from 65→190) |
| TypeScript files: 8,607 | Updated to 8,658 — RESOLVED |
| TypeScript lines: 1,655,809 | Updated to 1,710,796 — RESOLVED |

---

## 5. Coverage Assessment

### Well-Documented Areas

- All v2026.4.2 breaking changes documented
- Task Flow substrate comprehensively covered
- Provider transport centralization documented
- Exec defaults change and security implications noted
- All 15+ channel-specific fixes documented in CHANGELOG

### Remaining Gaps (carried forward)

| Area | Issue | Severity |
|------|-------|----------|
| Provider list | Some newer providers still not fully listed in architecture overview | LOW |
| Module catalog | Some src/ directories not in Module Catalog table | LOW |

---

## 6. Methodology

- File counts measured at git tag `v2026.4.2` (checked out tag, counted, returned to main)
- Line counts via `find ... -print0 | xargs -0 cat | wc -l`
- Commit/file stats via `git log --oneline | wc -l` and `git diff --shortstat`
- CHANGELOG entries cross-referenced with source file existence
- Online verification via npm registry and GitHub
