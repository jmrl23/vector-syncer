# Feature: Operations — triggers, tuning, recovery

> The knobs and procedures for whoever runs the stack. Full runbook: [`08-configuration-and-deployment.md`](../design/08-configuration-and-deployment.md) §5; queue internals: [`07-bullmq-design.md`](../design/07-bullmq-design.md).

## Trigger a sync manually

Don't wait for the cron tick:

```bash
docker compose run --rm worker node dist/cli.js sync     # dev: yarn sync:once
```

Always safe — the sync queue runs at concurrency 1, so a manual trigger during a running walk simply queues right behind it; runs never overlap. Every run is labeled in logs and Bull Board (`cron` / `manual` / `resync`).

## Adjust throughput live

The files worker's concurrency is a **runtime setting**, no restart:

```bash
… node dist/cli.js concurrency 8      # applies immediately; /health shows the effective value
```

Baseline comes from `FILE_CONCURRENCY` (default 4) at startup; live overrides last until the next restart (persist by editing `.env`). The `sync` worker is deliberately not adjustable — its concurrency 1 is a correctness invariant, not a knob.

## Pause / resume

```bash
… node dist/cli.js pause      # before bulk-reorganizing the OneDrive folder
… node dist/cli.js resume     # one big delta then handles everything at once
```

## Resync & rebuild

| Command | When |
|---|---|
| `… cli.js resync` | full re-enumeration + reconcile (orphans, stale paths) — cheap, hash-skip avoids re-embedding |
| `… cli.js resync --rebuild [--collection od_hr]` | after an embedding-model or chunking change, or to redo conversions after an OCR change — builds `_v2`, flips the alias, zero downtime |
| `… cli.js resync --catalog` | refresh catalog stats + regenerate descriptions on demand |

## Re-authentication (the one recurring duty)

When `/health` shows `auth_required` (password reset, revoked session, policy change):

```bash
docker compose run --rm worker node dist/cli-auth.js    # dev: yarn auth
```

Sign in with the device code — ~2 minutes, all state intact, queued work resumes immediately.

## Monitoring

- **`/health`** (`http://localhost:3001/health`) — the one endpoint to wire to an uptime monitor (uptime-kuma, healthchecks.io) **on day one**, because the worst failure mode (dead refresh token) is otherwise silent:

```json
{ "status": "ok", "auth": "valid", "lastSyncAt": "…", "lastSyncChanges": 3,
  "queues": { "files": { "waiting": 0, "failed": 2 } },
  "files": { "indexed": 1240, "empty_conversion": 2, "skipped_unsupported": 30, "ocr_images_processed": 57 } }
```

  Alert on: `auth` ≠ `valid` · `lastSyncAt` older than ~3× the sync interval · `failed` climbing · `ocr_images_processed` jumping (OCR volume — cost signal) · `empty_conversion` climbing (unreadable documents).

- **Bull Board** (`http://localhost:3001`) — inspect queues, read failure stack traces, retry or discard failed jobs manually.

## Backups (optional by design)

Qdrant snapshots and Redis AOF are conveniences: OneDrive remains the source of truth, so the worst-case recovery from *any* data loss is one full resync (embedding cost applies only to what Redis/Qdrant state can't prove unchanged).
