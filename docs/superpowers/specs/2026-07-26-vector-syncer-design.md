# vector-syncer — Design Specification

> Date: 2026-07-26 · Status: **validated through brainstorming review with the owner** (all open questions answered, all sections walked through and approved)
>
> This spec distills the detailed planning documents in `docs/design/` (`01-requirements.md` … `10-decisions-and-risks.md`); those remain the deep reference for each area. Where this spec and a detail doc disagree, this spec (newer) wins and the doc should be fixed.

## 1. Purpose

A self-hosted background worker that mirrors one Microsoft OneDrive folder tree into Qdrant vector collections, continuously and unattended, so the documents become searchable by the owner's own app. It watches the folder with the signed-in user's *delegated* permissions, converts every supported file to Markdown, and keeps Qdrant exactly in sync with the folder's current state — additions, edits, renames, moves, and deletions.

**Goals:** OneDrive is the single source of truth; changes are searchable within minutes; deletions cascade; zero interaction after a one-time sign-in; unchanged content is never re-processed; the tree is scanned recursively with each top-level subfolder ("base folder") isolated in its own collection, discoverable through a self-describing catalog.

**Out of scope (v1):** write-back to OneDrive; a search API or MCP server of its own (the consumer app queries Qdrant directly); multi-account/multi-folder; webhooks; any UI beyond the Bull Board queue dashboard.

## 2. Confirmed decisions (owner-validated 2026-07-26)

| Topic | Decision |
|---|---|
| Runtime | Hybrid: **Node.js/TypeScript orchestrator** (BullMQ is Node-first) + **Python FastAPI conversion sidecar** (MarkItDown is Python-only) (D1/D3) |
| Auth | **Delegated** device-code bootstrap once → encrypted persistent MSAL cache → silent refresh; `AUTH_REQUIRED` health state on failure (D2). OneDrive is **work/school**, folder **owned** by the signing-in user → `Files.Read offline_access User.Read`, `/me/drive` |
| Change detection | Graph **delta queries** scoped to the folder, cron-polled (default 5 min) + **manual trigger any time**; webhooks deferred (D9) |
| Queues | **BullMQ on Redis**: `sync` queue (concurrency 1, invariant) + `files` queue (concurrency 4 baseline, **live-adjustable at runtime**); at-least-once + dedup + idempotent writes (D6) |
| Conversion | **MarkItDown** sidecar, one endpoint `POST /convert`; OCR **on by default, inline, no budget machinery** — corpus is mostly digital PDFs with minimal images; near-empty conversions warn + count only (D8, simplified per owner) |
| AI access | **One provider (DeepInfra initially) through the official OpenAI SDK only**, configured via the SDK's native `OPENAI_BASE_URL`/`OPENAI_API_KEY`; every model ID is env config; nothing provider-specific in code (D11). Model IDs are initial preferences — only the embedding model is costly to change (re-index) |
| Embeddings | `BAAI/bge-m3`, 1024-d dense (D4) — finalize before the first big backfill |
| Search mode | **Hybrid from day one** (D14): every point carries `dense` + `sparse` (local BM25-style encoder `bm25-v1`, IDF server-side via Qdrant `modifier: idf`); queries = `prefetch` dense+sparse → **RRF fusion** |
| Collection layout | **One collection per base folder** (`od_{slug}` alias → versioned physical), `od__root` catch-all, mapping keyed by folder **ID** (rename-safe) (D12) |
| Catalog | **`od_catalog`** registry, one embedded point per collection with LLM-generated description (rich input: names, paths, content excerpts); serves **two first-class operations — enumerate** ("what collections exist?", full scroll) **and route** ("which collection covers X?", semantic search) (D13) |
| Consistency | Delete-then-upsert per file update (D7); deterministic UUIDv5 point IDs; weekly reconcile backstop; state in Redis, all rebuildable (D5) |
| Consumer | The **owner's own app/service (Node/TypeScript)** queries Qdrant directly — no MCP in v1. Contract: same embedding model; the app **imports the `bm25-v1` encoder module directly** (workspace/published package); catalog-first routing; client-side fan-out for global search |
| Conversation store | **Consumer-owned** collection (default `app_conversations`, outside `od_*` — sync/reconcile/rebuild can never touch it): one hybrid point per turn; **a whole conversation is retrieved by `conversationId` alone** (indexed filter + `order_by: turn_index` scroll); helpers ship in the shared consumer package (P4) — `docs/design/06-qdrant-design.md` §10 |
| Deployment | **Docker compose on the owner's WSL2 machine now, possibly a VPS later**; migration = copy `.env` + volumes. Sleeping machine delays (never loses) syncs |
| Package mgmt | **yarn** (worker), **uv** (converter) |

## 3. Architecture

Four services in one compose stack + two external APIs (Microsoft Graph; the OpenAI-SDK-compatible AI provider):

```
┌────────────────────── docker compose ──────────────────────┐
│  worker (Node/TS)          converter (Python)              │
│  ├ BullMQ scheduler+workers   └ FastAPI /convert           │
│  ├ MSAL auth (encrypted cache)   (MarkItDown + OCR)        │
│  ├ chunker + bm25-v1 encoder                               │
│  ├ Bull Board + /health :3001                              │
│  Redis 7 (queues + sync state, AOF, noeviction)            │
│  Qdrant (od_* collections + od_catalog, loopback :6333)    │
└────────────────────────────────────────────────────────────┘
   ↑ Microsoft Graph (delta, download)   ↑ OpenAI-SDK provider
     — delegated tokens                    (embeddings, OCR, descriptions)
```

**Data flow per change:** delta walk classifies each changed item → enqueues typed jobs → `process-file`: re-check metadata → stream download to scratch → `POST /convert` (OCR inline) → delete scratch (`finally`) → heading-aware chunking (512/64 tokens, breadcrumb prefixes) → dense embed (batched, via provider) + sparse encode (local) → delete-then-upsert into the base folder's collection → update Redis state. Catalog entries refresh on collection creation, weekly reconcile (>20 % churn regenerates the description), and on demand.

**Change classification:** new/changed content → `process-file` · rename/move within base folder → `update-metadata` (payload patch only) · move across base folders → re-route (index new, purge old) · file delete → `delete-file` · folder delete → `delete-folder` (one `ancestor_ids` filter) · base-folder delete → drop collection + catalog entry.

## 4. Key mechanisms

- **Idempotency:** point IDs = UUIDv5(`driveId/itemId#chunkIndex`); job dedup IDs = `itemId:cTag`; delta token advances only after enqueue. Any crash/retry re-produces identical writes.
- **Rate limits:** Graph/provider 429 → `worker.rateLimit(retryAfter)` + `Worker.RateLimitError` (attempt not consumed); 5xx → exponential backoff (4 attempts) → visible failed set. Poison files never block the pipeline.
- **Auth failure:** silent-refresh failure flips `/health` to `AUTH_REQUIRED`; re-auth is a 2-minute device-code login; queued work resumes untouched.
- **Empty conversions & OCR volume:** a PDF/image yielding almost no Markdown is skipped with a warning + `empty_conversion` counter; the converter reports `ocr_images` per file, aggregated into an `ocr_images_processed` counter on `/health`, so OCR volume is visible before the invoice. Warning lights, not workflows.
- **Live ops:** manual sync trigger (`cli.js sync`, serializes behind any running walk); live files-worker concurrency (`cli.js concurrency <n>`, BullMQ public setter, no restart); pause/resume; `resync [--rebuild [--collection X]] [--catalog]`.
- **Rebuilds:** aliases over versioned physical collections make model/chunking changes a build-into-`_v2`-then-flip operation, zero downtime.
- **Sparse encoder contract:** `bm25-v1` (unicode tokenize → lowercase → FNV-1a u32 → BM25 TF saturation k1=1.2 b=0.75 avgLen=256; IDF server-side) ships as an importable module; the consumer app must use it for query encoding; `sparse_encoder` recorded per point catches drift; encoder bump = sparse-only backfill.

## 5. Payload (what a point carries)

Identity (file id, name, path, web URL) · typing (extension, mime) · attribution (created/modified by + at, timestamps) · structure (base folder id/name, ancestor IDs, chunk index/count, heading path) · content (`text`) · audit (`content_hash`, `ctag`, `embed_model`, `sparse_encoder`, `synced_at`). Indexed: `file_id`, `extension`, `mime_type`, `ancestor_ids`, `last_modified_ts`. The consumer renders results from payload alone.

## 6. Error handling summary

Every failure mode maps to one of: **retry with backoff** (transient network/5xx), **rate-limit pause** (429), **complete-with-status** (unsupported/corrupt/oversized/empty → `skipped_*`/warning counters), **park in failed set** (poison, visible in Bull Board, weekly reconcile + manual retry), or **halt loudly** (`AUTH_REQUIRED`). Redis/Qdrant outages queue work and drain on recovery. Full table: `docs/design/02-architecture.md` §6.

## 7. Testing strategy

- **Unit (vitest):** chunker, `bm25-v1` encoder (golden vectors), point-ID derivation, slug naming, change classifier, config validation.
- **Golden files (pytest):** fixture docx/pdf/xlsx/pptx/scanned-pdf → expected Markdown traits through the real converter.
- **Integration (testcontainers):** Redis + Qdrant round-trips — write paths, delete cascades, re-route, catalog refresh, hybrid query.
- **Spike tests (Phase 1):** delta edge cases against the real tenant — move-out-of-scope and folder-rename behavior are *assumptions until verified*.
- **Chaos afternoon (Phase 5):** kill converter mid-job, revoke token, corrupt PDF, forced 429 — system must degrade per the failure-mode table.

## 8. MVP acceptance criteria

1. New file → chunked, hybrid-indexed, searchable within one polling cycle with correct payload.
2. Edit → old points fully replaced, no stale chunks.
3. Delete → points gone; folder delete cascades; base-folder delete drops the collection + catalog entry.
4. Kill worker mid-sync, restart → no duplicates, no losses.
5. Revoke consent → clear `AUTH_REQUIRED` state, not a crash loop.
6. `{root}/HR/…` file lands in `od_hr`; moving it to `{root}/Engineering/…` relocates points with nothing left behind.
7. Exact-token query (planted invoice number) hits via sparse; paraphrase hits via dense; catalog scroll answers "what collections are available?" completely.

## 9. Implementation phases

| Phase | Scope | ~Effort |
|---|---|---|
| P0 | Repo scaffold: yarn worker (TS strict), uv converter, compose (Redis AOF/noeviction, Qdrant), zod config, `.env.example` | ½ d |
| P1 | Auth (device code + encrypted cache) + Graph delta walker + **edge-case spike** | 1–2 d |
| P2 | Queues, schedulers, dedup, Bull Board, `/health`, manual trigger, live concurrency | 1 d |
| P3 | Download + converter service + OCR path + empty-conversion warning | 1–2 d |
| P4 | Chunker, embeddings client, `bm25-v1` encoder, multi-collection layer, catalog, shared consumer package (encoder + conversation-store helpers) → **MVP** | 2–3 d |
| P5 | Hardening: rate-limit fault injection, reconcile, `resync` CLI, chaos afternoon, runbook validation | 2–3 d |
| P6 | Backlog: DocIntel fallback + OCR budget add-backs, webhooks, learned sparse, MCP server, multi-folder, conversion cache, Prometheus | — |

## 10. Top risks (register: `docs/design/10-decisions-and-risks.md` §3)

1. **Refresh-token death is silent** at the source — mitigated by first-class `AUTH_REQUIRED` health + runbook; wire `/health` to an uptime monitor day one.
2. **Delta edge cases are assumptions** until the Phase-1 spike; weekly reconcile backstops correctness regardless.
3. **Embedding model choice is the one expensive commitment** (change = re-index); everything else is env-swappable.
4. **Image-light corpus assumption** removed the OCR budget machinery — `ocr_images_processed` + `empty_conversion` counters and the first invoice are the review triggers; add-backs documented.
5. **Sparse-encoder drift** between indexer and app would silently degrade hybrid recall — app confirmed Node/TS, so it imports the exact versioned module (no reimplementation); golden-vector tests + per-point `sparse_encoder` field.
6. **Provider dependency** (outage/model retirement) — queues drain on recovery; models pinned in env; self-host switch-back documented.
