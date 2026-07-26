# Graph Report - vector-syncer  (2026-07-27)

## Corpus Check
- 22 files · ~30,225 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 159 nodes · 222 edges · 11 communities
- Extraction: 96% EXTRACTED · 4% INFERRED · 0% AMBIGUOUS · INFERRED: 9 edges (avg confidence: 0.94)
- Token cost: 0 input · 0 output

## Graph Freshness
- Built from commit: `def03258`
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

## God Nodes (most connected - your core abstractions)
1. `P0+P1 Foundation Implementation Plan` - 20 edges
2. `06 — Qdrant design` - 11 edges
3. `vector-syncer — Design Specification` - 11 edges
4. `OneDrive Folder Sync Feature Guide` - 11 edges
5. `Operations Feature Guide` - 10 edges
6. `Conversation Store Feature Guide` - 9 edges
7. `05 — Conversion pipeline: MarkItDown, OCR, chunking, embeddings` - 7 edges
8. `08 — Configuration & deployment` - 7 edges
9. `04 — OneDrive / Microsoft Graph Integration` - 7 edges
10. `Feature Guides Index` - 7 edges

## Surprising Connections (you probably didn't know these)
- `MVP (End of Phase 4)` --cites--> `01 — Requirements`  [EXTRACTED]
  docs/design/09-roadmap.md → docs/design/01-requirements.md
- `P0+P1 Foundation Implementation Plan` --implements--> `Implementation Phases P0-P6`  [INFERRED]
  docs/superpowers/plans/2026-07-26-vector-syncer-p0-p1-foundation.md → docs/design/09-roadmap.md
- `walkDelta` --implements--> `Graph Delta Query Polling`  [INFERRED]
  docs/superpowers/plans/2026-07-26-vector-syncer-p0-p1-foundation.md → docs/features/onedrive-sync.md
- `AuthRequiredError` --conceptually_related_to--> `AUTH_REQUIRED Health State`  [INFERRED]
  docs/superpowers/plans/2026-07-26-vector-syncer-p0-p1-foundation.md → docs/features/operations.md
- `cliAuth (yarn auth)` --conceptually_related_to--> `AUTH_REQUIRED Health State`  [INFERRED]
  docs/superpowers/plans/2026-07-26-vector-syncer-p0-p1-foundation.md → docs/features/operations.md

## Import Cycles
- None detected.

## Communities (11 total, 0 thin omitted)

### Community 0 - "Sync Engine & Queue Semantics"
Cohesion: 0.19
Nodes (10): 10 — Decisions, open questions, risks, 1. Decision log (ADR-lite), 2. Open questions — **all answered** (as of 2026-07-26), 3. Risk register, 4. Standing review triggers, Feature guides (usage), Glossary, Planning documents (+2 more)

### Community 1 - "Feature Guides & Operations"
Cohesion: 0.15
Nodes (13): 06 — Qdrant design, 10. Consumer-owned collections: the conversation store (user addition, 2026-07-26), 1. Collection layout: one per base folder (decision D12), 2. Point IDs — deterministic, idempotent, 3. Payload schema (document collections), 4. Write paths, 5. Delete paths (FR-15 — OneDrive deletion ⇒ Qdrant deletion, always), 6. The catalog collection (`od_catalog`, decision D13) (+5 more)

### Community 2 - "Approved Spec Concepts"
Cohesion: 0.13
Nodes (27): 01 — Requirements, 02 — Architecture, 04 — OneDrive / Microsoft Graph Integration, 07 — BullMQ Design, 09 — Roadmap, Collection Discovery Feature Guide, od_catalog Collection Registry, Model and Provider Flexibility Feature Guide (+19 more)

### Community 3 - "P0+P1 Implementation Plan"
Cohesion: 0.21
Nodes (18): P0+P1 Foundation Implementation Plan, AuthRequiredError, classifyPage, cliAuth (yarn auth), createMsalApp, AES-256-GCM Cache Codec, DeltaGoneError, deltaState (+10 more)

### Community 4 - "Document Index"
Cohesion: 0.17
Nodes (12): 03 — Tech stack, 1. Summary, 2. The one structural decision: hybrid Node + Python, 3. Dependency lists (planned), 4. Version pinning policy, Also rejected, converter/pyproject.toml (runtime), Infrastructure images (+4 more)

### Community 5 - "Runtime Components & Stack"
Cohesion: 0.09
Nodes (29): Idempotency Principle, OneDrive as Source of Truth, At-Least-Once Delivery, Redis Sync State (vs:* keys), Worker (Node.js/TypeScript Orchestrator), Change-Notification Webhooks (v2), cTag Change Detection, Delegated Auth (Silent Refresh) (+21 more)

### Community 6 - "Conversion & OCR"
Cohesion: 0.17
Nodes (11): 10. Top risks (register: `docs/design/10-decisions-and-risks.md` §3), 1. Purpose, 2. Confirmed decisions (owner-validated 2026-07-26), 3. Architecture, 4. Key mechanisms, 5. Payload (what a point carries), 6. Error handling summary, 7. Testing strategy (+3 more)

### Community 7 - "Hybrid Search & Consumer Contract"
Cohesion: 0.22
Nodes (9): 05 — Conversion pipeline: MarkItDown, OCR, chunking, embeddings, 1. Converter service (Python sidecar), 2. What MarkItDown covers (v0.1.6, `markitdown[all]`), 3. OCR strategy (decision D8 — chosen: Qwen3-VL on DeepInfra), 4. Chunking (worker-side, TypeScript), 5. Embeddings — `BAAI/bge-m3` via DeepInfra (OpenAI SDK), 6. Per-file pipeline summary (inside one `process-file` job), API contract (+1 more)

### Community 8 - "Collection Layout & State"
Cohesion: 0.25
Nodes (6): graphify, Non-negotiable design constraints, Package managers & commands, Process rules, Repository state, What this builds

### Community 9 - "Conversation Store Feature Guide"
Cohesion: 0.50
Nodes (8): Conversation Store Feature Guide, app_conversations Collection, appendTurn, conversationId, deleteConversation, ensureConversationCollection, getConversation, searchTurns

### Community 10 - "Conversation Store"
Cohesion: 0.29
Nodes (7): 08 — Configuration & deployment, 1. Planned repository layout, 2. Environment variables, 3. docker-compose.yml (sketch), 4. Local development loop, 5. Operations runbook, 6. Deployment targets

## Knowledge Gaps
- **66 isolated node(s):** `Repository state`, `What this builds`, `Non-negotiable design constraints`, `Package managers & commands`, `Process rules` (+61 more)
  These have ≤1 connection - possible missing edges or undocumented components.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `P0+P1 Foundation Implementation Plan` connect `P0+P1 Implementation Plan` to `Approved Spec Concepts`, `Runtime Components & Stack`?**
  _High betweenness centrality (0.081) - this node is a cross-community bridge._
- **Why does `Feature Guides Index` connect `Approved Spec Concepts` to `Conversation Store Feature Guide`?**
  _High betweenness centrality (0.075) - this node is a cross-community bridge._
- **What connects `Repository state`, `What this builds`, `Non-negotiable design constraints` to the rest of the system?**
  _66 weakly-connected nodes found - possible documentation gaps or missing edges._
- **Should `Approved Spec Concepts` be split into smaller, more focused modules?**
  _Cohesion score 0.12535612535612536 - nodes in this community are weakly interconnected._
- **Should `Runtime Components & Stack` be split into smaller, more focused modules?**
  _Cohesion score 0.09113300492610837 - nodes in this community are weakly interconnected._