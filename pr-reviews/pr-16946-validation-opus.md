# PR #16946 Validation: fix(cron): add minimum delay floor to prevent timer spin loop

**Validator:** Opus 4.6 (subagent)  
**Date:** 2026-02-16  
**Branch:** `fix/cron-timer-spin-16839`  
**Issue:** #16839

---

## Summary

The PR adds a `MIN_TIMER_DELAY_MS = 100` floor to the cron timer's `armTimer()` function to prevent a spin loop when jobs are past-due (i.e., `nextAt - now <= 0`).

## Changes

### `src/cron/service/timer.ts`
- Added constant `MIN_TIMER_DELAY_MS = 100`
- Changed: `Math.min(delay, MAX_TIMER_DELAY_MS)` → `Math.min(Math.max(delay, MIN_TIMER_DELAY_MS), MAX_TIMER_DELAY_MS)`

### `src/cron/service.issue-regressions.test.ts`
- Updated `advanceTimersByTimeAsync(2)` → `advanceTimersByTimeAsync(150)` to account for the new 100ms minimum delay

## Analysis

### The Bug (Pre-Fix)
When a cron job is past-due (`nextAt <= now`), `delay` becomes 0, creating a `setTimeout(..., 0)` hot loop. Each timer tick calls `onTimer()`, which re-arms via `armTimer()` with delay 0 again if the job hasn't started executing yet. This causes CPU spin.

### The Fix
The `Math.max(delay, MIN_TIMER_DELAY_MS)` ensures the timer never fires faster than every 100ms. This is correct and well-chosen:
- 100ms is fast enough to be imperceptible for cron jobs (minute-granularity)
- 100ms is slow enough to prevent meaningful CPU spin
- The fix is minimal and surgical — only one line of logic changed

### Dependency Impact (per DEVELOPER-REFERENCE.md)
- **cron/ depends on:** `config/`, `agents/`, `routing/`, `infra/`, `auto-reply/`
- **cron/ is imported by:** `gateway/`, `agents/tools/`
- **Risk level:** 🟡 (medium)
- **Change impact:** Timer internals only — no API/type changes, no cross-module impact
- **Related paths:** The "Cron Execution: Trigger → Delivery" critical path starts at `timer.ts` — this fix touches the entry point but doesn't alter the execution flow, only the timing

### Gotchas Checked (per DEVELOPER-REFERENCE.md)
- ✅ **Timer Precision Issues**: Doc warns about `computeNextRunAtMs()` using ms precision. The fix doesn't affect precision, just adds a floor.
- ✅ **Cron service uses `locked()`**: Not affected — the fix is before the lock acquisition in `onTimer()`.
- ✅ **`onTimer` re-arm path**: Already uses `MAX_TIMER_DELAY_MS` when `running === true` (issue #12025 fix), so no spin there.
- ✅ **NaN guard**: `MIN_TIMER_DELAY_MS` is a constant, `delay` is already `Math.max(nextAt - now, 0)` so NaN won't occur unless `nextAt` is NaN, but that's guarded by the `if (!nextAt)` check above.

### Pre-PR Checklist
- ✅ Tests pass: 24 files, 106 tests all green
- ✅ Minimal diff (2 files changed)
- ✅ No API/type changes
- ✅ Test updated to match new behavior
- ✅ No new lint issues expected (constant + Math.max wrapping)

## Verdict

**✅ APPROVE** — The fix is correct, minimal, and well-tested. No bugs or missing edge cases found.

One minor observation: the `onTimer()` re-arm when `running === true` (line ~177) uses `MAX_TIMER_DELAY_MS` directly, not going through `armTimer()`. This is fine and intentional (it's a fixed backoff), but worth noting that there are two timer-setting paths, and this PR correctly targets only the one that can spin.

---

## DEVELOPER-REFERENCE.md Effectiveness

- **How useful was it?** **8/10** — Provided immediate context on cron's dependencies, risk level, critical path, and relevant gotchas without needing to explore the codebase.

- **What did it help you find that you wouldn't have found otherwise?**
  - The "Timer Precision Issues" gotcha section flagged cron timing concerns directly
  - The dependency map confirmed cron's blast radius is limited (🟡)
  - The critical path for cron execution showed exactly where `timer.ts` fits in the flow
  - The `locked()` race condition warning helped me verify the fix doesn't interact with locking

- **What was missing or could be improved?**
  - Could mention the two distinct timer-setting paths in `timer.ts` (normal `armTimer()` vs re-arm when `running`)
  - The gotchas section could specifically call out the spin loop risk with `Math.max(delay, 0)` → `setTimeout(..., 0)` pattern
  - A "known past issues" section linking to resolved issues like #12025 would help understand historical context

- **How much faster did it make the review?** ~40% faster — I could skip exploratory `grep` for dependencies and jump straight to understanding impact. Without it, I'd have spent time tracing imports and checking what modules consume cron.

- **Especially valuable sections:**
  - §1 Module Dependency Map — instant risk assessment
  - §2 Critical Paths (Cron Execution) — showed the exact flow
  - §10 Gotchas (Timer Precision, Race Conditions) — directly relevant warnings
  - §3 Change Impact Matrix — confirmed cron changes are relatively safe

- **Less useful for this PR:**
  - §6-9 (Config, Types, Patterns) — not relevant for a timer bug fix
