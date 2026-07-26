# Feature: Collection discovery (the catalog)

> `od_catalog` is the system's self-description: one embedded, LLM-described entry per document collection. It answers two different questions for your app. Design internals: [`06-qdrant-design.md`](../design/06-qdrant-design.md) §6.

## Operation 1 — Enumerate: “what collections are available?”

A complete, deterministic listing — plain scroll, no vector search, no LLM required:

```ts
const { points } = await qdrant.scroll('od_catalog', { limit: 256, with_payload: true });

for (const p of points) {
  console.log(p.payload.collection, '—', p.payload.description);
}
// od_hr           — HR policies, leave and reimbursement forms, onboarding guides …
// od_engineering  — Architecture notes, runbooks, ADRs for the platform team …
// od_sales_docs   — Proposals, pricing sheets and customer case studies …
```

Render it directly, or hand the list to your app's LLM as grounding — the user question *"what collections do you have?"* is answerable from this call alone.

## Operation 2 — Route: “which collection covers X?”

The catalog point's vector is the embedding of `"{name} — {description}"`, so routing is itself a semantic query:

```ts
const routed = await qdrant.query('od_catalog', {
  query: await embed('travel reimbursement rules'),   // same embed() as searching-from-your-app.md
  limit: 3,
  with_payload: true,
});
const targets = routed.points.map(p => p.payload.collection as string);
// → ['od_hr', …] — now hybrid-search those collections and merge
```

The catalog is dense-only by design (routing is semantic); document collections are where hybrid search happens.

## What each catalog entry contains

| Field | Content |
|---|---|
| `collection` | the alias to query (`od_hr`) |
| `base_folder_name`, `path` | current display identity of the source folder (rename-safe: updates even though the collection name doesn't) |
| `description` | 2–4 LLM-generated sentences from folder name, sampled file paths **and** content excerpts |
| `file_count`, `chunk_count`, `size_bytes` | live stats |
| `top_extensions` | e.g. `["pdf","docx"]` |
| `updated_at` | freshness of this entry |

## Freshness semantics

Catalog entries are maintained automatically: created with the collection, stats refreshed on every weekly reconcile, descriptions **regenerated when the folder's content changed materially** (>20 % churn), removed the moment a base folder is deleted. Descriptions can therefore lag a little between reconciles — `updated_at` tells you how much. Force a refresh any time with `resync --catalog` ([operations.md](operations.md)).
