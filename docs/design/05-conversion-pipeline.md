# 05 — Conversion pipeline: MarkItDown, OCR, chunking, embeddings

> Status: draft · Last updated: 2026-07-26

## 1. Converter service (Python sidecar)

Stateless FastAPI app wrapping MarkItDown. One endpoint that does one thing.

### API contract

```
POST /convert  (multipart/form-data)
  file:          binary            (required)
  filename:      string            (helps format detection)
  content_type:  string            (optional hint)
  ocr:           "auto" | "off"    (default "auto": use OCR engine if configured)

200 → {
  "markdown":    "…",              # result.text_content
  "title":       "…" | null,
  "converter":   "PdfConverter",   # which MarkItDown converter fired (diagnostics)
  "ocr_used":    false,
  "ocr_images":  0,                # images sent to the vision model for this file
  "warnings":    [],
  "elapsed_ms":  843
}
413 file too large        422 conversion failed (corrupt/unreadable)
415 unsupported format    500 unexpected (worker retries)

GET /healthz → { "status": "ok", "ocr_tier": "none|docintel|llm" }
```

### Implementation sketch

```python
from fastapi import FastAPI, UploadFile, Form, HTTPException
from markitdown import MarkItDown, StreamInfo
import io, os

app = FastAPI()

# Engines built once at startup — MarkItDown imports are heavy, keep them warm.
plain = MarkItDown(enable_plugins=False)
ocr = None
if os.getenv("OCR_LLM_MODEL"):                        # chosen: Qwen3-VL via DeepInfra (D8/D11)
    from openai import OpenAI
    ocr = MarkItDown(
        enable_plugins=True,                          # activates the markitdown-ocr plugin
        llm_client=OpenAI(),                          # reads OPENAI_BASE_URL / OPENAI_API_KEY (→ DeepInfra)
        llm_model=os.environ["OCR_LLM_MODEL"],        # Qwen/Qwen3-VL-235B-A22B-Instruct
    )
elif os.getenv("DOCINTEL_ENDPOINT"):                  # fallback tier: Azure Document Intelligence
    ocr = MarkItDown(docintel_endpoint=os.environ["DOCINTEL_ENDPOINT"])

@app.post("/convert")
async def convert(file: UploadFile, filename: str = Form(None), ocr_mode: str = Form("auto", alias="ocr")):
    engine = ocr if (ocr and ocr_mode != "off") else plain
    data = await file.read()
    try:
        result = engine.convert_stream(
            io.BytesIO(data),
            stream_info=StreamInfo(filename=filename or file.filename,
                                   mimetype=file.content_type),  # verify exact StreamInfo kwargs at impl.
        )
    except Exception as e:
        raise HTTPException(status_code=422, detail=f"{type(e).__name__}: {e}")
    return {"markdown": result.text_content, "title": getattr(result, "title", None),
            "ocr_used": engine is ocr, "warnings": [], "elapsed_ms": ...}
```

Runtime limits: request body cap = `MAX_FILE_SIZE_MB`; per-request timeout 120 s (worker aborts + retries); uvicorn with 2 workers so one giant PPTX doesn't block the stack.

## 2. What MarkItDown covers (v0.1.6, `markitdown[all]`)

Built-in converters: **PDF** (pdfminer/pdfplumber — digital text, no OCR), **DOCX** (mammoth), **PPTX**, **XLSX/XLS** (tables → Markdown tables), **CSV**, **HTML**, **EPUB**, **Outlook MSG**, **images** (metadata; description only with an LLM), **Jupyter**, **ZIP** (recurses), plain text, audio (transcription — excluded from our default allowlist). Format detection uses magika + filename/mimetype hints, hence passing `filename` through the API.

Practical fidelity notes (set expectations for retrieval quality):

- XLSX → per-sheet Markdown tables. Fine for reference tables, weak for huge numeric sheets — chunking keeps header rows attached (§4).
- PPTX → slide-by-slide text incl. notes; diagram/image content is recovered only through the OCR tier (on by default, §3).
- **Digital PDF ≠ scanned PDF.** pdfminer extracts embedded text only — a scanned PDF yields ~empty Markdown from the built-ins; the OCR tier (on by default, §3) is what reads it. The empty-conversion warning (§3) catches the case where even that produced nothing.

## 3. OCR strategy (decision D8 — chosen: Qwen3-VL on DeepInfra)

| Tier | Config | What it does | Cost | Status |
|---|---|---|---|---|
| **LLM vision (chosen)** | `OCR_LLM_MODEL=Qwen/Qwen3-VL-235B-A22B-Instruct` + DeepInfra key | `markitdown-ocr` plugin: images embedded in PDF/DOCX/PPTX/XLSX (and standalone image files) are read by the vision model — covers scanned pages, screenshots, diagrams | per-image tokens on DeepInfra (**verify pricing**) | ✅ v1 default (on) |
| off | `OCR_LLM_MODEL=` (empty) | MarkItDown built-ins only; scanned docs convert to ~nothing and trip the empty-conversion warning | free | opt-out / kill switch if costs ever surprise |
| Azure Document Intelligence | `DOCINTEL_ENDPOINT` | full layout OCR via `docintel_endpoint` — structured Markdown | ~$1.50 / 1k pages (*verify*) | fallback if Qwen3-VL underperforms on complex layouts |
| local | pre-processing | `ocrmypdf`/Tesseract pass before MarkItDown | free, CPU-heavy | fully-offline fallback |

**Empty-conversion warning** (worker-side): file is PDF or image AND `markdown.length / max(pageCount,1) < ~200 chars` ⇒ index nothing, log a warning, increment an `empty_conversion` counter on `/health`. That's the entire mechanism — a warning light, not a workflow. **No OCR budget machinery in v1** (user decision 2026-07-26: the corpus is mostly digital PDFs with minimal images, so OCR volume is naturally small and runs inline). If reality disagrees — the counter climbs or the first invoice surprises — the documented add-backs are a per-cycle image cap and a targeted re-process status, both small ([10](10-decisions-and-risks.md) risk register). Bulk redo after an OCR/config change is `resync --rebuild [--collection …]`. For volume visibility, the converter reports `ocr_images` per file and the worker aggregates an **`ocr_images_processed`** counter on `/health` — OCR spend is observable long before the invoice (user-approved addition, 2026-07-26).

## 4. Chunking (worker-side, TypeScript)

Converter returns whole-document Markdown; the worker chunks it. Defaults: **target 512 tokens, overlap 64, min 20** (env-tunable).

Algorithm (heading-aware, token-measured with the **embedding model's own tokenizer** — `@huggingface/transformers` `AutoTokenizer`, D15; `@langchain/textsplitters` recursive splitter with markdown separators as the base implementation):

1. Split at heading boundaries (`#`–`####`), maintaining the heading stack per section.
2. Pack adjacent small sibling sections into one chunk up to the token target; split oversized sections at paragraph → sentence boundaries with the 64-token overlap.
3. Keep Markdown tables and fenced code blocks intact when possible; when a table must split, repeat its header row in each part.
4. Discard/merge fragments under 20 tokens.
5. Prefix each chunk's *embedding input* with a breadcrumb line:

```
[Q3-report.docx › Financials › Revenue by region]
{chunk text}
```

The payload stores the raw `text` and `heading_path` separately, so the breadcrumb aids retrieval without polluting displayed text.

Token sizing is **exact, not a proxy** (D15, 2026-07-27 — supersedes the earlier js-tiktoken plan): the worker loads the embedding model's actual tokenizer via `@huggingface/transformers` (`AutoTokenizer.from_pretrained(env.EMBEDDING_TOKENIZER)`, default `Xenova/bge-m3` = bge-m3's XLM-RoBERTa sentencepiece vocabulary) and measures chunks with `tokenizer.encode(text).length`, so budgets are counted in the same tokens the embedder consumes. `EMBEDDING_TOKENIZER` travels with `EMBEDDING_MODEL`: swapping the embedding model swaps the tokenizer inside the same rebuild ([06](06-qdrant-design.md) §7). Tokenizer files download from the HF Hub once and cache locally (`env.cacheDir`); for offline/deterministic builds, vendor them and set `env.allowRemoteModels = false`. This is local tokenization code, not an AI call — D11's OpenAI-SDK-only rule is untouched.

## 5. Embeddings — `BAAI/bge-m3` via DeepInfra (OpenAI SDK)

All AI in this project rides one provider — **DeepInfra** today — through the **official OpenAI SDK only** (decision D11): embeddings here; OCR (§3) and catalog descriptions ([06](06-qdrant-design.md) §6) elsewhere. **Nothing is hardcoded**: the SDK's native env vars carry the endpoint and key (`OPENAI_BASE_URL=https://api.deepinfra.com/v1/openai`, `OPENAI_API_KEY=<deepinfra token>`), and every model ID is its own env var — any OpenAI-compatible provider slots in by changing config.

```ts
import OpenAI from 'openai';

const ai = new OpenAI();   // reads OPENAI_BASE_URL + OPENAI_API_KEY — the SDK's own env vars

const res = await ai.embeddings.create({
  model: env.EMBEDDING_MODEL,    // BAAI/bge-m3 — 1024-d dense, 8192-token input, 100+ languages
  input: chunkBatch,             // ≤ EMBEDDING_BATCH_SIZE texts per call
  encoding_format: 'float',
});
const vectors = res.data.map(d => d.embedding);
```

Behind the worker's provider interface, so model and endpoint stay config values:

```ts
interface Embedder {
  readonly model: string;        // 'BAAI/bge-m3'
  readonly dimensions: number;   // 1024 — asserted against every collection at startup
  embed(texts: string[]): Promise<number[][]>;
}
```

Positions taken (full reasoning in [10-decisions-and-risks.md](10-decisions-and-risks.md) D4/D11/D14):

| Concern | Position |
|---|---|
| **Sparse / hybrid** | **Hybrid ships in v1** (D14, user decision): the provider returns dense only, so the sparse half is a worker-local BM25-style encoder ([06](06-qdrant-design.md) §4) — no API call, language-agnostic, versioned `bm25-v1`. Queries fuse both sides with RRF |
| **Data residency** | Chunk text is sent to DeepInfra — an accepted trade (D11). Switch-back path: a self-hosted FlagEmbedding sidecar serving the same model (dense **and** sparse), evaluated and documented under D4 alternatives |
| **Throttling** | External API ⇒ DeepInfra 429/5xx flows through the same BullMQ manual rate-limit / backoff paths as Graph ([07](07-bullmq-design.md) §5) |
| **Cost** | Metered per token, but bge-m3 is among the cheapest embedding SKUs — a 10k-doc / ~20M-token full index lands in the cents-to-few-dollars range (**verify current DeepInfra pricing**). OCR remains the dominant AI cost (§3) |

Mechanics: batches of `EMBEDDING_BATCH_SIZE` (default 64) texts per call; `embed_model` recorded in every point payload; startup refuses to run if `dimensions` ≠ collection vector size ([06](06-qdrant-design.md) §7 covers model migration). The model ID is an initial preference — env-swappable — but finalize it **before** the first full backfill: an embedding-model change means re-indexing. Queries must be embedded with the **same model** — consumers call DeepInfra themselves or go through the future search API / MCP layer.

## 6. Per-file pipeline summary (inside one `process-file` job)

```
download → converter /convert (Qwen3-VL OCR inline as needed) → [near-empty? warn+skip]
        → chunk (512/64, breadcrumbs) → embed via DeepInfra bge-m3 (batched)
                                      → sparse-encode locally (bm25-v1)
        → Qdrant (collection = base folder): delete file_id → upsert hybrid points (dense+sparse) → update Redis state
```

The downloaded scratch file is deleted the moment the converter responds — success or failure (`finally`). Everything downstream re-derives from the Markdown, so a retry of the whole job at any point is safe and produces byte-identical point IDs.
