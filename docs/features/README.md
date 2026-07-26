# Feature guides

> Usage-oriented documentation: one guide per feature vector-syncer offers, written for the people and applications *using* the system (the design internals live in the repo-root docs `01`–`10-*.md`).
>
> **Status note:** these guides were written at design time — the code ships per [`09-roadmap.md`](../../09-roadmap.md). Snippets reflect the committed design and become runnable as their phase lands (search & consumer-package snippets: Phase 4).

| Guide | What it covers | For whom |
|---|---|---|
| [onedrive-sync.md](onedrive-sync.md) | The core feature: what gets mirrored from OneDrive, when, and where it lands | Owner / operator |
| [searching-from-your-app.md](searching-from-your-app.md) | **Consuming the vector database from another application** — hybrid search, filters, payload contract | App developers |
| [collection-discovery.md](collection-discovery.md) | The catalog: enumerating available collections and routing queries to the right one | App developers |
| [conversation-store.md](conversation-store.md) | Storing user↔agent conversations; retrieving a whole conversation by `conversationId` | App developers |
| [operations.md](operations.md) | Manual sync triggers, live concurrency, pause/resume, resync/rebuild, health monitoring, re-auth | Owner / operator |
| [model-and-provider-flexibility.md](model-and-provider-flexibility.md) | Swapping AI providers and models via env — what's free and what costs a rebuild | Owner / operator |
