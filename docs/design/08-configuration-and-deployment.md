# 08 — Configuration & deployment

> Status: draft · Last updated: 2026-07-27

## 1. Planned repository layout

```
vector-syncer/
├── README.md                      # index (repo root)
├── docs/design/01…10-*.md         # these design docs
├── docs/features/                 # per-feature usage guides
├── docs/superpowers/              # approved spec + implementation plans
├── docker-compose.yml
├── .env.example                   # every variable below, with safe defaults
├── data/                          # volume: encrypted MSAL token cache (gitignored)
├── worker/                        # TypeScript orchestrator
│   ├── Dockerfile                 # node:24-slim multi-stage
│   ├── package.json  tsconfig.json
│   ├── prompts/                   # *.hbs LLM prompt templates, e.g. catalog-description.hbs (D16)
│   └── src/
│       ├── index.ts               # boot: config → queues → schedulers → workers → fastify
│       ├── config.ts              # zod-validated env
│       ├── prompts.ts             # compile-once loader for ../prompts/*.hbs — typed render fns, the only Handlebars touchpoint (D16)
│       ├── auth/                  # msal.ts, encryptedCachePlugin.ts, cli-auth.ts (device code)
│       ├── graph/                 # client.ts, delta.ts, download.ts, types.ts
│       ├── queues/                # queues.ts, schedulers.ts
│       ├── jobs/                  # sync.ts, processFile.ts, deleteFile.ts, reconcile.ts, updateMeta.ts, refreshCatalog.ts
│       ├── pipeline/              # converterClient.ts, chunk.ts, embeddings.ts (OpenAI SDK → DeepInfra), catalog.ts (DeepSeek descriptions)
│       ├── qdrant/                # client.ts, schema.ts, writer.ts
│       ├── state/                 # redis.ts, fileState.ts, deltaState.ts
│       └── ops/                   # health.ts, bullboard.ts, logger.ts
└── converter/                     # Python sidecar
    ├── Dockerfile                 # python:3.12-slim + uv
    ├── pyproject.toml
    ├── prompts/                   # ocr-image.hbs — static OCR instruction template (D16)
    ├── app/main.py                # FastAPI /convert, /healthz
    └── tests/  fixtures/          # golden files: sample.docx/pdf/xlsx → expected md traits
```

## 2. Environment variables

| Variable | Default | Notes |
|---|---|---|
| `AZURE_TENANT_ID` | — (required) | or `consumers` for personal OneDrive (Q1) |
| `AZURE_CLIENT_ID` | — (required) | app registration |
| `GRAPH_SCOPES` | `Files.Read offline_access User.Read` | space-separated; `Files.Read.All` if folder is shared |
| `ONEDRIVE_FOLDER_PATH` | — (required) | e.g. `/Documents/KnowledgeBase` |
| `DELTA_SCOPE` | `folder` | `root` = fallback mode ([04](04-onedrive-graph-integration.md) §4) |
| `SYNC_CRON` | `*/5 * * * *` | delta poll schedule |
| `MSAL_CACHE_PATH` | `/app/data/msal-cache.bin` | on the `./data` volume |
| `MSAL_CACHE_KEY` | — (required) | 64 hex chars (32 bytes) for AES-256-GCM |
| `REDIS_URL` | `redis://redis:6379` | |
| `QDRANT_URL` | `http://qdrant:6333` | |
| `QDRANT_API_KEY` | empty | set if Qdrant auth enabled / cloud |
| `QDRANT_COLLECTION_PREFIX` | `od_` | collections named `{prefix}{base-folder slug}` (aliases; physical adds `_vN`) |
| `ROOT_FILES_COLLECTION` | `_root` | catch-all slug for files directly under the sync root; empty = skip them |
| `CATALOG_COLLECTION` | `od_catalog` | registry collection ([06](06-qdrant-design.md) §6) |
| `CONVERTER_URL` | `http://converter:8000` | |
| `OPENAI_API_KEY` | — (required) | the OpenAI SDK's native var — set to the DeepInfra token; shared default key for embeddings + OCR + descriptions |
| `OPENAI_BASE_URL` | `https://api.deepinfra.com/v1/openai` | the SDK's native var — **any** OpenAI-compatible provider slots in here; nothing provider-specific in code |
| `EMBEDDING_BASE_URL` / `EMBEDDING_API_KEY` | empty → falls back to `OPENAI_BASE_URL`/`OPENAI_API_KEY` | override to serve embeddings from a different provider than OCR/descriptions |
| `OCR_BASE_URL` / `OCR_API_KEY` | empty → falls back to `OPENAI_BASE_URL`/`OPENAI_API_KEY` | override for the converter's OCR calls only |
| `LLM_BASE_URL` / `LLM_API_KEY` | empty → falls back to `OPENAI_BASE_URL`/`OPENAI_API_KEY` | override for catalog-description calls only |
| `EMBEDDING_MODEL` | `BAAI/bge-m3` | served by DeepInfra by default |
| `EMBEDDING_TOKENIZER` | `Xenova/bge-m3` | HF repo the chunker loads the *exact* embedding tokenizer from (`AutoTokenizer`, D15); must match `EMBEDDING_MODEL` — swap them together |
| `EMBEDDING_DIMENSIONS` | `1024` | must match model & every collection |
| `EMBEDDING_BATCH_SIZE` | `64` | texts per embeddings call |
| `SPARSE_MODE` | `bm25` | hybrid sparse writes via the `bm25-v1` encoder ([06](06-qdrant-design.md) §4); `off` = dense-only fallback |
| `LLM_MODEL` | `deepseek-ai/DeepSeek-V4-Flash` | catalog descriptions (worker) |
| `CHUNK_TOKENS` / `CHUNK_OVERLAP_TOKENS` | `512` / `64` | |
| `ALLOWED_EXTENSIONS` | pdf,docx,pptx,xlsx,xls,csv,md,txt,html,htm,json,xml,epub,msg,png,jpg,jpeg | |
| `MAX_FILE_SIZE_MB` | `100` | |
| `FILE_CONCURRENCY` | `4` | files-worker baseline; live-adjustable at runtime via `cli.js concurrency <n>` ([07](07-bullmq-design.md) §1) |
| `SCRATCH_DIR` | `/tmp/vector-syncer` | tmpfs-able |
| `OCR_LLM_MODEL` | `Qwen/Qwen3-VL-235B-A22B-Instruct` | vision OCR via DeepInfra (converter); set empty to disable OCR |
| `DOCINTEL_ENDPOINT` / `DOCINTEL_KEY` | empty | fallback OCR tier (converter) |
| `BULL_BOARD_PORT` | `3001` | dashboard + `/health` |
| `LOG_LEVEL` | `info` | |

The three model IDs above (`EMBEDDING_MODEL`, `OCR_LLM_MODEL`, `LLM_MODEL`) are **initial preferences, not commitments** — any model served by the endpoint in that role's base URL (`EMBEDDING_BASE_URL`, `OCR_BASE_URL`, `LLM_BASE_URL`, each falling back to `OPENAI_BASE_URL`) works. Swapping the OCR or LLM model is free at any time; swapping the embedding model changes dimensions and therefore means a collection rebuild ([06](06-qdrant-design.md) §7) — finalize it before the first full backfill. An embedding swap also means updating `EMBEDDING_TOKENIZER` to the new model's tokenizer repo in the same change (D15): chunk budgets are measured with that tokenizer, and it must always match the model that embeds the chunks.

Secrets (`MSAL_CACHE_KEY`, `OPENAI_API_KEY`, `EMBEDDING_API_KEY`, `OCR_API_KEY`, `LLM_API_KEY`, `DOCINTEL_KEY`, `QDRANT_API_KEY`) come from `.env` (0600, gitignored) in v1; anything fancier (sops, vault) is a deploy-target question (Q5).

## 3. docker-compose.yml (sketch)

```yaml
name: vector-syncer
services:
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes", "--maxmemory-policy", "noeviction"]
    volumes: [redis_data:/data]
    healthcheck: { test: ["CMD", "redis-cli", "ping"], interval: 10s, retries: 5 }
    restart: unless-stopped

  qdrant:
    image: qdrant/qdrant:v1.15.1          # pin exact at implementation
    volumes: [qdrant_data:/qdrant/storage]
    ports: ["127.0.0.1:6333:6333"]        # loopback only — for local inspection/consumers on this host
    restart: unless-stopped

  converter:
    build: ./converter
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY:?}
      - OPENAI_BASE_URL=${OPENAI_BASE_URL:-https://api.deepinfra.com/v1/openai}
      - OCR_BASE_URL=${OCR_BASE_URL:-}       # falls back to OPENAI_BASE_URL above if unset
      - OCR_API_KEY=${OCR_API_KEY:-}         # falls back to OPENAI_API_KEY above if unset
      - OCR_LLM_MODEL=${OCR_LLM_MODEL:-Qwen/Qwen3-VL-235B-A22B-Instruct}
      - DOCINTEL_ENDPOINT=${DOCINTEL_ENDPOINT:-}
      - DOCINTEL_KEY=${DOCINTEL_KEY:-}
    expose: ["8000"]
    healthcheck: { test: ["CMD", "curl", "-f", "http://localhost:8000/healthz"], interval: 30s }
    restart: unless-stopped

  worker:
    build: ./worker
    env_file: .env
    volumes:
      - ./data:/app/data                  # encrypted MSAL cache survives rebuilds
    depends_on:
      redis: { condition: service_healthy }
      converter: { condition: service_healthy }
      qdrant: { condition: service_started }
    ports: ["127.0.0.1:3001:3001"]        # Bull Board + /health, loopback only
    restart: unless-stopped

volumes: { redis_data: {}, qdrant_data: {} }
```

Nothing is exposed beyond loopback; a reverse proxy with auth is the way to publish Bull Board if ever needed.

## 4. Local development loop

```bash
docker compose up -d redis qdrant converter   # infra + sidecar in containers
cd worker && yarn install
yarn auth                                     # one-time device-code sign-in → data/msal-cache.bin
yarn dev                                      # tsx watch src/index.ts against the containers
yarn sync:once                                # trigger a manual delta-sync job (no cron wait)
```

Tests: `yarn test` (vitest unit: chunker, point IDs, classifier, slug naming; testcontainers integration: redis+qdrant), `uv run pytest` in `converter/` (golden-file conversions via FastAPI test client).

## 5. Operations runbook

| Task | Procedure |
|---|---|
| **First-time setup** | `git init` → copy `.env.example` → fill Azure IDs + `openssl rand -hex 32` for `MSAL_CACHE_KEY` → `docker compose up -d` → `docker compose run --rm worker node dist/cli-auth.js` → sign in with device code → watch first full enumeration in Bull Board |
| **Re-authentication** (health shows `auth_required`) | Same auth command; existing state untouched; queued work resumes immediately |
| **Trigger a sync now** | `… node dist/cli.js sync` (dev: `yarn sync:once`) — enqueues a delta-sync immediately; safe while cron is active (runs serialize on the concurrency-1 queue) |
| **Full resync** | `… node dist/cli.js resync` — re-enumerates, reconciles orphans/paths; hash-skip keeps it cheap |
| **Rebuild after model/chunking change** | `… resync --rebuild [--collection od_hr]` — new collection versions + alias flips, all or per base folder ([06](06-qdrant-design.md) §7) |
| **Refresh catalog** | `… resync --catalog` — recompute stats + regenerate DeepSeek descriptions ([06](06-qdrant-design.md) §6) |
| **Redo conversions after an OCR/config change** | adjust `OCR_LLM_MODEL`, `docker compose up -d converter`, then `… resync --rebuild [--collection od_x]` (plain resync would hash-skip unchanged files) |
| **Inspect / retry failures** | Bull Board `http://localhost:3001` → `files` queue → failed |
| **Pause / resume** | `… cli.js pause|resume` (BullMQ queue pause) — e.g. before bulk-reorganizing the OneDrive folder, pause; reorganize; resume; one big delta handles it |
| **Adjust throughput live** | `… cli.js concurrency 8` — applies immediately, no restart; reverts to `FILE_CONCURRENCY` on restart; `sync` stays at 1 by design |
| **Backups** | Qdrant: snapshot API or `qdrant_data` volume; Redis: AOF in `redis_data`. Both are *conveniences* — OneDrive remains the source of truth; worst case is one full resync |
| **Monitor** | `curl :3001/health` (wire to uptime-kuma/healthchecks.io per deploy target) |

## 6. Deployment targets

**Decided (Q5, 2026-07-26): Docker on the user's WSL2 machine now, possibly a VPS later.** WSL2 notes: enable Docker autostart with the distro, and accept that a sleeping machine *delays* syncs rather than losing them — the next delta poll catches up. A later VPS move is mechanical: copy `.env`, the `data/` directory (encrypted MSAL cache) and the `redis_data`/`qdrant_data` volumes, then `docker compose up -d`. Kubernetes remains a straightforward later translation (each service → Deployment; `data/` → Secret + PVC) but is explicitly not designed for now.
