# 07 — BullMQ design

> Status: draft · Last updated: 2026-07-26

## 1. Topology

| Queue | Jobs | Concurrency | Why separate |
|---|---|---|---|
| `sync` | `delta-sync` (from scheduler), `reconcile` (weekly), `resync` (manual) | **1** | delta walks must never overlap; serializes token handling |
| `files` | `process-file`, `delete-file`, `delete-folder`, `update-metadata`, `refresh-catalog` | N (`FILE_CONCURRENCY`, default 4, **live-adjustable**) | fan-out work; independently rate-limited and retried |

One Redis, one BullMQ prefix (`bull:vs:`), all workers in the single `worker` process for v1 (BullMQ makes multi-process scale-out a deploy change, not a code change).

### Adjustable concurrency (verified BullMQ capability)

`worker.concurrency` is a public setter — changeable **at runtime, no restart** (BullMQ validates a finite number ≥ 1; the new value takes effect as running jobs complete). Wired two ways:

- **Baseline** from env: `FILE_CONCURRENCY` (default 4), applied at startup.
- **Live override**: `cli.js concurrency <n>` writes `vs:config:fileConcurrency` to Redis; the worker picks it up, applies `filesWorker.concurrency = n` immediately, and logs the change. `/health` reports the current effective value. Overrides last until the next restart (persist by editing `.env`).

The `sync` worker is deliberately **not** adjustable — concurrency 1 there is a correctness invariant (delta walks must never overlap), not a tuning knob.

## 2. Scheduling (BullMQ Job Schedulers)

```ts
import { Queue } from 'bullmq';
const syncQueue = new Queue('sync', { connection });

await syncQueue.upsertJobScheduler(
  'onedrive-delta-sync',
  { pattern: env.SYNC_CRON },              // default '*/5 * * * *'; or { every: 300_000 }
  {
    name: 'delta-sync',
    data: { reason: 'cron' },
    opts: { removeOnComplete: 50, removeOnFail: 200 },
  },
);

await syncQueue.upsertJobScheduler(
  'weekly-reconcile',
  { pattern: '0 4 * * 0' },                // Sunday 04:00
  { name: 'reconcile', data: { reason: 'cron' } },
);
```

`upsertJobScheduler` is idempotent by scheduler ID — safe to run on every startup; changing `SYNC_CRON` just updates the scheduler. Note the documented semantics: the next job is produced when the previous one *starts*, so a slow walk could overlap the next tick — that's why `sync` runs at concurrency 1 (overlapping job simply waits).

**Manual triggering** (user requirement): the same `sync` queue accepts ad-hoc jobs at any time, alongside the scheduler —

```ts
await syncQueue.add('delta-sync', { reason: 'manual' });   // cli.js sync   (dev: yarn sync:once)
await syncQueue.add('resync',     { reason: 'resync' });   // cli.js resync [--rebuild|--catalog]
```

Concurrency 1 makes manual triggers always safe: if a walk is already running, the manual job simply queues right behind it — no overlap, no special casing. `reason` lands in logs and Bull Board so every run is attributable (cron vs manual vs resync).

## 3. Job payloads (typed contracts)

```ts
type SyncJob        = { reason: 'cron' | 'manual' | 'resync' | 'webhook' };
type ProcessFileJob = { driveId: string; itemId: string; cTag: string;
                        path: string; name: string; size: number; mimeType?: string };
type DeleteFileJob  = { driveId: string; itemId: string };
type DeleteFolderJob= { driveId: string; folderId: string };
type UpdateMetaJob  = { driveId: string; itemId: string };   // re-fetches metadata itself
type CatalogJob     = { baseFolderId: string; regenerateDescription: boolean };
```

Payloads carry **identifiers, not content** — jobs re-fetch fresh state, which is what makes stale jobs harmless. `process-file` resolves the file's base folder at run time; if Redis shows the file previously lived in a *different* collection, the job purges the old collection's points after indexing into the new one (cross-base-folder move, FR-16).

## 4. Idempotency & deduplication

```ts
await filesQueue.add('process-file', payload, {
  deduplication: { id: `${payload.itemId}:${payload.cTag}` },  // same content version queued once
  attempts: 4,
  backoff: { type: 'exponential', delay: 10_000 },             // 10s → 20s → 40s
  removeOnComplete: { age: 24 * 3600, count: 1000 },
  removeOnFail: { age: 7 * 24 * 3600 },
});
```

- Re-walking the same delta window (crash before token save) produces identical dedup IDs ⇒ silently dropped duplicates.
- A *newer* version of the file has a new `cTag` ⇒ new dedup ID ⇒ queues fine behind the old one; the old job detects `cTag` mismatch on metadata re-fetch and completes as `superseded`.
- `delete-file` dedups on `del:${itemId}`.

## 5. Rate limiting (verified BullMQ pattern)

Static ceiling plus dynamic reaction to 429s from Graph or DeepInfra:

```ts
import { Worker, UnrecoverableError } from 'bullmq';

const filesWorker = new Worker('files', processor, {
  connection,
  concurrency: env.FILE_CONCURRENCY,        // baseline 4 — live-adjustable, §1
  limiter: { max: 8, duration: 1000 },      // ≤ 8 job starts/sec across all workers
});

// inside the processor, on any upstream 429:
const retryMs = (Number(res.headers.get('retry-after')) || 10) * 1000;
await filesWorker.rateLimit(retryMs);       // pauses the whole queue for retryMs
throw Worker.RateLimitError();              // special error: job → back to waiting, attempt NOT consumed
```

Permanent failures (unsupported format already classified, file gone) throw `UnrecoverableError` → straight to failed, no pointless retries.

## 6. Retry matrix

| Failure | attempts | backoff | terminal behavior |
|---|---|---|---|
| Graph / DeepInfra 429 | not counted | `rateLimit()` + `RateLimitError` | — |
| Graph/DeepInfra 5xx, network, converter 5xx/timeout | 4 | exp 10 s | failed set (Bull Board manual retry) |
| Qdrant unavailable | 5 | exp 5 s | failed set |
| Converter 415/422 (bad file) | 1 | — | job *completes* with `skipped_*` status (not an error) |
| Auth `interaction_required` | 1 | — | `UnrecoverableError('AUTH_REQUIRED')` + health flips red |
| Anything in `sync` job | 3 | exp 30 s | next cron tick retries anyway; alert after N consecutive failures |

## 7. Observability

- **Bull Board** (`@bull-board/fastify`) on `:3001` — inspect queues, payloads, stack traces; retry/discard failed jobs manually.
- **QueueEvents** listener → pino logs: job completed/failed with duration, file path, status; counters aggregated into `/health`:

```json
{ "status": "ok", "auth": "valid", "lastSyncAt": "...", "lastSyncChanges": 3,
  "queues": { "files": { "waiting": 0, "failed": 2 } },
  "files": { "indexed": 1240, "empty_conversion": 2, "skipped_unsupported": 30 } }
```

- Failed-set size > threshold or sync stale > 3× interval ⇒ log at `error` (hookable to any alerting later).

## 8. Lifecycle & Redis requirements

- **Graceful shutdown:** `SIGTERM` → `await Promise.all([syncWorker.close(), filesWorker.close()])` — in-flight jobs finish or move back to waiting; scheduler state lives in Redis and survives.
- **Redis config (hard requirements):** `maxmemory-policy noeviction` (BullMQ corrupts under eviction), `appendonly yes` for durability of queue + deltaLink state. Valkey/Redis ≥ 6.2 compatible; compose pins `redis:7-alpine`.
- **Startup order:** worker retries Redis connection (BullMQ auto-reconnects); compose `depends_on` + healthchecks make the common case clean.
