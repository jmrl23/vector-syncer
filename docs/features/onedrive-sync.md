# Feature: OneDrive folder sync

> What the core feature does, from the owner's point of view. Design internals: [`01-requirements.md`](../../01-requirements.md), [`02-architecture.md`](../../02-architecture.md), [`04-onedrive-graph-integration.md`](../../04-onedrive-graph-integration.md).

## What it does

vector-syncer continuously mirrors **one OneDrive folder tree** (yours, via delegated access) into Qdrant vector collections. Every supported file becomes searchable chunks; every change in OneDrive — add, edit, rename, move, delete — is reflected automatically. OneDrive stays the single source of truth; Qdrant is a derived, rebuildable index.

## What you do (once)

1. Register the Entra app (single tenant, public client flows on, delegated `Files.Read` + `offline_access` + `User.Read`).
2. Fill `.env` (`AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `ONEDRIVE_FOLDER_PATH`, `MSAL_CACHE_KEY`, provider key).
3. `docker compose up -d`, then `yarn auth` — sign in once with the printed device code.

From then on the worker refreshes tokens silently. You interact again only if auth dies (`/health` shows `auth_required` → run `yarn auth` again, ~2 minutes, nothing lost).

## What happens continuously

- Every 5 minutes (`SYNC_CRON`) a **delta query** asks Graph "what changed?" — only changed items are processed; quiet polls are nearly free. A sync can also be [triggered manually](operations.md) at any time.
- **New/edited file** → downloaded, converted to Markdown (OCR inline for scans/images), chunked, embedded (dense) + sparse-encoded, indexed. An edit **replaces all** of the file's previous chunks — no stale content survives.
- **Rename/move within the same base folder** → metadata patched on existing points, no re-embedding.
- **Move across base folders** → re-indexed into the new folder's collection, purged from the old one.
- **Delete** → file: its points vanish · folder: the whole subtree's points vanish · **base folder: its entire collection is dropped** (along with its catalog entry). Deletion in OneDrive always means deletion in Qdrant.
- A **weekly reconcile** re-checks everything end-to-end as the correctness backstop.

## Where content lands

Each **top-level subfolder** ("base folder") of the synced root gets its own collection, named `od_{slug}`:

```
OneDrive /KnowledgeBase          Qdrant
├── HR/            ─────────►    od_hr
├── Engineering/   ─────────►    od_engineering
├── Sales Docs/    ─────────►    od_sales_docs
└── loose-file.pdf ─────────►    od__root          (catch-all for files directly in the root)
                                 od_catalog        (self-describing registry — see collection-discovery.md)
```

Collections are created automatically the first time a file routes to a new base folder. Renaming a base folder in OneDrive does **not** rename its collection (mapping is by folder ID); the catalog always shows the current display name.

## What is not synced

- Extensions outside the allowlist (default: pdf, docx, pptx, xlsx, xls, csv, md, txt, html, htm, json, xml, epub, msg, png, jpg, jpeg) → recorded `skipped_unsupported`.
- Files over `MAX_FILE_SIZE_MB` (default 100) → `skipped_too_large`.
- Files that convert to almost nothing (e.g. a scan that even OCR couldn't read) → skipped with a warning and counted (`empty_conversion` on `/health`) — never silently indexed as empty.

## Timing expectations

A change is searchable within roughly one polling interval plus processing time (seconds per typical office document). On a machine that sleeps (WSL2 laptop), syncs are *delayed, never lost* — the next poll catches everything up.
