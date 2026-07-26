# Graph Report - .  (2026-07-26)

## Corpus Check
- Corpus is ~29,128 words - fits in a single context window. You may not need a graph.

## Summary
- 151 nodes · 299 edges · 12 communities
- Extraction: 92% EXTRACTED · 8% INFERRED · 0% AMBIGUOUS · INFERRED: 25 edges (avg confidence: 0.93)
- Token cost: 147,765 input · 0 output

## Community Hubs (Navigation)
- [[_COMMUNITY_Sync Engine & Queue Semantics|Sync Engine & Queue Semantics]]
- [[_COMMUNITY_Feature Guides & Operations|Feature Guides & Operations]]
- [[_COMMUNITY_Approved Spec Concepts|Approved Spec Concepts]]
- [[_COMMUNITY_P0+P1 Implementation Plan|P0+P1 Implementation Plan]]
- [[_COMMUNITY_Document Index|Document Index]]
- [[_COMMUNITY_Runtime Components & Stack|Runtime Components & Stack]]
- [[_COMMUNITY_Conversion & OCR|Conversion & OCR]]
- [[_COMMUNITY_Hybrid Search & Consumer Contract|Hybrid Search & Consumer Contract]]
- [[_COMMUNITY_Collection Layout & State|Collection Layout & State]]
- [[_COMMUNITY_AI Provider & Catalog|AI Provider & Catalog]]
- [[_COMMUNITY_Conversation Store|Conversation Store]]
- [[_COMMUNITY_Delegated Auth Bootstrap|Delegated Auth Bootstrap]]

## God Nodes (most connected - your core abstractions)
1. `P0+P1 Foundation Implementation Plan` - 23 edges
2. `vector-syncer Design Specification` - 19 edges
3. `06 — Qdrant Design` - 12 edges
4. `vector-syncer README` - 11 edges
5. `04 — OneDrive / Microsoft Graph Integration` - 11 edges
6. `10 — Decisions, Open Questions, Risks` - 11 edges
7. `Worker (Node.js/TypeScript Orchestrator)` - 11 edges
8. `OneDrive Folder Sync Feature Guide` - 11 edges
9. `Operations Feature Guide` - 11 edges
10. `01 — Requirements` - 10 edges

## Surprising Connections (you probably didn't know these)
- `vector-syncer System` --conceptually_related_to--> `Idempotency Principle`  [EXTRACTED]
  README.md → docs/design/01-requirements.md
- `vector-syncer System` --references--> `Worker (Node.js/TypeScript Orchestrator)`  [EXTRACTED]
  README.md → docs/design/02-architecture.md
- `vector-syncer System` --conceptually_related_to--> `Graph Delta Query`  [EXTRACTED]
  README.md → docs/design/04-onedrive-graph-integration.md
- `vector-syncer System` --references--> `Converter Service (Python FastAPI Sidecar)`  [EXTRACTED]
  README.md → docs/design/05-conversion-pipeline.md
- `vector-syncer System` --conceptually_related_to--> `od_catalog Catalog Collection`  [EXTRACTED]
  README.md → docs/design/06-qdrant-design.md

## Import Cycles
- None detected.

## Hyperedges (group relationships)
- **Per-File Processing Pipeline (download, convert, chunk, embed, encode, upsert)** — design_02_architecture_worker, design_04_onedrive_graph_integration_microsoft_graph, design_05_conversion_pipeline_converter_service, design_05_conversion_pipeline_heading_aware_chunking, design_05_conversion_pipeline_bge_m3, design_06_qdrant_design_bm25_v1_encoder, design_06_qdrant_design_qdrant [EXTRACTED 1.00]
- **Hybrid Dense+Sparse Search with RRF Fusion** — design_05_conversion_pipeline_bge_m3, design_06_qdrant_design_bm25_v1_encoder, design_06_qdrant_design_hybrid_rrf_search, design_06_qdrant_design_qdrant [EXTRACTED 1.00]
- **Docker Compose Stack (worker, converter, redis, qdrant)** — design_02_architecture_worker, design_05_conversion_pipeline_converter_service, design_07_bullmq_design_redis, design_06_qdrant_design_qdrant [EXTRACTED 1.00]
- **Conversation helpers ship in the shared consumer package** — features_searching_from_your_app_consumer_package, features_conversation_store_ensureconversationcollection, features_conversation_store_appendturn, features_conversation_store_getconversation, features_conversation_store_searchturns, features_conversation_store_deleteconversation [EXTRACTED 1.00]
- **P1 delta walk pipeline (walk, classify, act) with persistent state** — plans_2026_07_26_vector_syncer_p0_p1_foundation_resolveroot, plans_2026_07_26_vector_syncer_p0_p1_foundation_walkdelta, plans_2026_07_26_vector_syncer_p0_p1_foundation_classifypage, plans_2026_07_26_vector_syncer_p0_p1_foundation_deltastate, plans_2026_07_26_vector_syncer_p0_p1_foundation_foldermap, plans_2026_07_26_vector_syncer_p0_p1_foundation_filestate, plans_2026_07_26_vector_syncer_p0_p1_foundation_devdelta [EXTRACTED 1.00]
- **Encrypted MSAL device-code auth flow** — plans_2026_07_26_vector_syncer_p0_p1_foundation_crypto, plans_2026_07_26_vector_syncer_p0_p1_foundation_encryptedcacheplugin, plans_2026_07_26_vector_syncer_p0_p1_foundation_createmsalapp, plans_2026_07_26_vector_syncer_p0_p1_foundation_getgraphtoken, plans_2026_07_26_vector_syncer_p0_p1_foundation_authrequirederror, plans_2026_07_26_vector_syncer_p0_p1_foundation_cliauth [EXTRACTED 1.00]

## Communities (12 total, 0 thin omitted)

### Community 0 - "Sync Engine & Queue Semantics"
Cohesion: 0.11
Nodes (24): Idempotency Principle, At-Least-Once Delivery, Change-Notification Webhooks (v2), cTag Change Detection, Delta Item Classification, Graph Delta Query, 410 Gone Resync, Delete-Then-Upsert per File (+16 more)

### Community 1 - "Feature Guides & Operations"
Cohesion: 0.15
Nodes (21): Collection Discovery Feature Guide, od_catalog Collection Registry, Model and Provider Flexibility Feature Guide, Embedding Model Rebuild Cost, OpenAI-Compatible Provider Contract, OneDrive Folder Sync Feature Guide, OneDrive as Single Source of Truth, Weekly Reconcile (+13 more)

### Community 2 - "Approved Spec Concepts"
Cohesion: 0.14
Nodes (19): Base Folder to Collection Mapping (od_slug), Graph Delta Query Polling, Shared Consumer Package (@vector-syncer/consumer), Sparse-Only Degraded Mode, Docker Compose Stack, Converter /healthz Endpoint, vector-syncer Design Specification, bm25-v1 Sparse Encoder (+11 more)

### Community 3 - "P0+P1 Implementation Plan"
Cohesion: 0.23
Nodes (17): P0+P1 Foundation Implementation Plan, AuthRequiredError, classifyPage, cliAuth (yarn auth), createMsalApp, AES-256-GCM Cache Codec, DeltaGoneError, deltaState (+9 more)

### Community 4 - "Document Index"
Cohesion: 0.59
Nodes (12): 01 — Requirements, 02 — Architecture, 03 — Tech Stack, 04 — OneDrive / Microsoft Graph Integration, 05 — Conversion Pipeline, 06 — Qdrant Design, 07 — BullMQ Design, 08 — Configuration & Deployment (+4 more)

### Community 5 - "Runtime Components & Stack"
Cohesion: 0.31
Nodes (10): Worker (Node.js/TypeScript Orchestrator), Hybrid Node + Python Runtime, Converter Service (Python FastAPI Sidecar), Qdrant, BullMQ, Redis, docker-compose Stack, D10: Self-Hosted Qdrant in Compose (+2 more)

### Community 6 - "Conversion & OCR"
Cohesion: 0.20
Nodes (10): Azure Document Intelligence (Fallback OCR), Empty-Conversion Warning, MarkItDown, OCR Strategy Tiers, Qwen3-VL OCR Model, Bull Board Dashboard, /health Endpoint, D4: bge-m3 via DeepInfra (+2 more)

### Community 7 - "Hybrid Search & Consumer Contract"
Cohesion: 0.31
Nodes (9): Delegated Auth (Silent Refresh), Microsoft Graph API, bm25-v1 Sparse Encoder, Conversation Store (app_conversations), Hybrid RRF Search, Shared Consumer Package, D14: Hybrid From Day One, Risk Register (+1 more)

### Community 8 - "Collection Layout & State"
Cohesion: 0.25
Nodes (8): OneDrive as Source of Truth, Redis Sync State (vs:* keys), Alias-Flip Rebuild, Collection per Base Folder, Point Payload Schema, Operations Runbook, D12: One Collection per Base Folder, D5: Sync State in Redis

### Community 9 - "AI Provider & Catalog"
Cohesion: 0.32
Nodes (8): DeepInfra (AI Provider), DeepSeek-V4-Flash LLM, OpenAI SDK, BAAI/bge-m3 Embedding Model, Heading-Aware Chunking, od_catalog Catalog Collection, D11: One AI Provider (DeepInfra via OpenAI SDK), D13: Catalog Collection

### Community 10 - "Conversation Store"
Cohesion: 0.50
Nodes (8): Conversation Store Feature Guide, app_conversations Collection, appendTurn, conversationId, deleteConversation, ensureConversationCollection, getConversation, searchTurns

### Community 11 - "Delegated Auth Bootstrap"
Cohesion: 0.50
Nodes (5): Device Code Flow Bootstrap, Encrypted MSAL Token Cache, Entra App Registration, MSAL Node, D2: Delegated Auth via Device Code

## Knowledge Gaps
- **3 isolated node(s):** `Azure Document Intelligence (Fallback OCR)`, `Entra App Registration`, `Search Result Payload Contract`
  These have ≤1 connection - possible missing edges or undocumented components.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `01 — Requirements` connect `Document Index` to `Sync Engine & Queue Semantics`, `Feature Guides & Operations`, `Approved Spec Concepts`?**
  _High betweenness centrality (0.509) - this node is a cross-community bridge._
- **Why does `Implementation Phases P0-P6` connect `Sync Engine & Queue Semantics` to `Hybrid Search & Consumer Contract`?**
  _High betweenness centrality (0.504) - this node is a cross-community bridge._
- **Why does `MVP (End of Phase 4)` connect `Sync Engine & Queue Semantics` to `Document Index`?**
  _High betweenness centrality (0.503) - this node is a cross-community bridge._
- **What connects `Azure Document Intelligence (Fallback OCR)`, `Entra App Registration`, `Retry Matrix` to the rest of the system?**
  _13 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Sync Engine & Queue Semantics` be split into smaller, more focused modules?**
  _Cohesion score 0.10507246376811594 - nodes in this community are weakly interconnected._
- **Should `Approved Spec Concepts` be split into smaller, more focused modules?**
  _Cohesion score 0.14035087719298245 - nodes in this community are weakly interconnected._