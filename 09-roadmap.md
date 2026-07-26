# 09 — Roadmap

> Status: draft · Last updated: 2026-07-26 · Estimates assume focused solo work.

```
P0 bootstrap → P1 auth+graph spike → P2 queues → P3 conversion → P4 indexing = MVP
                                                                → P5 hardening → P6 extensions
```

## Phase 0 — Repo & infrastructure bootstrap (~½ day)

- `git init`; scaffold `worker/` (**yarn**, TS strict, tsx, vitest, eslint/biome, pino) and `converter/` (uv, FastAPI skeleton, pytest).
- `docker-compose.yml` with redis (+noeviction/AOF) and qdrant; `.env.example`; zod config loader.
- **Accept:** `docker compose up -d` healthy; `yarn dev` boots and validates env; CI-less but `yarn test` green.

## Phase 1 — Delegated auth + Graph read (spike-heavy) (~1–2 days)

- Encrypted `ICachePlugin`; `yarn auth` device-code CLI; `acquireTokenSilent` helper with `AUTH_REQUIRED` handling.
- Folder resolution (path → driveId/folderId); delta walker (paging, `$select`, deltaLink persistence in Redis, `410` handling).
- `yarn dev:delta` — prints classified changes (incl. resolved base folder per file) as a dry run.
- **Spike goals (encode findings as tests + doc updates):** folder-scoped delta behavior on the actual drive type; which `file.hashes` exist; what a move-out-of-scope and folder-rename emit ([04](04-onedrive-graph-integration.md) §4).
- **Accept:** restart-safe incremental change listing; token refresh survives worker restarts; revoking consent produces the friendly error.

## Phase 2 — Queues & scheduling (~1 day)

- Queues, Job Schedulers (`delta-sync`, `weekly-reconcile`), typed job payloads, dedup wiring, Bull Board + `/health`; manual sync trigger (`cli.js sync`) and live files-concurrency adjustment (`cli.js concurrency <n>`).
- `sync` job enqueues `process-file`/`delete-*` jobs; `files` processor is a stub that logs + updates Redis state.
- **Accept:** cron fires; duplicate suppression proven (run sync twice → zero duplicate file jobs); Bull Board shows the flow.

## Phase 3 — Download + conversion (~1–2 days)

- Streaming download with metadata re-check, size caps, allowlist, scratch cleanup.
- Converter service: `/convert` + `/healthz`, golden-file pytest suite; worker client with timeout/abort; OCR path (markitdown-ocr + Qwen3-VL via DeepInfra) exercised against a scanned-PDF fixture; empty-conversion warning counter.
- **Accept:** drop docx/pdf/xlsx/pptx fixtures into OneDrive → worker logs clean Markdown for each; scanned-PDF fixture → readable Markdown via OCR (and with OCR off, the empty-conversion warning fires instead of indexing garbage).

## Phase 4 — Chunk, embed, index, catalog → **MVP** (~2–3 days)

- Chunker (heading-aware, token-sized, breadcrumbs) with unit tests on fixture markdown.
- Embeddings client: OpenAI SDK → DeepInfra `BAAI/bge-m3` (batching, 429 → rate-limit path).
- Sparse encoder `bm25-v1` ([06](06-qdrant-design.md) §4): unicode tokenize → FNV-1a hash → BM25 TF weights; exported as an importable module (the consumer app reuses it verbatim for query encoding); unit tests pin the encoding with golden vectors.
- Shared consumer package: exports the `bm25-v1` encoder **and the conversation-store helpers** (`ensureConversationCollection`, `appendTurn`, `getConversation(conversationId)`, `searchTurns`, `deleteConversation`) — app-facing only, the worker never calls them ([06](06-qdrant-design.md) §10).
- Qdrant multi-collection layer: base-folder discovery, slug naming + Redis mapping, on-demand creation (dense + sparse slots, payload indexes, alias), writer (delete-then-upsert **writing both vectors**, `setPayload` metadata path), delete + cross-base-folder re-route paths.
- Catalog: `od_catalog` init, `refresh-catalog` job, DeepSeek-V4-Flash description generation.
- **Accept = MVP criteria from [01-requirements.md](01-requirements.md) §7:** add/edit/delete round-trips verified end-to-end; kill-mid-sync test shows no dupes/loss; a file under `HR/` lands in `od_hr` with a catalog entry; moving it to another base folder relocates its points; a hybrid (RRF) query returns the freshly-added document, and an exact-token query (e.g. an invoice number planted in a fixture) hits via the sparse side; a catalog scroll lists every collection with name + description + stats ("what collections are available?" is answerable from it alone).

## Phase 5 — Hardening & operations (~2–3 days)

- Manual rate-limit path exercised (fault injection); retry matrix; `UnrecoverableError` classification; superseded-job check.
- Reconcile job (paths, orphans); `resync` CLI (`--full`, `--rebuild`); pause/resume.
- `/health` counters; failure alerts in logs; runbook validated by doing each procedure once; README quickstart rewritten for real usage.
- **Accept:** chaos afternoon — kill converter mid-job, revoke token, feed a corrupt PDF, force a 429 — system degrades per the failure-mode table and recovers unattended where designed to.

## Phase 6 — Extensions (unordered backlog, post-MVP)

| Item | Trigger to build it |
|---|---|
| Azure DocIntel fallback tier; OCR budget cap + targeted re-process status | Qwen3-VL quality insufficient on complex layouts, or the empty-conversion counter / first invoice disproves the "minimal images" assumption |
| Graph webhooks → instant sync trigger | 5-min latency becomes annoying; public HTTPS endpoint available |
| Learned sparse (true bge-m3 lexical weights) in a second named slot, A/B'd against `bm25-v1` | a serving path exposes the lexical weights (self-hosted FlagEmbedding, or provider support) |
| Search API / **MCP server** (catalog-first: `list_collections` + `search`) | consumers shouldn't talk to Qdrant raw; also solves cross-collection fan-out cleanly |
| Multi-folder / multi-account | second use case appears (design already drive-qualified) |
| Conversion cache (hash → markdown) | OCR spend makes re-conversion costly |
| Metrics endpoint (Prometheus) + Grafana | someone actually operates this long-term |
