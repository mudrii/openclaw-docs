# Documentation Integrity Audit: v2026.4.9

**Audit date:** 2026-04-09
**Source repo:** `~/src/open_claw/openclaw/` (tag: `v2026.4.9`)
**Docs repo:** `~/src/open_claw/openclaw-docs/`
**Docs snapshot:** Published docs snapshot `v2026.4.9-1`, validated against upstream `v2026.4.9`

---

## 1. Source Code Verification

### Version

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| Released baseline: `v2026.4.9` | git tag `v2026.4.9` confirmed, published 2026-04-09 UTC | CONFIRMED |
| Window: `v2026.4.8..v2026.4.9` | 331 commits, 1,052 files changed, +40,773 / -24,675 lines | CONFIRMED via `git rev-list --count` and `git diff --shortstat` |

### TypeScript File Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| 10,454 TypeScript files | 10,454 `.ts` + `.tsx` files in `src/`, `extensions/`, `ui/`, `test/`, `scripts/` at tag `v2026.4.9` (via `git ls-tree -r v2026.4.9`) | CONFIRMED |

### Extension Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| 97 extension packages | 97 `package.json` files under `extensions/` at tag `v2026.4.9` | CONFIRMED |
| 109 extension directories | 109 top-level entries under `extensions/` at tag `v2026.4.9` | CONFIRMED |

### Skill Count

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| 75 released skill entrypoints | 75 `SKILL.md` entrypoints total under `skills/`, `.agents/skills/`, and `extensions/` at tag `v2026.4.9` | CONFIRMED |

### Key Release Claims Verified Against Source

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| MS Teams thread isolation via `replyToId` | `replyToId` found in `extensions/msteams/src/monitor-handler/message-handler.ts` (11 refs) and `message-handler.thread-session.test.ts` (8 refs), plus `conversation-store.ts`, `sdk.ts`, `sdk-types.ts` | CONFIRMED |
| Ollama thinking support enabled | `think=true`/`think=false` test cases found in `extensions/ollama/index.test.ts`; thinking level controls confirmed | CONFIRMED |
| Matrix legacy `trusted` policy migration | `trusted` migration code found in `extensions/matrix/src/doctor-contract.ts` (5 refs) and `extensions/matrix/src/cli.ts` (5 refs) | CONFIRMED |
| Dreaming grounded scene lane | Dreaming grounded scene code references present in extension tree at `v2026.4.9` | CONFIRMED |
| Plugin setup wizard skip option | Wizard explicit skip option code landed in `v2026.4.9` beta and confirmed in stable | CONFIRMED |
| OpenRouter model picker refs fix | OpenRouter model picker reference fix landed as `fix openrouter model picker refs (#63416)` in window `v2026.4.8..v2026.4.9` | CONFIRMED |

### v2026.4.7 Release Claims

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| Window: `v2026.4.5..v2026.4.7` | 1,663 commits, 4,916 files changed, +206,144 / -104,901 lines | CONFIRMED |
| Pluggable compaction provider registry (#56224) | `feat: add pluggable compaction provider registry (#56224)` in v2026.4.5..v2026.4.7 git log | CONFIRMED |
| `openclaw infer` CLI command (#62129) | `feat: Add first-class infer CLI for inference workflows (#62129)` in git log | CONFIRMED |
| Arcee AI provider | `feat: add Arcee AI provider plugin` in git log | CONFIRMED |
| Ollama vision auto-detection (#62193) | `feat(ollama): detect vision capability from /api/show (#62193)` in git log | CONFIRMED |
| Slack `thread.requireExplicitMention` (#58276) | `feat(slack): add thread.requireExplicitMention config option (#58276)` in git log | CONFIRMED |
| iOS gateway connection error UX (#62650) | `feat(ios): improve gateway connection error ux (#62650)` in git log | CONFIRMED |
| Discord cover images for events (#60883) | `feat: add cover image support to Discord event create (#60883)` in git log | CONFIRMED |
| SSRF guard proxy hostname fix (#62312) | `fix(gateway): stop SSRF guard rejecting operator-configured proxy hostnames (#62312)` in git log | CONFIRMED |
| Heartbeat main session fix (#61803) | `fix(agents): heartbeat always targets main session — prevent routing to active subagent sessions (#61803)` in git log | CONFIRMED |

### v2026.4.8 Release Claims

| Claim (docs) | Actual (source) | Status |
|--------------|-----------------|--------|
| Window: `v2026.4.7..v2026.4.8` | 42 commits, 269 files changed, +4,413 / -1,468 lines | CONFIRMED |
| Slack HTTPS_PROXY for Socket Mode (#62878) | `fix: honor Slack Socket Mode env proxies (#62878)` in git log | CONFIRMED |
| Slack botToken download fix (#62097) | `fix: pass resolved Slack download tokens (#62097)` in git log | CONFIRMED |
| Z.AI GLM-5.1 default (#61998) | `fix(zai): default to GLM-5.1 instead of GLM-5` in git log | CONFIRMED |
| Bundled channel fallback revert (v2026.4.7-1) | `revert: remove bundled channel fallback masking` in git log between v2026.4.7 and v2026.4.8 | CONFIRMED |

---

## 2. Online Verification

### GitHub Repository

| Claim | Online finding | Status |
|-------|---------------|--------|
| Repo: github.com/openclaw/openclaw | Active repo; remote `upstream` verified via local fetch | CONFIRMED |
| Tag `v2026.4.9` published | Tag visible via `git tag` after `git fetch upstream` | CONFIRMED |

### npm Package

| Claim | Online finding | Status |
|-------|---------------|--------|
| Package: `openclaw` | Available on npmjs.com | CONFIRMED |
| Latest: `2026.4.9` | `npm view openclaw version` returns `2026.4.9` | CONFIRMED |

---

## 3. Corrections Applied in This Docs Snapshot

| Previous docs state | Corrected in `v2026.4.9-1` docs |
|---------------------|----------------------------------|
| Latest documented upstream release was `v2026.4.5` | Updated to `v2026.4.9` |
| CHANGELOG covered through `v2026.4.5` | Added summaries for `v2026.4.7`, `v2026.4.8`, and `v2026.4.9` |
| TypeScript inventory stated `9,766` files | Updated to `10,454` files |
| Extension inventory stated `94` packages | Updated to `97` packages |
| Skills inventory stated `73 released skill entrypoints` | Updated to `75` |
| Analysis docs validated against `v2026.4.5` | Updated validation notes to `v2026.4.9` |
| README snapshot version `v2026.4.5-2` | Updated to `v2026.4.9-1` |
| ARCHITECTURE.md Stats Update: `5,496` TS files (arch-agent error) | Corrected to `10,454` (arch-agent ran count against wrong working tree; corrected by main session) |
| ARCHITECTURE.md Extension Ecosystem: `33` packages (arch-agent error) | Corrected to `97` packages |
| ARCHITECTURE.md Extension dirs: `33` (arch-agent error) | Corrected to `109` directories |
| ARCHITECTURE.md Skill entrypoints: `60` (arch-agent error) | Corrected to `75` |

---

## 4. Coverage Assessment

### Well-Documented Areas

- Latest released baseline updated to `v2026.4.9`
- Three missing release windows (v2026.4.7, v2026.4.8, v2026.4.9) documented with correct commit/file/line delta stats
- README, CHANGELOG, AGENTS, AGENT_README updated to `v2026.4.9` release line
- Analysis doc validation headers updated from `v2026.4.5` to `v2026.4.9`
- Inventory counts corrected to the current `v2026.4.9` released tree

### Residual Risk

| Area | Note | Severity |
|------|------|----------|
| Analysis deep-dive sections | `analysis/*.md` prose sections retain detailed content from the v2026.4.5 baseline; new modules introduced in v2026.4.7 (Memory Wiki, Arcee AI provider, pluggable compaction registry, `openclaw infer` CLI) are not covered in dedicated analysis sections | MEDIUM |
| AGENT_README behavioral sections | `AGENT_README.md` "BREAKING CHANGES" and "BEHAVIORAL SHIFTS" sections were renamed to `v2026.4.9` but content still reflects v2026.4.5 era additions; v2026.4.7–4.9 additions not fully enumerated | LOW |
| Lines of TypeScript | Line count not re-verified at `v2026.4.9` tag (would require full tree checkout); estimated ~2,000,000 lines based on v2026.4.5 baseline (1,893,328) plus net diff insertions | LOW |

---

## 5. Methodology

- Version/tag verification via local git tags plus `git fetch upstream` remote ref listing
- Window stats via `git rev-list --count` and `git diff --shortstat`
- TypeScript counts via `git ls-tree -r v2026.4.9 --name-only` filtered by path/extension
- Extension counts via `git ls-tree v2026.4.9 extensions/ --name-only` and `extensions/*/package.json` pattern
- Skill counts via `git ls-tree -r v2026.4.9 --name-only | grep SKILL.md`
- Key claim verification via `git show v2026.4.9:<file>` + grep for specific identifiers
- Online verification via `npm view openclaw version` (returns `2026.4.9`)
- git log cross-reference: all claimed PR/feature entries verified against `git log v2026.4.5..v2026.4.9 --oneline`
