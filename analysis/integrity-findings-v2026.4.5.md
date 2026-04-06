# Documentation Integrity Audit: v2026.4.5

**Audit date:** 2026-04-06
**Source repo:** `~/src/open_claw/openclaw/` (tag: `v2026.4.5`)
**Docs repo:** `~/src/open_claw/openclaw-docs/`
**Docs snapshot:** Published docs snapshot `v2026.4.5-2`, validated against upstream `v2026.4.5`

---

## 1. Source Code Verification

### Version

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| Released baseline: `v2026.4.5` | git tag `v2026.4.5` confirmed, published 2026-04-06 UTC | CONFIRMED |
| Window: `v2026.4.2..v2026.4.5` | 2,562 commits, 5,746 files changed, +331,486 / -268,299 lines | CONFIRMED via `git rev-list --count` and `git diff --shortstat` |

### TypeScript File Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| "9,766 TypeScript files" | 9,766 `.ts` + `.tsx` files in `src/`, `extensions/`, `ui/`, `test/`, `scripts/` at tag `v2026.4.5` | CONFIRMED |
| "1,893,328 lines of TypeScript" | 1,893,328 total lines across the same scope at tag `v2026.4.5` | CONFIRMED |

### Extension Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| "94 extension packages" | 94 `package.json` files under `extensions/` at tag `v2026.4.5` | CONFIRMED |

### Skill Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| "73 released skill entrypoints" | 73 `SKILL.md` entrypoints total: 53 under `skills/`, 8 under `.agents/skills/`, and 12 under `extensions/` | CONFIRMED |

### Released Tree Layout Changes

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| Released memory surfaces are split across `src/memory-host-sdk/` and `extensions/memory-core/src/memory/` | `src/memory-host-sdk/` exists with 87 `.ts` / `.tsx` files, `extensions/memory-core/src/memory/` exists in the released tree, and the older standalone `src/memory/` path is absent on `v2026.4.5` | CONFIRMED |
| Released media-generation surfaces include `src/music-generation/`, `src/video-generation/`, and `src/media-generation/` | All three directories confirmed present in the release tree | CONFIRMED |
| Standalone `src/providers/` module no longer exists | `src/providers/` absent in `v2026.4.5`; provider auth/runtime logic is distributed elsewhere | CONFIRMED |

### Key Release Claims Verified Against Source

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| Legacy public config aliases removed from the documented surface | Upstream `CHANGELOG.md` `2026.4.5` Breaking section confirms the alias removals and `openclaw doctor --fix` migration path (#60726) | CONFIRMED |
| Built-in `music_generate` and `video_generate` tools | Upstream `CHANGELOG.md` `2026.4.5` Changes section confirms both tools and their provider/runtime surfaces | CONFIRMED |
| Embedded ACPX runtime and generic `reply_dispatch` hook | Upstream `CHANGELOG.md` `2026.4.5` Changes section confirms ACPX runtime embedding and `reply_dispatch` hook | CONFIRMED |
| OpenAI commentary buffered until `final_answer` | Upstream `CHANGELOG.md` `2026.4.5` Fixes section confirms commentary buffering and related reply-delivery hardening | CONFIRMED |

---

## 2. Online Verification

### GitHub Repository

| Claim | Online finding | Status |
|-------|---------------|--------|
| Repo: github.com/openclaw/openclaw | Active repo reachable over HTTPS | CONFIRMED |
| Tag `v2026.4.5` published | Tag visible via GitHub remote refs | CONFIRMED |

### npm Package

| Claim | Online finding | Status |
|-------|---------------|--------|
| Package: `openclaw` | Available on npmjs.com | CONFIRMED |
| Latest: `2026.4.5` | `npm view openclaw version` returns `2026.4.5` | CONFIRMED |

### Documentation Site

| Claim | Online finding | Status |
|-------|---------------|--------|
| `docs.openclaw.ai` | Live site returns HTTP 200 | CONFIRMED |

---

## 3. Corrections Applied in This Docs Snapshot

| Previous docs state | Corrected in `v2026.4.5-2` docs |
|---------------------|----------------------------------|
| Latest documented upstream release was `v2026.4.2` | Updated to `v2026.4.5` |
| TypeScript inventory stated `8,658` files / `1,710,796` lines | Updated to `9,766` files / `1,893,328` lines |
| Extension inventory stated `84` packages | Updated to `94` packages |
| Skills inventory stated `190 bundled skills` | Corrected to `73 released skill entrypoints` with explicit scope |
| Memory docs assumed `src/memory/` as current release path | Updated to document the `v2026.4.5` split between `src/memory-host-sdk/` and `extensions/memory-core/src/memory/` |
| Architecture docs still implied a standalone `src/providers/` module | Updated to note that standalone `src/providers/` is absent on `v2026.4.5` |

---

## 4. Coverage Assessment

### Well-Documented Areas

- Latest released baseline and changelog window updated to `v2026.4.5`
- Release-window facts (commit/file/line delta) updated to the current stable tag
- README, CHANGELOG, AGENTS, and AGENT_README updated to the `v2026.4.5` release line
- Analysis docs updated with `v2026.4.5` metadata and release-window notes
- Inventory counts corrected to the current released tree

### Residual Risk

| Area | Note | Severity |
|------|------|----------|
| Historical deep-dive sections | Some older deep-dive sections still describe pre-`v2026.4.5` internals for historical context; current-path corrections are called out in the release-window notes and architecture overview | LOW |

---

## 5. Methodology

- Version/tag verification via local git tags plus GitHub remote tag listing
- Window stats via `git rev-list --count v2026.4.2..v2026.4.5` and `git diff --shortstat v2026.4.2 v2026.4.5`
- TypeScript counts via release-tree file enumeration in `src/`, `extensions/`, `ui/`, `test/`, and `scripts/`
- Line counts via per-file line summation against the `v2026.4.5` tree
- Extension counts via `extensions/*/package.json`
- Skill counts via `SKILL.md` entrypoint enumeration across `skills/`, `.agents/skills/`, and `extensions/`
- Online verification via npm registry, GitHub remote refs, and live HTTP checks to `docs.openclaw.ai`
