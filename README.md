# vector-syncer

> Status: **planning** · Last updated: 2026-07-26 · No code yet — this repo currently holds the design docs.

A self-hosted background worker that mirrors a Microsoft OneDrive folder tree into **Qdrant** vector collections — **one collection per top-level subfolder ("base folder")** — so the folder's documents become searchable by any RAG / semantic-search application.

The pipeline:

```
OneDrive folder ──(Graph delta query, delegated auth)──► BullMQ jobs
    ──(download)──► MarkItDown (+ optional OCR) ──► Markdown
    ──(chunk · embed dense: bge-m3 · encode sparse: local BM25)──► Qdrant hybrid points
                                                (collection per base folder + catalog)
```

Key properties:

- **Delegated access** — the worker acts as a signed-in user (one-time device-code login, silent token refresh afterwards). No app-only/admin-consented permissions required.
- **Incremental** — Microsoft Graph *delta queries* return only what changed since the last run; unchanged files are never re-downloaded or re-embedded.
- **Idempotent** — deterministic point IDs and content-hash checks make every job safe to retry.
- **Hybrid search from day one** — every point carries a dense bge-m3 vector *and* a locally computed BM25-style sparse vector; consumers query with RRF fusion, so exact tokens (codes, invoice numbers) and semantic paraphrases both hit.
- **Self-contained core, one AI provider** — runs as a small `docker compose` stack (Node.js worker, Python conversion sidecar, Redis, Qdrant). All AI — `BAAI/bge-m3` embeddings, `Qwen/Qwen3-VL-235B-A22B-Instruct` OCR, `deepseek-ai/DeepSeek-V4-Flash` catalog descriptions — goes through **DeepInfra's OpenAI-compatible API** using the official OpenAI SDK and a single key. Provider, endpoint and model IDs are pure configuration (`OPENAI_BASE_URL` + model env vars) — nothing is hardcoded.
- **Self-describing** — a small `od_catalog` collection holds one embedded, LLM-described entry per base-folder collection, so agentic consumers can pick the right collection before searching.

## Planning documents

| Doc | Contents |
|---|---|
| [01-requirements.md](docs/design/01-requirements.md) | Goals, functional & non-functional requirements, constraints, out of scope |
| [02-architecture.md](docs/design/02-architecture.md) | Components, data flow, sequence diagrams, consistency model, failure modes |
| [03-tech-stack.md](docs/design/03-tech-stack.md) | Chosen stack per layer, rationale, rejected alternatives, dependency list |
| [04-onedrive-graph-integration.md](docs/design/04-onedrive-graph-integration.md) | Entra app registration, delegated auth (MSAL device code + token cache), delta queries, downloads, throttling |
| [05-conversion-pipeline.md](docs/design/05-conversion-pipeline.md) | MarkItDown converter service, OCR strategy tiers, chunking algorithm, embedding providers & cost |
| [06-qdrant-design.md](docs/design/06-qdrant-design.md) | Collection & payload schema, point IDs, write/delete flows, reindex & rebuild procedures |
| [07-bullmq-design.md](docs/design/07-bullmq-design.md) | Queue topology, job scheduler, retries, rate limiting, deduplication, dashboard |
| [08-configuration-and-deployment.md](docs/design/08-configuration-and-deployment.md) | Repo layout, environment variables, docker-compose, operations runbook |
| [09-roadmap.md](docs/design/09-roadmap.md) | Implementation phases with acceptance criteria |
| [10-decisions-and-risks.md](docs/design/10-decisions-and-risks.md) | Decision log, **open questions that need answers**, risk register |

## Feature guides (usage)

Usage-oriented guides — one per feature, written for the people and apps *using* the system — live in [docs/features/](docs/features/README.md): [OneDrive sync](docs/features/onedrive-sync.md), [searching from your app](docs/features/searching-from-your-app.md), [collection discovery](docs/features/collection-discovery.md), [conversation store](docs/features/conversation-store.md), [operations](docs/features/operations.md), and [model & provider flexibility](docs/features/model-and-provider-flexibility.md).

## Glossary

| Term | Meaning |
|---|---|
| **Delta query** | Graph API mechanism (`/delta`) that returns only drive items changed since a stored `deltaLink` token |
| **Delegated access** | The app calls Graph *as a user* (user signs in once; refresh tokens keep it alive) — as opposed to app-only access |
| **cTag** | Drive item property that changes only when file *content* changes (eTag also changes on metadata edits) |
| **Base folder** | A top-level subfolder of the synced root — each one maps to its own Qdrant collection |
| **Catalog** | Tiny extra collection (`od_catalog`): one described, embedded entry per base-folder collection — lets consumers route queries to the right collection |
| **MarkItDown** | Microsoft's Python library converting PDF/Office/HTML/images/… to Markdown |
| **Point** | A Qdrant record: one vector + JSON payload; here, one chunk of one document |
| **Job Scheduler** | BullMQ's factory for recurring (cron) jobs |

## Provenance

API usage in these docs was verified against current library documentation via Context7 on 2026-07-26: BullMQ 5.x (`upsertJobScheduler`, `worker.rateLimit` + `Worker.RateLimitError`, `deduplication` options, live `worker.concurrency` setter), MarkItDown v0.1.6 (`docintel_endpoint`, `llm_client`, `markitdown-ocr` plugin, `convert_stream`), Microsoft Graph v1.0 (`driveItem/delta` incl. non-root folders, `410 Gone` resync, `@microsoft.graph.downloadUrl`), MSAL Node (`acquireTokenByDeviceCode`, `ICachePlugin`, `acquireTokenSilent`), Qdrant (createCollection / payload indexes / upsert / delete-by-filter, plus hybrid-search APIs: named dense+sparse vectors, RRF fusion, additive `createVectorName` migration), and FlagEmbedding / HF Text Embeddings Inference (evaluated for self-hosting bge-m3 before DeepInfra was chosen as provider). Verified 2026-07-27: Transformers.js / `@huggingface/transformers` (`AutoTokenizer.from_pretrained` + `encode` for exact bge-m3 chunk token counts, `env.cacheDir` / `env.allowRemoteModels` for tokenizer caching and offline vendoring — D15, replacing js-tiktoken). Anything marked *"verify at implementation"* (mostly pricing and minor API kwargs) should be re-checked when the code is written.
