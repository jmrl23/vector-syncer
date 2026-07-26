# Graph Report - vector-syncer  (2026-07-27)

## Corpus Check
- 22 files · ~30,134 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 164 nodes · 256 edges · 13 communities (12 shown, 1 thin omitted)
- Extraction: 91% EXTRACTED · 9% INFERRED · 0% AMBIGUOUS · INFERRED: 22 edges (avg confidence: 0.93)
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `0bb7ac41`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- Sync Engine & Queue Semantics
- Feature Guides & Operations
- Approved Spec Concepts
- P0+P1 Implementation Plan
- Document Index
- Runtime Components & Stack
- Conversion & OCR
- Hybrid Search & Consumer Contract
- Collection Layout & State
- Conversation Store Feature Guide
- Conversation Store
- Repository State
- 05 — Conversion pipeline: MarkItDown, OCR, chunking, embeddings

## God Nodes (most connected - your core abstractions)
1. `P0+P1 Foundation Implementation Plan` - 21 edges
2. `vector-syncer Design Specification` - 16 edges
3. `OneDrive Folder Sync Feature Guide` - 11 edges
4. `Conversation Store Feature Guide` - 10 edges
5. `Operations Feature Guide` - 10 edges
6. `Vector Syncer Worker` - 10 edges
7. `Searching From Your App Feature Guide` - 9 edges
8. `05 — Conversion pipeline: MarkItDown, OCR, chunking, embeddings` - 7 edges
9. `08 — Configuration & deployment` - 7 edges
10. `01 — Requirements` - 7 edges

## Surprising Connections (you probably didn't know these)
- `MVP (End of Phase 4)` --cites--> `01 — Requirements`  [EXTRACTED]
  docs/design/09-roadmap.md → docs/design/01-requirements.md
- `Graph Delta Query` --shares_data_with--> `Point Payload Schema`  [EXTRACTED]
  docs/design/04-onedrive-graph-integration.md → docs/design/06-qdrant-design.md
- `Job Deduplication (itemId:cTag)` --semantically_similar_to--> `Deterministic UUIDv5 Point IDs`  [INFERRED] [semantically similar]
  docs/design/07-bullmq-design.md → docs/design/06-qdrant-design.md
- `P0+P1 Foundation Implementation Plan` --implements--> `Implementation Phases P0-P6`  [INFERRED]
  docs/superpowers/plans/2026-07-26-vector-syncer-p0-p1-foundation.md → docs/design/09-roadmap.md
- `appendTurn` --conceptually_related_to--> `Idempotent Writes (UUIDv5 Point IDs)`  [INFERRED]
  docs/features/conversation-store.md → docs/superpowers/specs/2026-07-26-vector-syncer-design.md

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **Vector Syncer Core Components** — claude_md_onedrive_integration, claude_md_qdrant_collections, claude_md_embedding_model, claude_md_bm25_sparse_encoder, claude_md_redis_bullmq, claude_md_consistency_machinery, claude_md_security_secrets [EXTRACTED 1.00]
- **Document Hierarchy** — claude_md_spec_doc, claude_md_design_docs, claude_md_implementation_plans [EXTRACTED 1.00]

## Communities (13 total, 1 thin omitted)

### Community 0 - "Sync Engine & Queue Semantics"
Cohesion: 0.11
Nodes (17): 08 — Configuration & deployment, 1. Planned repository layout, 2. Environment variables, 3. docker-compose.yml (sketch), 4. Local development loop, 5. Operations runbook, 6. Deployment targets, 10 — Decisions, open questions, risks (+9 more)

### Community 1 - "Feature Guides & Operations"
Cohesion: 0.21
Nodes (12): Idempotency Principle, At-Least-Once Delivery, cTag Change Detection, Delta Item Classification, Delete-Then-Upsert per File, Deterministic UUIDv5 Point IDs, Runtime-Adjustable Files Concurrency, Job Deduplication (itemId:cTag) (+4 more)

### Community 2 - "Approved Spec Concepts"
Cohesion: 0.16
Nodes (24): 01 — Requirements, 02 — Architecture, 04 — OneDrive / Microsoft Graph Integration, 06 — Qdrant Design, 07 — BullMQ Design, 09 — Roadmap, Collection Discovery Feature Guide, od_catalog Collection Registry (+16 more)

### Community 3 - "P0+P1 Implementation Plan"
Cohesion: 0.19
Nodes (19): P0+P1 Foundation Implementation Plan, AuthRequiredError, classifyPage, cliAuth (yarn auth), createMsalApp, AES-256-GCM Cache Codec, DeltaGoneError, deltaState (+11 more)

### Community 4 - "Document Index"
Cohesion: 0.20
Nodes (14): Qdrant, Base Folder to Collection Mapping (od_slug), Searching From Your App Feature Guide, Shared Consumer Package (@vector-syncer/consumer), Search Result Payload Contract, Sparse-Only Degraded Mode, Converter /healthz Endpoint, vector-syncer Design Specification (+6 more)

### Community 5 - "Runtime Components & Stack"
Cohesion: 0.17
Nodes (16): Worker (Node.js/TypeScript Orchestrator), Change-Notification Webhooks (v2), Delegated Auth (Silent Refresh), Graph Delta Query, Device Code Flow Bootstrap, Encrypted MSAL Token Cache, Entra App Registration, Microsoft Graph API (+8 more)

### Community 6 - "Conversion & OCR"
Cohesion: 0.20
Nodes (12): OneDrive as Source of Truth, Redis Sync State (vs:* keys), Alias-Flip Rebuild, bm25-v1 Sparse Encoder, Collection per Base Folder, Conversation Store (app_conversations), Hybrid RRF Search, od_catalog Catalog Collection (+4 more)

### Community 7 - "Hybrid Search & Consumer Contract"
Cohesion: 0.18
Nodes (11): BM25 Sparse Encoder, Consistency Machinery, Converter Package, Embedding Model, Graphify Tool, OneDrive Integration, Qdrant Collections, Redis and BullMQ (+3 more)

### Community 8 - "Collection Layout & State"
Cohesion: 0.17
Nodes (12): 03 — Tech stack, 1. Summary, 2. The one structural decision: hybrid Node + Python, 3. Dependency lists (planned), 4. Version pinning policy, Also rejected, converter/pyproject.toml (runtime), Infrastructure images (+4 more)

### Community 9 - "Conversation Store Feature Guide"
Cohesion: 0.50
Nodes (8): Conversation Store Feature Guide, app_conversations Collection, appendTurn, conversationId, deleteConversation, ensureConversationCollection, getConversation, searchTurns

### Community 10 - "Conversation Store"
Cohesion: 0.50
Nodes (4): Design Documents, Implementation Plans, Process Rules, Spec Document

### Community 12 - "05 — Conversion pipeline: MarkItDown, OCR, chunking, embeddings"
Cohesion: 0.22
Nodes (9): 05 — Conversion pipeline: MarkItDown, OCR, chunking, embeddings, 1. Converter service (Python sidecar), 2. What MarkItDown covers (v0.1.6, `markitdown[all]`), 3. OCR strategy (decision D8 — chosen: Qwen3-VL on DeepInfra), 4. Chunking (worker-side, TypeScript), 5. Embeddings — `BAAI/bge-m3` via DeepInfra (OpenAI SDK), 6. Per-file pipeline summary (inside one `process-file` job), API contract (+1 more)

## Knowledge Gaps
- **45 isolated node(s):** `Planning documents`, `Feature guides (usage)`, `Glossary`, `Provenance`, `1. Summary` (+40 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **1 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `vector-syncer Design Specification` connect `Document Index` to `Approved Spec Concepts`, `P0+P1 Implementation Plan`, `Runtime Components & Stack`, `Conversion & OCR`?**
  _High betweenness centrality (0.140) - this node is a cross-community bridge._
- **Why does `P0+P1 Foundation Implementation Plan` connect `P0+P1 Implementation Plan` to `Approved Spec Concepts`, `Document Index`, `Conversion & OCR`?**
  _High betweenness centrality (0.105) - this node is a cross-community bridge._
- **Why does `Microsoft Graph API` connect `Runtime Components & Stack` to `P0+P1 Implementation Plan`, `Document Index`?**
  _High betweenness centrality (0.072) - this node is a cross-community bridge._
- **What connects `Planning documents`, `Feature guides (usage)`, `Glossary` to the rest of the system?**
  _45 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Sync Engine & Queue Semantics` be split into smaller, more focused modules?**
  _Cohesion score 0.10822510822510822 - nodes in this community are weakly interconnected._