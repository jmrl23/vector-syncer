# 06 — Qdrant design

> Status: draft · Last updated: 2026-07-26 (rev 3: hybrid dense+sparse from day one · rev 2: collection-per-base-folder, catalog collection, bge-m3 @ DeepInfra)

## 1. Collection layout: one per base folder (decision D12)

Every **base folder** (top-level subfolder of the sync root) gets its own collection. Files sitting directly in the sync root go to a catch-all. A small **catalog** collection (§6) describes them all.

```
OneDrive /KnowledgeBase          Qdrant
├── HR/            ─────────►    od_hr            (alias → od_hr_v1)
├── Engineering/   ─────────►    od_engineering   (alias → od_engineering_v1)
├── Sales Docs/    ─────────►    od_sales_docs    (alias → od_sales_docs_v1)
└── loose-file.pdf ─────────►    od__root         (alias → od__root_v1)
                                 od_catalog       (registry: 1 point per collection above)
```

### Naming & mapping

- **Slug**: base-folder name → lowercase, ASCII-folded, non-alphanumerics collapsed to `_`, trimmed (`Sales Docs` → `sales_docs`). Collision (two folders producing the same slug) → append a 6-char hash of the folder ID.
- **Alias** = `{QDRANT_COLLECTION_PREFIX}{slug}` (default prefix `od_`); **physical** collection adds a version suffix (`_v1`) so rebuilds are alias flips. Catch-all uses the reserved slug `_root` (sluggification can't produce a leading underscore, so it can't collide).
- **Authoritative mapping lives in Redis** (`vs:collections`: base-folder **ID** → `{ alias, physical, name }`), mirrored into each catalog point. Keyed by ID, not name — renaming a base folder does **not** rename its collection (the alias keeps working; the catalog entry's display name updates; an alias-rename cosmetic pass is a possible later nicety).
- **Lifecycle**: created lazily the first time an item routes to a new base folder; dropped (with catalog entry) when the base folder is deleted (FR-15 — destructive but rebuildable, like everything here).

### Creation (per collection, on demand)

```ts
await qdrant.createCollection(physical, {
  vectors:        { dense:  { size: 1024, distance: 'Cosine', on_disk: true } },  // bge-m3
  sparse_vectors: { sparse: { modifier: 'idf' } },   // BM25-style sparse, written from day one (D14) — IDF applied server-side
  on_disk_payload: true,
});
for (const [field, schema] of [
  ['file_id', 'keyword'], ['mime_type', 'keyword'], ['extension', 'keyword'],
  ['ancestor_ids', 'keyword'], ['last_modified_ts', 'integer'],
] as const) {
  await qdrant.createPayloadIndex(physical, { field_name: field, field_schema: schema });
}
await qdrant.updateCollectionAliases({
  actions: [{ create_alias: { collection_name: physical, alias_name: alias } }],
});
```

Both slots are written from day one (hybrid v1, D14). The dense vector comes from the embedding provider; the sparse vector is computed **locally** by the worker's BM25 encoder (§4) — the provider's OpenAI-compatible endpoint exposes dense only and never sees the sparse side. If a named vector were ever missing, Qdrant's `createVectorName` can add it to an existing collection — no one-way doors.

Startup checks (fail fast, clear message): every mapped collection exists, `dense` size == embedder dimensions (1024), aliases resolve, catalog exists.

## 2. Point IDs — deterministic, idempotent

```ts
import { v5 as uuidv5 } from 'uuid';
const NAMESPACE = '6f9d2e8a-…-fixed-project-uuid';   // constant, committed

const pointId = (driveId: string, itemId: string, chunkIndex: number) =>
  uuidv5(`${driveId}/${itemId}#${chunkIndex}`, NAMESPACE);
```

Re-processing a file computes the *same* IDs ⇒ upsert overwrites ⇒ retries are free. IDs are collection-independent, which makes cross-collection moves a plain delete+insert (§4).

## 3. Payload schema (document collections)

| Field | Type | Indexed | Purpose |
|---|---|---|---|
| `file_id` | keyword | ✅ | delete/update all chunks of a file; group results |
| `drive_id` | keyword | — | multi-drive future |
| `file_name` | keyword | — | display |
| `path` | keyword | — | display + debugging (`/KnowledgeBase/HR/policies/leave.docx`) |
| `extension` | keyword | ✅ | quick type filters (`pdf`, `docx`, …) |
| `mime_type` | keyword | ✅ | precise type filters |
| `web_url` | string | — | click-through link to OneDrive |
| `size_bytes` | integer | — | diagnostics |
| `base_folder_id` / `base_folder_name` | keyword | — | self-describing points; reconcile cross-check with the catalog |
| `ancestor_ids` | keyword[] | ✅ | folder-subtree deletes/filters in one condition |
| `created_at` / `created_by` | string | — | authorship (from Graph `createdBy.user.displayName`) |
| `last_modified` / `modified_by` | string | — | recency + authorship for display |
| `last_modified_ts` | integer | ✅ | range filters ("changed this year") |
| `ctag` / `content_hash` | keyword | — | change detection; lets a fresh worker skip re-embedding after Redis loss |
| `chunk_index` / `chunk_count` | integer | — | ordering, neighbor lookup for context expansion |
| `heading_path` | string | — | breadcrumb ("Financials › Revenue") |
| `text` | string | — | the chunk content (what RAG displays) |
| `embed_model` | keyword | — | audit; guards against mixed-model collections |
| `sparse_encoder` | keyword | — | sparse encoder version (`bm25-v1`); guards against tokenizer drift between indexer and consumer app |
| `synced_at` | string | — | diagnostics |

This is the "important information about the files" requirement (FR-14) made concrete — every point is fully self-describing and attributable without a second lookup.

## 4. Write paths

**Sparse vectors come from a local BM25 encoder** (hybrid v1, D14) — computed in the worker, no API involved, versioned **`bm25-v1`** and recorded per point:

1. Unicode-segment chunk text into words, lowercase, drop punctuation-only tokens (no stemming/stopwords in v1 — language-agnostic by construction).
2. Stable-hash each token (FNV-1a) to a `u32` sparse index.
3. Value = BM25 TF saturation `tf·(k1+1) / (tf + k1·(1−b+b·len/avgLen))`, constants fixed at `k1=1.2`, `b=0.75`, `avgLen=256`.
4. IDF stays server-side: the slot's `modifier: 'idf'` applies live corpus-level IDF at query time — always current as the collection grows.

The encoder is **shared code with the consumer app**: queries must be encoded with the identical tokenizer/hash, so it ships as a small importable module, and the `sparse_encoder` payload field catches version drift. An encoder change is a cheap sparse-only backfill (no re-embedding); `SPARSE_MODE=off` falls back to dense-only writes.

**Full (re)index of one file** — target collection resolved from the ancestor chain first:

```ts
const col = await resolveCollection(item);          // ensure-created, from vs:collections
await qdrant.delete(col, { wait: true,              // 1. drop every old chunk (handles shrinkage)
  filter: { must: [{ key: 'file_id', match: { value: item.id } }] } });
await qdrant.upsert(col, { wait: true,              // 2. insert the new set (batches ≤ 256)
  points: chunks.map((c, i) => ({
    id: pointId(item.driveId, item.id, i),
    vector: { dense: c.embedding,                   // hybrid v1: both slots written
              sparse: bm25(c.text) },               // { indices: u32[], values: number[] } from the bm25-v1 module
    payload: c.payload,
  })) });
```

`wait: true` keeps BullMQ's success signal honest. Delete-then-upsert is D7 — the milliseconds-long zero-point window per file is acceptable.

**Metadata-only update** (rename/move *within* a base folder — no re-embedding):

```ts
await qdrant.setPayload(col, { wait: true,
  payload: { file_name, path, web_url, ancestor_ids, last_modified, last_modified_ts, modified_by },
  filter: { must: [{ key: 'file_id', match: { value: item.id } }] } });
```

**Cross-base-folder move** (FR-16): `process-file` indexes into the *new* collection, then deletes `file_id` from the collection recorded in the file's previous Redis state. v1 re-embeds on move (simple, rare); optimization if it ever matters: `scroll` the old points with vectors and re-upsert them re-payloaded.

## 5. Delete paths (FR-15 — OneDrive deletion ⇒ Qdrant deletion, always)

```ts
// file deleted → all its chunk points go
await qdrant.delete(col, { wait: true,
  filter: { must: [{ key: 'file_id', match: { value: itemId } }] } });

// nested folder deleted → whole subtree in one call
await qdrant.delete(col, { wait: true,
  filter: { must: [{ key: 'ancestor_ids', match: { value: folderId } }] } });

// base folder deleted → drop collection + its catalog point + mapping
await qdrant.deleteCollection(physical);
await qdrant.delete(CATALOG, { wait: true, points: [catalogPointId(baseFolderId)] });
```

## 6. The catalog collection (`od_catalog`, decision D13)

A tiny registry — **one point per document collection** — so consumers (especially agents) can discover *what exists and what it's about* before searching.

| Payload field | Content |
|---|---|
| `collection` | alias to query, e.g. `od_hr` |
| `base_folder_id` / `base_folder_name` / `path` | identity of the source folder |
| `description` | 2–4 sentences, generated by `deepseek-ai/DeepSeek-V4-Flash` from **all the important signals**: folder name, sampled file names/paths, and brief content excerpts/titles from already-indexed chunks |
| `file_count` / `chunk_count` / `size_bytes` | live stats |
| `top_extensions` | e.g. `["pdf","docx"]` |
| `created_at` / `updated_at` | catalog entry freshness |

- **Point ID**: `uuidv5('catalog:' + baseFolderId)`. **Vector**: bge-m3 dense embedding of `"{name} — {description}"`, so "which collection covers travel reimbursements?" is itself a semantic query.
- **Maintained by** `refresh-catalog` jobs: on collection creation (initial description), during weekly reconcile (stats always; description regenerated when the file set changed materially, e.g. >20 % churn), on base-folder delete (point removed), and on demand via `resync --catalog`.
- The description prompt is a compact digest — folder name, file paths, and short excerpts from a sample of indexed chunks (user decision: rich catalog) — not whole documents, so regeneration stays fast and cheap.
- **Consumer pattern — two first-class catalog operations** (code in §8):
  1. **Enumerate** — *"what collections do you have?"* is answered **completely and deterministically** by scrolling `od_catalog` (no vector search): every entry's name, description and stats come back, ready to render — or to hand to the app's LLM — as the answer.
  2. **Route** — *"which collection covers X?"* is a semantic search over the embedded descriptions → pick the top collection(s) → search there.

  This is also exactly the tool-shape for a future MCP server: `list_collections()` + `search(collection, query)`.

## 7. Reconcile, rebuild, model changes

- **Weekly reconcile** (BullMQ scheduler): full delta enumeration (same code as the `410 Gone` path) → fix stale paths, delete orphaned `file_id`s, re-process hash mismatches, refresh catalog stats/descriptions, drop orphaned collections (mapping entries whose base folder no longer exists). Cheap because hash-skip makes untouched files no-ops.
- **Full rebuild** (`resync --rebuild [--collection od_hr]`): create `{alias}_v2`, enumerate into it, flip the alias, drop `_v1`. Zero downtime, per collection or all.
- **Redis lost, Qdrant intact**: before re-embedding a file, fetch one of its points (`scroll`, `file_id` filter, limit 1) and compare `content_hash` — match ⇒ restore Redis state, skip the DeepInfra spend.
- **Changing the embedding model**: dimensions/semantics are baked into collections ⇒ either full rebuild (alias flip), or Qdrant's additive migration — `createVectorName` a second named vector, backfill in the background, remove the old — both routine; `embed_model` payload + startup assertion enforce never mixing models silently.

## 8. What consumers do (reference queries)

```ts
// A. enumerate: "what collections are available?" — complete listing, no vector search
const all = await qdrant.scroll('od_catalog', { limit: 256, with_payload: true });
// → [{ collection: 'od_hr', base_folder_name: 'HR', description: '…', file_count, chunk_count, … }, …]
//   render directly, or pass to your app's LLM as grounding for its answer

// 0. route: which collection(s)? — catalog-first (the catalog stays dense-only: routing is semantic)
const cat = await qdrant.query('od_catalog', { query: await embed(q), limit: 3, with_payload: true });
const targets = cat.points.map(p => p.payload.collection);

// 1. search a document collection — hybrid (RRF fusion of dense + sparse), the v1 default
const hits = await qdrant.query('od_hr', {
  prefetch: [
    { query: await embed(q), using: 'dense',  limit: 20 },  // MUST be the same embedding model as indexing
    { query: bm25(q),        using: 'sparse', limit: 20 },  // MUST be the same bm25-v1 encoder module (§4)
  ],
  query: { fusion: 'rrf' },
  limit: 8,
  filter: { must: [{ key: 'extension', match: { value: 'pdf' } }] },   // optional, applies to both prefetches
  with_payload: true,
});
// hit.payload.text, .heading_path, .path, .web_url, .modified_by → straight into a RAG prompt with attribution
```

- **Cross-collection ("global") search** = fan out over `targets` and merge client-side by score — Qdrant has no server-side cross-collection query. This is the accepted cost of D12; the catalog keeps the fan-out small and smart.
- Context expansion: fetch `chunk_index ± 1` neighbors via `file_id` + `chunk_index` filter.

## 9. Future: beyond BM25 hybrid

Hybrid (dense + local BM25 sparse, RRF fusion) **is v1** — user decision 2026-07-26 (D14). The remaining upgrade paths, both additive:

1. **Learned sparse (true bge-m3 lexical weights)**: only if a serving path exposes them (self-hosted FlagEmbedding sidecar, or the provider adding the output) — would land in a *second* named slot via `createVectorName` (learned weights shouldn't get IDF re-weighting), then A/B against `bm25-v1`.
2. **Encoder improvements** (stemming, stopwords, language-aware tokenization): version-bump to `bm25-v2`, run a sparse-only backfill, update the shared module — no re-embedding, no rebuild.

ColBERT-style reranking (bge-m3's third output, Qdrant multivectors + MaxSim) stays a distant option — storage-hungry, only worth it if top-k precision becomes the bottleneck.
