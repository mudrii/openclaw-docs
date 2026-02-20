# OpenClaw Analysis: Memory, Cron & Media Cluster

> Updated: 2026-02-20 | Codebase: ~/src/openclaw | Version: v2026.2.19

---

## Table of Contents
1. [src/memory](#srcmemory)
2. [src/cron](#srccron)
3. [src/media](#srcmedia)
4. [src/media-understanding](#srcmedia-understanding)
5. [src/link-understanding](#srclink-understanding)
6. [src/tts](#srctts)
7. [src/markdown](#srcmarkdown)

---

## src/memory

### Module Overview
The memory module provides **semantic search over markdown files and session transcripts** using vector embeddings. It supports multiple embedding providers (OpenAI, Gemini, Voyage, local llama.cpp), stores chunks in SQLite with optional sqlite-vec for native vector search, and implements hybrid search (vector + BM25 full-text). An alternative backend (`qmd`) delegates to an external QMD binary for indexing/search.

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
- **New `sync-progress.ts`** — typed sync progress state & reporting callback
- **New `batch-utils.ts`** — shared batch HTTP client config types

### File Inventory (63 files)

| File | Description |
|------|-------------|
| `index.ts` | Public API re-exports |
| `index.test.ts` | Integration tests for memory index |
| `types.ts` | Core types: MemorySearchManager interface, MemorySearchResult, MemorySource |
| `manager.ts` | `MemoryIndexManager` class — main builtin memory backend |
| `manager-embedding-ops.ts` | Mixin: embedding batch logic, cache, retry |
| `manager-sync-ops.ts` | Mixin: file watching, sync, vector loading, session tracking |
| `manager-search.ts` | Vector and keyword search query execution |
| `manager-cache-key.ts` | Computes cache key for manager deduplication |
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
| `node-llama.ts` | Dynamic import wrapper for node-llama-cpp |
| `batch-http.ts` | Generic HTTP POST with retry for batch APIs |
| `batch-output.ts` | Shared batch output line parser |
| `batch-utils.ts` | Shared batch HTTP client config types |
| `batch-openai.ts` | OpenAI batch embedding API |
| `batch-gemini.ts` | Gemini batch embedding API |
| `batch-voyage.ts` | Voyage batch embedding API |
| `openai-batch.ts` | Deprecated re-export of batch-openai |
| `embedding-chunk-limits.ts` | Enforces max input tokens per chunk |
| `embedding-input-limits.ts` | UTF-8 byte estimation, text splitting |
| `embedding-model-limits.ts` | Known max input tokens per model |
| `provider-key.ts` | Computes provider fingerprint for cache invalidation |
| `headers-fingerprint.ts` | Fingerprints header names for cache keys |
| `hybrid.ts` | BM25+vector hybrid search merging |
| `sqlite.ts` | `requireNodeSqlite()` — loads node:sqlite with warning filter |
| `sqlite-vec.ts` | Loads sqlite-vec native extension |
| `memory-schema.ts` | SQLite schema creation (meta, files, chunks, embedding_cache, FTS5) |
| `session-files.ts` | Reads session JSONL transcripts, builds entries with line maps |
| `sync-index.ts` | Generic file-entry-changed check for incremental sync |
| `sync-memory-files.ts` | Syncs memory markdown files to index |
| `sync-session-files.ts` | Syncs session transcript files to index |
| `sync-progress.ts` | Typed sync progress state, reporting callback for UI updates |
| `sync-stale.ts` | Deletes stale indexed paths |
| `status-format.ts` | Formatting helpers for memory status display |
| `embedding-chunk-limits.test.ts` | Tests for chunk limit enforcement |
| `embedding-manager.test-harness.ts` | Test fixture for embedding manager tests |
| `embedding.test-mocks.ts` | Vitest mocks for embeddings |
| `backend-config.test.ts` | Tests for backend config resolution |
| `embeddings.test.ts` | Tests for embedding provider creation |
| `embeddings-voyage.test.ts` | Tests for Voyage provider |
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
type EmbeddingProviderId = "openai" | "local" | "gemini" | "voyage";
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
- `src/config/config.js` — OpenClawConfig type
- `src/config/types.memory.js` — MemoryBackend, MemoryQmdConfig types
- `src/config/sessions/paths.js` — session transcript paths
- `src/config/sessions.js` — session store operations
- `src/agents/agent-scope.js` — workspace/agent directory resolution
- `src/agents/memory-search.js` — ResolvedMemorySearchConfig
- `src/agents/model-auth.js` — API key resolution
- `src/cli/parse-duration.js` — duration string parsing
- `src/infra/env.js`, `src/infra/retry.js`, `src/infra/warning-filter.js`
- `src/logging/subsystem.js`, `src/logging/redact.js`
- `src/sessions/transcript-events.js`, `src/sessions/session-key-utils.js`
- `src/utils.js` — resolveUserPath, truncateUtf16Safe

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

### Test Coverage
18 test files covering: backend config resolution, embedding chunk limits, embedding providers (OpenAI, Voyage, Gemini), batch APIs, hybrid search, chunking/file listing, session entry parsing, QMD manager/parser/scope, search manager factory, async search, atomic reindex, batch splitting, token limits, sync error handling, vector deduplication, watcher configuration.

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

### File Inventory (57 files)

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
| `isolated-agent/session.test.ts` | Tests for cron session resolution |
| + 24 test files | Various regression, integration, and unit tests |

### Key Types & Interfaces

```typescript
// types.ts
type CronSchedule = { kind: "at"; at: string } | { kind: "every"; everyMs: number; anchorMs? } | { kind: "cron"; expr: string; tz? };
type CronSessionTarget = "main" | "isolated";
type CronPayload = { kind: "systemEvent"; text } | { kind: "agentTurn"; message, model?, thinking?, timeoutSeconds?, deliver?, channel?, to?, ... };
type CronDelivery = { mode: "none" | "announce"; channel?; to?; bestEffort? };
type CronJobState = { nextRunAtMs?, runningAtMs?, lastRunAtMs?, lastStatus?, lastError?, lastDurationMs?, consecutiveErrors?, scheduleErrorCount? };
type CronJob = { id, agentId?, name, description?, enabled, deleteAfterRun?, schedule, sessionTarget, wakeMode, payload, delivery?, state, createdAtMs, updatedAtMs };
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
- `src/config/config.js`, `src/config/types.cron.js`, `src/config/sessions.js`
- `src/agents/agent-scope.js`, `src/agents/cli-runner.js`, `src/agents/model-selection.js`, `src/agents/model-auth.js`, `src/agents/pi-embedded.js`, etc.
- `src/routing/session-key.js` — session key building/sanitization
- `src/channels/registry.js`, `src/channels/plugins/types.js`
- `src/infra/heartbeat-wake.js`, `src/infra/outbound/deliver.js`, `src/infra/outbound/targets.js`
- `src/security/external-content.js`
- `src/auto-reply/heartbeat.js`, `src/auto-reply/tokens.js`
- `src/cli/parse-duration.js`
- `src/sessions/session-key-utils.js`
- `src/utils.js`

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
26+ test files: delivery plan resolution, every-job firing, regression tests (#13992, #16156, general regressions), job CRUD, duplicate timer prevention, read-ops non-blocking, rearm timer, restart catchup, one-shot execution, empty payload handling, store migration (2 files), schedule error isolation, session management, protocol conformance, skill-filter in isolated runs, cron session resolution.

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

### File Inventory (26 files)

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
| `png-encode.ts` | Minimal PNG encoder (CRC32, chunks, RGBA) |
| `read-response-with-limit.ts` | Stream response body with byte limit |
| `sniff-mime-from-base64.ts` | Detect MIME from base64 header bytes (shared helper) |
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

### Internal Dependencies
- `src/infra/net/ssrf.js`, `src/infra/net/fetch-guard.js` — SSRF protection
- `src/infra/fs-safe.js` — safe file open within root
- `src/infra/ports.js`, `src/infra/tailscale.js` — port checking, Tailscale hostname
- `src/globals.js` — danger flags
- `src/runtime.js` — runtime environment
- `src/process/exec.js` — shell execution (for sips)
- `src/markdown/fences.js` — fence parsing for MEDIA token extraction
- `src/utils/directive-tags.js` — inline directive parsing
- `src/web/media.js` — web media loading
- `src/cli/command-format.js`

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
9 test files: audio compatibility, media constants, fetch with SSRF, input-files fetch guard, media hosting, MIME detection, MEDIA token parsing, store operations (headers, general), store redirects.

---

## src/media-understanding

### Module Overview
**AI-powered media analysis**: audio transcription, image description, and video description using multiple providers (OpenAI, Groq, Deepgram, Google Gemini, Anthropic, MiniMax, ZAI). Supports provider fallback, scope-based access control, and CLI-based custom processors.

**Architecture Pattern:** Provider registry pattern with capability-based routing, attachment cache, concurrent processing.

**Entry Points:**
- `index.ts` — re-exports `applyMediaUnderstanding`, `formatMediaUnderstandingBody`, `resolveMediaUnderstandingScope`
- `apply.ts` — `applyMediaUnderstanding()` — main orchestrator
- `runner.ts` — `runCapability()` — runs a specific media capability

### File Inventory (43 files)

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
| `providers/index.ts` | Provider registry builder |
| `providers/shared.ts` | Shared fetch utilities with timeout and SSRF guard |
| `providers/image.ts` | `describeImageWithModel()` — pi-ai based image description |
| `providers/anthropic/index.ts` | Anthropic provider (image only) |
| `providers/openai/index.ts` | OpenAI provider (image + audio) |
| `providers/openai/audio.ts` | OpenAI-compatible audio transcription |
| `providers/google/index.ts` | Google provider (image + audio + video) |
| `providers/google/audio.ts` | Gemini audio transcription |
| `providers/google/video.ts` | Gemini video description |
| `providers/google/inline-data.ts` | Gemini inline data API helper |
| `providers/groq/index.ts` | Groq provider (audio via OpenAI-compatible API) |
| `providers/deepgram/index.ts` | Deepgram provider (audio) |
| `providers/deepgram/audio.ts` | Deepgram Nova transcription |
| `providers/minimax/index.ts` | MiniMax provider (image) |
| `providers/zai/index.ts` | ZAI/GLM provider (image) |
| `providers/audio.test-helpers.ts` | Shared audio test helpers for provider tests |
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
| `buildProviderRegistry` | `(overrides?) → Map<string, Provider>` | Build provider registry |
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

### Internal Dependencies
- `src/auto-reply/templating.js` — MsgContext type
- `src/config/config.js`, `src/config/types.tools.js`
- `src/agents/model-auth.js`, `src/agents/model-catalog.js`, `src/agents/model-selection.js`
- `src/agents/pi-embedded.js`, `src/agents/minimax-vlm.js`
- `src/globals.js` — verbose logging
- `src/infra/net/ssrf.js`, `src/infra/net/fetch-guard.js`
- `src/media/fetch.js`, `src/media/mime.js`
- `src/process/exec.js` — CLI command execution
- `src/channels/chat-type.js`

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

### File Inventory (7 files)

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
- `src/auto-reply/templating.js`, `src/auto-reply/reply/inbound-context.js`
- `src/config/config.js`, `src/config/types.tools.js`
- `src/globals.js`
- `src/infra/net/ssrf.js` — blocked host/IP detection
- `src/media-understanding/defaults.js` — CLI_OUTPUT_MAX_BUFFER
- `src/media-understanding/resolve.js` — resolveTimeoutMs
- `src/media-understanding/scope.js` — scope resolution
- `src/process/exec.js` — CLI execution

### Configuration
- `tools.links.enabled` — enable/disable
- `tools.links.maxLinks` — max URLs per message (default 3)
- `tools.links.models[]` — CLI command entries with template args
- `tools.links.scope` — access control rules
- `tools.links.timeoutSeconds` — per-entry timeout

---

## src/tts

### Module Overview
**Text-to-speech** with three providers: OpenAI, ElevenLabs, and Microsoft Edge TTS. Supports auto-TTS modes (off/always/inbound/tagged), text summarization for long content, markdown stripping, inline `[[tts:...]]` directives for voice/model control, user preferences, and channel-specific output formats (Opus for Telegram).

**Architecture Pattern:** Config-driven with user preference overlay, provider fallback chain, directive parsing.

**Entry Points:**
- `tts.ts` — all public API (config resolution, TTS execution, auto-apply)
- `tts-core.ts` — provider implementations (OpenAI, ElevenLabs, Edge, summarization)

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
- `src/config/config.js`, `src/config/types.tts.js`
- `src/agents/model-auth.js`, `src/agents/model-selection.js`
- `src/agents/pi-embedded-runner/model.js`
- `src/auto-reply/types.js` — ReplyPayload
- `src/channels/plugins/index.js`, `src/channels/plugins/types.js`
- `src/globals.js` — verbose logging
- `src/line/markdown-to-line.js` — `stripMarkdown()`
- `src/media/audio.js` — voice compatibility check
- `src/utils.js`

### External Dependencies
- `@mariozechner/pi-ai` — LLM completion for summarization
- `node-edge-tts` — Microsoft Edge TTS

### Data Flow
1. `maybeApplyTtsToPayload()` → check auto mode → parse directives → optionally summarize → strip markdown → call provider → save audio file → attach to payload
2. Provider chain: try primary → fallback to others (openai → elevenlabs → edge)
3. Output: Opus for Telegram (voice-compatible), MP3 for others

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

### File Inventory (15 files)

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
- `src/config/types.base.js` — MarkdownTableMode type
- `src/auto-reply/chunk.js` — text chunking

### External Dependencies
- `markdown-it` — markdown parser
- `yaml` — YAML frontmatter parsing

### Test Coverage
7 test files: frontmatter parsing, WhatsApp conversion, IR blockquote/HR/list spacing, nested lists, table → bullets conversion, table with code.
