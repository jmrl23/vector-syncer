# 10 — Decisions, open questions, risks

> Status: draft · Last updated: 2026-07-26

## 1. Decision log (ADR-lite)

| # | Decision | Status | Alternatives & why rejected |
|---|---|---|---|
| D1 | **Hybrid runtime**: Node/TS orchestrator + Python conversion sidecar | ✅ accepted | All-Python (BullMQ py client trails Node features); all-Node (no MarkItDown equivalent) — full analysis in [03-tech-stack.md](03-tech-stack.md) §2 |
| D2 | **Delegated auth via device code** + encrypted persistent MSAL cache + silent refresh | ✅ accepted (user constraint) | Auth-code-with-PKCE bootstrap (needs a redirect listener; device code is simpler headless); app-only client credentials (violates the delegated requirement) |
| D3 | Conversion as **HTTP sidecar service** | ✅ accepted | `markitdown` CLI child-process in the worker image: fewer moving parts but cold imports per file, mixed-runtime image, no crash isolation. Documented as a fallback "minimal mode" if the stack must shrink |
| D4 | Embeddings: **`BAAI/bge-m3` (1024-d dense) via DeepInfra**, called with the official OpenAI SDK (native env config) | ✅ architecture accepted (closes Q2); **model ID = initial preference** — env-swappable, but an embedding-model swap re-indexes ⇒ finalize before the first big backfill | Self-hosted FlagEmbedding sidecar (same model, dense **and** sparse, zero data egress — the documented switch-back path); OpenAI/Azure embedding models (different ecosystem, no advantage here) |
| D5 | **Sync state in Redis** (deltaLink + per-file hashes), no SQL database | ✅ accepted | Postgres/SQLite: adds a service for state that is fully rebuildable from OneDrive + Qdrant payloads; revisit only if audit history is ever required |
| D6 | **Advance delta token after enqueue**, not after all jobs complete (at-least-once + dedup + reconcile) | ✅ accepted | FlowProducer parent-child gating (a single poison file would wedge the token forever); exactly-once illusions rejected on principle |
| D7 | Per-file update = **delete-by-filter then upsert** | ✅ accepted | Upsert + trailing delete of `chunk_index ≥ n` avoids the brief zero-point window; more code for a window nobody will notice at this scale |
| D8 | OCR via **LLM vision: `Qwen/Qwen3-VL-235B-A22B-Instruct` on DeepInfra** (markitdown-ocr plugin), **on by default, inline, no budget machinery** — corpus is mostly digital PDFs with minimal images (user 2026-07-26), so OCR volume is naturally small; near-empty conversions warn + count (`empty_conversion`) | ✅ architecture accepted (closes Q3); **model ID = initial preference**, swappable anytime with zero migration; budget cap + targeted re-process status are documented add-backs if the image-light assumption proves wrong | Azure DocIntel (kept as fallback tier for complex layouts); local ocrmypdf/Tesseract (offline fallback); budget guard + `--no-ocr` staged backfill (dropped 2026-07-26 as over-engineering for this corpus) |
| D9 | **Polling delta** (5 min cron), webhooks deferred | ✅ accepted for v1 | Webhooks need a public HTTPS endpoint and still require delta calls; latency win only |
| D10 | Qdrant **self-hosted in compose**, versioned collections + aliases | 🟡 accepted-pending-Q5 | Qdrant Cloud fine too — only `QDRANT_URL/API_KEY` change |
| D11 | **One AI provider for everything: DeepInfra via the OpenAI SDK** — `BAAI/bge-m3` (embeddings), `Qwen/Qwen3-VL-235B-A22B-Instruct` (OCR), `deepseek-ai/DeepSeek-V4-Flash` (catalog descriptions / future LLM needs), single key | ✅ accepted (user) | Best-of-breed per capability (more keys/dashboards/failure modes); full self-hosting (ops burden, GPU needs). **Flexibility clause (user)**: only the official OpenAI SDK is used, endpoint + key live in the SDK's native `OPENAI_BASE_URL`/`OPENAI_API_KEY` env vars, and every model ID is env config — DeepInfra is a *value*, not a dependency, and **the three model IDs are initial preferences, not commitments** (only the embedding model carries switching cost — a re-index). Consequence accepted: document content is processed by the configured provider; 429/outage handled like Graph throttling |
| D12 | **One Qdrant collection per base folder**, created on demand; loose root files → `_root` catch-all; mapping keyed by folder *ID* (renames don't break it) | ✅ accepted (user) | Single collection + `base_folder_id` payload filter: one HNSW (lighter), cross-folder search in one query — revisit if base folders proliferate. Consequence accepted: global search = client-side fan-out; base-folder delete = collection drop |
| D13 | **Catalog collection** (`od_catalog`): one embedded point per document collection — name, DeepSeek-generated description, stats — enabling catalog-first routing | ✅ accepted (user) | Static config file (drifts from reality); no catalog (consumers must guess). Descriptions draw on names, paths **and** brief content excerpts (user choice: rich catalog) |
| D14 | **Hybrid from day one**: every point carries `dense` (bge-m3 via provider) **and** `sparse` — a worker-computed BM25-style vector (`bm25-v1`: unicode tokenize → FNV-1a `u32` → TF saturation; IDF server-side via `modifier: idf`); queries use Query-API `prefetch` dense+sparse → **RRF fusion** ([06](06-qdrant-design.md) §4/§8). Encoder is a shared module the consumer app imports for query encoding; `SPARSE_MODE=off` = dense-only fallback | ✅ accepted (user 2026-07-26 — supersedes the earlier dense-only-v1 lean) | Dense-only v1 (rejected by user: exact-token recall matters from day one); learned sparse / true bge-m3 lexical weights (no serving path exposes them today — future second slot via `createVectorName`) |

## 2. Open questions — **all answered** (as of 2026-07-26)

| # | Question | Why it matters | Default if unanswered |
|---|---|---|---|
| ~~Q1~~ | OneDrive type — **answered 2026-07-26: work/school (OneDrive for Business)**. Tenant authority, `Files.Read`, quickXorHash available. **Owned-folder assumption confirmed by user 2026-07-26** — `/me/drive`, plain `Files.Read`, no `Files.Read.All` needed. | — | — |
| ~~Q2~~ | Embedding provider — **answered 2026-07-26**: `BAAI/bge-m3` via DeepInfra, OpenAI SDK (D4/D11) | — | — |
| ~~Q3~~ | OCR — **answered 2026-07-26**: `Qwen/Qwen3-VL-235B-A22B-Instruct` via DeepInfra, markitdown-ocr plugin (D8); DocIntel demoted to fallback | Remaining sub-question: rough share of scanned content, to predict OCR spend | on by default |
| ~~Q4~~ | Corpus size/mix — **answered 2026-07-26: count unknown; mix is mostly digital PDFs with minimal images.** Defaults stand (sized for ≤ 50 k files); Bull Board stats + the `empty_conversion` counter measure reality during the first backfill | — | — |
| ~~Q5~~ | Deployment — **answered 2026-07-26: Docker on the WSL2 machine now, possibly a VPS later.** Docker daemon must autostart with WSL; a sleeping machine *delays* (never loses) syncs; VPS migration = copy `.env` + `data/` + the two volumes ([08](08-configuration-and-deployment.md) §6) | — | — |
| ~~Q6~~ | Consumers — **answered 2026-07-26: the user's own app/service**, querying Qdrant directly. MCP server stays in the Phase 6 backlog; [06](06-qdrant-design.md) §8 is the app's integration contract: same embedding model **and** same `bm25-v1` encoder for queries, catalog-first routing, client-side fan-out for global search | — | — |

## 3. Risk register

| Risk | L | I | Mitigation |
|---|---|---|---|
| Refresh token revoked (password reset, CA policy, admin action) → sync silently dies | M | H | First-class `AUTH_REQUIRED` state: health endpoint flips, loud logs, 2-minute re-auth runbook; frequent syncs keep the token warm |
| Tenant blocks user consent for the app's scopes | M | M | Discovered in Phase 1 minute one; needs a one-time admin approval — flag early to whoever owns the tenant |
| Delta edge cases differ from assumptions (move-out-of-scope, folder rename cascades) | M | M | Phase-1 spike verifies empirically; weekly reconcile is the correctness backstop regardless |
| `410 Gone` resync on a huge corpus → burst of work | M | L | Hash-skip makes re-enumeration cheap; rate limiter caps the burst |
| OCR cost/latency surprise — the vision model is called **per embedded image** (one 200-page scan = 200 calls) and v1 ships no budget cap, trusting the image-light corpus assumption | L | M | Assumption recorded (Q4/D8); `empty_conversion` counter + first-invoice review trigger (e); kill switch `OCR_LLM_MODEL=` (empty); documented add-backs: per-cycle image cap, targeted re-process status |
| DeepInfra dependency: outage, rate limits, or model retirement (bge-m3 / Qwen3-VL / DeepSeek-V4-Flash are *their* catalog) | M | M | Rate-limit + retry paths queue work rather than lose it; model IDs pinned in env; provider abstraction + documented self-host switch-back (D4); embedding-model change = routine rebuild ([06](06-qdrant-design.md) §7) |
| Document content processed by a third party (DeepInfra) | accepted | M | Explicit decision D11; review DeepInfra's data-retention/ToS at implementation; provider swappable via `OPENAI_BASE_URL` if governance changes |
| Base-folder proliferation → collection sprawl (per-collection HNSW/segment overhead) | L | M | D12 records the single-collection fallback; reconcile reports collection count; catalog keeps fan-out manageable |
| Embedding model deprecated/changed → full reindex needed | L | M | Versioned collections + alias flips are routine, zero-downtime ([06](06-qdrant-design.md) §7), or additive via `createVectorName`; re-embedding cost at bge-m3 rates is trivial |
| Sparse-encoder drift — consumer app encodes queries with a different tokenizer/hash than the indexer → hybrid recall silently degrades | L | M | One shared, versioned encoder module (`bm25-v1`); `sparse_encoder` recorded per point; reconcile flags version mixes; encoder bump = sparse-only backfill, no re-embedding |
| Redis data loss | L | L | Rebuildable: re-enumeration + Qdrant `content_hash` short-circuit avoids re-embedding |
| MarkItDown fidelity gaps (complex XLSX, diagram-heavy PPTX) degrade retrieval | M | M | Golden-file tests set expectations; `converter` field in payload aids diagnosis; OCR/LLM tier for image-heavy docs |
| Poison files (corrupt, exotic, enormous) clog retries | M | L | Attempt caps, `UnrecoverableError` classification, size caps, Bull Board visibility |
| Sensitive content lands in Qdrant payload (`text`) on a shared host | M | H | Qdrant loopback-only + optional API key; treat the Qdrant volume with the same sensitivity as the source folder; encryption-at-rest is a deploy-target decision |
| WSL2 dev box asleep ⇒ missed schedules mistaken for bugs | H | L | Documented (Q5); deploy target must be always-on; `/health.lastSyncAt` makes staleness obvious |
| Pricing/API details drift from these docs (written 2026-07) | M | L | Everything cost- or minor-API-related is tagged *"verify at implementation"*; Context7 re-check is one command away |

## 4. Standing review triggers

Revisit this file when: (a) any open question gets answered → flip the linked decision to ✅ and propagate; (b) Phase-1 spike results land → update [04](04-onedrive-graph-integration.md) §4 and the classifier tests; (c) a second consumer or a second *sync root* appears → D5/D9/Q6 all get re-examined; (d) base-folder count grows past a few dozen → revisit D12; (e) the first DeepInfra invoice arrives → sanity-check the OCR cost model against reality.
