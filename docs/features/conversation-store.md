# Feature: Conversation store

> Persist your app's user↔agent conversations in the same Qdrant instance, retrieve any whole conversation with nothing but its `conversationId`, and get hybrid search over chat history for free. Design internals: [`06-qdrant-design.md`](../design/06-qdrant-design.md) §10.

## The model

One `conversationId` = one **whole thread**, both sides:

```
conversationId "abc-123"
  ├─ turn 0  role: user       "How do I file a travel reimbursement?"
  ├─ turn 1  role: assistant  "According to HR/policies/travel.docx …"
  ├─ turn 2  role: user       "And for international trips?"
  └─ turn 3  role: assistant  "…"
```

Storage is one point per turn (append-friendly, searchable per message), but the reference your app holds is always the conversation's ID — never a turn's.

## Ownership & safety

The collection (default **`app_conversations`**) is **owned by your app**, not by the worker. It lives outside the managed `od_*` namespace, so nothing vector-syncer ever does — sync, reconcile, rebuild, base-folder delete — can touch it. Your conversations survive any OneDrive reorganization or index rebuild.

## The helpers

Exported from the shared consumer package (working name `@vector-syncer/consumer`, ships in Phase 4) alongside the `bm25-v1` encoder:

| Helper | Does |
|---|---|
| `ensureConversationCollection(qdrant, name?)` | creates the collection + payload indexes if absent — idempotent, call at app startup |
| `appendTurn(conversationId, turnIndex, role, text)` | embeds (dense) + encodes (sparse) + upserts one turn — retries safe (deterministic point ID) |
| `getConversation(conversationId)` | the whole ordered transcript — the retrieval contract |
| `searchTurns(query, { conversationId?, userId? })` | hybrid RRF search over chat history, optionally scoped |
| `deleteConversation(conversationId)` | removes every turn of one conversation |

## Retrieval — one reference is enough

```ts
// what getConversation() does under the hood:
const { points } = await qdrant.scroll('app_conversations', {
  filter: { must: [{ key: 'conversation_id', match: { value: conversationId } }] },
  order_by: { key: 'turn_index' },     // ascending
  limit: 1024,
  with_payload: true,
});
// points[i].payload → { role: 'user' | 'assistant', text, created_at, … }
```

## Appending as the chat progresses

```ts
await appendTurn('abc-123', 4, 'user', 'What about hotel caps?');
await appendTurn('abc-123', 5, 'assistant', answerText);
```

Each append is one small upsert — the rest of the thread is never re-embedded. Point IDs are `uuidv5('conv:{conversationId}#{turnIndex}')`, so re-sending a turn overwrites instead of duplicating, and editing a turn is just an upsert to the same index.

## Memory search

Because turns carry the same dense+sparse vector pair as documents, your agent can recall past exchanges semantically:

```ts
const memories = await searchTurns('user asked about hotel budget limits', { userId: 'u-42' });
```

## Notes

- Turn `text` goes to the configured AI provider for embedding — the same data-sharing decision (D11) that covers document content applies to conversations.
- Multi-user apps: include `user_id` in the payload (indexed) and scope every read by it.
- The conversation store never appears in `od_catalog` — the catalog describes document collections only.
