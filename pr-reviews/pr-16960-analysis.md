# PR #16960 Analysis — `perf: skip cache-busting for bundled hooks, use mtime for workspace hooks`

**Date:** 2026-02-16  
**Branch:** `perf/hook-import-cache-busting` (mudrii/openclaw fork)  
**Fixes:** #16953  
**Size:** +109 / -13 lines (4 files)

---

## 1. What the PR Does

Replaces inline `?t=${Date.now()}` cache-busting on all hook `import()` calls with a centralized `buildImportUrl()` helper that applies source-aware strategy:

| Source | Strategy | Rationale |
|--------|----------|-----------|
| `openclaw-bundled` | No query string | Immutable between installs; V8 caches across restarts |
| `openclaw-workspace` | `?t=<mtime>` | User-editable; re-parse only when file changes |
| `openclaw-managed` | `?t=<mtime>` | Same as workspace |
| `openclaw-plugin` | `?t=<mtime>` | Same as workspace |
| Any source, stat fails | `?t=Date.now()` | Safety fallback, preserves previous behavior |

## 2. Files Changed

### `src/hooks/import-url.ts` (new — 28 LOC)
Clean, focused utility. Uses `IMMUTABLE_SOURCES` Set for extensibility. `statSync` + `mtimeMs` for mutable hooks. Fallback to `Date.now()` on stat failure.

### `src/hooks/import-url.test.ts` (new — 7 tests)
Covers: bundled (no query), workspace/managed/plugin (mtime), cacheability (same URL across calls), and stat failure fallback.

### `src/hooks/loader.ts` (modified)
Two call sites migrated:
1. Directory-based hook loading — passes `entry.hook.source` (correct, source comes from `HookEntry`)
2. Legacy handler loading — hardcodes `"openclaw-workspace"` (correct, legacy handlers are always workspace-relative)

### `src/hooks/plugin-hooks.ts` (modified)
One call site migrated — passes `entry.hook.source` (correct, plugin hooks carry their source).

## 3. Detailed Analysis

### Is the fix correct?
**Yes.** The categorization is sound:
- Bundled hooks live in `dist/bundled/` and only change on `npm install`
- All other sources are user-mutable and need cache invalidation
- The `HookSource` union type (`"openclaw-bundled" | "openclaw-managed" | "openclaw-workspace" | "openclaw-plugin"`) is exhaustive — all 4 values are handled

### Race conditions with `statSync()`?
**No practical concern.** This runs during sequential hook loading at gateway startup. There's no concurrent writer. Even if a file were being written simultaneously, the worst case is getting a stale mtime → one extra restart to pick up changes. `statSync` is the right choice here (simpler than async, startup is already sequential).

### Could `mtime` be unreliable?
- **NFS/network filesystems:** mtime can have 1-2 second granularity. Not a concern here — hooks aren't edited with sub-second frequency during restarts.
- **Docker with bind mounts:** mtime propagation is reliable on modern Docker.
- **Git operations:** `git checkout` updates mtime to current time, which is correct behavior (triggers re-import).
- **Copy without preserving mtime:** Would show current time → triggers re-import → safe.
- **Edge case:** If mtime is somehow frozen (e.g., `touch -t` to old time), the hook wouldn't be re-imported. This is pathological and not a real concern.

### Is the `Date.now()` fallback sufficient?
**Yes.** It's the exact behavior before this PR. Stat failure on a file you're about to `import()` is extremely unlikely (permissions, deleted file). If stat fails, the import will likely also fail — the fallback just ensures the import attempt uses a fresh URL.

### Any hooks loaded outside `loader.ts` and `plugin-hooks.ts`?
**No.** Grep confirms:
- `loadInternalHooks` (in `loader.ts`) is only called from `server-startup.ts`
- `plugin-hooks.ts` handles all plugin hook loading
- No other files use `Date.now()` cache-busting for imports
- All three import sites are migrated

### Test coverage
**Good but could be slightly better:**
- ✅ All 4 source types covered
- ✅ Cacheability (same URL across calls for bundled)
- ✅ Stability (same URL for unchanged workspace file)
- ✅ Stat failure fallback
- ❌ **Missing:** Test that mtime changes after file modification → different URL. This is the core correctness property. (Minor — it's implicitly tested by the mtime assertion, but an explicit "modify file → URL changes" test would be stronger.)
- ❌ **Missing:** Test that the fallback URL has a *different* timestamp on each call (verifying it's `Date.now()` not cached). (Very minor.)

### Could this break hot-reload?
**No.** Hot-reload/dev workflows that watch for file changes will modify files → mtime changes → cache busted correctly. The only change is that *unchanged* files aren't needlessly re-imported.

### Performance impact
**The ~120 redundant compilations claim is accurate.** With 4 bundled hooks each transitively importing ~30 chunks, that's ~120 module parse+compile cycles per restart. With this fix:
- Bundled hooks: 0 re-compilations (V8 cache hit)
- Workspace hooks: re-compiled only when changed (mtime-based)

The 1.3s → ~170ms improvement observed in the issue is primarily OS page cache, but eliminating V8 re-compilation will further reduce warm restart times, especially on lower-end hardware.

## 4. Issues Found

### Bugs/Logic Errors
**None.**

### Style/Lint
**Clean.** PR states `pnpm lint` passes with 0 warnings.

### Missing Error Handling
**None needed.** The try/catch around `statSync` is the only error path and it's handled correctly.

### Potential Regressions
**None identified.** The fallback behavior exactly matches the previous implementation.

### Thread Safety
**Not applicable.** Node.js is single-threaded. Hook loading is sequential at startup.

### Minor Observations
1. `IMMUTABLE_SOURCES` as a module-level `Set` is good — but with only one member, a simple `=== "openclaw-bundled"` would also work. The Set is future-proof though (e.g., if `openclaw-managed` becomes immutable).
2. The `pathToFileURL` import was correctly removed from `loader.ts` and `plugin-hooks.ts` since it's now encapsulated in `import-url.ts`.

## 5. Cross-Reference with Issue #16953

The PR directly addresses all points raised:
- ✅ Bundled hooks no longer cache-bust
- ✅ Workspace hooks use mtime instead of `Date.now()`
- ✅ The suggested code pattern from the issue is implemented (with improvements: centralized helper, type-safe source parameter, proper abstraction)

## 6. Verdict

### ✅ APPROVE

**Reasoning:**
- Correct fix for a real performance problem
- Clean, well-scoped refactoring (single new helper, 3 call sites migrated)
- Type-safe (uses `HookSource` union, compiler enforces correctness)
- Good test coverage (7 tests, all branches)
- No regressions (fallback preserves previous behavior exactly)
- Well-documented (clear JSDoc, good PR description)

**Optional suggestion:** Add one test for "file modification changes URL" — but this is not blocking.

**Risk: Minimal.** The worst case if something goes wrong is falling back to `Date.now()` behavior, which is what we had before.
