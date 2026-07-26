# Feature: Searching the vector database from your application

> The consumer contract: how another application (yours — Node/TypeScript) gets data out of the collections vector-syncer maintains. Design internals: [`06-qdrant-design.md`](../design/06-qdrant-design.md) §3/§4/§8.

## The three rules

1. **Embed queries with the same model the indexer used** (`EMBEDDING_MODEL`, initially `BAAI/bge-m3`, 1024-d) through the same OpenAI-compatible endpoint. A query embedded with a different model returns garbage, not an error.
2. **Sparse-encode queries with the exact same `bm25-v1` module** the indexer used — import it from the shared consumer package (working name `@vector-syncer/consumer`, finalized in Phase 4). Never reimplement the tokenizer.
3. **Read-only on `od_*` collections.** They are managed mirrors — rebuilds and OneDrive deletions recreate or drop them at will. Anything your app needs to *write* goes in its own collections (see [conversation-store.md](conversation-store.md)).

## Setup

```ts
import { QdrantClient } from '@qdrant/js-client-rest';
import OpenAI from 'openai';
import { bm25 } from '@vector-syncer/consumer';   // the shared bm25-v1 encoder

const qdrant = new QdrantClient({ url: process.env.QDRANT_URL ?? 'http://localhost:6333' });
const ai = new OpenAI();  // reads OPENAI_BASE_URL + OPENAI_API_KEY — same values the worker uses

async function embed(text: string): Promise<number[]> {
  const res = await ai.embeddings.create({
    model: process.env.EMBEDDING_MODEL ?? 'BAAI/bge-m3',
    input: text,
    encoding_format: 'float',
  });
  return res.data[0].embedding;
}
```

## Hybrid search (the default query)

Every point carries a `dense` and a `sparse` vector; fuse them with RRF so semantic paraphrases *and* exact tokens (invoice numbers, codes, names) both hit:

```ts
const hits = await qdrant.query('od_hr', {
  prefetch: [
    { query: await embed(q), using: 'dense',  limit: 20 },
    { query: bm25(q),        using: 'sparse', limit: 20 },
  ],
  query: { fusion: 'rrf' },
  limit: 8,
  with_payload: true,
});
```

### Optional filters (indexed fields)

```ts
filter: {
  must: [
    { key: 'extension', match: { value: 'pdf' } },                    // file type
    { key: 'ancestor_ids', match: { value: someSubfolderId } },       // limit to a subtree
    { key: 'last_modified_ts', range: { gte: 1767225600 } },          // recency
  ],
}
```

Indexed and filterable: `file_id`, `extension`, `mime_type`, `ancestor_ids`, `last_modified_ts`.

## What a result contains (payload contract)

Everything needed to render a result **without any second lookup**:

| Field | Use it for |
|---|---|
| `text` | the chunk content — feed to your RAG prompt / display |
| `heading_path` | breadcrumb context ("Financials › Revenue by region") |
| `file_name`, `path` | display + grouping |
| `web_url` | click-through link to the document in OneDrive |
| `created_by`, `modified_by`, `last_modified` | attribution ("from Q3-report.docx, edited by A. Chen last week") |
| `file_id`, `chunk_index`, `chunk_count` | grouping results per document; context expansion (below) |
| `base_folder_name`, `extension`, `mime_type`, `size_bytes` | badges / facets |

## Context expansion (neighboring chunks)

To hand the LLM more surrounding context than one chunk:

```ts
const neighbors = await qdrant.scroll('od_hr', {
  filter: { must: [
    { key: 'file_id', match: { value: hit.payload.file_id } },
    { key: 'chunk_index', range: { gte: hit.payload.chunk_index - 1, lte: hit.payload.chunk_index + 1 } },
  ]},
  with_payload: true, limit: 3,
});
```

## Searching “everything” (multiple collections)

Qdrant has no server-side cross-collection query. The intended pattern is **catalog-first**: ask the catalog which 1–3 collections fit the query, search those, merge client-side ([collection-discovery.md](collection-discovery.md) has the routing code). For a true global sweep, fan out over all collections from a catalog enumeration and merge by score — acceptable at a handful of collections, which the catalog keeps you from needing most of the time.

## Degraded mode worth building in

Dense query embedding needs the AI provider; the sparse side is computed locally. If the provider is down, fall back to **sparse-only** search (`query: bm25(q), using: 'sparse'` — no prefetch/fusion) — keyword search keeps working through an outage.
