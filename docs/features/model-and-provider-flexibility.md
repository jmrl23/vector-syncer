# Feature: Model & provider flexibility

> Every AI dependency is configuration. The system speaks to exactly one API shape — OpenAI-compatible, through the official OpenAI SDK — and everything specific lives in env vars. Design internals: [`10-decisions-and-risks.md`](../design/10-decisions-and-risks.md) D4/D8/D11/D14.

## The default contract is five env vars

```bash
OPENAI_BASE_URL=https://api.deepinfra.com/v1/openai   # ANY OpenAI-compatible provider
OPENAI_API_KEY=<provider token>                        # shared default key for everything

EMBEDDING_MODEL=BAAI/bge-m3                            # dense embeddings (worker + your app)
OCR_LLM_MODEL=Qwen/Qwen3-VL-235B-A22B-Instruct         # vision OCR (converter); empty = OCR off
LLM_MODEL=deepseek-ai/DeepSeek-V4-Flash                # catalog descriptions
```

Nothing provider-specific exists in code — DeepInfra is a *value*, not a dependency. Point `OPENAI_BASE_URL` at another provider (or a self-hosted OpenAI-compatible server) and the whole system follows, worker and consumer app alike — as long as no per-role override below is set.

## Per-role overrides — different providers at once

Each role can independently override the shared endpoint and key, falling back to `OPENAI_BASE_URL`/`OPENAI_API_KEY` when unset:

```bash
EMBEDDING_BASE_URL=   EMBEDDING_API_KEY=   # embeddings only
OCR_BASE_URL=         OCR_API_KEY=         # OCR only (converter)
LLM_BASE_URL=         LLM_API_KEY=         # catalog descriptions only
```

Leave all six blank and you get today's behavior — one provider for everything. Set any one pair and only that role moves; the other two keep using the shared default. This is how you'd, say, run embeddings through DeepInfra while routing catalog descriptions through a different provider, with no code changes either way.

## What swaps freely vs. what costs a rebuild

| Change | Cost |
|---|---|
| Provider, shared or per-role (`OPENAI_BASE_URL`+key, or `EMBEDDING_`/`OCR_`/`LLM_` `_BASE_URL`+key) | free — restart containers |
| OCR model (`OCR_LLM_MODEL`) | free — affects future conversions only (`resync --rebuild` to redo old ones) |
| Description model (`LLM_MODEL`) | free — next catalog refresh uses it |
| **Embedding model (`EMBEDDING_MODEL`)** | **a re-index**: dimensions and semantics are baked into every collection. Procedure exists and is zero-downtime (`resync --rebuild` → build `_v2` → alias flip), but plan it — and your app must switch its query-side model at the same moment |
| Sparse encoder version (`bm25-v1` → `-v2`) | sparse-only backfill, no re-embedding — but indexer and app must move together (shared module) |

**Practical rule:** treat the embedding model as the one decision to finalize *before* the first big backfill; everything else is a config flip forever.

## Kill switches

```bash
OCR_LLM_MODEL=        # empty → no vision calls at all (scans convert to ~nothing and trip the warning counter)
SPARSE_MODE=off       # dense-only writes, if the sparse side ever misbehaves
```

## Coordination with your app

Your app shares two of these values and must stay in lockstep:

- `EMBEDDING_MODEL` (+ whichever endpoint/key actually serves it — `EMBEDDING_BASE_URL`/`EMBEDDING_API_KEY` if set, else `OPENAI_BASE_URL`/`OPENAI_API_KEY`) — queries must be embedded with the indexer's model, via the same provider.
- The `bm25-v1` encoder — imported from the shared consumer package, never reimplemented.

Both are guarded: every point records `embed_model` and `sparse_encoder`, and the worker refuses to start against a collection whose vector size doesn't match its configured model.

## If the provider retires a model

Model IDs are pinned in env, so nothing breaks silently — calls start failing loudly and queues hold work. Swap the ID (free for OCR/LLM; rebuild procedure for embeddings) or move that role's base URL (`OPENAI_BASE_URL`, or the role-specific override) to wherever the model is still served, including self-hosting (the documented switch-back for bge-m3 is a FlagEmbedding sidecar — D4).
