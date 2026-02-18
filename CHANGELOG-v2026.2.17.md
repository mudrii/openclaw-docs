# Synthesis Review: v2026.2.15 → v2026.2.17

> **Window analyzed:** `v2026.2.15..v2026.2.17` (979 commits)
> **Method:** prioritize behavior and operator/developer impact; ignore low-signal test churn

---

## Executive synthesis

This release window is best understood as a **runtime-hardening and orchestration-stability cycle** with four dominant themes:

1. **Subagent execution became more deterministic** (spawn/announce semantics, routing, completion reliability).
2. **Cron scheduling became safer and more explicit** (default stagger behavior + operator controls).
3. **Security boundaries tightened** (notably config `$include` confinement and sandbox/input hardening).
4. **Tool/runtime behavior shifted toward guarded streaming + loop control** (Z.AI tool-stream default, stronger loop detection, read-context guard behavior).

The most important takeaway for maintainers is not a new feature checklist; it is that **assumptions that used to be “soft defaults” are now enforced behaviors**.

---

## What changed in practice (high signal)

### 1) Subagents: from best-effort to managed flow

Subagent workflows now lean heavily into push-based lifecycle semantics:

- `/subagents spawn` adds deterministic command-path spawning.
- `sessions_spawn` user experience now explicitly sets expectation that completion is auto-announced.
- Completion routing/retry/output selection has been hardened.

**Operational implication:**
Do not design runbooks around aggressive polling of spawned work. Treat spawned runs as async jobs with eventual callback/announce.

### 2) Cron: schedules now carry intent, not just expression

Top-of-hour recurring jobs now receive deterministic stagger by default and persist it as `schedule.staggerMs`; operators can force exact timing via `--exact` / `staggerMs: 0`.

**Operational implication:**
If your mental model is “`0 * * * *` means exact wall-clock fire”, that is no longer universally true. Exactness must be explicit.

### 3) Security posture: config/file boundaries tightened

The most consequential hardening in this window is strict confinement of config `$include` paths to config-root boundaries (with traversal/symlink guards), plus additional sandbox/env sanitization work.

**Operational implication:**
Older include layouts that reached outside config root may fail after upgrade; this is expected security behavior, not parser regression.

### 4) Tool/runtime behavior: safer defaults, stricter failure handling

- Z.AI tool-call streaming defaults to enabled.
- Loop detection now escalates/no-progress blocks on repeated poll/log patterns.
- Read-context handling emphasizes bounded incremental reads over full-dump retries.

**Operational implication:**
Agent/tool orchestration prompts and automation scripts need explicit progress checks, bounded retries, and chunked-read recovery patterns.

---

## Recommended maintainer workflow updates

1. **Before blaming runtime bugs, validate config confinement constraints first** (`$include` path containment, symlink behavior).
2. **When debugging cron timing, inspect `schedule.staggerMs`**, not only cron expression.
3. **Treat subagent completion as announce-driven**, and only poll for intervention/debug.
4. **For tool loops, require no-progress detection + backoff in agent logic docs/tests.**
5. **For large file analysis flows, enforce paged reads (`offset`/`limit`) in prompts and examples.**

---

## Docs in this repo updated for this window

- `README.md` — version/index updates for v2026.2.17 docs.
- `DEVELOPER-REFERENCE.md` — concrete gotchas + workflow guidance for this release window.
- `CHANGELOG-v2026.2.17.md` — this synthesized review (high-signal, non-raw).

---

*Generated: 2026-02-18*