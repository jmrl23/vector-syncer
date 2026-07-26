# 03 — Tech stack

> Status: draft · Last updated: 2026-07-27

## 1. Summary

| Layer | Choice | Version (pin at impl.) | Why |
|---|---|---|---|
| Orchestrator language | **TypeScript on Node.js** | Node 24 LTS (or 22), TS 5.x | BullMQ is Node-first; first-class MSAL + Graph SDKs; single language for scheduler→Qdrant path |
| Queue / scheduling | **BullMQ** on **Redis** | bullmq ^5, redis 7.x (alpine) | Job Schedulers (cron), retries/backoff, per-worker rate limiting incl. manual 429 handling, deduplication — all built in |
| Graph access | **@microsoft/microsoft-graph-client** | ^3 | Lightweight; middleware retry handler honors `Retry-After`; raw `api(url)` calls work with absolute `nextLink`/`deltaLink` URLs |
| Auth | **@azure/msal-node** | ^3 | Device code flow, `acquireTokenSilent`, pluggable `ICachePlugin` for encrypted persistent cache |
| Conversion | **MarkItDown** (`markitdown[all]`) behind **FastAPI** | markitdown ≥0.1.6, Python 3.12, FastAPI ^0.11x | MarkItDown is Python-only ⇒ sidecar service; FastAPI+uvicorn is the minimal stable wrapper |
| **AI provider** | **DeepInfra** (OpenAI-compatible API), by default | — | One key/dashboard for all three models below by default; accessed **exclusively through the official OpenAI SDK** via its native `OPENAI_BASE_URL`/`OPENAI_API_KEY` env vars — models and endpoint are pure config, nothing hardcoded. Each model role can override the pair independently (`EMBEDDING_`/`OCR_`/`LLM_` `_BASE_URL`/`_API_KEY`) to use a different provider per role |
| Embeddings | **`BAAI/bge-m3`** on DeepInfra via the **openai** SDK | openai ^5; bge-m3 1024-d | Multilingual (100+ langs), 8k-token input, cheap; OpenAI-compatible endpoint returns **dense only** — the sparse half of hybrid is computed locally (next row, D14) |
| Sparse vectors (hybrid) | **custom BM25 encoder** in the worker (`bm25-v1`) | in-repo, no dependency | ~100 lines: unicode tokenize → FNV-1a `u32` → BM25 TF weights; IDF server-side via Qdrant `modifier: idf`; shared with the consumer app for query encoding ([06](06-qdrant-design.md) §4) |
| OCR | **markitdown-ocr plugin** driven by **`Qwen/Qwen3-VL-235B-A22B-Instruct`** on DeepInfra | — | Vision-LLM OCR of images embedded in PDF/DOCX/PPTX/XLSX; Azure DocIntel / local Tesseract kept as fallbacks — [05](05-conversion-pipeline.md) §3 |
| LLM (descriptions etc.) | **`deepseek-ai/DeepSeek-V4-Flash`** on DeepInfra | — | Generates catalog collection descriptions ([06](06-qdrant-design.md) §6); available for future summarization needs |
| Chunking | **@langchain/textsplitters** + **@huggingface/transformers** (tokenizer only) | latest | Markdown-aware recursive splitting measured with the *actual* bge-m3 tokenizer (`AutoTokenizer`, D15) — exact counts against what the embedder sees, not an OpenAI-BPE proxy; replaceable by a custom splitter later |
| Vector DB | **Qdrant** + **@qdrant/js-client-rest** | server 1.15+, client matching | User requirement; payload filtering, payload indexes, aliases for zero-downtime reindex |
| HTTP client (worker) | native `fetch` / `undici` | Node built-in | Streaming downloads; no extra dep |
| Config | **dotenv** + **zod** | latest | Fail-fast validated env schema |
| Logging | **pino** (+ pino-pretty in dev) | ^9 | Structured JSON logs |
| Dashboard / health | **@bull-board/fastify** + Fastify | latest | Queue inspection, manual retries, `/health` on the same tiny server |
| IDs / hashing | **uuid** (v5), node `crypto` | — | Deterministic point IDs; content hashing fallback |
| Tests | **vitest** (worker), **pytest** (converter), **testcontainers** (redis+qdrant integration) | latest | — |
| Packaging | **Docker + docker compose** | compose v2 | Whole stack reproducible on any Linux host |
| Package managers | **yarn** (worker), **uv** (converter) | yarn 4.x | User's standard (yarn); lockfile-first |

The three AI model IDs are the user's **initial preferences**, deliberately kept as env config (D11 flexibility clause) — swap freely, with one caveat: the *embedding* model determines vector dimensions, so it should be finalized before the first full backfill (changing it later = collection rebuild, [06](06-qdrant-design.md) §7).

## 2. The one structural decision: hybrid Node + Python

The two anchor requirements pull in different directions: **BullMQ** is a Node-native library; **MarkItDown** is a Python-only library. Options considered:

### Option A — All TypeScript ❌
Replace MarkItDown with a Node patchwork (`pdf-parse`, `mammoth`, `turndown`, `xlsx`, …). Rejected: MarkItDown's breadth (PDF, DOCX, PPTX, XLSX, EPUB, MSG, images, ZIP…), its actively maintained converter registry, and its OCR integrations (Azure Document Intelligence, LLM-vision plugin) would take weeks to approximate and forever to maintain.

### Option B — All Python ❌ (close second)
BullMQ ships an official Python client, and it does support workers, retries and even deduplication. Rejected for v1 because the Python client trails the Node reference implementation on exactly the features this design leans on (Job Schedulers / repeatable-job management, manual `rateLimit` ergonomics), and the surrounding ecosystem (Bull Board, docs, examples) is Node-centric. MSAL Python + msgraph-sdk-python are fine, so this remains a viable consolidation path if the sidecar ever feels heavy. Recorded as D1 in [10-decisions-and-risks.md](10-decisions-and-risks.md).

### Option C — Hybrid: Node orchestrator + Python conversion sidecar ✅
Node owns everything stateful and scheduled; Python owns the single stateless function `bytes → markdown`. The seam between them is one HTTP endpoint (`POST /convert`), which is also the natural process-isolation boundary for the one step most likely to crash or hog CPU (parsing arbitrary office files).

### Also rejected

| Alternative | Why not |
|---|---|
| Plain `node-cron` + async loop, no queue | No retries, no backpressure, no visibility, no dedup — all rebuilt by hand |
| Temporal / Inngest | Great engines, but heavyweight for one pipeline; BullMQ+Redis is enough and was requested |
| n8n / Power Automate / Logic Apps | Low-code lock-in, awkward custom chunking/embedding logic, harder self-hosting story |
| Azure AI Search integrated vectorization | Replaces Qdrant (hard requirement) and drags in app-only permissions |
| `markitdown` as CLI child-process inside the Node image | Viable minimal mode; loses warm imports and dependency isolation. Kept as fallback (D3) |

## 3. Dependency lists (planned)

### worker/package.json (runtime)

```
bullmq                          queue, scheduler, workers
@azure/msal-node                delegated auth (device code + silent)
@microsoft/microsoft-graph-client   Graph REST wrapper
@microsoft/microsoft-graph-types    (dev) typings for driveItem etc.
@qdrant/js-client-rest          Qdrant client
openai                          OpenAI SDK → DeepInfra (bge-m3 embeddings, DeepSeek catalog descriptions)
@langchain/textsplitters        markdown-aware chunking
@huggingface/transformers       AutoTokenizer — exact bge-m3 token counts for chunk sizing (tokenizer files only, no model weights)
handlebars                      renders worker/prompts/*.hbs — all LLM prompt text lives in template files, never in code (D16)
uuid                            UUIDv5 deterministic point ids
zod + dotenv                    config validation
pino                            logging
fastify + @bull-board/fastify + @bull-board/api   dashboard & health
```

Dev: typescript, tsx, vitest, testcontainers, @types/node, eslint + prettier (or biome). Installed with **yarn** (workspace root + `worker/`).

### converter/pyproject.toml (runtime)

```
markitdown[all]                 conversion incl. pdf/docx/pptx/xlsx/epub/msg/image deps
fastapi + uvicorn[standard]     HTTP wrapper
python-multipart                multipart uploads
openai                          markitdown llm_client → DeepInfra (Qwen3-VL OCR)
markitdown-ocr                  OCR plugin (LLM-vision over embedded images)
# OCR instruction text: converter/prompts/ocr-image.hbs — static template read at startup (stdlib pathlib,
#   no templating dependency needed) and passed to MarkItDown's llm_prompt (D16)
# fallback OCR tier: markitdown[all] already includes azure-ai-documentintelligence
```

Dev: pytest, httpx (test client), ruff.

### Infrastructure images

| Service | Image |
|---|---|
| Redis | `redis:7-alpine` — **must** run with `--maxmemory-policy noeviction` (BullMQ requirement) and `--appendonly yes` |
| Qdrant | `qdrant/qdrant:v1.15.x` (pin exact at implementation) |
| worker | `node:24-slim` multi-stage build |
| converter | `python:3.12-slim` + `uv` |

## 4. Version pinning policy

- Lockfiles committed (`pnpm-lock.yaml`, `uv.lock`).
- Qdrant server and JS client pinned to the same minor.
- Renovate/Dependabot later; not a v1 concern.
