# 01 — Requirements

> Status: draft · Last updated: 2026-07-26

## 1. Problem statement

Documents that a team (or individual) maintains in a OneDrive folder should be continuously available for semantic search / RAG. Today that requires manual export–convert–embed steps. **vector-syncer** automates it: it watches one OneDrive folder using the signed-in user's own permissions (delegated access), converts every supported file to Markdown, and keeps a Qdrant collection in sync with the folder's current state — including updates and deletions.

## 2. Goals

1. A OneDrive folder is the single source of truth; Qdrant is a derived, rebuildable index.
2. New/changed files appear in Qdrant within minutes (one polling interval + processing time).
3. Deleted or moved-away files disappear from Qdrant.
4. Zero interactive involvement after a one-time sign-in — until the refresh token is revoked.
5. Cheap to run: unchanged content is never re-converted or re-embedded.
6. The folder tree is scanned **recursively**, and each top-level subfolder ("base folder") is isolated in its **own Qdrant collection** — discoverable through a self-describing catalog collection.

## 3. Non-goals (v1)

- No write-back to OneDrive (read-only sync).
- No search/query API of its own — consumers query Qdrant directly (a search endpoint / MCP server is a candidate for a later phase, see [09-roadmap.md](09-roadmap.md)).
- The worker never ingests user conversations. Conversation storage is **consumer-owned**: the app writes its own collection outside `od_*` (default `app_conversations`), and a whole conversation is retrievable by its `conversationId` alone — pattern and helpers specced in [06-qdrant-design.md](06-qdrant-design.md) §10.
- No multi-user / multi-drive fan-out (single account, single folder; the design keeps IDs drive-qualified so this can be added later).
- No real-time webhooks (polling first; webhook upgrade path documented in [04-onedrive-graph-integration.md](04-onedrive-graph-integration.md)).
- No UI beyond the Bull Board queue dashboard.

## 4. Functional requirements

### Sync & change detection

| ID | Requirement |
|---|---|
| FR-1 | Poll a configured OneDrive folder **and its entire subtree (recursive, any nesting depth)** on a cron schedule (default every 5 min) using Microsoft Graph **delta queries** scoped to that folder. A sync is also **manually triggerable at any time** (CLI); manual and scheduled runs serialize — they never overlap. |
| FR-2 | Detect and handle: file created, file content updated, file renamed/moved (within scope), file deleted, folder deleted (cascade to descendants), file moved out of scope. |
| FR-3 | Persist the `deltaLink` token between runs; on `410 Gone` perform the documented resync (fresh enumeration from the `Location` header link). |
| FR-4 | Skip files whose content is unchanged, using `cTag` and/or `file.hashes` (quickXorHash / sha256) comparison against stored per-file state. |
| FR-5 | Support a manual **full resync** command that re-enumerates the folder and reconciles Qdrant (removing orphans, fixing stale paths). |

### Acquisition & conversion

| ID | Requirement |
|---|---|
| FR-6 | Download changed files via Graph (`/content` or `@microsoft.graph.downloadUrl`), streamed to a scratch location, bounded by a configurable max size (default 100 MB). **Downloaded files are deleted from scratch immediately after processing** (success or failure) — content persists nowhere outside Qdrant payloads. |
| FR-7 | Process only an allowlist of extensions (default: pdf, docx, pptx, xlsx, xls, csv, md, txt, html, htm, json, xml, epub, msg, png, jpg, jpeg); everything else is recorded as `skipped_unsupported`. |
| FR-8 | Convert files to Markdown with **MarkItDown**; OCR is a configurable tier (off / LLM-vision plugin / Azure Document Intelligence — see [05-conversion-pipeline.md](05-conversion-pipeline.md)). |
| FR-9 | Detect near-empty conversion output (a PDF/image yielding almost no Markdown) → skip indexing, log a warning, count it on `/health` (`empty_conversion`). A warning light only — no re-processing workflow in v1; bulk redo is `resync --rebuild`. |

### Indexing

| ID | Requirement |
|---|---|
| FR-10 | Chunk Markdown heading-aware with configurable token size/overlap (default 512/64) and a heading-breadcrumb prefix per chunk. |
| FR-11 | Embed chunks with **`BAAI/bge-m3`** (1024-d dense) served by **DeepInfra** through the OpenAI SDK, behind a provider abstraction that keeps model and endpoint config values. Additionally compute a **local BM25-style sparse vector** per chunk (versioned worker-side encoder, no API call) so every point supports **hybrid (dense + sparse, RRF) search** from day one. |
| FR-12 | Route each file to the Qdrant collection derived from its **base folder** (the top-level subfolder it lives under); create collections on demand when a new base folder appears; files sitting directly in the sync root go to a configurable catch-all collection. |
| FR-13 | Upsert chunks with **deterministic point IDs** (UUIDv5 of drive+item+chunk index) so retries overwrite instead of duplicate. **A detected file update replaces *all* of the file's previous points** with freshly converted, chunked and embedded content — no stale vectors survive. |
| FR-14 | Store rich payload per point for filtering and attribution: file id, name, path, extension, mime type, size, created/modified timestamps, author (`createdBy`/`lastModifiedBy`), base folder, chunk index/count, heading path, raw chunk text, source web URL, content hash, embed model. |
| FR-15 | **A deletion in OneDrive deletes every reference in Qdrant**: file deleted → all its points removed; folder deleted → all descendant points removed (via ancestor IDs); base folder deleted → its entire collection dropped. |
| FR-16 | Handle files moved *between* base folders: index into the new base folder's collection, purge the points left in the old one. |
| FR-21 | Maintain a **catalog collection** (default `od_catalog`): one embedded point per base-folder collection carrying its name, an LLM-generated description (`deepseek-ai/DeepSeek-V4-Flash`), and stats (file/chunk counts, types) — created with the collection, refreshed on reconcile, removed when the collection is dropped. Serves two consumer operations: **enumeration** ("what collections exist?" — full catalog scroll, deterministic) and **semantic routing** ("which collection covers X?" — vector search over the embedded descriptions). |

### Auth & operations

| ID | Requirement |
|---|---|
| FR-17 | One-time interactive bootstrap via **device code flow** (CLI prints code + URL); thereafter tokens refresh silently from an **encrypted persistent MSAL token cache**. |
| FR-18 | When silent refresh fails with an interaction-required error, surface a loud, actionable alert (log + failed job + health endpoint state) with the re-auth runbook. |
| FR-19 | Operational visibility & live tuning: structured logs, Bull Board dashboard, `/health` endpoint (last successful sync, queue depths, auth state, effective concurrency), manual retry of failed jobs, and **runtime-adjustable files-worker concurrency** (CLI, no restart). |
| FR-20 | All configuration via environment variables (validated at startup); no secrets in code or logs. |

## 5. Non-functional requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-1 | **Idempotency** | Every job safe to run twice; at-least-once delivery end to end |
| NFR-2 | **Freshness** | Change → searchable in ≤ polling interval + 2 min (p95, typical office docs) |
| NFR-3 | **Scale (v1)** | Up to ~50 k files / ~500 k chunks on a single Qdrant node without redesign |
| NFR-4 | **Cost control** | No re-embedding of unchanged content; batched embedding calls; OCR runs inline on an image-light corpus (user 2026-07-26) with `OCR_LLM_MODEL=` (empty) as the kill switch |
| NFR-5 | **Security** | Least-privilege delegated scopes; token cache encrypted at rest (AES-256-GCM); Redis/Qdrant not exposed publicly; scratch files deleted after processing; document content is shared with exactly two external parties — Microsoft Graph (source) and DeepInfra (AI provider, by decision D11) |
| NFR-6 | **Recoverability** | Qdrant and Redis are both rebuildable from OneDrive (source of truth); losing the delta token only costs a re-enumeration, not re-embedding (hash-skip) |
| NFR-7 | **Observability** | Per-job logs with file path + duration; counters for processed/skipped/failed/empty_conversion/ocr_images_processed; last-sync timestamp gauge |
| NFR-8 | **Portability** | Everything runs via `docker compose` on any Linux host; external dependencies are exactly two: Microsoft Graph and DeepInfra |
| NFR-9 | **Rate-limit compliance** | Honor `Retry-After` on Graph 429s; global concurrency caps; single delta walk at a time |

## 6. Constraints & assumptions

- **Delegated access is a hard constraint** (user's requirement) — the design must not depend on application permissions or admin consent. Consequence: the worker holds a refresh token for one specific user and dies (loudly) if that token is revoked.
- Assumes the account can consent to `Files.Read` (or `Files.Read.All`), `offline_access`, `User.Read`. Some tenants require admin approval even for these — flagged in [10-decisions-and-risks.md](10-decisions-and-risks.md).
- Assumes a Redis instance and Qdrant instance dedicated to this stack (provisioned by the compose file).
- Embedding dimension is fixed per Qdrant collection — changing the embedding model means a re-index (procedure in [06-qdrant-design.md](06-qdrant-design.md)).
- All AI inference (embeddings, OCR, catalog descriptions) is delegated to **DeepInfra by default** via its OpenAI-compatible API — an accepted data-sharing decision (D11). **Models and endpoints are configuration, never code**: only the official OpenAI SDK is used, pointed by `OPENAI_BASE_URL`/`OPENAI_API_KEY` as the shared default, with every model ID in its own env var — any OpenAI-compatible provider (or a self-hosted bge-m3 sidecar, D4) slots in without touching code. Each role can independently override the endpoint/key (`EMBEDDING_`/`OCR_`/`LLM_` `_BASE_URL`/`_API_KEY`), so embeddings, OCR, and catalog descriptions can each be served by a different provider.
- Development environment: Linux/WSL2 with Docker.

## 7. Success criteria (MVP acceptance)

1. Drop `report.docx` into the OneDrive folder → within one polling cycle it is chunked and searchable in Qdrant with correct payload.
2. Edit the file → old chunks replaced (no stale duplicates), point count matches new chunking.
3. Delete the file → its points are gone.
4. Kill the worker mid-sync, restart → no duplicates, no lost changes.
5. Revoke consent → the worker degrades to a clear "re-authentication required" state instead of crash-looping.
6. A file under `{root}/HR/…` is indexed into the `od_hr` collection; moving it to `{root}/Engineering/…` relocates its vectors to `od_engineering` and leaves nothing behind in `od_hr`.
7. A hybrid query for an exact token planted in a fixture (e.g. an invoice number) finds the document via the sparse side; a semantic paraphrase finds it via the dense side.
8. A catalog scroll lists every collection with name, description and stats — "what collections are available?" is answerable from it alone (FR-21 enumeration).
