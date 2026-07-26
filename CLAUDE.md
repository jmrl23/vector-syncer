# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository state

**Planning-complete, pre-code.** This repo currently holds an owner-approved design; implementation hasn't started. Document hierarchy (highest authority first):

1. `docs/superpowers/specs/2026-07-26-vector-syncer-design.md` — the approved spec. **Where the spec and a design doc disagree, the spec wins and the doc should be fixed.**
2. `docs/design/01…10-*.md` — deep design per area (requirements, architecture, tech stack, Graph integration, conversion, Qdrant, BullMQ, config/deploy, roadmap, decisions/risks). `10-decisions-and-risks.md` is the decision log — don't re-litigate D1–D14 without the owner.
3. `docs/features/*.md` — usage guides (consumer contract lives here and in `docs/design/06` §8/§10).
4. `docs/superpowers/plans/` — implementation plans. Execution is a **plan series**: `…-p0-p1-foundation.md` exists (14 TDD tasks); plans for P2–P5 are written only at each phase boundary, because the Phase-1 spike findings (recorded into `docs/design/04` §4) feed later phases.

## What this builds

A worker that mirrors one OneDrive folder (delegated device-code auth, Graph delta queries) into Qdrant: download → MarkItDown conversion (vision-LLM OCR inline) → heading-aware chunking → bge-m3 dense embedding + **local** BM25-style sparse vector → collection per top-level subfolder (`od_{slug}`, alias over versioned physical) + a self-describing `od_catalog`. Hybrid (RRF) search from day one. Node/TS orchestrator (`worker/`) + Python FastAPI conversion sidecar (`converter/`) + Redis + Qdrant in one compose stack. The consumer is the owner's separate Node/TS app.

## Non-negotiable design constraints

- **AI access only through the official OpenAI SDK**, configured via its native `OPENAI_BASE_URL`/`OPENAI_API_KEY`; every model ID is its own env var. Nothing provider-specific in code (DeepInfra is a config value).
- Swapping the **embedding model means a full re-index** (alias-flip rebuild); OCR/description models swap freely. Query embedding must always match the indexing model.
- The worker creates/writes/drops **only** `od_*` collections and `od_catalog`. Any other collection in the same Qdrant (e.g. `app_conversations`) is consumer-owned — never touch it from worker code.
- The `bm25-v1` sparse encoder is **shared code with the consumer app** (imported, never reimplemented); changing it = version bump + sparse-only backfill, and golden-vector tests pin the encoding.
- `sync` queue concurrency **1 is a correctness invariant** (delta walks must never overlap). `files` concurrency is env baseline + runtime-adjustable — keep it that way.
- Consistency machinery is load-bearing: deterministic UUIDv5 point IDs, delete-then-upsert per file, dedup id `itemId:cTag`, delta token advanced only after enqueue, exact delta `$select` list in `docs/design/04` §4.
- Redis must run `noeviction` + AOF (BullMQ requirement). All ports loopback-only. Secrets only via `.env`; MSAL cache is AES-256-GCM encrypted.
- Scratch downloads are deleted in a `finally` — file content persists nowhere outside Qdrant payloads.

## Package managers & commands

`worker/` uses **yarn 4** (never npm/pnpm); `converter/` uses **uv**. Commands are defined by the P0+P1 plan and exist once Task 1–3 land:

```bash
docker compose up -d                 # redis (noeviction+AOF), qdrant, converter, worker
cd worker && yarn install
yarn auth                            # one-time device-code sign-in (needs the owner)
yarn dev                             # tsx watch
yarn dev:delta                       # classified delta dry-run against the real drive
yarn test                            # vitest; tests/int/** uses testcontainers (needs Docker)
yarn vitest run tests/graph/classify.test.ts   # single test file
yarn typecheck
cd converter && uv run pytest -q     # converter golden-file tests
```

Two things always require the human owner: the Entra app registration + `yarn auth` device-code sign-in, and any experiment against the real OneDrive (Phase-1 spike, acceptance runs).

## Process rules

- Implementation follows the plan tasks TDD-style (failing test → minimal code → pass → commit per task, conventional commits).
- Verify library APIs with Context7 before writing integration code; README's Provenance section records what was verified on 2026-07-26 — anything tagged *"verify at implementation"* (mostly pricing/minor kwargs) must be re-checked when touched.
- When implementation deviates from a design doc, update the doc in the same commit (spike findings go in `docs/design/04` §4 as regression-tested facts).

## Knowledge graph

`graphify-out/` (once present) holds this project's knowledge graph — for questions about architecture, file relationships, or where something is specced, query it first via the `graphify` skill instead of re-reading all docs.
