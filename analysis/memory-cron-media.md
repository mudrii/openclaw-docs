# OpenClaw Analysis: Memory, Cron & Media Cluster
<!-- markdownlint-disable MD024 -->

> Updated: 2026-04-09 | Version: v2026.4.9 | Codebase: OpenClaw release tag `v2026.4.9`
> Release note: the current released memory surface is split across `src/memory-host-sdk/` and `extensions/memory-core/src/memory/`. Historical `src/memory/` references below describe the older pre-split layout and should be read as historical context only.

---

## Table of Contents
1. [Memory / Released Surfaces](#memory--released-surfaces)
2. [src/cron](#srccron)
3. [src/media](#srcmedia)
4. [src/media-understanding](#srcmedia-understanding)
5. [src/link-understanding](#srclink-understanding)
6. [src/tts](#srctts)
7. [src/markdown](#srcmarkdown)

---

## Memory / Released Surfaces

### Module Overview
On `v2026.4.9`, released memory is split between `src/memory-host-sdk/` (embeddings, dreaming, promotion, multimodal helpers, and host/runtime utilities) and `extensions/memory-core/src/memory/` (QMD, search managers, SQLite indexing, and sync orchestration). Together these surfaces provide **semantic search over markdown files and session transcripts** using vector embeddings. They support multiple embedding providers (OpenAI, Gemini, Voyage, Mistral, local llama.cpp), store chunks in SQLite with optional sqlite-vec for native vector search, and implement hybrid search (vector + BM25 full-text). An alternative backend (`qmd`) delegates to an external QMD binary for indexing/search. Historical bullets that still say `src/memory/` are describing the older pre-split layout.

**Architecture Pattern:** Manager singleton with lazy initialization, file watcher for live reindexing, mixin-style ops classes (`manager-embedding-ops.ts`, `manager-sync-ops.ts` mixed into `MemoryIndexManager`).

**Entry Points:**
- `index.ts` — re-exports `MemoryIndexManager`, `getMemorySearchManager`
- `search-manager.ts` — `getMemorySearchManager()` factory (selects builtin vs qmd backend)

### v2026.2.15 Changes
- **Isolate managed collections per agent** — QMD collections are now scoped per-agent to prevent cross-agent leaks
- **Rebind drifted collection paths** — auto-corrects collection paths that drifted from expected locations
- **Harden context window cache collisions** — prevents cache key collisions across concurrent context windows
- **Unicode tokens in FTS query builder** — `buildFtsQuery()` now supports non-ASCII tokens
- **Inject runtime date-time into memory flush prompt** — memory flush gets current timestamp for temporal context
- **Verify QMD index artifact after manual reindex** — validates index integrity post-reindex
- **`MemorySyncProgressUpdate` in `types.ts`** — typed sync progress state & reporting callback; progress infra is in `manager-sync-ops.ts`
- **New `batch-utils.ts`** — shared batch HTTP client config types

### v2026.2.21 Changes <!-- v2026.2.21 -->

- **QMD manager reliability** — `extensions/memory-core/src/memory/qmd-manager.ts` received significant changes improving QMD (query-managed document) search reliability. Key improvements: better error handling for missing files, improved index consistency. Source: `qmd-manager.test.ts` (332 lines of updated tests).

- **Memory ENOENT handling** — The memory system now handles missing file errors (ENOENT) gracefully. Previously a missing memory file could cause a crash; now it returns an empty result and continues.

- **Multilingual FTS stop-word filtering** — Full-text search now applies stop-word filtering for Korean (with particle-aware extraction and mixed Korean/English stems), Japanese (mixed-script katakana/ASCII), Spanish, and Portuguese. Query expansion uses language-appropriate tokenization, improving recall in conversational searches for these languages.

### v2026.2.22 Changes <!-- v2026.2.22 -->

#### Memory FTS

- **Multilingual query expansion expanded** — Query expansion now covers Spanish, Portuguese, Japanese (mixed-script ASCII + katakana), Korean (particle-aware, mixed Korean/English stems), and Arabic (stop-word filtering). Total: 6 languages with stop-word/tokenization support.

#### Memory QMD

- **Migrate legacy unscoped collection bindings** — Legacy unscoped collection bindings (e.g., `memory-root`) are migrated to per-agent scoped names (e.g., `memory-root-main`) during startup, fixing `Collection not found` errors after upgrades.
- **Normalize Han-script BM25 queries** — Mixed CJK+Latin prompts now have Han-script tokens normalized before BM25 query construction.
- **Optional mcporter routing** — `memory.qmd.mcporter` can now route QMD searches through mcporter keep-alive flows.
- **Windows shim resolution** — On Windows, bare `qmd`/`mcporter` binary names are now resolved to their npm `.cmd` shim executables.

#### Memory Embeddings

- **Centralized remote memory HTTP calls** — Remote memory HTTP calls are now routed through a shared guarded helper (`withRemoteHttpResponse`).
- **Host pinning across embedding providers** — Configured remote-base host pinning (`allowedHostnames`) is now applied across OpenAI, Voyage, and Gemini embedding requests.
- **Per-input 8k safety cap** — A per-input 8k token safety cap is applied before batching; a 2k fallback is used for local providers.
- **Automatic full reindex on source-set change** — Memory source-set changes are now detected and trigger a full reindex automatically.

### v2026.2.23 Changes <!-- v2026.2.23 -->

- **Bootstrap file caching** — Bootstrap file snapshots (`AGENTS.md`/`MEMORY.md`) are now cached per session key and cleared on session reset/delete. Reduces prompt-cache invalidations caused by in-session writes to these files. (#22220)

### v2026.2.24 Changes <!-- v2026.2.24 -->

- **Config / Meta timestamp coercion** (#25491): numeric `meta.lastTouchedAt` timestamps (written by agent edits using `Date.now()`) are now accepted and coerced to ISO strings. Contributor: @mcaxtr. <!-- v2026.2.24 -->

### v2026.3.1 Changes <!-- v2026.3.1 -->

- **Hybrid keyword-only hit preservation** — When hybrid search yields keyword-only matches whose maximum score equals `textWeight` (e.g. 0.3) but `minScore` is higher (e.g. 0.35), those exact lexical hits were previously discarded. The search now relaxes `minScore` to `min(minScore, textWeight)` for keyword-matched entries so they are retained when no vector results exist.

- **SQLite readonly sync recovery** (#25799) — If `sync()` encounters a `SQLITE_READONLY` error (e.g. stale WAL after crash or NFS mount), the manager now reopens the SQLite connection and retries once. Recovery statistics (`readonlyRecovery.attempts/successes/failures/lastError`) are exposed in `status().custom`. Contributor: @rodrigouroz.

- **Manager cache hydration deduplication** — Concurrent calls to `MemoryIndexManager.get()` with the same cache key now share a single pending `createEmbeddingProvider` promise via `INDEX_CACHE_PENDING`, preventing redundant provider creation during parallel startup.

- **QMD discard-output mode** (#28900) — `qmd update` and `qmd embed` commands now run with `discardOutput: true`, draining stdout without accumulation. Prevents "produced too much output" failures on large indexes (>200K chars). Contributor: @Glucksberg.

- **LanceDB extension: custom OpenAI baseUrl and dimensions** (#17874) — The `memory-lancedb` extension (`extensions/memory-lancedb/`) now supports `embedding.baseUrl` and `embedding.dimensions` configuration fields, enabling use of OpenAI-compatible endpoints (e.g. Ollama at `http://localhost:11434/v1`) and custom vector dimensions for non-standard models. When `dimensions` is specified, the built-in model-dimension lookup is bypassed.

### v2026.3.11 Changes <!-- v2026.3.11 -->

- **Multimodal memory indexing** (#43460) — Opt-in multimodal image and audio indexing for `memorySearch.extraPaths`. When enabled, image files (`.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.heic`, `.heif`) and audio files (`.mp3`, `.wav`, `.ogg`, `.opus`, `.m4a`, `.aac`, `.flac`) in configured extra paths are indexed using Gemini structured inputs. Requires the `gemini-embedding-2-preview` model. Strict fallback gating prevents non-Gemini-2 providers from attempting multimodal indexing. Reindexing is triggered automatically when the multimodal scope settings change. Configuration: `memorySearch.multimodal.enabled`, `memorySearch.multimodal.modalities` (`"image"`, `"audio"`, or `"all"`), `memorySearch.multimodal.maxFileBytes` (default 10 MB).

- **`gemini-embedding-2-preview` support** (#42501) — The Gemini embedding provider now supports `gemini-embedding-2-preview` with configurable output dimensions. Valid dimension values are `768`, `1536`, and `3072` (default). When the configured dimension value changes, the memory index is automatically reindexed. The model set is tracked in `GEMINI_EMBEDDING_2_MODELS` in `embeddings-gemini.ts`.

- **Normalize Gemini embeddings** (#43409) — Gemini embedding vectors are now L2-normalized before storage. The normalization helper `normalizeGeminiEmbeddingModelForMemory()` is exposed from `multimodal.ts` for use in provider routing.

- **Revalidate multimodal files before indexing** — Multimodal file entries (images/audio) are revalidated against the filesystem immediately before indexing. Files that grow or change between discovery and indexing are skipped to prevent partial or corrupt index entries.

- **Forward `memoryFlushWritePath` through `runEmbeddedPiAgent`** (#41761) — The `memoryFlushWritePath` parameter is now correctly forwarded when running embedded agent sessions, ensuring memory flush writes reach the intended path in all execution contexts.

- **Close cached memory managers in one-shot CLI shutdown** (#40389) — The `close()` lifecycle method is now called on cached memory search and index manager instances during one-shot CLI teardown, preventing unclosed SQLite connections and dangling file watchers.

### Selected File Inventory (`src/memory` has 101 tracked files in `v2026.3.11`)

| File | Description |
|------|-------------|
| `index.ts` | Public API re-exports |
| `index.test.ts` | Integration tests for memory index |
| `types.ts` | Core types: MemorySearchManager interface, MemorySearchResult, MemorySource |
| `manager.ts` | `MemoryIndexManager` class — main builtin memory backend |
| `manager-embedding-ops.ts` | Mixin: embedding batch logic, cache, retry |
| `manager-sync-ops.ts` | Mixin: file watching, sync, vector loading, session tracking |
| `manager-search.ts` | Vector and keyword search query execution |
| `search-manager.ts` | Factory: selects qmd or builtin backend, FallbackMemoryManager wrapper |
| `backend-config.ts` | Resolves memory backend config (builtin vs qmd) with defaults |
| `qmd-manager.ts` | `QmdMemoryManager` — alternative backend using external qmd CLI |
| `qmd-query-parser.ts` | Parses JSON output from qmd query command |
| `qmd-scope.ts` | Session-key scope filtering for qmd searches |
| `internal.ts` | Core utilities: chunkMarkdown, listMemoryFiles, hashText, cosineSimilarity |
| `embeddings.ts` | `createEmbeddingProvider()` factory — routes to openai/gemini/voyage/local |
| `embeddings-openai.ts` | OpenAI embedding provider |
| `embeddings-gemini.ts` | Google Gemini embedding provider |
| `embeddings-voyage.ts` | Voyage AI embedding provider |
| `embeddings-mistral.ts` | Mistral embedding provider |
| `embeddings-remote-provider.ts` | Shared factory for remote embedding providers (OpenAI, Voyage, Gemini, Mistral) |
| `embeddings-ollama.ts` | Ollama embedding provider |
| `temporal-decay.ts` | TTL decay scoring for memory results |
| `mmr.ts` | Maximum Marginal Relevance reranking |
| `query-expansion.ts` | Query expansion for improved recall |
| `prompt-section.ts` | Memory prompt section builder |
| `read-file.ts` | Memory file reading utilities |
| `node-llama.ts` | Dynamic import wrapper for node-llama-cpp |
| `batch-http.ts` | Generic HTTP POST with retry for batch APIs |
| `batch-output.ts` | Shared batch output line parser |
| `batch-utils.ts` | Shared batch HTTP client config types |
| `batch-openai.ts` | OpenAI batch embedding API |
| `batch-gemini.ts` | Gemini batch embedding API |
| `batch-voyage.ts` | Voyage batch embedding API |
| `embedding-chunk-limits.ts` | Enforces max input tokens per chunk |
| `embedding-input-limits.ts` | UTF-8 byte estimation, text splitting |
| `embedding-model-limits.ts` | Known max input tokens per model |
| `hybrid.ts` | BM25+vector hybrid search merging |
| `sqlite.ts` | `requireNodeSqlite()` — loads node:sqlite with warning filter |
| `sqlite-vec.ts` | Loads sqlite-vec native extension |
| `memory-schema.ts` | SQLite schema creation (meta, files, chunks, embedding_cache, FTS5) |
| `session-files.ts` | Reads session JSONL transcripts, builds entries with line maps |
| `status-format.ts` | Formatting helpers for memory status display |
| `embedding-chunk-limits.test.ts` | Tests for chunk limit enforcement |
| `embedding-manager.test-harness.ts` | Test fixture for embedding manager tests |
| `embedding.test-mocks.ts` | Vitest mocks for embeddings |
| `backend-config.test.ts` | Tests for backend config resolution |
| `embeddings.test.ts` | Tests for embedding provider creation |
| `embeddings-voyage.test.ts` | Tests for Voyage provider |
| `embeddings-mistral.test.ts` | Tests for Mistral provider |
| `manager.mistral-provider.test.ts` | Tests for Mistral provider integration in manager |
| `batch-voyage.test.ts` | Tests for Voyage batch API |
| `hybrid.test.ts` | Tests for hybrid search |
| `internal.test.ts` | Tests for chunking, file listing |
| `session-files.test.ts` | Tests for session entry building |
| `qmd-manager.test.ts` | Tests for QMD manager |
| `qmd-query-parser.test.ts` | Tests for QMD JSON parsing |
| `qmd-scope.test.ts` | Tests for QMD scope filtering |
| `search-manager.test.ts` | Tests for search manager factory |
| `manager.async-search.test.ts` | Tests async search doesn't block on sync |
| `manager.atomic-reindex.test.ts` | Tests atomic reindex safety |
| `manager.batch.test.ts` | Tests OpenAI batch embedding flow |
| `manager.embedding-batches.test.ts` | Tests multi-batch splitting |
| `manager.embedding-token-limit.test.ts` | Tests chunk splitting at token limits |
| `manager.sync-errors-do-not-crash.test.ts` | Tests graceful sync failure |
| `manager.vector-dedupe.test.ts` | Tests vector row deduplication |
| `manager.watcher-config.test.ts` | Tests chokidar watcher configuration |
| `manager.get-concurrency.test.ts` | Tests concurrent manager creation deduplication |
| `manager.readonly-recovery.test.ts` | Tests SQLite readonly sync recovery |

### Key Types & Interfaces

```typescript
// types.ts
type MemorySource = "memory" | "sessions";
type MemorySearchResult = { path, startLine, endLine, score, snippet, source, citation? };
type MemoryEmbeddingProbeResult = { ok: boolean; error?: string };
type MemorySyncProgressUpdate = { completed, total, label? };
type MemoryProviderStatus = { backend, provider, model?, files?, chunks?, dirty?, ... };
interface MemorySearchManager {
  search(query, opts?): Promise<MemorySearchResult[]>;
  readFile(params): Promise<{ text, path }>;
  status(): MemoryProviderStatus;
  sync?(params?): Promise<void>;
  probeEmbeddingAvailability(): Promise<MemoryEmbeddingProbeResult>;
  probeVectorAvailability(): Promise<boolean>;
  close?(): Promise<void>;
}

// embeddings.ts
type EmbeddingProvider = { id, model, maxInputTokens?, embedQuery, embedBatch };
type EmbeddingProviderId = "openai" | "local" | "gemini" | "voyage" | "mistral" | "ollama";
type EmbeddingProviderRequest = EmbeddingProviderId | "auto";

// internal.ts
type MemoryFileEntry = { path, absPath, mtimeMs, size, hash };
type MemoryChunk = { startLine, endLine, text, hash };

// backend-config.ts
type ResolvedMemoryBackendConfig = { backend, citations, qmd? };
type ResolvedQmdConfig = { command, searchMode, collections[], sessions, update, limits, ... };

// hybrid.ts
type HybridVectorResult = { id, path, startLine, endLine, source, snippet, vectorScore };
type HybridKeywordResult = { id, path, startLine, endLine, source, snippet, textScore };

// session-files.ts
type SessionFileEntry = { path, absPath, mtimeMs, size, hash, content, lineMap };
```

### Key Functions & Classes

| Export | Signature | Description |
|--------|-----------|-------------|
| `MemoryIndexManager` | class implements `MemorySearchManager` | Main builtin memory manager — indexes .md files, embeds chunks, searches |
| `getMemorySearchManager` | `(params) → Promise<MemorySearchManagerResult>` | Factory: returns qmd or builtin manager based on config |
| `createEmbeddingProvider` | `(options) → Promise<EmbeddingProviderResult>` | Creates embedding provider (openai/gemini/voyage/local) |
| `resolveMemoryBackendConfig` | `(params) → ResolvedMemoryBackendConfig` | Resolves backend config with defaults |
| `QmdMemoryManager` | class implements `MemorySearchManager` | External qmd-based memory backend |
| `chunkMarkdown` | `(text, tokens, overlap) → MemoryChunk[]` | Splits markdown into overlapping chunks |
| `listMemoryFiles` | `(workspaceDir, extraPaths?) → Promise<string[]>` | Lists .md files in memory dirs |
| `buildFtsQuery` | `(raw) → string | null` | Builds FTS5 AND-joined query (supports unicode tokens) |
| `mergeHybridResults` | `(params) → merged[]` | Merges vector + keyword results with weights |
| `searchVector` | `(params) → Promise<SearchRowResult[]>` | Executes vector similarity search |
| `searchKeyword` | `(params) → Promise<SearchRowResult[]>` | Executes FTS5 keyword search |
| `buildSessionEntry` | `(absPath) → Promise<SessionFileEntry | null>` | Parses JSONL session into indexable entry |
| `parseQmdQueryJson` | `(stdout, stderr) → QmdQueryResult[]` | Parses qmd CLI output |

### Internal Dependencies
- `src/config/config.ts` — OpenClawConfig type
- `src/config/types.memory.ts` — MemoryBackend, MemoryQmdConfig types
- `src/config/sessions/paths.ts` — session transcript paths
- `src/config/sessions.ts` — session store operations
- `src/agents/agent-scope.ts` — workspace/agent directory resolution
- `src/agents/memory-search.ts` — ResolvedMemorySearchConfig
- `src/agents/model-auth.ts` — API key resolution
- `src/cli/parse-duration.ts` — duration string parsing
- `src/infra/env.ts`, `src/infra/retry.ts`, `src/infra/warning-filter.ts`
- `src/logging/subsystem.ts`, `src/logging/redact.ts`
- `src/sessions/transcript-events.ts`, `src/sessions/session-key-utils.ts`
- `src/utils.ts` — resolveUserPath, truncateUtf16Safe

### External Dependencies
- `chokidar` — file watching
- `node-llama-cpp` — local embedding (optional, dynamic import)
- `sqlite-vec` — native vector search extension (optional)
- `node:sqlite` — SQLite database (Node.js built-in, experimental)
- `node:crypto`, `node:fs`, `node:path`, `node:readline`

### Data Flow
1. **Indexing:** `sync()` → `listMemoryFiles()` + `listSessionFilesForAgent()` → read files → `chunkMarkdown()` → `embedBatch()` → store in SQLite `chunks` table
2. **Search:** `search(query)` → `embedQuery(query)` → vector search (sqlite-vec or cosine scan) + optional FTS5 keyword search → `mergeHybridResults()` → return ranked results
3. **Watching:** chokidar watches memory dirs → debounced `sync()` on file changes
4. **Batch embeddings:** OpenAI/Gemini/Voyage batch APIs for bulk indexing (upload JSONL → poll → download results)

### Storage
- **SQLite database** at configurable path (default: `<workspace>/memory-index.sqlite`)
  - `meta` table — key/value for index metadata (model, provider, chunk settings)
  - `files` table — indexed file paths with hash, mtime, source
  - `chunks` table — text chunks with embeddings (JSON array), start/end lines, model
  - `chunks_vec` virtual table — sqlite-vec for native vector search (optional)
  - `chunks_fts` FTS5 virtual table — full-text keyword search (optional)
  - `embedding_cache` table — cached embeddings by provider+model+hash
- **QMD backend** stores index in `~/.openclaw/state/<agent>/qmd/` with XDG-style dirs

### Configuration
- `memory.backend` — `"builtin"` | `"qmd"`
- `memory.citations` — `"auto"` | `"on"` | `"off"`
- `memory.qmd.*` — command, searchMode, paths, sessions, update intervals, limits
- `agents.defaults.memorySearch.*` — provider, model, store path, chunking, sync, query, cache, hybrid, extra paths
- `agents.defaults.memorySearch.store.vector.enabled` — enable sqlite-vec
- `agents.defaults.memorySearch.remote.batch.*` — batch embedding config
- `agents.defaults.memorySearch.sync.sessions.postCompactionForce` — force session memory reindex after compaction-triggered transcript updates (boolean, default `true`) — added v2026.3.12 (#25561)
- `agents.defaults.compaction.postIndexSync` — post-compaction session memory index sync mode: `"off"` | `"async"` | `"await"` (default `"async"`) — added v2026.3.12 (#25561)
- `agents.defaults.compaction.customInstructions` — additional compaction-summary instructions for language/persona continuity; resolved with precedence SDK event → this config → built-in default; max 800 characters — added v2026.3.13 (#10456)

### Test Coverage
20 test files covering: backend config resolution, embedding chunk limits, embedding providers (OpenAI, Voyage, Gemini), batch APIs, hybrid search, chunking/file listing, session entry parsing, QMD manager/parser/scope, search manager factory, async search, atomic reindex, batch splitting, token limits, sync error handling, vector deduplication, watcher configuration, concurrent manager creation deduplication, SQLite readonly sync recovery.

---

## src/cron

### Module Overview
The cron module provides **scheduled job execution** — one-shot (`at`), recurring (`every`), and cron-expression-based (`cron`) schedules. Jobs can run in the main agent session (systemEvent) or in isolated agent sessions (agentTurn). Supports delivery of results to messaging channels.

**Architecture Pattern:** Service class with internal state machine, file-based JSON store, timer-driven execution, serialized operations via `locked()`.

**Entry Points:**
- `service.ts` — `CronService` class (start/stop/list/add/update/remove/run/wake)
- `store.ts` — `loadCronStore()` / `saveCronStore()`
- `types.ts` — all type definitions

### v2026.2.15 Changes
- **Normalize skill-filter snapshots** — consistent snapshot format for skill filters in cron context
- **Pass agent-level skill filter to isolated cron sessions** — isolated runs inherit the agent's skill filter
- **Treat missing `enabled` as true in `update()`** — job patches without `enabled` no longer disable jobs
- **Infer payload kind for model-only patches** — PATCH with only `model` auto-infers `agentTurn` kind
- **Preserve cron prompts for tagged interval events** — interval events with tags retain their original prompts
- **Cron finished-run webhook (#14535)** — fires webhook on job completion
- **Avoid async cron timer callbacks** — timer callbacks are now synchronous to prevent race conditions
- **New `skills-snapshot.ts`** — build & normalize skill snapshots for cron sessions
- **New `subagent-followup.ts`** — handle sub-agent followup after cron isolated runs
- **New `legacy-delivery.ts`** — detect and handle legacy delivery hint fields

### v2026.2.19 Changes
- **Heartbeat skip on empty** — Interval heartbeats skipped when `HEARTBEAT.md` is missing or empty and no tagged cron events are queued
- **Telegram topic delivery** — Explicit `<chatId>:topic:<threadId>` targets now correctly route scheduled sends into the configured topic
- **Cron webhook SSRF guard** — Webhook delivery URLs validated through SSRF guard before dispatch; private/metadata destinations blocked. See DEVELOPER-REFERENCE.md §9 (gotcha 42)
- **Exec preflight guard** — Shell env var injection patterns in cron scripts detected before execution

### v2026.2.21 Changes <!-- v2026.2.21 -->

- **`maxConcurrentRuns` now honored** (`fix(cron)`, #22413) — The `cron.maxConcurrentRuns` config key was previously not enforced in the timer loop. Concurrent cron job executions are now properly limited. This is a significant reliability fix: previously, slow cron jobs could accumulate unbounded parallel runs across timer ticks.

### v2026.2.22 Changes <!-- v2026.2.22 -->

- **`cron.maxConcurrentRuns` enforced in timer loop** — `cron.maxConcurrentRuns` is now actually enforced in the timer loop (previously only some paths checked it). Manual `cron.run` executions and startup catch-up replay runs now also enforce per-job timeout guards. Fresh session IDs are forced for `sessionTarget="isolated"` cron executions. For `every` jobs: `lastRunAtMs + everyMs` is preferred as the next-run anchor when still in the future after restarts.

- **Status visibility** — Execution outcome (`lastRunStatus`) and delivery outcome (`lastDeliveryStatus`) are now split into separate fields in persisted state. `delivered` state is persisted in cron records. Settled per-path run-log write queue entries are cleaned up. `cron.runs` path resolution is hardened to reject path-separator characters in `id`/`jobId` inputs.

- **Auth/Delivery** — Auth-profile resolution is propagated to isolated cron sessions. `agentDir` is passed through isolated cron and queued follow-up runs. Text-only announce jobs with thread/topic targets route through direct outbound delivery. Telegram: `delivery.to` is now validated with shared target parsing.

- **Scheduling** — Runtime cron expressions are validated before schedule evaluation — malformed jobs report a clear error instead of crashing. Abort/timeout signals are now honored in `wakeMode=now` heartbeat contention loops.

### v2026.2.23 Changes <!-- v2026.2.23 -->

- **Sessions maintenance hardening** — `openclaw sessions cleanup` CLI command added with per-agent store targeting and disk-budget controls (`session.maintenance.maxDiskBytes` / `session.maintenance.highWaterBytes`). Provides safer transcript/archive cleanup with run-log retention behavior. (#24753)
- **Isolated cron full prompt mode** — Isolated cron sessions now use full prompt mode so skills/extensions are available during cron execution. (#24944)

### v2026.3.1 Changes <!-- v2026.3.1 -->

#### Scheduling & Execution

- **Year-rollback guard in croner nextRun** (#30777) — Some timezone/date combinations (e.g. `Asia/Shanghai`) cause `croner.nextRun()` to return a timestamp in a past year. When `nextRun` returns a value at or before `nowMs`, the scheduler now retries from the next whole second and, if still stale, from midnight-tomorrow UTC before giving up. Contributor: @Sid-Qin.

- **One-shot reschedule re-arm** (#28915) — Completed `at`-scheduled (one-shot) jobs can now run again when rescheduled. Previously, a finished one-shot job with `deleteAfterRun: false` could not be re-armed to a new timestamp. Contributor: @Glucksberg.

- **Cron run exit code** — `cron.run` CLI now returns exit code 0 only when both `ok: true` and `ran: true`. All other outcomes (skipped, error, timeout) return non-zero, enabling shell-level success/failure detection.

- **armTimer tight-loop prevention** (#29853) — When a job has a stuck `runningAtMs` (e.g. process crashed mid-run), `armTimer` no longer enters a tight re-fire loop. Contributor: @FlamesCN.

- **1/3 timeout fix for fresh isolated CLI runs** (#30140) — Fresh isolated cron sessions no longer silently inherit a reduced (1/3) timeout from the CLI runner defaults. Contributor: @ningding97.

#### Delivery & Messaging

- **Delivery mode `none`: disable messaging tool** (#21808) — When `delivery.mode` is `"none"`, the messaging tool is now disabled in isolated cron sessions, preventing accidental message delivery from agent turns.

- **Delivery mode `none` from cron editor** — The `"none"` delivery mode is now explicitly selectable from the Web UI cron editor when adding or updating jobs.

- **Completion direct send for text-only announce** (#29151) — Text-only announce delivery jobs with explicit targets now send via direct outbound delivery at completion.

- **`--account` flag for multi-account routing** (#26284) — Cron jobs can now specify an explicit channel account ID for delivery routing in multi-account setups.

#### Light Bootstrap Context

- **Heartbeat light bootstrap context** (#26064) — Opt-in `--light-context` CLI flag for cron agent turns and `agents.*.heartbeat.lightContext` config for heartbeat runs. When enabled, bootstrap files (`AGENTS.md`, `MEMORY.md`) are loaded with reduced context to lower token usage. Contributor: @jose-velez.

#### Reliability & Resilience

- **Configurable failure alerts** (#24789) — Repeated job failures now trigger user-facing alerts after a configurable threshold (`failureAlert.after`, default 2 consecutive errors) with a cooldown period (`failureAlert.cooldownMs`, default 1 hour). Contributor: @0xbrak.

- **Retry policy for one-shot transient errors** (#24435) — One-shot `at`-jobs now support a retry policy for transient errors, re-arming the job for a later attempt instead of permanently failing. Contributor: @hugenshen.

- **Auto-disable on repeated errors** (#29098) — Jobs are auto-disabled after repeated consecutive errors, with a user notification explaining why. Contributor: @ningding97.

- **Reject `sessionTarget: "main"` for non-default agents** (#30217) — Creating a cron job with `sessionTarget: "main"` for a non-default agent now fails at creation time instead of silently misbehaving at runtime. Contributor: @liaosvcaf.

#### Model & Payload

- **Payload fallbacks** (#26304) — Per-job `payload.fallbacks` array enables model fallback override at the job level, independent of the agent's global fallback chain.

- **Respect `subagents.model` in isolated cron** (#11474) — Isolated cron sessions now correctly propagate the `subagents.model` configuration.

- **Handle cron model override in sessions list** (#21279) — The sessions list CLI now displays the model override when a cron job specifies one. Contributor: @altaywtf.

#### Store & Persistence

- **Store backup churn reduction** (#19484) — Cron store backup (`.bak`) files are no longer rewritten on every persist; backups are only created when the store content has actually changed.

- **Drain pending writes before run log read** (#25416) — Run log reads now drain pending write queue entries first, ensuring consistent state.

- **Legacy schedule field migration** (#28889) — Legacy `schedule.cron` fields in stored jobs are migrated to the current `schedule.expr` format on load.

- **List sort guard** (#28896) — `cron list` sorting now guards against malformed legacy jobs that lack required fields.

### v2026.3.11 Changes <!-- v2026.3.11 -->

#### Delivery Isolation (BREAKING)

- **Tighten isolated cron delivery** (#40998, BREAKING) — Isolated cron jobs can no longer deliver responses through ad hoc agent sends or fall back to main-session summaries. Cron delivery is now strictly scoped to the isolated session's configured delivery target. `openclaw doctor --fix` migrates legacy cron storage records and legacy `notify`/webhook delivery metadata to the current format. Operators upgrading from older stores should run `openclaw doctor --fix` to avoid silent delivery failures.

#### Reliability

- **Cron subagent fix: empty/`NO_REPLY` responses** (#41383) — Empty responses and explicit `NO_REPLY` returns from cron subagents are no longer misclassified as interim acknowledgements. Previously this could trigger a spurious retry turn; now they are correctly treated as deliberate non-delivery.

- **Record `lastErrorReason` in cron job state** (#14382) — The cron job state now persists `lastErrorReason` (typed as `FailoverReason`) after a failed run, and the gateway schema is kept aligned with the full failover-reason set. This makes failure reasons queryable without reading run logs.

### File Inventory (83 files)

| File | Description |
|------|-------------|
| `types.ts` | Core types: CronJob, CronSchedule, CronPayload, CronStoreFile |
| `service.ts` | `CronService` class — public API |
| `service/state.ts` | CronServiceState, CronServiceDeps, CronEvent types |
| `service/ops.ts` | Service operations: start, stop, status, list, add, update, remove, run, wake |
| `service/jobs.ts` | Job lifecycle: create, patch, schedule computation, due checks |
| `service/timer.ts` | Timer management, job execution, backoff, delivery |
| `service/store.ts` | Store loading with migrations, persistence |
| `service/locked.ts` | Serialized operation queue per store path |
| `service/normalize.ts` | Name inference, text normalization |
| `store.ts` | File I/O: loadCronStore, saveCronStore (JSON with atomic write) |
| `schedule.ts` | `computeNextRunAtMs()` — next run time for all schedule kinds |
| `delivery.ts` | `resolveCronDeliveryPlan()` — resolve delivery mode/channel/target |
| `normalize.ts` | Input normalization for job create/patch |
| `parse.ts` | `parseAbsoluteTimeMs()` — ISO/epoch timestamp parsing |
| `payload-migration.ts` | Migrates legacy `provider` field to `channel` |
| `validate-timestamp.ts` | Validates at-schedule timestamps (not past/too far future) |
| `run-log.ts` | JSONL run log: append, read, prune |
| `session-reaper.ts` | Prunes expired isolated cron session entries |
| `isolated-agent.ts` | Re-exports isolated agent runner |
| `isolated-agent/run.ts` | `runCronIsolatedAgentTurn()` — full isolated agent execution |
| `isolated-agent/session.ts` | `resolveCronSession()` — creates isolated session |
| `isolated-agent/delivery-target.ts` | `resolveDeliveryTarget()` — resolves channel/recipient |
| `isolated-agent/helpers.ts` | Output picking, heartbeat detection, summary extraction |
| `isolated-agent/skills-snapshot.ts` | Build & normalize agent skill-filter snapshots for cron sessions |
| `isolated-agent/subagent-followup.ts` | Wait for/collect sub-agent results after isolated cron runs |
| `legacy-delivery.ts` | Detect legacy delivery hints (`deliver`, `bestEffortDeliver`) in payloads |
| `isolated-agent.mocks.ts` | Test mocks for isolated agent |
| `isolated-agent.test-harness.ts` | Test harness for isolated agent |
| `service.test-harness.ts` | Test harness for cron service |
| `isolated-agent/run.skill-filter.test.ts` | Tests for skill filter passing in isolated runs |
| `isolated-agent/run.cron-model-override.test.ts` | Tests for cron model override in isolated runs |
| `isolated-agent/run.payload-fallbacks.test.ts` | Tests for per-job payload fallback overrides |
| `isolated-agent.subagent-model.test.ts` | Tests for subagents.model propagation in cron |
| `isolated-agent/session.test.ts` | Tests for cron session resolution |
| `service.armtimer-tight-loop.test.ts` | Tests for armTimer tight-loop prevention |
| `service.failure-alert.test.ts` | Tests for configurable failure alerts |
| `service.issue-19676-at-reschedule.test.ts` | Tests for one-shot reschedule re-arm |
| `service.list-page-sort-guards.test.ts` | Tests for list sorting with malformed jobs |
| `service.main-job-passes-heartbeat-target-last.test.ts` | Tests heartbeat target=last for main jobs |
| + 24 test files | Various regression, integration, and unit tests |

### Key Types & Interfaces

```typescript
// types.ts
type CronSchedule = { kind: "at"; at: string } | { kind: "every"; everyMs: number; anchorMs? } | { kind: "cron"; expr: string; tz? };
type CronSessionTarget = "main" | "isolated";
type CronPayload = { kind: "systemEvent"; text } | { kind: "agentTurn"; message, model?, thinking?, timeoutSeconds?, deliver?, channel?, to?, ... };
type CronDelivery = { mode: "none" | "announce"; channel?; to?; bestEffort? };
type CronJobState = { nextRunAtMs?, runningAtMs?, lastRunAtMs?, lastStatus?, lastError?, lastDurationMs?, consecutiveErrors?, scheduleErrorCount? };
type CronFailureAlert = { after?; channel?; to?; cooldownMs? };
type CronJob = { id, agentId?, name, description?, enabled, deleteAfterRun?, schedule, sessionTarget, wakeMode, payload, delivery?, failureAlert?, state, createdAtMs, updatedAtMs };
type CronStoreFile = { version: 1; jobs: CronJob[] };
type CronJobCreate = Omit<CronJob, "id" | "createdAtMs" | "updatedAtMs" | "state"> & { state? };
type CronJobPatch = Partial<...> & { payload?: CronPayloadPatch; delivery?: CronDeliveryPatch; state? };

// service/state.ts
type CronEvent = { jobId, action, runAtMs?, durationMs?, status?, error?, summary?, sessionId?, sessionKey?, nextRunAtMs? };
type CronServiceDeps = { log, storePath, cronEnabled, enqueueSystemEvent, requestHeartbeatNow, runIsolatedAgentJob, onEvent?, ... };

// delivery.ts
type CronDeliveryPlan = { mode, channel, to?, source, requested };

// run-log.ts
type CronRunLogEntry = { ts, jobId, action, status?, error?, summary?, sessionId?, sessionKey?, runAtMs?, durationMs?, nextRunAtMs? };
```

### Key Functions & Classes

| Export | Signature | Description |
|--------|-----------|-------------|
| `CronService` | class | Main service: start/stop/list/add/update/remove/run/wake |
| `loadCronStore` | `(storePath) → Promise<CronStoreFile>` | Load jobs from JSON file |
| `saveCronStore` | `(storePath, store) → Promise<void>` | Atomic JSON write with .bak |
| `computeNextRunAtMs` | `(schedule, nowMs) → number \| undefined` | Compute next fire time |
| `resolveCronDeliveryPlan` | `(job) → CronDeliveryPlan` | Resolve delivery mode/channel |
| `parseAbsoluteTimeMs` | `(input) → number \| null` | Parse ISO/epoch timestamps |
| `validateScheduleTimestamp` | `(schedule, nowMs?) → TimestampValidationResult` | Validate at-schedule |
| `appendCronRunLog` | `(filePath, entry, opts?) → Promise<void>` | Append JSONL run log |
| `readCronRunLogEntries` | `(filePath, opts?) → Promise<CronRunLogEntry[]>` | Read run log entries |
| `sweepCronRunSessions` | `(params) → Promise<ReaperResult>` | Prune expired cron sessions |
| `runCronIsolatedAgentTurn` | `(params) → Promise<RunCronAgentTurnResult>` | Execute isolated agent job |
| `resolveCronSession` | `(params) → session data` | Create isolated cron session |
| `resolveDeliveryTarget` | `(cfg, agentId, jobPayload) → Promise<target>` | Resolve message target |
| `normalizeCronJobInput` | `(input, opts?) → CronJobCreate` | Normalize/validate job input |

### Internal Dependencies
- `src/config/config.ts`, `src/config/types.cron.ts`, `src/config/sessions.ts`
- `src/agents/agent-scope.ts`, `src/agents/cli-runner.ts`, `src/agents/model-selection.ts`, `src/agents/model-auth.ts`, `src/agents/pi-embedded.ts`, etc.
- `src/routing/session-key.ts` — session key building/sanitization
- `src/channels/registry.ts`, `src/channels/plugins/types.ts`
- `src/infra/heartbeat-wake.ts`, `src/infra/outbound/deliver.ts`, `src/infra/outbound/targets.ts`
- `src/security/external-content.ts`
- `src/auto-reply/heartbeat.ts`, `src/auto-reply/tokens.ts`
- `src/cli/parse-duration.ts`
- `src/sessions/session-key-utils.ts`
- `src/utils.ts`

### External Dependencies
- `croner` — cron expression parsing
- `json5` — JSON5 parsing for store files

### Data Flow
1. **Job Creation:** `add(input)` → normalize → create job with UUID → compute nextRunAtMs → persist → arm timer
2. **Timer Tick:** `setTimeout` fires → `locked()` → find due jobs → execute each:
   - **Main jobs:** enqueue system event text → requestHeartbeatNow/runHeartbeatOnce
   - **Isolated jobs:** `runCronIsolatedAgentTurn()` → create session → run CLI agent → collect output → deliver via messaging channel
3. **Delivery:** resolve target (channel + recipient from job config or session last) → `deliverOutboundPayloads()`
4. **Persistence:** after each state change → atomic JSON write to `jobs.json`
5. **Session Reaper:** periodic sweep → prune expired `cron:<id>:run:<uuid>` sessions from session store

### Storage
- **Jobs store:** `~/.openclaw/cron/jobs.json` (JSON, atomic write with `.bak` backup)
- **Run logs:** `~/.openclaw/cron/runs/<jobId>.jsonl` (JSONL, auto-pruned at 2MB/2000 lines)
- **Session entries:** in `sessions.json` (shared session store)

### Configuration
- `cron.enabled` — enable/disable cron service
- `cron.storePath` — custom jobs file path
- `cron.sessionRetention` — retention period for cron run sessions (default 24h)
- Jobs configured via API (add/update/remove), not static config

### Test Coverage
33+ test files: delivery plan resolution, every-job firing, regression tests (#13992, #16156, general regressions), job CRUD, duplicate timer prevention, read-ops non-blocking, rearm timer, restart catchup, one-shot execution, empty payload handling, store migration (2 files), schedule error isolation, session management, protocol conformance, skill-filter in isolated runs, cron session resolution, cron model override, payload fallbacks, subagent model propagation, armTimer tight-loop prevention, failure alerts, one-shot reschedule re-arm, list sort guards, heartbeat target for main jobs.

---

## src/media

### Module Overview
Low-level **media file handling**: MIME detection, file fetching with size limits, local media hosting via Express server, media store (save/serve/cleanup with TTL), image operations, PDF/document extraction, audio compatibility detection, base64 utilities, and PNG encoding.

**Architecture Pattern:** Utility library with stateless functions + media store singleton.

**Entry Points:**
- `store.ts` — save/serve/cleanup media files
- `server.ts` — Express media server routes
- `host.ts` — `ensureMediaHosted()` for Tailscale-accessible URLs
- `fetch.ts` — `fetchRemoteMedia()` with SSRF protection
- `input-files.ts` — PDF/document content extraction

### v2026.2.15 Changes
- **Share outbound attachment resolver** — `outbound-attachment.ts` now shared across channels
- **Share base64 MIME sniff helper** — `sniff-mime-from-base64.ts` reusable across modules
- **Treat binary `application/*` MIMEs as non-text** — prevents binary blobs from being treated as text
- **Share response size limiter** — `read-response-with-limit.ts` shared for consistent enforcement
- **New `local-roots.ts`** — resolve local media root directories (workspace, state, home)

### v2026.3.1 Changes <!-- v2026.3.1 -->

- **Outbound media load-options helper** — New `load-options.ts` provides `buildOutboundMediaLoadOptions()` and `resolveOutboundMediaLocalRoots()` to centralize outbound media load parameter assembly. `outbound-attachment.ts` now uses this helper for consistent `localRoots` propagation.

- **JavaScript MIME mapping** — `.js` files now resolve to `text/javascript` in the MIME extension mapping, fixing assets served with an incorrect content type.

- **Media server `readFileWithinRoot`** — `server.ts` now uses `readFileWithinRoot` (buffer-returning) instead of `openFileWithinRoot` (handle-returning), eliminating the open-then-close pattern and consolidating the size-limit check inside the safe-read call.

- **Outside-workspace error handling** — Media store and server now explicitly handle `SafeOpenError` with code `"outside-workspace"`, returning a 400 status with a descriptive "file is outside workspace root" message instead of a generic 404.

- **Sandbox TOCTOU hardening** — Sandbox media reads are hardened against time-of-check-to-time-of-use (TOCTOU) path escapes.

### File Inventory (34 files)

| File | Description |
|------|-------------|
| `store.ts` | Media store: save buffer/URL, serve, cleanup, TTL, SSRF-safe download |
| `server.ts` | Express routes: `GET /media/:id` with TTL, size limits |
| `host.ts` | `ensureMediaHosted()` — saves media + starts server if needed |
| `fetch.ts` | `fetchRemoteMedia()` — download with SSRF guard, size limit, content-disposition |
| `mime.ts` | MIME detection (file-type + extension fallback), extension mapping |
| `constants.ts` | Media kind enum, size limits (6MB image, 16MB audio/video, 100MB doc) |
| `audio.ts` | Telegram voice audio compatibility detection |
| `audio-tags.ts` | `parseAudioTag()` — extract `[[audio_as_voice]]` directive |
| `parse.ts` | Parse `MEDIA:` tokens from text output |
| `base64.ts` | Base64 decoded size estimation (zero-alloc) |
| `image-ops.ts` | Image resize/metadata via sharp or macOS sips, EXIF orientation |
| `input-files.ts` | PDF text/image extraction, document content loading |
| `local-roots.ts` | Resolve local media root directories (workspace, state, home) for safe file access |
| `outbound-attachment.ts` | Resolve outbound media URL → local file (shared across channels) |
| `load-options.ts` | `buildOutboundMediaLoadOptions()`, `resolveOutboundMediaLocalRoots()` — centralized outbound media load params |
| `png-encode.ts` | Minimal PNG encoder (CRC32, chunks, RGBA) |
| `read-response-with-limit.ts` | Stream response body with byte limit |
| `sniff-mime-from-base64.ts` | Detect MIME from base64 header bytes (shared helper) |
| `load-options.test.ts` | Tests for outbound media load options |
| `server.outside-workspace.test.ts` | Tests for outside-workspace error in media server |
| `store.outside-workspace.test.ts` | Tests for outside-workspace error in media store |
| + 9 test files | Tests for audio, constants, fetch, fetch-guard, host, mime, parse, store, store-redirect |

### Key Types & Interfaces

```typescript
// constants.ts
type MediaKind = "image" | "audio" | "video" | "document" | "unknown";

// fetch.ts
type FetchMediaResult = { buffer: Buffer; contentType?; fileName? };
class MediaFetchError extends Error { code: "max_bytes" | "http_error" | "fetch_failed" }

// host.ts
type HostedMedia = { url: string; id: string; size: number };

// input-files.ts
type InputImageContent = { type: "image"; data: string; mimeType: string };
type InputFileExtractResult = { filename; text?; images? };
type InputFileLimits = { allowUrl; urlAllowlist?; allowedMimes; maxBytes; ... };

// store.ts
const MEDIA_MAX_BYTES = 5 * 1024 * 1024; // 5MB

// load-options.ts
type OutboundMediaLoadParams = { maxBytes?; mediaLocalRoots?: readonly string[] };
type OutboundMediaLoadOptions = { maxBytes?; localRoots?: readonly string[] };
```

### Key Functions & Classes

| Export | Signature | Description |
|--------|-----------|-------------|
| `saveMediaBuffer` | `(buffer, contentType?, prefix?, maxBytes?) → Promise<saved>` | Save buffer to media dir |
| `saveMediaSource` | `(source) → Promise<{ id, path, size }>` | Save from URL or file path |
| `cleanOldMedia` | `(ttlMs?) → Promise<void>` | Remove expired media files |
| `getMediaDir` | `() → string` | Resolve media directory path |
| `attachMediaRoutes` | `(app, ttlMs?) → void` | Mount Express `/media/:id` route |
| `startMediaServer` | `(port, ttlMs?) → Promise<Server>` | Start standalone media server |
| `ensureMediaHosted` | `(source, opts?) → Promise<HostedMedia>` | Host media with Tailscale URL |
| `fetchRemoteMedia` | `(opts) → Promise<FetchMediaResult>` | Download remote media safely |
| `detectMime` | `(opts) → Promise<string \| undefined>` | Detect MIME from buffer/path |
| `extensionForMime` | `(mime) → string` | Map MIME to file extension |
| `mediaKindFromMime` | `(mime?) → MediaKind` | Classify MIME into media kind |
| `maxBytesForKind` | `(kind) → number` | Size limit per media kind |
| `parseMediaTokens` | `(text) → string[]` | Extract MEDIA: tokens from text |
| `isVoiceCompatibleAudio` | `(opts) → boolean` | Check Telegram voice compatibility |
| `extractFileContentFromSource` | `(source, limits) → Promise<result>` | Extract text/images from PDF/doc |
| `readResponseWithLimit` | `(res, maxBytes) → Promise<Buffer>` | Stream with byte limit |
| `estimateBase64DecodedBytes` | `(base64) → number` | Zero-alloc size estimation |
| `encodePng` | `(width, height, rgba) → Buffer` | Minimal PNG encoding |
| `buildOutboundMediaLoadOptions` | `(params?) → OutboundMediaLoadOptions` | Build load options for outbound media |
| `resolveOutboundMediaLocalRoots` | `(roots?) → readonly string[] \| undefined` | Resolve local roots for media loading |

### Internal Dependencies
- `src/infra/net/ssrf.ts`, `src/infra/net/fetch-guard.ts` — SSRF protection
- `src/infra/fs-safe.ts` — safe file read within root
- `src/infra/ports.ts`, `src/infra/tailscale.ts` — port checking, Tailscale hostname
- `src/globals.ts` — danger flags
- `src/runtime.ts` — runtime environment
- `src/process/exec.ts` — shell execution (for sips)
- `src/markdown/fences.ts` — fence parsing for MEDIA token extraction
- `src/utils/directive-tags.ts` — inline directive parsing
- `src/web/media.ts` — web media loading
- `src/cli/command-format.ts`

### External Dependencies
- `express` — HTTP server
- `file-type` — MIME detection from buffer magic bytes
- `sharp` — image processing (optional, fallback to sips on macOS)
- `@napi-rs/canvas` — PDF image rendering (optional)
- `pdfjs-dist` — PDF text extraction (optional)

### Data Flow
1. **Inbound:** URL/file → `fetchRemoteMedia()` (with SSRF guard + size limit) → `saveMediaBuffer()` → file in `~/.openclaw/media/`
2. **Serving:** Express `GET /media/:id` → check TTL → read file → detect MIME → serve
3. **Hosting:** `ensureMediaHosted()` → save → get Tailscale hostname → return `https://<hostname>/media/<id>`
4. **Cleanup:** `cleanOldMedia()` removes files older than TTL (default 2 min)

### Storage
- **Media directory:** `~/.openclaw/media/`
- **File naming:** `<prefix>---<uuid>.<ext>` (original filename preserved in pattern)
- **TTL:** 2 minutes default, expired files cleaned on access or periodic cleanup

### Configuration
- No direct config keys; uses `CONFIG_DIR` from utils
- `OPENCLAW_IMAGE_BACKEND` env var — `"sharp"` | `"sips"`

### Test Coverage
12 test files: audio compatibility, media constants, fetch with SSRF, input-files fetch guard, media hosting, MIME detection, MEDIA token parsing, store operations (headers, general), store redirects, outbound media load options, outside-workspace error (server + store).

---

## src/media-understanding

### Module Overview
**AI-powered media analysis**: audio transcription, image description, and video description using multiple providers (OpenAI, Groq, Deepgram, Google Gemini, Anthropic, MiniMax, ZAI). Supports provider fallback, scope-based access control, and CLI-based custom processors.

**Architecture Pattern:** Provider registry pattern with capability-based routing, attachment cache, concurrent processing.

**Entry Points:**
- `index.ts` — re-exports `applyMediaUnderstanding`, `formatMediaUnderstandingBody`, `resolveMediaUnderstandingScope`
- `apply.ts` — `applyMediaUnderstanding()` — main orchestrator
- `runner.ts` — `runCapability()` — runs a specific media capability

### File Inventory (51 files)

| File | Description |
|------|-------------|
| `index.ts` | Public re-exports |
| `fs.ts` | File existence helper utility |
| `types.ts` | Core types: MediaAttachment, MediaUnderstandingOutput, MediaUnderstandingProvider |
| `apply.ts` | `applyMediaUnderstanding()` — orchestrates all capabilities for a message |
| `runner.ts` | `runCapability()` — runs image/audio/video processing |
| `runner.entries.ts` | Individual entry execution (provider API or CLI) |
| `resolve.ts` | Config resolution: timeout, prompt, maxChars, maxBytes, scope |
| `scope.ts` | Scope-based access control (allow/deny by channel/chatType/keyPrefix) |
| `attachments.ts` | Attachment normalization, caching, SSRF-safe fetching |
| `format.ts` | Format media understanding outputs into message body |
| `defaults.ts` | Default models, timeouts, prompts, size limits per capability |
| `errors.ts` | `MediaUnderstandingSkipError` — maxBytes/timeout/unsupported |
| `concurrency.ts` | `runWithConcurrency()` — parallel task execution |
| `audio-preflight.ts` | Pre-mention audio transcription for voice notes in groups |
| `output-extract.ts` | Extract text from Gemini JSON responses |
| `video.ts` | Base64 size estimation for video uploads |
| `shared.ts` | Shared fetch utilities with timeout and SSRF guard |
| `image.ts` | `describeImageWithModel()` — pi-ai based image description |
| `image-runtime.ts` | Image runtime resolution |
| `openai-compatible-audio.ts` | OpenAI-compatible audio transcription |
| `transcribe-audio.ts` | Audio transcription orchestration |
| `provider-registry.ts` | Provider registry builder, `buildMediaUnderstandingRegistry()` |
| `provider-id.ts` | Provider ID types and constants |
| `media-understanding-misc.test.ts` | Miscellaneous integration tests |
| + 10 test files | Tests for apply, attachments SSRF, format, resolve, scope, runner, providers |

### Key Types & Interfaces

```typescript
// types.ts
type MediaUnderstandingKind = "audio.transcription" | "video.description" | "image.description";
type MediaUnderstandingCapability = "image" | "audio" | "video";
type MediaAttachment = { path?; url?; mime?; index; alreadyTranscribed? };
type MediaUnderstandingOutput = { kind; attachmentIndex; text; provider; model? };
type MediaUnderstandingProvider = {
  id: string;
  capabilities: MediaUnderstandingCapability[];
  transcribeAudio?: (req) => Promise<AudioTranscriptionResult>;
  describeImage?: (req) => Promise<ImageDescriptionResult>;
  describeVideo?: (req) => Promise<VideoDescriptionResult>;
};
type AudioTranscriptionRequest = { buffer; fileName; mime?; apiKey; baseUrl?; headers?; model?; language?; prompt?; timeoutMs; ... };
type MediaUnderstandingDecision = { capability; outcome; attachments: MediaUnderstandingAttachmentDecision[] };
```

### Key Functions & Classes

| Export | Signature | Description |
|--------|-----------|-------------|
| `applyMediaUnderstanding` | `(params) → Promise<ApplyMediaUnderstandingResult>` | Process all media in a message |
| `runCapability` | `(params) → Promise<RunCapabilityResult>` | Run single capability (image/audio/video) |
| `formatMediaUnderstandingBody` | `(params) → string` | Format outputs into message body |
| `resolveMediaUnderstandingScope` | `(params) → "allow" \| "deny"` | Check scope rules |
| `transcribeFirstAudio` | `(params) → Promise<string \| undefined>` | Pre-mention audio transcription |
| `buildMediaUnderstandingRegistry` | `(overrides?) → Map<string, Provider>` | Build provider registry |
| `normalizeAttachments` | `(ctx) → MediaAttachment[]` | Extract attachments from message context |
| `transcribeOpenAiCompatibleAudio` | `(params) → Promise<result>` | OpenAI Whisper API |
| `transcribeDeepgramAudio` | `(params) → Promise<result>` | Deepgram Nova API |
| `transcribeGeminiAudio` | `(params) → Promise<result>` | Gemini inline data API |
| `describeGeminiVideo` | `(params) → Promise<result>` | Gemini video via inline data |
| `describeImageWithModel` | `(params) → Promise<result>` | Generic image description via pi-ai |

### Providers
| Provider | Capabilities | Default Models |
|----------|-------------|----------------|
| OpenAI | image, audio | gpt-5-mini, gpt-4o-mini-transcribe |
| Groq | audio | whisper-large-v3-turbo |
| Deepgram | audio | nova-3 |
| Google | image, audio, video | gemini-3-flash-preview |
| Anthropic | image | claude-opus-4-6 |
| MiniMax | image | MiniMax-VL-01 |
| ZAI | image | glm-4.6v |
| Moonshot | video | moonshot-v1-8k |
| Mistral | image | pixtral-large |
| MiniMax Portal | image | MiniMax-VL-01 |

### Internal Dependencies
- `src/auto-reply/templating.ts` — MsgContext type
- `src/config/config.ts`, `src/config/types.tools.ts`
- `src/agents/model-auth.ts`, `src/agents/model-catalog.ts`, `src/agents/model-selection.ts`
- `src/agents/pi-embedded.ts`, `src/agents/minimax-vlm.ts`
- `src/globals.ts` — verbose logging
- `src/infra/net/ssrf.ts`, `src/infra/net/fetch-guard.ts`
- `src/media/fetch.ts`, `src/media/mime.ts`
- `src/process/exec.ts` — CLI command execution
- `src/channels/chat-type.ts`

### External Dependencies
- `@mariozechner/pi-ai` — model completion API
- `file-type` — MIME detection (via media/mime)

### Data Flow
1. **Inbound message** with attachments → `applyMediaUnderstanding()`
2. For each capability (image → audio → video): resolve config → check scope → select provider → download attachment (with cache) → call provider API → collect output
3. Format outputs → inject into message body → return to auto-reply pipeline

### Configuration
- `tools.media.image.*` — image understanding config (models, maxChars, maxBytes, scope)
- `tools.media.audio.*` — audio transcription config
- `tools.media.video.*` — video description config
- Each supports: `models[]` array (provider/model/CLI entries), `enabled`, `maxChars`, `maxBytes`, `timeoutSeconds`, `scope` rules

### Test Coverage
12 test files: apply e2e, attachment SSRF, format output, resolve config, scope decisions, runner (auto-audio, deepgram, vision-skip), provider-specific (deepgram audio/live, google video, openai audio), misc integration, audio test helpers.

---

## src/link-understanding

### Module Overview
**URL content extraction** from messages — detects links, runs CLI-based extractors, and injects summaries into the message body. Reuses scope infrastructure from media-understanding.

**Architecture Pattern:** Simple pipeline: detect → run → format → inject.

**Entry Points:**
- `index.ts` — re-exports all public functions
- `apply.ts` — `applyLinkUnderstanding()` — main orchestrator

### File Inventory (6 files)

| File | Description |
|------|-------------|
| `index.ts` | Public re-exports |
| `detect.ts` | `extractLinksFromMessage()` — extracts URLs, filters blocked hosts |
| `apply.ts` | `applyLinkUnderstanding()` — orchestrates detection + running + formatting |
| `runner.ts` | `runLinkUnderstanding()` — executes CLI entries per URL |
| `format.ts` | `formatLinkUnderstandingBody()` — appends outputs to message body |
| `defaults.ts` | Default timeout (30s), max links (3) |
| `detect.test.ts` | Tests for URL extraction |

### Key Types & Interfaces

```typescript
type ApplyLinkUnderstandingResult = { outputs: string[]; urls: string[] };
type LinkUnderstandingResult = { urls: string[]; outputs: string[] };
```

### Key Functions

| Export | Signature | Description |
|--------|-----------|-------------|
| `applyLinkUnderstanding` | `(params) → Promise<ApplyLinkUnderstandingResult>` | Full pipeline |
| `extractLinksFromMessage` | `(message, opts?) → string[]` | Extract HTTP(S) URLs |
| `runLinkUnderstanding` | `(params) → Promise<LinkUnderstandingResult>` | Run CLI extractors |
| `formatLinkUnderstandingBody` | `(params) → string` | Format outputs into body |

### Internal Dependencies
- `src/auto-reply/templating.ts`, `src/auto-reply/reply/inbound-context.ts`
- `src/config/config.ts`, `src/config/types.tools.ts`
- `src/globals.ts`
- `src/infra/net/ssrf.ts` — blocked host/IP detection
- `src/media-understanding/defaults.ts` — CLI_OUTPUT_MAX_BUFFER
- `src/media-understanding/resolve.ts` — resolveTimeoutMs
- `src/media-understanding/scope.ts` — scope resolution
- `src/process/exec.ts` — CLI execution

### Configuration
- `tools.links.enabled` — enable/disable
- `tools.links.maxLinks` — max URLs per message (default 3)
- `tools.links.models[]` — CLI command entries with template args
- `tools.links.scope` — access control rules
- `tools.links.timeoutSeconds` — per-entry timeout

---

## src/tts

### Module Overview
**Text-to-speech** with three providers: OpenAI, ElevenLabs, and Microsoft Edge TTS. Supports auto-TTS modes (off/always/inbound/tagged), text summarization for long content, markdown stripping, inline `[[tts:...]]` directives for voice/model control, user preferences, and channel-specific output formats (Opus for voice-bubble channels: Telegram, Feishu, WhatsApp).

**Architecture Pattern:** Config-driven with user preference overlay, provider fallback chain, directive parsing.

**Entry Points:**
- `tts.ts` — all public API (config resolution, TTS execution, auto-apply)
- `tts-core.ts` — provider implementations (OpenAI, ElevenLabs, Edge, summarization)

### v2026.3.1 Changes <!-- v2026.3.1 -->

- **TTS voice bubbles for Feishu and WhatsApp** (#27366) — `resolveOutputFormat()` now checks against a `VOICE_BUBBLE_CHANNELS` set (`"telegram"`, `"feishu"`, `"whatsapp"`) instead of a hardcoded `channelId === "telegram"` comparison. Opus output and `voiceCompatible` tagging are applied for all three channels. `maybeApplyTtsToPayload()` similarly uses the set for voice-bubble detection.

### File Inventory (4 files)

| File | Description |
|------|-------------|
| `tts.ts` | Public API: config, preferences, textToSpeech, maybeApplyTtsToPayload |
| `tts-core.ts` | Provider implementations, directive parsing, summarization |
| `prepare-text.test.ts` | Tests for markdown stripping in TTS pipeline |
| `tts.test.ts` | Comprehensive tests for TTS config, providers, directives, summarization |

### Key Types & Interfaces

```typescript
type TtsAutoMode = "off" | "always" | "inbound" | "tagged";
type TtsMode = "final" | "block";
type TtsProvider = "openai" | "elevenlabs" | "edge";
type ResolvedTtsConfig = {
  auto; mode; provider; providerSource; summaryModel?; modelOverrides;
  elevenlabs: { apiKey?; baseUrl; voiceId; modelId; voiceSettings; seed?; ... };
  openai: { apiKey?; model; voice };
  edge: { enabled; voice; lang; outputFormat; pitch?; rate?; volume?; ... };
  prefsPath?; maxTextLength; timeoutMs;
};
type TtsResult = { success; audioPath?; error?; latencyMs?; provider?; outputFormat?; voiceCompatible? };
type TtsDirectiveOverrides = { ttsText?; provider?; openai?; elevenlabs? };
type TtsDirectiveParseResult = { cleanedText; ttsText?; hasDirective; overrides; warnings };
```

### Key Functions

| Export | Signature | Description |
|--------|-----------|-------------|
| `resolveTtsConfig` | `(cfg) → ResolvedTtsConfig` | Resolve TTS config with defaults |
| `textToSpeech` | `(params) → Promise<TtsResult>` | Convert text to audio file |
| `textToSpeechTelephony` | `(params) → Promise<TtsTelephonyResult>` | TTS for telephony (PCM) |
| `maybeApplyTtsToPayload` | `(params) → Promise<ReplyPayload>` | Auto-TTS on reply payloads |
| `isTtsEnabled` | `(config, prefsPath, sessionAuto?) → boolean` | Check if TTS is active |
| `buildTtsSystemPromptHint` | `(cfg) → string \| undefined` | System prompt hint for TTS |
| `parseTtsDirectives` | `(text, policy) → TtsDirectiveParseResult` | Parse `[[tts:...]]` tags |
| `summarizeText` | `(params) → Promise<SummarizeResult>` | LLM-based text summarization |
| `openaiTTS` | `(params) → Promise<Buffer>` | OpenAI TTS API call |
| `elevenLabsTTS` | `(params) → Promise<Buffer>` | ElevenLabs TTS API call |
| `edgeTTS` | `(params) → Promise<void>` | Microsoft Edge TTS |
| `get/setTtsAutoMode` | pref helpers | Manage user TTS preferences |
| `get/setTtsProvider` | pref helpers | Manage provider preference |
| `get/setTtsMaxLength` | pref helpers | Manage max length preference |

### Internal Dependencies
- `src/config/config.ts`, `src/config/types.tts.ts`
- `src/agents/model-auth.ts`, `src/agents/model-selection.ts`
- `src/agents/pi-embedded-runner/model.ts`
- `src/auto-reply/types.ts` — ReplyPayload
- `src/channels/plugins/index.ts`, `src/channels/plugins/types.ts`
- `src/globals.ts` — verbose logging
- `src/line/markdown-to-line.ts` — `stripMarkdown()`
- `src/media/audio.ts` — voice compatibility check
- `src/utils.ts`

### External Dependencies
- `@mariozechner/pi-ai` — LLM completion for summarization
- `node-edge-tts` — Microsoft Edge TTS

### Data Flow
1. `maybeApplyTtsToPayload()` → check auto mode → parse directives → optionally summarize → strip markdown → call provider → save audio file → attach to payload
2. Provider chain: try primary → fallback to others (openai → elevenlabs → edge)
3. Output: Opus for voice-bubble channels (Telegram, Feishu, WhatsApp), MP3 for others

### Storage
- **User preferences:** `~/.openclaw/settings/tts.json` (or configurable path)
- **Audio files:** temp dirs with 5-minute cleanup timer

### Configuration
- `messages.tts.auto` — auto mode (off/always/inbound/tagged)
- `messages.tts.mode` — when to apply (final/block)
- `messages.tts.provider` — primary provider
- `messages.tts.openai.*` — OpenAI voice/model config
- `messages.tts.elevenlabs.*` — ElevenLabs voice settings
- `messages.tts.edge.*` — Edge TTS voice/lang/format
- `messages.tts.modelOverrides.*` — directive permission policy
- `messages.tts.summaryModel` — model for text summarization
- `messages.tts.maxTextLength` — hard limit (default 4096)
- `messages.tts.timeoutMs` — provider timeout (default 30s)
- `OPENAI_TTS_BASE_URL` env — custom OpenAI-compatible endpoint

---

## src/markdown

### Module Overview
**Markdown processing**: parsing to intermediate representation (IR), rendering with custom style markers, WhatsApp format conversion, YAML frontmatter extraction, code fence/span detection, and table conversion.

**Architecture Pattern:** Parser → IR → Renderer pipeline. Uses `markdown-it` for parsing.

**Entry Points:**
- `ir.ts` — `markdownToIR()`, `markdownToIRWithMeta()` — parse markdown to IR
- `render.ts` — `renderMarkdownWithMarkers()` — render IR with custom markers
- `whatsapp.ts` — `markdownToWhatsApp()` — convert to WhatsApp formatting
- `frontmatter.ts` — `parseFrontmatter()` — extract YAML frontmatter

### File Inventory (14 files)

| File | Description |
|------|-------------|
| `ir.ts` | Markdown → IR parser (uses markdown-it) |
| `render.ts` | IR → text renderer with style markers and links |
| `whatsapp.ts` | Standard markdown → WhatsApp formatting conversion |
| `frontmatter.ts` | YAML frontmatter extraction from markdown files |
| `fences.ts` | Code fence span detection (`parseFenceSpans`) |
| `code-spans.ts` | Inline code span detection, `buildCodeSpanIndex` |
| `tables.ts` | `convertMarkdownTables()` — table format conversion |
| `frontmatter.test.ts` | Frontmatter parsing tests |
| `whatsapp.test.ts` | WhatsApp conversion tests |
| `ir.blockquote-spacing.test.ts` | Blockquote spacing tests |
| `ir.hr-spacing.test.ts` | Horizontal rule spacing tests |
| `ir.list-spacing.test.ts` | List spacing tests |
| `ir.nested-lists.test.ts` | Nested list tests |
| `ir.table-bullets.test.ts` | Table → bullet conversion tests |
| `ir.table-code.test.ts` | Table with code tests |

### Key Types & Interfaces

```typescript
// ir.ts
type MarkdownStyle = "bold" | "italic" | "strikethrough" | "code" | "code_block" | "spoiler" | "blockquote";
type MarkdownStyleSpan = { start: number; end: number; style: MarkdownStyle };
type MarkdownLinkSpan = { start: number; end: number; href: string };
type MarkdownIR = { text: string; styles: MarkdownStyleSpan[]; links: MarkdownLinkSpan[] };

// render.ts
type RenderStyleMarker = { open: string; close: string };
type RenderStyleMap = Partial<Record<MarkdownStyle, RenderStyleMarker>>;

// fences.ts
type FenceSpan = { start; end; openLine; marker; indent };

// frontmatter.ts
type ParsedFrontmatter = Record<string, string>;
```

### Key Functions

| Export | Signature | Description |
|--------|-----------|-------------|
| `markdownToIR` | `(text, opts?) → MarkdownIR` | Parse markdown to intermediate representation |
| `markdownToIRWithMeta` | `(text, opts?) → { ir; hasTables }` | Parse with metadata |
| `renderMarkdownWithMarkers` | `(ir, options) → string` | Render IR with custom style markers |
| `markdownToWhatsApp` | `(text) → string` | Convert markdown to WhatsApp format |
| `parseFrontmatter` | `(text) → { frontmatter; body }` | Extract YAML frontmatter |
| `parseFenceSpans` | `(buffer) → FenceSpan[]` | Detect code fence regions |
| `buildCodeSpanIndex` | `(text, state?) → CodeSpanIndex` | Build inline code span index |
| `convertMarkdownTables` | `(markdown, mode) → string` | Convert tables to bullets/text |

### Internal Dependencies
- `src/config/types.base.ts` — MarkdownTableMode type
- `src/auto-reply/chunk.ts` — text chunking

### External Dependencies
- `markdown-it` — markdown parser
- `yaml` — YAML frontmatter parsing

### Test Coverage

## v2026.3.7 Delta Notes

- SQLite contention resilience: the memory SQLite backend now retries operations on `SQLITE_BUSY` contention, reducing dropped writes during concurrent agent activity.
- Memory hybrid search BM25 ordering: BM25 keyword hit ordering in hybrid vector+keyword search is now stable and consistent with relevance ranking.
- Memory flush daily file canonicalization: daily memory flush output files are canonicalized (resolved to absolute real paths) before write, preventing creation of duplicate flush artifacts from symlinked directories.
- QMD collection-name conflict recovery: collection-name conflicts detected at startup are resolved automatically without data loss.
- QMD search result decoding: search results with `qmd://` scheme URIs are decoded correctly, fixing blank results in some QMD-backed memory setups.
7 test files: frontmatter parsing, WhatsApp conversion, IR blockquote/HR/list spacing, nested lists, table → bullets conversion, table with code.

## v2026.3.8 Delta Notes

- **Cron restart catch-up staggering:** missed-job replay on startup is now limited and rescheduled from the post-catch-up clock so restart bursts do not starve the gateway or silently skip overdue recurring jobs.
- **Cron/Telegram announce delivery:** text-only announce jobs now route through the real outbound adapters after descendant finalization, fixing false-positive `delivered: true` results for plain Telegram targets.
- **Cron owner-context recovery:** trusted isolated cron runs once again receive owner context so `cron` / `gateway` tooling remains available after earlier owner-auth hardening.
- **Telegram media download recovery:** stalled body reads now time out without aborting slow-but-still-streaming downloads, preventing hung file downloads from wedging the polling loop.

## v2026.3.11 Delta Notes

- **Multimodal memory indexing (opt-in):** image and audio files in `memorySearch.extraPaths` can now be indexed using Gemini `gemini-embedding-2-preview`; scope-based reindexing fires automatically when multimodal settings change (#43460).
- **`gemini-embedding-2-preview` memory support:** configurable output dimensions (768/1536/3072) with automatic reindex on dimension changes; Gemini embeddings are now L2-normalized (#42501, #43409).
- **Multimodal file revalidation:** files are rechecked against the filesystem immediately before indexing — size changes after discovery abort the index entry.
- **`memoryFlushWritePath` forwarding fix:** correctly threaded through `runEmbeddedPiAgent` so memory flush writes always land at the configured path (#41761).
- **Memory manager CLI teardown:** cached search/index managers are now explicitly closed during one-shot CLI shutdown, preventing dangling SQLite handles (#40389).
- **BREAKING — Cron delivery isolation tightened:** isolated cron jobs can no longer fall back to ad hoc agent sends or main-session summaries; run `openclaw doctor --fix` to migrate legacy storage and delivery metadata (#40998).
- **Cron `NO_REPLY`/empty response fix:** empty or `NO_REPLY` cron subagent responses are no longer misidentified as interim acknowledgements, preventing spurious retry turns (#41383).
- **Cron `lastErrorReason` persistence:** failed run error reasons are now stored in job state and the gateway schema covers the full `FailoverReason` set (#14382).

## v2026.3.12 Delta Notes

### Memory / Session Sync

- **Post-compaction session reindexing** (#25561): After compaction completes, the memory manager now performs a session reindex pass. The behaviour is controlled by two new config keys: `agents.defaults.compaction.postIndexSync` (`"off"` | `"async"` | `"await"`, default `"async"`) and `agents.defaults.memorySearch.sync.sessions.postCompactionForce` (boolean, default `true`). `postCompactionForce=true` forces a full session reindex after a compaction-triggered transcript write even if the normal delta thresholds have not been crossed, so compacted summaries are immediately searchable.

### Cron

- **Direct cron sends excluded from write-ahead resend queue** (#40646): Isolated direct cron send operations are no longer enqueued into the write-ahead resend queue. Previously, after a gateway restart these entries could replay and produce duplicate messages. The fix scopes direct cron delivery to fire-and-forget, consistent with isolated session semantics.

- **Doctor no longer flags canonical `agentTurn`/`systemEvent` payload kinds as legacy** (#44012): The `openclaw doctor` check that previously flagged `agentTurn` and `systemEvent` as legacy payload kind strings now only normalizes whitespace-padded variants (e.g. `" agentTurn "`) rather than the canonical forms.

- **Fast mode enabled for isolated cron runs**: `runCronIsolatedAgentTurn()` now passes the resolved `fastMode` state through to the embedded runner for isolated cron sessions, matching the behavior already available for main-session runs.

### Auto-reply / Compaction

- **Status reaction shown during context compaction pauses** (#35474): A visual status reaction is now emitted to the channel while the agent is paused waiting for context compaction to complete, giving users feedback that the agent is active rather than hung.

### Delivery / Dedupe

- **Direct-cron delivery cache trimmed correctly** (#44666): Stale entries in the completed direct-cron delivery cache are now correctly pruned. Mirrored transcript deduplication continues to function even when individual JSONL lines in the transcript are malformed.

---

## v2026.3.22-v2026.3.23 Delta Notes

### Memory

- **`memory_search` and `memory_get` register independently** (`v2026.3.22`) so one unavailable memory tool no longer suppresses the other.
- **Memory plugins can inject their own system-prompt section** (`v2026.3.22`) through the released prompt-section builder path.
- **`memory-lancedb` boots on first use in packaged installs** (`v2026.3.23`) so `plugins.slots.memory="memory-lancedb"` works again after global npm installs without moving LanceDB into core dependencies.

### Memory / Bootstrap

- **Load only one root memory file; prefer `MEMORY.md`, fall back to `memory.md`** (#26054): `resolveMemoryBootstrapEntry()` in `workspace.ts` now returns at most one root memory file per workspace. It tries `MEMORY.md` first; only if that file is absent does it try `memory.md`. Previously, both files could be loaded as bootstrap context on case-insensitive filesystems (e.g. macOS Docker volumes), wasting tokens by injecting the same content twice.

### Cron / Isolated Sessions

- **Nested cron-triggered embedded runner work routed onto nested lane**: `runCronIsolatedAgentTurn()` now calls `resolveNestedAgentLane(params.lane)` when launching the embedded runner. This routes any nested work triggered from inside a cron-isolated session (compaction, inner tool completions, etc.) onto a dedicated nested lane, preventing deadlocks that could arise when inner queued work competed with the outer cron session on the same lane.

### Cron / CLI

- **One-shot timezone scheduling fixed** (`v2026.3.23-2`) — `openclaw cron add|edit --at ... --tz <iana>` now honors the requested local wall-clock time for offset-less one-shot datetimes, including DST boundaries.

---

## v2026.3.24 Delta Notes

### Memory

- **Compaction timeout recovery:** `src/agents/pi-embedded-runner/run/compaction-timeout.ts` adds recovery logic for compaction operations that exceed their timeout, preventing agent sessions from hanging indefinitely during context compaction.

- **Memory builtin SQLite — reduced sync/status query churn:** the builtin SQLite memory backend reduces unnecessary sync and status query frequency, lowering database contention during normal operation.

- **Memory-lancedb — HTTP/HTTPS proxy dispatcher bootstrap:** the `memory-lancedb` extension now bootstraps an HTTP/HTTPS proxy dispatcher on startup, enabling LanceDB connections through corporate proxy environments.

### Agents / Cooldowns

- **Cooldown per-model scoping:** `src/agents/auth-profiles/usage.ts` now scopes cooldown tracking per-model rather than per-profile, so a rate limit on one model does not unnecessarily cool down other models on the same auth profile.

---

## v2026.3.28 Delta Notes

### Memory / QMD — CJK Chunking (#40271)

- **CJK-aware chunk size estimation:** `packages/memory-host-sdk/src/host/internal.ts` now calls `estimateStringChars()` from the new `src/utils/cjk-chars.ts` helper when deciding whether a coarsely-sliced line segment is within the token budget. Each non-Latin (CJK, Hangul, Hiragana, Katakana, CJK Extension B+) character is counted as `CHARS_PER_TOKEN_ESTIMATE` (4) chars so the `chars / 4` token estimate stays accurate for CJK-heavy notes.
- **Surrogate-pair guard during fine splits:** when a CJK-heavy segment requires a second fine-grained split, the splitter now checks whether the split point falls on a UTF-16 high surrogate (range `0xD800–0xDBFF`) and advances one position to include the paired low surrogate. This prevents breaking CJK Extension B+ characters (U+20000–U+2FA1F) that are encoded as surrogate pairs.
- **Latin line boundary preservation:** the first-pass coarse slice still runs at `maxChars` (tokens × `CHARS_PER_TOKEN_ESTIMATE`) unchanged, so Latin content continues to use the original `chars`-based chunk boundaries and is unaffected by the CJK re-split path.

The helper module `src/utils/cjk-chars.ts` exports `estimateStringChars(text)`, `CHARS_PER_TOKEN_ESTIMATE = 4`, and `estimateTokensFromChars(chars)`. The regex `NON_LATIN_RE` covers CJK Unified Ideographs, Extension A/B, CJK Compatibility Ideographs, Hangul Syllables, Hiragana, and Katakana.

### Memory / QMD — Slugified Path Resolution (#50313)

- **Search hit path de-slugification:** `extensions/memory-core/src/memory/qmd-manager.ts` now resolves `qmd://`-scheme file references returned by the qmd search binary back to the actual indexed filesystem path stored in the QMD SQLite database. Previously, QMD slugified collection-relative paths (e.g. `extra-docs/category/sub-category/topic-name/topic-name.md`) and returned them as `qmd://<collection>/<slugified-path>`, which did not match the original mixed-case or space-containing path used by `memory_get`.
- **Resolution flow:** after `search()` receives results, `resolveHintedDocPath()` performs an exact SQLite lookup (`documents WHERE collection = ? AND path = ? AND active = 1`) using the normalized slugified path. On miss it falls back to a fuzzy `matchesPreferredFileHint()` scan over all active documents in the collection. The resolved original path is substituted into the result so downstream `readFile()` calls hit the correct file.
- **New test file:** `extensions/memory-core/src/memory/qmd-manager.slugified-paths.test.ts` covers workspace and extra-collection slug resolution end-to-end.

### Memory / QMD — embedInterval (#37326)

- **Dedicated embed-cadence timer:** `extensions/memory-core/src/memory/qmd-manager.ts` now arms a separate `embedTimer` (`setInterval`) when `shouldScheduleEmbedTimer()` returns true. This happens when `memory.qmd.update.embedInterval` is configured and the regular update interval is either disabled (`intervalMs <= 0`) or slower than the embed interval (`updateIntervalMs > embedIntervalMs`). Previously the embed pass only ran as a side-effect of the regular update cycle, so setting `embedInterval` shorter than `interval` had no effect.
- **Deduplication guard:** `shouldScheduleEmbedTimer()` returns false when the regular update timer is already firing at least as frequently as the requested embed cadence, preventing a redundant second timer.
- **`shouldRunEmbed()` logic unchanged:** the existing guard (`lastEmbedAt === null || now - lastEmbedAt > embedIntervalMs`) still gates each embed pass independently of which timer triggered it.

### Memory Flush — Append-Only (#53725)

- **Append-only enforcement for embedded flush attempts:** `src/agents/pi-tools.read.ts` `wrapToolMemoryFlushAppendOnlyWrite()` enforces that during a memory flush run the write tool can only append to the configured daily file (`memory/YYYY-MM-DD.md`). Writes to any other path throw immediately. The core `appendMemoryFlushContent()` function uses `appendFileWithinRoot()` (non-sandbox path) or reads the existing file and concatenates new content before writing (sandbox path), ensuring earlier notes in the same file are never overwritten.
- **Flush plan prompt hints:** `extensions/memory-core/src/flush-plan.ts` injects `MEMORY_FLUSH_APPEND_ONLY_HINT` ("If memory/YYYY-MM-DD.md already exists, APPEND new content only and do not overwrite existing entries.") into both the user prompt and system prompt for every flush turn, making the append-only constraint explicit to the model as well as enforced by the tool wrapper.

### Memory / Plugins — Flush Plan Contract (#52709 area / v2026.3.28)

- **Flush plan owned by `memory-core`:** the pre-compaction memory flush plan (prompt text, system prompt, target `relativePath`, soft-threshold tokens, force-flush byte threshold) is now registered through the active memory plugin contract via `api.registerMemoryFlushPlan(buildMemoryFlushPlan)` in `extensions/memory-core/index.ts`. Core logic calls `resolveMemoryFlushPlan()` from `src/plugins/memory-state.ts` to retrieve the plan at runtime; the hardcoded defaults have been removed from core. This means only a loaded memory plugin (e.g. `memory-core`) can supply flush prompts and target-path policy — third-party memory plugins can override this by registering their own resolver.
- **`MemoryFlushPlan` type:** defined in `src/plugins/memory-state.ts` as `{ softThresholdTokens, forceFlushTranscriptBytes, reserveTokensFloor, prompt, systemPrompt, relativePath }`. The resolver signature is `(params: { cfg?, nowMs? }) => MemoryFlushPlan | null`.

### Memory / FTS — CJK Trigram Tokenization

- **Configurable trigram tokenizer:** `packages/memory-host-sdk/src/host/memory-schema.ts` now accepts `ftsTokenizer: "trigram"` in addition to the default `"unicode61"`. When `"trigram"` is selected, the FTS5 virtual table is created with `tokenize='trigram case_sensitive 0'`, enabling substring search for CJK text that does not segment on whitespace.
- **Provider-less keyword hit visibility fix (#56473):** FTS-only keyword hits are now visible at the default memory-search threshold without requiring `--min-score 0`.

## v2026.3.31 Delta Notes

### Memory / QMD

- **Per-agent extra collections are part of the stable line:** `memorySearch.qmd.extraCollections` allows controlled cross-agent transcript search without flattening all transcript history into one namespace.
- **Stable search quality work continued after `v2026.3.28`:** the released line adds additional QMD/FTS fixes around substring search, embed cadence, path round-tripping, and search-mode embedding behavior.

### Cron / Detached Work

- **Cron work now shares the same durable task ledger as ACP and subagents:** release-line cron documentation should treat task records, audit, and maintenance as part of normal detached-run behavior rather than an ACP-only implementation detail.

## v2026.4.9 Delta Notes

### Memory / Dreaming

- **Released memory surfaces are split across `src/memory-host-sdk/` and `extensions/memory-core/src/memory/`:** embeddings, dreaming, and promotion live under `src/memory-host-sdk/`; QMD/search backends and sync/index orchestration live under `extensions/memory-core/src/memory/` rather than the older standalone `src/memory/` tree referenced by earlier snapshots.
- **Dreaming is a stable-line feature:** weighted recall promotion, `/dreaming`, Dreams UI, multilingual conceptual tagging, aging controls, REM preview tooling, and `dreams.md` are now part of the released memory surface.
- **Bedrock embeddings join the released memory provider set:** Bedrock Titan, Cohere, Nova, and TwelveLabs embeddings are now part of the stable memory/search story.

### Cron / Background Work

- **Cron remains on the shared task ledger:** the `v2026.4.9` line keeps cron, ACP, subagents, and detached CLI runs tied to the same durable task/control-plane surfaces, with more emphasis on automatic completion wake behavior and primary-channel failure delivery.

### Media Generation

- **Music and video generation join the cluster:** the released line adds dedicated music/video generation surfaces and async completion behavior, so media docs can no longer treat image/media understanding as the only major media runtime.
