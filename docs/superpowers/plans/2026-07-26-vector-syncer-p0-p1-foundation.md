# vector-syncer P0+P1 — Foundation: Scaffold, Auth, Delta Walker — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the repo (worker + converter + compose) and deliver a restart-safe OneDrive delta walker: `yarn dev:delta` prints classified changes (with resolved base folder per file) against the real drive, persisting the delta token between runs.

**Architecture:** TypeScript worker (config → encrypted MSAL device-code auth → Graph client → delta walker → pure classifier → Redis state), Python FastAPI converter skeleton (health only; `/convert` arrives in the P3 plan), Redis + Qdrant + both services in one compose stack. This is Plan 1 of a series — P2 (queues), P3 (conversion), P4 (indexing = MVP), P5 (hardening) get their own plans at each phase boundary, informed by this phase's spike findings.

**Tech Stack:** Node 24 / TypeScript 5 (strict), yarn 4, zod, pino, ioredis, @azure/msal-node, @microsoft/microsoft-graph-client, vitest, testcontainers; Python 3.12, uv, FastAPI, pytest; docker compose.

**Spec:** `docs/superpowers/specs/2026-07-26-vector-syncer-design.md` · Detail docs `01`–`10-*.md` in `docs/design/` (esp. `04-onedrive-graph-integration.md`, `08-configuration-and-deployment.md`).

## Global Constraints

- Delegated scopes exactly: `Files.Read offline_access User.Read` (env-overridable via `GRAPH_SCOPES`).
- MSAL token cache encrypted **AES-256-GCM**; key = `MSAL_CACHE_KEY`, 64 hex chars; cache file mode `0600`.
- Redis must run `--maxmemory-policy noeviction --appendonly yes` (BullMQ requirement, set now).
- Secrets never in code or logs; config only via env (zod-validated, fail-fast with the offending variable named).
- All ports loopback-only (`127.0.0.1:`) in compose.
- Worker package manager is **yarn 4** (`packageManager` field, `nodeLinker: node-modules`); converter uses **uv**.
- TypeScript `strict: true`; no `any` in exported signatures.
- TDD: every code task = failing test → minimal implementation → pass → commit. Commit after every task (conventional commits).
- Delta `$select` is exactly: `id,name,size,file,folder,deleted,cTag,eTag,parentReference,createdDateTime,createdBy,lastModifiedBy,lastModifiedDateTime,webUrl` with `$top=200`.
- `410 Gone` on delta = restart enumeration (fresh folder delta URL); never crash-loop.
- Auth failure surfaces as `AuthRequiredError` with the message telling the operator to run `yarn auth`.

## File Structure (locked in by this plan)

```
vector-syncer/
├── .gitignore  .env.example  docker-compose.yml
├── worker/
│   ├── package.json  tsconfig.json  vitest.config.ts  .yarnrc.yml  Dockerfile
│   ├── src/
│   │   ├── index.ts              # boot: env → logger → redis ping → idle (queues arrive in P2)
│   │   ├── config.ts             # zod schema + loadConfig()
│   │   ├── ops/logger.ts         # pino factory
│   │   ├── auth/crypto.ts        # encrypt()/decrypt() AES-256-GCM codec
│   │   ├── auth/encryptedCachePlugin.ts
│   │   ├── auth/msal.ts          # createMsalApp(), getGraphToken(), AuthRequiredError
│   │   ├── auth/cliAuth.ts       # `yarn auth` device-code bootstrap
│   │   ├── graph/client.ts       # SDK client factory + graphGet() + typed error mapping
│   │   ├── graph/types.ts        # DriveItem subset, DeltaPage
│   │   ├── graph/resolveRoot.ts  # ONEDRIVE_FOLDER_PATH → {driveId, folderId}
│   │   ├── graph/delta.ts        # walkDelta() pager
│   │   ├── graph/classify.ts     # pure classifier (delta items → typed actions)
│   │   ├── state/redis.ts        # ioredis factory
│   │   ├── state/deltaState.ts   # vs:deltaLink, vs:root
│   │   ├── state/folderMap.ts    # vs:folder:{id}
│   │   ├── state/fileState.ts    # vs:file:{itemId}
│   │   └── cli/devDelta.ts       # `yarn dev:delta` dry run
│   └── tests/                    # mirrors src/ (unit) + tests/int/ (testcontainers)
└── converter/
    ├── pyproject.toml  Dockerfile
    ├── app/main.py               # FastAPI /healthz (only)
    └── tests/test_healthz.py
```

---

### Task 1: Worker scaffold + config loader + logger

**Files:**
- Create: `.gitignore`, `worker/package.json`, `worker/.yarnrc.yml`, `worker/tsconfig.json`, `worker/vitest.config.ts`, `worker/src/config.ts`, `worker/src/ops/logger.ts`, `worker/src/index.ts`
- Test: `worker/tests/config.test.ts`

**Interfaces:**
- Produces: `loadConfig(env?: NodeJS.ProcessEnv): Config` (throws `Error` listing every invalid var); `Config` type with fields `AZURE_TENANT_ID, AZURE_CLIENT_ID, GRAPH_SCOPES, ONEDRIVE_FOLDER_PATH, MSAL_CACHE_PATH, MSAL_CACHE_KEY, REDIS_URL, LOG_LEVEL`; `createLogger(level: string): pino.Logger`.

- [ ] **Step 1: Scaffold files (no test yet — pure config)**

`.gitignore` (repo root):

```
node_modules/
dist/
.env
data/
__pycache__/
.venv/
.pytest_cache/
*.log
```

`worker/package.json`:

```json
{
  "name": "vector-syncer-worker",
  "private": true,
  "type": "module",
  "packageManager": "yarn@4.5.0",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "auth": "tsx src/auth/cliAuth.ts",
    "dev:delta": "tsx src/cli/devDelta.ts",
    "build": "tsc -p tsconfig.json",
    "test": "vitest run",
    "typecheck": "tsc --noEmit"
  }
}
```

`worker/.yarnrc.yml`:

```yaml
nodeLinker: node-modules
```

`worker/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "outDir": "dist",
    "rootDir": "src",
    "skipLibCheck": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"]
}
```

`worker/vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config';
export default defineConfig({ test: { include: ['tests/**/*.test.ts'], testTimeout: 30_000 } });
```

Run: `cd worker && corepack enable && yarn install && yarn add zod dotenv pino pino-pretty && yarn add -D typescript tsx vitest @types/node`

- [ ] **Step 2: Write the failing config test**

`worker/tests/config.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { loadConfig } from '../src/config.js';

const VALID = {
  AZURE_TENANT_ID: 't-id',
  AZURE_CLIENT_ID: 'c-id',
  ONEDRIVE_FOLDER_PATH: '/Documents/KnowledgeBase',
  MSAL_CACHE_KEY: 'a'.repeat(64),
};

describe('loadConfig', () => {
  it('parses a valid env and applies defaults', () => {
    const cfg = loadConfig(VALID as NodeJS.ProcessEnv);
    expect(cfg.GRAPH_SCOPES).toBe('Files.Read offline_access User.Read');
    expect(cfg.REDIS_URL).toBe('redis://localhost:6379');
    expect(cfg.MSAL_CACHE_PATH).toBe('./data/msal-cache.bin');
    expect(cfg.LOG_LEVEL).toBe('info');
  });

  it('fails fast naming the missing variable', () => {
    const { AZURE_CLIENT_ID: _omit, ...rest } = VALID;
    expect(() => loadConfig(rest as NodeJS.ProcessEnv)).toThrow(/AZURE_CLIENT_ID/);
  });

  it('rejects a MSAL_CACHE_KEY that is not 64 hex chars', () => {
    expect(() => loadConfig({ ...VALID, MSAL_CACHE_KEY: 'short' } as NodeJS.ProcessEnv))
      .toThrow(/MSAL_CACHE_KEY/);
  });

  it('rejects a folder path not starting with /', () => {
    expect(() => loadConfig({ ...VALID, ONEDRIVE_FOLDER_PATH: 'Documents' } as NodeJS.ProcessEnv))
      .toThrow(/ONEDRIVE_FOLDER_PATH/);
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd worker && yarn test`
Expected: FAIL — `Cannot find module '../src/config.js'`

- [ ] **Step 4: Implement config + logger + boot**

`worker/src/config.ts`:

```ts
import { z } from 'zod';

const EnvSchema = z.object({
  AZURE_TENANT_ID: z.string().min(1),
  AZURE_CLIENT_ID: z.string().min(1),
  GRAPH_SCOPES: z.string().default('Files.Read offline_access User.Read'),
  ONEDRIVE_FOLDER_PATH: z.string().startsWith('/', 'must start with /'),
  MSAL_CACHE_PATH: z.string().default('./data/msal-cache.bin'),
  MSAL_CACHE_KEY: z.string().regex(/^[0-9a-fA-F]{64}$/, 'must be 64 hex chars (openssl rand -hex 32)'),
  REDIS_URL: z.string().default('redis://localhost:6379'),
  LOG_LEVEL: z.enum(['fatal', 'error', 'warn', 'info', 'debug', 'trace']).default('info'),
});

export type Config = z.infer<typeof EnvSchema>;

export function loadConfig(env: NodeJS.ProcessEnv = process.env): Config {
  const parsed = EnvSchema.safeParse(env);
  if (!parsed.success) {
    const lines = parsed.error.issues.map(i => `  ${i.path.join('.')}: ${i.message}`);
    throw new Error(`Invalid environment:\n${lines.join('\n')}`);
  }
  return parsed.data;
}
```

`worker/src/ops/logger.ts`:

```ts
import { pino, type Logger as PinoLogger } from 'pino';

export type Logger = PinoLogger;

export function createLogger(level: string): Logger {
  const pretty = process.stdout.isTTY
    ? { transport: { target: 'pino-pretty', options: { translateTime: 'SYS:HH:MM:ss' } } }
    : {};
  return pino({ level, ...pretty });
}
```

`worker/src/index.ts`:

```ts
import 'dotenv/config';
import { loadConfig } from './config.js';
import { createLogger } from './ops/logger.js';
import { createRedis } from './state/redis.js';

const cfg = loadConfig();
const log = createLogger(cfg.LOG_LEVEL);
const redis = createRedis(cfg.REDIS_URL);

await redis.ping();
log.info('vector-syncer worker ready (no queue workers registered yet — P2)');

const keepalive = setInterval(() => {}, 60_000);
process.on('SIGTERM', async () => {
  clearInterval(keepalive);
  await redis.quit();
  log.info('shut down cleanly');
  process.exit(0);
});
```

`worker/src/state/redis.ts` (needed by index; full state layer is Task 8):

```ts
import { Redis } from 'ioredis';

export function createRedis(url: string): Redis {
  return new Redis(url, { maxRetriesPerRequest: null });
}
```

Run: `yarn add ioredis`

- [ ] **Step 5: Run tests + typecheck to verify pass**

Run: `cd worker && yarn test && yarn typecheck`
Expected: 4 tests PASS; no type errors.

- [ ] **Step 6: Commit**

```bash
git add .gitignore worker/
git commit -m "feat(worker): scaffold — yarn 4, strict TS, zod config loader, pino logger, boot"
```

---

### Task 2: Converter skeleton (FastAPI `/healthz`)

**Files:**
- Create: `converter/pyproject.toml`, `converter/app/__init__.py`, `converter/app/main.py`
- Test: `converter/tests/test_healthz.py`

**Interfaces:**
- Produces: `GET /healthz` → `{"status": "ok", "ocr_tier": "none" | "llm" | "docintel"}`. The P3 plan adds `POST /convert` to this same `app`.

- [ ] **Step 1: Scaffold project**

`converter/pyproject.toml`:

```toml
[project]
name = "vector-syncer-converter"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = ["fastapi>=0.115", "uvicorn[standard]>=0.30"]

[dependency-groups]
dev = ["pytest>=8", "httpx>=0.27"]
```

Run: `cd converter && uv sync && touch app/__init__.py`

- [ ] **Step 2: Write the failing test**

`converter/tests/test_healthz.py`:

```python
from fastapi.testclient import TestClient
from app.main import app

def test_healthz_ok_and_reports_ocr_tier_none_by_default(monkeypatch):
    monkeypatch.delenv("OCR_LLM_MODEL", raising=False)
    monkeypatch.delenv("DOCINTEL_ENDPOINT", raising=False)
    r = TestClient(app).get("/healthz")
    assert r.status_code == 200
    assert r.json() == {"status": "ok", "ocr_tier": "none"}

def test_healthz_reports_llm_tier_when_model_configured(monkeypatch):
    monkeypatch.setenv("OCR_LLM_MODEL", "Qwen/Qwen3-VL-235B-A22B-Instruct")
    r = TestClient(app).get("/healthz")
    assert r.json()["ocr_tier"] == "llm"
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd converter && uv run pytest -q`
Expected: FAIL — `ModuleNotFoundError: No module named 'app.main'`

- [ ] **Step 4: Implement**

`converter/app/main.py`:

```python
import os
from fastapi import FastAPI

app = FastAPI(title="vector-syncer converter")

def _ocr_tier() -> str:
    if os.getenv("OCR_LLM_MODEL"):
        return "llm"
    if os.getenv("DOCINTEL_ENDPOINT"):
        return "docintel"
    return "none"

@app.get("/healthz")
def healthz() -> dict:
    return {"status": "ok", "ocr_tier": _ocr_tier()}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd converter && uv run pytest -q`
Expected: 2 passed.

- [ ] **Step 6: Commit**

```bash
git add converter/
git commit -m "feat(converter): FastAPI skeleton with /healthz and OCR-tier reporting"
```

---

### Task 3: Compose stack + `.env.example` + Dockerfiles

**Files:**
- Create: `docker-compose.yml`, `.env.example`, `worker/Dockerfile`, `converter/Dockerfile`

**Interfaces:**
- Produces: `docker compose up -d` → healthy `redis` (noeviction+AOF), `qdrant` (loopback `:6333`), `converter` (healthcheck on `/healthz`), `worker` (boots, pings Redis, idles). Service DNS names `redis`, `qdrant`, `converter` are what later plans' env defaults point at.

- [ ] **Step 1: Write the files**

`docker-compose.yml`:

```yaml
name: vector-syncer
services:
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes", "--maxmemory-policy", "noeviction"]
    volumes: [redis_data:/data]
    healthcheck: { test: ["CMD", "redis-cli", "ping"], interval: 10s, retries: 5 }
    restart: unless-stopped

  qdrant:
    image: qdrant/qdrant:v1.15.1
    volumes: [qdrant_data:/qdrant/storage]
    ports: ["127.0.0.1:6333:6333"]
    restart: unless-stopped

  converter:
    build: ./converter
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY:-}
      - OPENAI_BASE_URL=${OPENAI_BASE_URL:-https://api.deepinfra.com/v1/openai}
      - OCR_LLM_MODEL=${OCR_LLM_MODEL:-}
    expose: ["8000"]
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request;urllib.request.urlopen('http://localhost:8000/healthz')"]
      interval: 30s
      retries: 3
    restart: unless-stopped

  worker:
    build: ./worker
    env_file: .env
    environment:
      - REDIS_URL=redis://redis:6379
    volumes:
      - ./data:/app/data
    depends_on:
      redis: { condition: service_healthy }
      converter: { condition: service_healthy }
      qdrant: { condition: service_started }
    ports: ["127.0.0.1:3001:3001"]
    restart: unless-stopped

volumes: { redis_data: {}, qdrant_data: {} }
```

`.env.example` (every var the system will use; later-phase vars commented with their phase):

```bash
# ── Azure / Graph (required) ─────────────────────────────
AZURE_TENANT_ID=
AZURE_CLIENT_ID=
GRAPH_SCOPES="Files.Read offline_access User.Read"
ONEDRIVE_FOLDER_PATH=/Documents/KnowledgeBase

# ── Auth cache (required) ────────────────────────────────
MSAL_CACHE_PATH=./data/msal-cache.bin
MSAL_CACHE_KEY=   # openssl rand -hex 32

# ── Infra ────────────────────────────────────────────────
REDIS_URL=redis://localhost:6379
LOG_LEVEL=info

# ── AI provider (used from Phase 3) ──────────────────────
OPENAI_API_KEY=   # DeepInfra token (or any OpenAI-compatible provider)
OPENAI_BASE_URL=https://api.deepinfra.com/v1/openai
OCR_LLM_MODEL=Qwen/Qwen3-VL-235B-A22B-Instruct

# ── Phase 4+ (documented ahead; unread until those plans) ─
# QDRANT_URL=http://localhost:6333
# QDRANT_COLLECTION_PREFIX=od_
# ROOT_FILES_COLLECTION=_root
# CATALOG_COLLECTION=od_catalog
# EMBEDDING_MODEL=BAAI/bge-m3
# EMBEDDING_DIMENSIONS=1024
# EMBEDDING_BATCH_SIZE=64
# SPARSE_MODE=bm25
# LLM_MODEL=deepseek-ai/DeepSeek-V4-Flash
# CHUNK_TOKENS=512
# CHUNK_OVERLAP_TOKENS=64
# SYNC_CRON=*/5 * * * *
# FILE_CONCURRENCY=4
# MAX_FILE_SIZE_MB=100
# CONVERTER_URL=http://localhost:8000
# SCRATCH_DIR=/tmp/vector-syncer
# BULL_BOARD_PORT=3001
```

`worker/Dockerfile`:

```dockerfile
FROM node:24-slim AS build
WORKDIR /app
RUN corepack enable
COPY package.json yarn.lock .yarnrc.yml ./
RUN yarn install --immutable
COPY tsconfig.json ./
COPY src ./src
RUN yarn build

FROM node:24-slim
WORKDIR /app
RUN corepack enable
COPY package.json yarn.lock .yarnrc.yml ./
RUN yarn workspaces focus --production 2>/dev/null || yarn install --immutable
COPY --from=build /app/dist ./dist
CMD ["node", "dist/index.js"]
```

`converter/Dockerfile`:

```dockerfile
FROM python:3.12-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock* ./
RUN uv sync --frozen --no-dev 2>/dev/null || uv sync --no-dev
COPY app ./app
CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

- [ ] **Step 2: Verify the stack boots**

Run:

```bash
cp .env.example .env   # fill AZURE_TENANT_ID, AZURE_CLIENT_ID, and: openssl rand -hex 32 → MSAL_CACHE_KEY
docker compose up -d --build
sleep 15 && docker compose ps && docker compose logs worker | tail -5
```

Expected: `redis` and `converter` healthy, `qdrant` running, worker log ends with `vector-syncer worker ready (no queue workers registered yet — P2)`.

If the worker image build trips on the yarn production-install line, replace that RUN with plain `RUN yarn install --immutable` (dev deps in the runtime image are acceptable for now) and note it for P5 slimming.

- [ ] **Step 3: Commit**

```bash
git add docker-compose.yml .env.example worker/Dockerfile converter/Dockerfile
git commit -m "feat: docker compose stack (redis noeviction+AOF, qdrant, converter, worker) + .env.example"
```

**P0 acceptance reached:** stack up and healthy; `yarn dev` boots and validates env; `yarn test` + `uv run pytest` green.

---

### Task 4: AES-256-GCM cache codec

**Files:**
- Create: `worker/src/auth/crypto.ts`
- Test: `worker/tests/auth/crypto.test.ts`

**Interfaces:**
- Produces: `encrypt(plaintext: Buffer, keyHex: string): Buffer` and `decrypt(blob: Buffer, keyHex: string): Buffer` (throws on tamper/wrong key). Blob layout: `iv(12) | authTag(16) | ciphertext`.

- [ ] **Step 1: Write the failing test**

`worker/tests/auth/crypto.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { encrypt, decrypt } from '../../src/auth/crypto.js';

const KEY = 'ab'.repeat(32);
const OTHER_KEY = 'cd'.repeat(32);

describe('cache codec', () => {
  it('round-trips', () => {
    const blob = encrypt(Buffer.from('token cache json'), KEY);
    expect(decrypt(blob, KEY).toString()).toBe('token cache json');
  });

  it('produces different ciphertext per call (random IV)', () => {
    const a = encrypt(Buffer.from('x'), KEY);
    const b = encrypt(Buffer.from('x'), KEY);
    expect(a.equals(b)).toBe(false);
  });

  it('throws on tampered ciphertext', () => {
    const blob = encrypt(Buffer.from('secret'), KEY);
    blob[blob.length - 1] ^= 0xff;
    expect(() => decrypt(blob, KEY)).toThrow();
  });

  it('throws on wrong key', () => {
    const blob = encrypt(Buffer.from('secret'), KEY);
    expect(() => decrypt(blob, OTHER_KEY)).toThrow();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd worker && yarn vitest run tests/auth/crypto.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

`worker/src/auth/crypto.ts`:

```ts
import { createCipheriv, createDecipheriv, randomBytes } from 'node:crypto';

const IV_LEN = 12;
const TAG_LEN = 16;

export function encrypt(plaintext: Buffer, keyHex: string): Buffer {
  const key = Buffer.from(keyHex, 'hex');
  const iv = randomBytes(IV_LEN);
  const cipher = createCipheriv('aes-256-gcm', key, iv);
  const ct = Buffer.concat([cipher.update(plaintext), cipher.final()]);
  return Buffer.concat([iv, cipher.getAuthTag(), ct]);
}

export function decrypt(blob: Buffer, keyHex: string): Buffer {
  const key = Buffer.from(keyHex, 'hex');
  const iv = blob.subarray(0, IV_LEN);
  const tag = blob.subarray(IV_LEN, IV_LEN + TAG_LEN);
  const ct = blob.subarray(IV_LEN + TAG_LEN);
  const decipher = createDecipheriv('aes-256-gcm', key, iv);
  decipher.setAuthTag(tag);
  return Buffer.concat([decipher.update(ct), decipher.final()]);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd worker && yarn vitest run tests/auth/crypto.test.ts`
Expected: 4 passed.

- [ ] **Step 5: Commit**

```bash
git add worker/src/auth/crypto.ts worker/tests/auth/crypto.test.ts
git commit -m "feat(auth): AES-256-GCM codec for the MSAL token cache"
```

---

### Task 5: Encrypted MSAL cache plugin

**Files:**
- Create: `worker/src/auth/encryptedCachePlugin.ts`
- Test: `worker/tests/auth/encryptedCachePlugin.test.ts`

**Interfaces:**
- Consumes: `encrypt`/`decrypt` (Task 4); `Logger` (Task 1).
- Produces: `encryptedCachePlugin(path: string, keyHex: string, log: Logger): ICachePlugin` — msal-node's `beforeCacheAccess`/`afterCacheAccess` contract. Corrupt/unreadable cache ⇒ warn + start empty (never throw — the empty cache naturally forces the `AuthRequiredError` path later).

- [ ] **Step 1: Install msal + write the failing test**

Run: `cd worker && yarn add @azure/msal-node`

`worker/tests/auth/encryptedCachePlugin.test.ts`:

```ts
import { describe, it, expect, vi } from 'vitest';
import { mkdtemp, readFile, writeFile } from 'node:fs/promises';
import { tmpdir } from 'node:os';
import { join } from 'node:path';
import type { TokenCacheContext } from '@azure/msal-node';
import { encryptedCachePlugin } from '../../src/auth/encryptedCachePlugin.js';
import { decrypt } from '../../src/auth/crypto.js';

const KEY = 'ab'.repeat(32);
const log = { warn: vi.fn() } as any;

function fakeCtx(serialized = '', hasChanged = false): TokenCacheContext {
  let state = serialized;
  return {
    cacheHasChanged: hasChanged,
    tokenCache: {
      serialize: () => state,
      deserialize: (s: string) => { state = s; },
    },
    get state() { return state; },
  } as any;
}

describe('encryptedCachePlugin', () => {
  it('writes an encrypted file after cache change, readable on next beforeCacheAccess', async () => {
    const dir = await mkdtemp(join(tmpdir(), 'msal-'));
    const path = join(dir, 'cache.bin');
    const plugin = encryptedCachePlugin(path, KEY, log);

    const writeCtx = fakeCtx('{"tokens":1}', true);
    await plugin.afterCacheAccess(writeCtx);
    expect(decrypt(await readFile(path), KEY).toString()).toBe('{"tokens":1}');

    const readCtx = fakeCtx('');
    await plugin.beforeCacheAccess(readCtx);
    expect((readCtx as any).state).toBe('{"tokens":1}');
  });

  it('missing file is a silent no-op; corrupt file warns and starts empty', async () => {
    const dir = await mkdtemp(join(tmpdir(), 'msal-'));
    const plugin = encryptedCachePlugin(join(dir, 'absent.bin'), KEY, log);
    await plugin.beforeCacheAccess(fakeCtx(''));           // no throw
    expect(log.warn).not.toHaveBeenCalled();

    const corrupt = join(dir, 'corrupt.bin');
    await writeFile(corrupt, Buffer.from('garbage'));
    const p2 = encryptedCachePlugin(corrupt, KEY, log);
    await p2.beforeCacheAccess(fakeCtx(''));               // no throw
    expect(log.warn).toHaveBeenCalled();
  });

  it('does not write when cacheHasChanged is false', async () => {
    const dir = await mkdtemp(join(tmpdir(), 'msal-'));
    const path = join(dir, 'cache.bin');
    const plugin = encryptedCachePlugin(path, KEY, log);
    await plugin.afterCacheAccess(fakeCtx('data', false));
    await expect(readFile(path)).rejects.toMatchObject({ code: 'ENOENT' });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd worker && yarn vitest run tests/auth/encryptedCachePlugin.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

`worker/src/auth/encryptedCachePlugin.ts`:

```ts
import { readFile, writeFile, mkdir } from 'node:fs/promises';
import { dirname } from 'node:path';
import type { ICachePlugin, TokenCacheContext } from '@azure/msal-node';
import { encrypt, decrypt } from './crypto.js';
import type { Logger } from '../ops/logger.js';

export function encryptedCachePlugin(path: string, keyHex: string, log: Logger): ICachePlugin {
  return {
    async beforeCacheAccess(ctx: TokenCacheContext): Promise<void> {
      try {
        const blob = await readFile(path);
        ctx.tokenCache.deserialize(decrypt(blob, keyHex).toString('utf8'));
      } catch (err: unknown) {
        if ((err as NodeJS.ErrnoException).code === 'ENOENT') return; // first run — empty cache
        log.warn({ err, path }, 'token cache unreadable — starting empty; re-auth will be required');
      }
    },
    async afterCacheAccess(ctx: TokenCacheContext): Promise<void> {
      if (!ctx.cacheHasChanged) return;
      await mkdir(dirname(path), { recursive: true });
      const blob = encrypt(Buffer.from(ctx.tokenCache.serialize(), 'utf8'), keyHex);
      await writeFile(path, blob, { mode: 0o600 });
    },
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd worker && yarn vitest run tests/auth/encryptedCachePlugin.test.ts`
Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add worker/src/auth/encryptedCachePlugin.ts worker/tests/auth/encryptedCachePlugin.test.ts
git commit -m "feat(auth): encrypted persistent MSAL cache plugin (warn + empty on corruption)"
```

---

### Task 6: MSAL app factory + silent token acquisition

**Files:**
- Create: `worker/src/auth/msal.ts`
- Test: `worker/tests/auth/msal.test.ts`

**Interfaces:**
- Consumes: `encryptedCachePlugin` (Task 5), `Config` (Task 1).
- Produces: `AuthRequiredError` (message contains ``run `yarn auth` ``); `createMsalApp(cfg: Config, log: Logger): PublicClientApplication`; `getGraphToken(app: TokenApp, scopes: string[]): Promise<string>` where `TokenApp` is the structural subset `{ getTokenCache(): { getAllAccounts(): Promise<AccountInfo[]> }; acquireTokenSilent(req): Promise<{ accessToken: string } | null> }` — structural typing is what makes this testable without mocking the msal module.

- [ ] **Step 1: Write the failing test**

`worker/tests/auth/msal.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { InteractionRequiredAuthError } from '@azure/msal-node';
import { getGraphToken, AuthRequiredError, type TokenApp } from '../../src/auth/msal.js';

const SCOPES = ['Files.Read'];
const account = { homeAccountId: 'h', username: 'u@x' } as any;

function appWith(overrides: Partial<TokenApp>): TokenApp {
  return {
    getTokenCache: () => ({ getAllAccounts: async () => [account] }),
    acquireTokenSilent: async () => ({ accessToken: 'tok-123' }),
    ...overrides,
  } as TokenApp;
}

describe('getGraphToken', () => {
  it('returns the access token on silent success', async () => {
    await expect(getGraphToken(appWith({}), SCOPES)).resolves.toBe('tok-123');
  });

  it('throws AuthRequiredError when no account is cached', async () => {
    const app = appWith({ getTokenCache: () => ({ getAllAccounts: async () => [] }) });
    await expect(getGraphToken(app, SCOPES)).rejects.toBeInstanceOf(AuthRequiredError);
  });

  it('maps InteractionRequiredAuthError to AuthRequiredError', async () => {
    const app = appWith({
      acquireTokenSilent: async () => { throw new InteractionRequiredAuthError('interaction_required'); },
    });
    await expect(getGraphToken(app, SCOPES)).rejects.toBeInstanceOf(AuthRequiredError);
  });

  it('lets unexpected errors escape unchanged', async () => {
    const boom = new Error('network down');
    const app = appWith({ acquireTokenSilent: async () => { throw boom; } });
    await expect(getGraphToken(app, SCOPES)).rejects.toBe(boom);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd worker && yarn vitest run tests/auth/msal.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

`worker/src/auth/msal.ts`:

```ts
import {
  PublicClientApplication,
  InteractionRequiredAuthError,
  type AccountInfo,
  type SilentFlowRequest,
} from '@azure/msal-node';
import type { Config } from '../config.js';
import type { Logger } from '../ops/logger.js';
import { encryptedCachePlugin } from './encryptedCachePlugin.js';

export class AuthRequiredError extends Error {
  constructor() {
    super('Authentication required — run `yarn auth` to sign in with a device code');
    this.name = 'AuthRequiredError';
  }
}

/** Structural subset of PublicClientApplication that token acquisition needs. */
export interface TokenApp {
  getTokenCache(): { getAllAccounts(): Promise<AccountInfo[]> };
  acquireTokenSilent(req: SilentFlowRequest): Promise<{ accessToken: string } | null>;
}

export function createMsalApp(cfg: Config, log: Logger): PublicClientApplication {
  return new PublicClientApplication({
    auth: {
      clientId: cfg.AZURE_CLIENT_ID,
      authority: `https://login.microsoftonline.com/${cfg.AZURE_TENANT_ID}`,
    },
    cache: { cachePlugin: encryptedCachePlugin(cfg.MSAL_CACHE_PATH, cfg.MSAL_CACHE_KEY, log) },
  });
}

export function scopesFrom(cfg: Config): string[] {
  return cfg.GRAPH_SCOPES.split(/\s+/).filter(Boolean);
}

export async function getGraphToken(app: TokenApp, scopes: string[]): Promise<string> {
  const [account] = await app.getTokenCache().getAllAccounts();
  if (!account) throw new AuthRequiredError();
  try {
    const result = await app.acquireTokenSilent({ account, scopes });
    if (!result?.accessToken) throw new AuthRequiredError();
    return result.accessToken;
  } catch (err) {
    if (err instanceof InteractionRequiredAuthError) throw new AuthRequiredError();
    throw err;
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd worker && yarn vitest run tests/auth/msal.test.ts`
Expected: 4 passed.

- [ ] **Step 5: Commit**

```bash
git add worker/src/auth/msal.ts worker/tests/auth/msal.test.ts
git commit -m "feat(auth): MSAL app factory + silent token flow with AuthRequiredError classification"
```

---

### Task 7: `yarn auth` device-code CLI (manual verification gate)

**Files:**
- Create: `worker/src/auth/cliAuth.ts`

**Interfaces:**
- Consumes: `createMsalApp`, `scopesFrom` (Task 6), `loadConfig`, `createLogger` (Task 1).
- Produces: the operator bootstrap command. After it succeeds once, `getGraphToken` works headlessly.

- [ ] **Step 1: Implement (thin composition — covered by the manual gate, not unit tests)**

`worker/src/auth/cliAuth.ts`:

```ts
import 'dotenv/config';
import { loadConfig } from '../config.js';
import { createLogger } from '../ops/logger.js';
import { createMsalApp, scopesFrom } from './msal.js';

const cfg = loadConfig();
const log = createLogger(cfg.LOG_LEVEL);
const app = createMsalApp(cfg, log);

const result = await app.acquireTokenByDeviceCode({
  scopes: scopesFrom(cfg),
  deviceCodeCallback: info => {
    console.log(`\n${info.message}\n`); // "To sign in, use a web browser to open https://microsoft.com/devicelogin and enter the code XXXX"
  },
});

if (!result?.account) {
  console.error('Sign-in did not complete.');
  process.exit(1);
}
console.log(`Signed in as ${result.account.username}.`);
console.log(`Encrypted token cache written to ${cfg.MSAL_CACHE_PATH} — silent refresh is now active.`);
```

- [ ] **Step 2: MANUAL GATE — real sign-in against the tenant**

Prerequisite (operator, one-time, Azure portal → Entra ID → App registrations):
1. New registration, single tenant, no redirect URI needed.
2. Authentication → Advanced settings → **Allow public client flows: Yes**.
3. API permissions: Microsoft Graph, **Delegated** — `Files.Read`, `offline_access`, `User.Read` (grant/consent per tenant policy).
4. Put Tenant ID and Client ID into `.env`.

Run: `cd worker && yarn auth` → open the printed URL, enter the code, sign in.
Expected: `Signed in as <you>.` and `data/msal-cache.bin` exists with mode `0600` (`ls -l ../data/`), content is binary (not JSON — `head -c 32 ../data/msal-cache.bin | xxd` shows no `{`).

Then verify silent refresh survives a process restart:

```bash
yarn tsx -e "
import 'dotenv/config';
import { loadConfig } from './src/config.js';
import { createLogger } from './src/ops/logger.js';
import { createMsalApp, scopesFrom, getGraphToken } from './src/auth/msal.js';
const cfg = loadConfig(); const app = createMsalApp(cfg, createLogger('warn'));
console.log('token acquired, length:', (await getGraphToken(app, scopesFrom(cfg))).length);
"
```

Expected: `token acquired, length: <~1500-2500>` with **no** interactive prompt.

- [ ] **Step 3: Commit**

```bash
git add worker/src/auth/cliAuth.ts
git commit -m "feat(auth): yarn auth device-code bootstrap CLI"
```

---

### Task 8: Redis state layer (deltaLink, root, folder map, file state)

**Files:**
- Create: `worker/src/state/deltaState.ts`, `worker/src/state/folderMap.ts`, `worker/src/state/fileState.ts`
- Test: `worker/tests/int/state.int.test.ts`

**Interfaces:**
- Consumes: `createRedis` (Task 1).
- Produces (all take an `ioredis` `Redis` instance):
  - `deltaState(r)`: `getDeltaLink(): Promise<string|null>` · `setDeltaLink(v: string)` · `clearDeltaLink()` · `getRoot(): Promise<{driveId: string; folderId: string} | null>` · `setRoot(root)`.
  - `folderMap(r)`: `upsert(id: string, e: {parentId: string|null; name: string})` · `get(id): Promise<{parentId: string|null; name: string} | null>` · `remove(id)`.
  - `fileState(r)`: `get(id): Promise<{cTag: string; baseFolderId: string|null} | null>` · `set(id, s)` · `remove(id)`.
  - Keys: `vs:deltaLink`, `vs:root`, `vs:folder:{id}`, `vs:file:{id}` (JSON values).

- [ ] **Step 1: Install testcontainers + write the failing integration test**

Run: `cd worker && yarn add -D testcontainers`

`worker/tests/int/state.int.test.ts`:

```ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { GenericContainer, type StartedTestContainer } from 'testcontainers';
import { Redis } from 'ioredis';
import { deltaState } from '../../src/state/deltaState.js';
import { folderMap } from '../../src/state/folderMap.js';
import { fileState } from '../../src/state/fileState.js';

let container: StartedTestContainer;
let redis: Redis;

beforeAll(async () => {
  container = await new GenericContainer('redis:7-alpine').withExposedPorts(6379).start();
  redis = new Redis(container.getMappedPort(6379), container.getHost());
}, 120_000);

afterAll(async () => { await redis?.quit(); await container?.stop(); });

describe('state layer', () => {
  it('deltaState round-trips link and root', async () => {
    const s = deltaState(redis);
    expect(await s.getDeltaLink()).toBeNull();
    await s.setDeltaLink('https://graph.example/delta?token=abc');
    expect(await s.getDeltaLink()).toBe('https://graph.example/delta?token=abc');
    await s.clearDeltaLink();
    expect(await s.getDeltaLink()).toBeNull();

    await s.setRoot({ driveId: 'd1', folderId: 'f1' });
    expect(await s.getRoot()).toEqual({ driveId: 'd1', folderId: 'f1' });
  });

  it('folderMap upserts, gets, removes', async () => {
    const m = folderMap(redis);
    await m.upsert('fid', { parentId: 'root', name: 'HR' });
    expect(await m.get('fid')).toEqual({ parentId: 'root', name: 'HR' });
    await m.remove('fid');
    expect(await m.get('fid')).toBeNull();
  });

  it('fileState round-trips', async () => {
    const f = fileState(redis);
    expect(await f.get('i1')).toBeNull();
    await f.set('i1', { cTag: 'c1', baseFolderId: 'b1' });
    expect(await f.get('i1')).toEqual({ cTag: 'c1', baseFolderId: 'b1' });
    await f.remove('i1');
    expect(await f.get('i1')).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd worker && yarn vitest run tests/int/state.int.test.ts`
Expected: FAIL — modules not found. (Requires local docker; the redis image pull may take a moment on first run.)

- [ ] **Step 3: Implement**

`worker/src/state/deltaState.ts`:

```ts
import type { Redis } from 'ioredis';

export interface RootRef { driveId: string; folderId: string }

export function deltaState(r: Redis) {
  return {
    getDeltaLink: () => r.get('vs:deltaLink'),
    setDeltaLink: (v: string) => r.set('vs:deltaLink', v),
    clearDeltaLink: () => r.del('vs:deltaLink'),
    getRoot: async (): Promise<RootRef | null> => {
      const v = await r.get('vs:root');
      return v ? (JSON.parse(v) as RootRef) : null;
    },
    setRoot: (root: RootRef) => r.set('vs:root', JSON.stringify(root)),
  };
}
export type DeltaState = ReturnType<typeof deltaState>;
```

`worker/src/state/folderMap.ts`:

```ts
import type { Redis } from 'ioredis';

export interface FolderEntry { parentId: string | null; name: string }

export function folderMap(r: Redis) {
  return {
    upsert: (id: string, e: FolderEntry) => r.set(`vs:folder:${id}`, JSON.stringify(e)),
    get: async (id: string): Promise<FolderEntry | null> => {
      const v = await r.get(`vs:folder:${id}`);
      return v ? (JSON.parse(v) as FolderEntry) : null;
    },
    remove: (id: string) => r.del(`vs:folder:${id}`),
  };
}
export type FolderMap = ReturnType<typeof folderMap>;
```

`worker/src/state/fileState.ts`:

```ts
import type { Redis } from 'ioredis';

export interface FileEntry { cTag: string; baseFolderId: string | null }

export function fileState(r: Redis) {
  return {
    get: async (id: string): Promise<FileEntry | null> => {
      const v = await r.get(`vs:file:${id}`);
      return v ? (JSON.parse(v) as FileEntry) : null;
    },
    set: (id: string, s: FileEntry) => r.set(`vs:file:${id}`, JSON.stringify(s)),
    remove: (id: string) => r.del(`vs:file:${id}`),
  };
}
export type FileState = ReturnType<typeof fileState>;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd worker && yarn vitest run tests/int/state.int.test.ts`
Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add worker/src/state/ worker/tests/int/state.int.test.ts
git commit -m "feat(state): Redis state layer — deltaLink/root, folder map, file state (testcontainers)"
```

---

### Task 9: Graph client wrapper with typed error mapping

**Files:**
- Create: `worker/src/graph/client.ts`, `worker/src/graph/types.ts`
- Test: `worker/tests/graph/client.test.ts`

**Interfaces:**
- Consumes: `getGraphToken` (Task 6).
- Produces:
  - `graph/types.ts`: `DriveItem` (subset: `id, name, size?, cTag?, eTag?, webUrl?, file?: {mimeType?: string; hashes?: {quickXorHash?: string; sha256Hash?: string}}, folder?: {childCount?: number}, deleted?: {state?: string}, parentReference?: {id?: string; driveId?: string; path?: string}, createdDateTime?, lastModifiedDateTime?, createdBy?: {user?: {displayName?: string}}, lastModifiedBy?: {user?: {displayName?: string}}`), `DeltaPage = { value: DriveItem[]; '@odata.nextLink'?: string; '@odata.deltaLink'?: string }`.
  - `client.ts`: `RateLimitedError { retryAfterMs: number }` · `DeltaGoneError { location: string | null }` · `mapGraphError(e: unknown): unknown` (pure — 429→RateLimitedError, 410→DeltaGoneError, else passthrough) · `createGraphClient(getToken: () => Promise<string>): Client` · `graphGet<T>(client: Client, pathOrUrl: string): Promise<T>`.

- [ ] **Step 1: Install the SDK + write the failing test (pure error mapper)**

Run: `cd worker && yarn add @microsoft/microsoft-graph-client && yarn add -D @microsoft/microsoft-graph-types`

`worker/tests/graph/client.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { mapGraphError, RateLimitedError, DeltaGoneError } from '../../src/graph/client.js';

function graphErr(statusCode: number, headers?: Record<string, string>) {
  return Object.assign(new Error(`status ${statusCode}`), { statusCode, headers });
}

describe('mapGraphError', () => {
  it('maps 429 with Retry-After seconds to RateLimitedError ms', () => {
    const e = mapGraphError(graphErr(429, { 'retry-after': '7' }));
    expect(e).toBeInstanceOf(RateLimitedError);
    expect((e as RateLimitedError).retryAfterMs).toBe(7000);
  });

  it('defaults Retry-After to 10s when the header is absent', () => {
    const e = mapGraphError(graphErr(429));
    expect((e as RateLimitedError).retryAfterMs).toBe(10_000);
  });

  it('maps 410 to DeltaGoneError carrying Location when present', () => {
    const e = mapGraphError(graphErr(410, { location: 'https://graph/…/delta?token=resync' }));
    expect(e).toBeInstanceOf(DeltaGoneError);
    expect((e as DeltaGoneError).location).toBe('https://graph/…/delta?token=resync');
  });

  it('maps 410 without headers to DeltaGoneError with null location', () => {
    expect((mapGraphError(graphErr(410)) as DeltaGoneError).location).toBeNull();
  });

  it('passes other errors through unchanged', () => {
    const boom = new Error('nope');
    expect(mapGraphError(boom)).toBe(boom);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd worker && yarn vitest run tests/graph/client.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

`worker/src/graph/types.ts`:

```ts
export interface DriveItem {
  id: string;
  name?: string;
  size?: number;
  cTag?: string;
  eTag?: string;
  webUrl?: string;
  file?: { mimeType?: string; hashes?: { quickXorHash?: string; sha256Hash?: string } };
  folder?: { childCount?: number };
  deleted?: { state?: string };
  parentReference?: { id?: string; driveId?: string; path?: string };
  createdDateTime?: string;
  lastModifiedDateTime?: string;
  createdBy?: { user?: { displayName?: string } };
  lastModifiedBy?: { user?: { displayName?: string } };
}

export interface DeltaPage {
  value: DriveItem[];
  '@odata.nextLink'?: string;
  '@odata.deltaLink'?: string;
}
```

`worker/src/graph/client.ts`:

```ts
import { Client } from '@microsoft/microsoft-graph-client';

export class RateLimitedError extends Error {
  constructor(public readonly retryAfterMs: number) {
    super(`rate limited — retry after ${retryAfterMs}ms`);
    this.name = 'RateLimitedError';
  }
}

export class DeltaGoneError extends Error {
  constructor(public readonly location: string | null) {
    super('delta token expired (410 Gone) — full re-enumeration required');
    this.name = 'DeltaGoneError';
  }
}

interface GraphErrorLike { statusCode?: number; headers?: Record<string, string> | { get?(k: string): string | null } }

function header(e: GraphErrorLike, name: string): string | null {
  const h = e.headers;
  if (!h) return null;
  if (typeof (h as { get?: unknown }).get === 'function') {
    return (h as { get(k: string): string | null }).get(name);
  }
  const rec = h as Record<string, string>;
  return rec[name] ?? rec[name.toLowerCase()] ?? null;
}

export function mapGraphError(e: unknown): unknown {
  const ge = e as GraphErrorLike;
  if (ge?.statusCode === 429) {
    const secs = Number(header(ge, 'retry-after'));
    return new RateLimitedError(Number.isFinite(secs) && secs > 0 ? secs * 1000 : 10_000);
  }
  if (ge?.statusCode === 410) {
    return new DeltaGoneError(header(ge, 'location'));
  }
  return e;
}

export function createGraphClient(getToken: () => Promise<string>): Client {
  return Client.init({
    defaultVersion: 'v1.0',
    authProvider: done => getToken().then(t => done(null, t), e => done(e as Error, null)),
  });
}

export async function graphGet<T>(client: Client, pathOrUrl: string): Promise<T> {
  try {
    return (await client.api(pathOrUrl).get()) as T;
  } catch (e) {
    throw mapGraphError(e);
  }
}
```

Note: Node 24's global `fetch` satisfies the SDK's fetch requirement. If `createGraphClient` ever complains about a missing fetch implementation at runtime, add `yarn add cross-fetch` and `import 'cross-fetch/polyfill';` as the first line of `client.ts` — then record which was needed in `docs/design/04-onedrive-graph-integration.md`.

- [ ] **Step 4: Run test + typecheck to verify pass**

Run: `cd worker && yarn vitest run tests/graph/client.test.ts && yarn typecheck`
Expected: 5 passed; no type errors.

- [ ] **Step 5: Commit**

```bash
git add worker/src/graph/client.ts worker/src/graph/types.ts worker/tests/graph/client.test.ts
git commit -m "feat(graph): SDK client factory + graphGet with typed 429/410 error mapping"
```

---

### Task 10: Root folder resolution

**Files:**
- Create: `worker/src/graph/resolveRoot.ts`
- Test: `worker/tests/graph/resolveRoot.test.ts`

**Interfaces:**
- Consumes: `DriveItem` (Task 9), `RootRef` (Task 8).
- Produces: `resolveRoot(get: <T>(pathOrUrl: string) => Promise<T>, folderPath: string): Promise<RootRef>` — Graph path-addressing (`/me/drive/root:/segment/segment`), URL-encodes each segment, throws with a clear message when the target is missing or not a folder. (Callers bind `get` to `graphGet(client, …)` — the injected-function seam is the test strategy for everything Graph-facing.)

- [ ] **Step 1: Write the failing test**

`worker/tests/graph/resolveRoot.test.ts`:

```ts
import { describe, it, expect, vi } from 'vitest';
import { resolveRoot } from '../../src/graph/resolveRoot.js';

const folderItem = { id: 'folder-1', folder: { childCount: 3 }, parentReference: { driveId: 'drive-9' } };

describe('resolveRoot', () => {
  it('addresses the path relative to /me/drive/root and returns ids', async () => {
    const get = vi.fn().mockResolvedValue(folderItem);
    const root = await resolveRoot(get, '/Documents/Knowledge Base');
    expect(get).toHaveBeenCalledWith('/me/drive/root:/Documents/Knowledge%20Base');
    expect(root).toEqual({ driveId: 'drive-9', folderId: 'folder-1' });
  });

  it('throws when the item is not a folder', async () => {
    const get = vi.fn().mockResolvedValue({ id: 'x', file: {}, parentReference: { driveId: 'd' } });
    await expect(resolveRoot(get, '/Documents/report.docx')).rejects.toThrow(/not a folder/);
  });

  it('throws a clear error when the drive id is missing', async () => {
    const get = vi.fn().mockResolvedValue({ id: 'x', folder: {}, parentReference: {} });
    await expect(resolveRoot(get, '/Docs')).rejects.toThrow(/driveId/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd worker && yarn vitest run tests/graph/resolveRoot.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

`worker/src/graph/resolveRoot.ts`:

```ts
import type { DriveItem } from './types.js';
import type { RootRef } from '../state/deltaState.js';

export async function resolveRoot(
  get: <T>(pathOrUrl: string) => Promise<T>,
  folderPath: string,
): Promise<RootRef> {
  const encoded = folderPath
    .split('/')
    .filter(Boolean)
    .map(encodeURIComponent)
    .join('/');
  const item = await get<DriveItem>(`/me/drive/root:/${encoded}`);
  if (!item.folder) {
    throw new Error(`ONEDRIVE_FOLDER_PATH "${folderPath}" exists but is not a folder`);
  }
  const driveId = item.parentReference?.driveId;
  if (!driveId) {
    throw new Error(`Graph returned no parentReference.driveId for "${folderPath}"`);
  }
  return { driveId, folderId: item.id };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd worker && yarn vitest run tests/graph/resolveRoot.test.ts`
Expected: 3 passed.

- [ ] **Step 5: Commit**

```bash
git add worker/src/graph/resolveRoot.ts worker/tests/graph/resolveRoot.test.ts
git commit -m "feat(graph): resolve ONEDRIVE_FOLDER_PATH to {driveId, folderId}"
```

---

### Task 11: Delta walker

**Files:**
- Create: `worker/src/graph/delta.ts`
- Test: `worker/tests/graph/delta.test.ts`

**Interfaces:**
- Consumes: `DeltaPage`, `DriveItem` (Task 9), `RootRef` (Task 8).
- Produces:
  - `DELTA_SELECT` (the exact Global-Constraints field list) and `initialDeltaUrl(root: RootRef): string` → `/drives/{driveId}/items/{folderId}/delta?$select=…&$top=200`.
  - `walkDelta(get: (url: string) => Promise<DeltaPage>, startUrl: string, onPage: (items: DriveItem[]) => Promise<void>): Promise<string>` — follows `@odata.nextLink` pages, invokes `onPage` per page, returns the final `@odata.deltaLink`. Errors (incl. `DeltaGoneError`) propagate to the caller, which owns restart policy.

- [ ] **Step 1: Write the failing test**

`worker/tests/graph/delta.test.ts`:

```ts
import { describe, it, expect, vi } from 'vitest';
import { walkDelta, initialDeltaUrl, DELTA_SELECT } from '../../src/graph/delta.js';
import type { DeltaPage } from '../../src/graph/types.js';

const item = (id: string) => ({ id });

describe('delta walker', () => {
  it('builds the initial URL with the exact $select and $top', () => {
    const url = initialDeltaUrl({ driveId: 'd1', folderId: 'f1' });
    expect(url).toBe(`/drives/d1/items/f1/delta?$select=${DELTA_SELECT}&$top=200`);
  });

  it('follows nextLink pages in order and returns the deltaLink', async () => {
    const pages: Record<string, DeltaPage> = {
      start: { value: [item('a'), item('b')], '@odata.nextLink': 'page2' },
      page2: { value: [item('c')], '@odata.deltaLink': 'https://delta/final' },
    };
    const get = vi.fn(async (url: string) => pages[url]);
    const seen: string[][] = [];
    const link = await walkDelta(get, 'start', async items => { seen.push(items.map(i => i.id)); });
    expect(seen).toEqual([['a', 'b'], ['c']]);
    expect(link).toBe('https://delta/final');
  });

  it('handles a page with no value array', async () => {
    const get = vi.fn(async () => ({ '@odata.deltaLink': 'dl' }) as DeltaPage);
    const link = await walkDelta(get, 'start', async () => {});
    expect(link).toBe('dl');
  });

  it('throws on a page with neither nextLink nor deltaLink', async () => {
    const get = vi.fn(async () => ({ value: [] }) as DeltaPage);
    await expect(walkDelta(get, 'start', async () => {})).rejects.toThrow(/neither/);
  });

  it('propagates errors from get (e.g. DeltaGoneError) to the caller', async () => {
    const boom = new Error('410');
    const get = vi.fn(async () => { throw boom; });
    await expect(walkDelta(get, 'start', async () => {})).rejects.toBe(boom);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd worker && yarn vitest run tests/graph/delta.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

`worker/src/graph/delta.ts`:

```ts
import type { DeltaPage, DriveItem } from './types.js';
import type { RootRef } from '../state/deltaState.js';

export const DELTA_SELECT =
  'id,name,size,file,folder,deleted,cTag,eTag,parentReference,createdDateTime,createdBy,lastModifiedBy,lastModifiedDateTime,webUrl';

export function initialDeltaUrl(root: RootRef): string {
  return `/drives/${root.driveId}/items/${root.folderId}/delta?$select=${DELTA_SELECT}&$top=200`;
}

export async function walkDelta(
  get: (url: string) => Promise<DeltaPage>,
  startUrl: string,
  onPage: (items: DriveItem[]) => Promise<void>,
): Promise<string> {
  let url = startUrl;
  for (;;) {
    const page = await get(url);
    await onPage(page.value ?? []);
    const deltaLink = page['@odata.deltaLink'];
    if (deltaLink) return deltaLink;
    const nextLink = page['@odata.nextLink'];
    if (!nextLink) throw new Error('delta page has neither nextLink nor deltaLink');
    url = nextLink;
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd worker && yarn vitest run tests/graph/delta.test.ts`
Expected: 5 passed.

- [ ] **Step 5: Commit**

```bash
git add worker/src/graph/delta.ts worker/tests/graph/delta.test.ts
git commit -m 'feat(graph): delta walker — paging, exact $select, deltaLink return'
```

---

### Task 12: Change classifier

**Files:**
- Create: `worker/src/graph/classify.ts`
- Test: `worker/tests/graph/classify.test.ts`

**Interfaces:**
- Consumes: `DriveItem` (Task 9), `FolderEntry` (Task 8), `FileEntry` (Task 8).
- Produces:

```ts
type Classified =
  | { action: 'process-file'; item: DriveItem; baseFolderId: string | null; reason: 'new' | 'content-changed' | 're-route'; previousBaseFolderId: string | null }
  | { action: 'update-metadata'; item: DriveItem; baseFolderId: string | null }
  | { action: 'delete-file'; itemId: string }
  | { action: 'delete-folder'; itemId: string }
  | { action: 'skip'; itemId: string; reason: 'root-folder' | 'folder-entry' | 'unresolvable' };

interface ClassifyCtx {
  rootFolderId: string;
  getFolder(id: string): Promise<FolderEntry | null>;
  upsertFolder(id: string, e: FolderEntry): Promise<void>;
  removeFolder(id: string): Promise<void>;
  getPrevFile(id: string): Promise<FileEntry | null>;
  fetchItem?(id: string): Promise<DriveItem | null>;  // fallback for folder-map misses
}

classifyPage(items: DriveItem[], ctx: ClassifyCtx): Promise<Classified[]>
```

  Rules (the classification table from `docs/design/02-architecture.md` §3): the root folder itself → skip; folder entries → folder-map upsert (delete → `delete-folder` + map removal); deleted files → `delete-file`; files resolve their **base folder** = the ancestor whose parent is the root (`null` when the file sits directly in the root). New file or changed `cTag` → `process-file` (`new`/`content-changed`); same `cTag`, same base folder → `update-metadata`; same `cTag`, different base folder → `process-file` (`re-route`, carrying `previousBaseFolderId`). Folders are upserted **before** files within a page (delta pages can interleave). Unresolvable ancestry after the `fetchItem` fallback → `skip`/`unresolvable` (counted, never crashing).

- [ ] **Step 1: Write the failing test**

`worker/tests/graph/classify.test.ts`:

```ts
import { describe, it, expect } from 'vitest';
import { classifyPage, type ClassifyCtx } from '../../src/graph/classify.js';
import type { DriveItem } from '../../src/graph/types.js';
import type { FolderEntry } from '../../src/state/folderMap.js';
import type { FileEntry } from '../../src/state/fileState.js';

const ROOT = 'root-1';

function ctxWith(init: {
  folders?: Record<string, FolderEntry>;
  files?: Record<string, FileEntry>;
  fetch?: Record<string, DriveItem>;
} = {}): ClassifyCtx & { folders: Map<string, FolderEntry> } {
  const folders = new Map(Object.entries(init.folders ?? {}));
  const files = new Map(Object.entries(init.files ?? {}));
  return {
    rootFolderId: ROOT,
    folders,
    getFolder: async id => folders.get(id) ?? null,
    upsertFolder: async (id, e) => { folders.set(id, e); },
    removeFolder: async id => { folders.delete(id); },
    getPrevFile: async id => files.get(id) ?? null,
    fetchItem: async id => init.fetch?.[id] ?? null,
  };
}

const rootFolder: DriveItem = { id: ROOT, folder: {}, name: 'KnowledgeBase' };
const hrFolder: DriveItem = { id: 'hr', folder: {}, name: 'HR', parentReference: { id: ROOT } };
const policies: DriveItem = { id: 'pol', folder: {}, name: 'policies', parentReference: { id: 'hr' } };
const file = (id: string, parent: string, cTag: string): DriveItem =>
  ({ id, name: `${id}.docx`, cTag, file: { mimeType: 'application/x' }, parentReference: { id: parent } });

describe('classifyPage', () => {
  it('skips the root folder and records other folders in the map', async () => {
    const ctx = ctxWith();
    const out = await classifyPage([rootFolder, hrFolder, policies], ctx);
    expect(out).toEqual([
      { action: 'skip', itemId: ROOT, reason: 'root-folder' },
      { action: 'skip', itemId: 'hr', reason: 'folder-entry' },
      { action: 'skip', itemId: 'pol', reason: 'folder-entry' },
    ]);
    expect(ctx.folders.get('hr')).toEqual({ parentId: ROOT, name: 'HR' });
  });

  it('classifies a new file, resolving its base folder through nested ancestors', async () => {
    const ctx = ctxWith();
    const out = await classifyPage([hrFolder, policies, file('f1', 'pol', 'c1')], ctx);
    expect(out[2]).toEqual({
      action: 'process-file', item: file('f1', 'pol', 'c1'),
      baseFolderId: 'hr', reason: 'new', previousBaseFolderId: null,
    });
  });

  it('a file directly in the root gets baseFolderId null', async () => {
    const out = await classifyPage([file('loose', ROOT, 'c1')], ctxWith());
    expect(out[0]).toMatchObject({ action: 'process-file', baseFolderId: null, reason: 'new' });
  });

  it('changed cTag → content-changed; same cTag same base → update-metadata', async () => {
    const ctx = ctxWith({
      folders: { hr: { parentId: ROOT, name: 'HR' } },
      files: { f1: { cTag: 'old', baseFolderId: 'hr' }, f2: { cTag: 'same', baseFolderId: 'hr' } },
    });
    const out = await classifyPage([file('f1', 'hr', 'NEW'), file('f2', 'hr', 'same')], ctx);
    expect(out[0]).toMatchObject({ action: 'process-file', reason: 'content-changed', previousBaseFolderId: 'hr' });
    expect(out[1]).toMatchObject({ action: 'update-metadata', baseFolderId: 'hr' });
  });

  it('same cTag but different base folder → re-route with previousBaseFolderId', async () => {
    const ctx = ctxWith({
      folders: { hr: { parentId: ROOT, name: 'HR' }, eng: { parentId: ROOT, name: 'Engineering' } },
      files: { f1: { cTag: 'same', baseFolderId: 'hr' } },
    });
    const out = await classifyPage([file('f1', 'eng', 'same')], ctx);
    expect(out[0]).toMatchObject({ action: 'process-file', reason: 're-route', baseFolderId: 'eng', previousBaseFolderId: 'hr' });
  });

  it('deleted file → delete-file; deleted folder (known in map) → delete-folder + map removal', async () => {
    const ctx = ctxWith({ folders: { hr: { parentId: ROOT, name: 'HR' } } });
    const out = await classifyPage(
      [{ id: 'f9', deleted: { state: 'deleted' } }, { id: 'hr', deleted: { state: 'deleted' } }],
      ctx,
    );
    expect(out).toEqual([
      { action: 'delete-file', itemId: 'f9' },
      { action: 'delete-folder', itemId: 'hr' },
    ]);
    expect(ctx.folders.has('hr')).toBe(false);
  });

  it('folder-map miss uses fetchItem fallback; still-missing ancestry → skip unresolvable', async () => {
    const orphan: DriveItem = { id: 'mystery', folder: {}, name: 'Mystery', parentReference: { id: ROOT } };
    const ctx = ctxWith({ fetch: { unknown: orphan } });
    const out = await classifyPage([file('f1', 'unknown', 'c1'), file('f2', 'gone', 'c1')], ctx);
    expect(out[0]).toMatchObject({ action: 'process-file', baseFolderId: 'unknown' });
    expect(out[1]).toEqual({ action: 'skip', itemId: 'f2', reason: 'unresolvable' });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd worker && yarn vitest run tests/graph/classify.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

`worker/src/graph/classify.ts`:

```ts
import type { DriveItem } from './types.js';
import type { FolderEntry } from '../state/folderMap.js';
import type { FileEntry } from '../state/fileState.js';

export type Classified =
  | { action: 'process-file'; item: DriveItem; baseFolderId: string | null; reason: 'new' | 'content-changed' | 're-route'; previousBaseFolderId: string | null }
  | { action: 'update-metadata'; item: DriveItem; baseFolderId: string | null }
  | { action: 'delete-file'; itemId: string }
  | { action: 'delete-folder'; itemId: string }
  | { action: 'skip'; itemId: string; reason: 'root-folder' | 'folder-entry' | 'unresolvable' };

export interface ClassifyCtx {
  rootFolderId: string;
  getFolder(id: string): Promise<FolderEntry | null>;
  upsertFolder(id: string, e: FolderEntry): Promise<void>;
  removeFolder(id: string): Promise<void>;
  getPrevFile(id: string): Promise<FileEntry | null>;
  fetchItem?(id: string): Promise<DriveItem | null>;
}

/** Walk parent chain to the entry whose parent IS the root; null = directly in root; undefined = unresolvable. */
async function resolveBaseFolder(parentId: string | undefined, ctx: ClassifyCtx): Promise<string | null | undefined> {
  if (!parentId) return undefined;
  if (parentId === ctx.rootFolderId) return null;

  let current = parentId;
  const guard = new Set<string>();
  for (;;) {
    if (guard.has(current)) return undefined;      // cycle safety
    guard.add(current);
    let entry = await ctx.getFolder(current);
    if (!entry && ctx.fetchItem) {
      const fetched = await ctx.fetchItem(current);
      if (fetched?.folder && fetched.parentReference?.id) {
        entry = { parentId: fetched.parentReference.id, name: fetched.name ?? '' };
        await ctx.upsertFolder(current, entry);
      }
    }
    if (!entry || entry.parentId === null) return undefined;
    if (entry.parentId === ctx.rootFolderId) return current;
    current = entry.parentId;
  }
}

export async function classifyPage(items: DriveItem[], ctx: ClassifyCtx): Promise<Classified[]> {
  // Pass 1 — folders first, so files later in the same page can resolve ancestry.
  for (const it of items) {
    if (it.deleted) continue;
    if (it.folder && it.id !== ctx.rootFolderId && it.parentReference?.id) {
      await ctx.upsertFolder(it.id, { parentId: it.parentReference.id, name: it.name ?? '' });
    }
  }

  const out: Classified[] = [];
  for (const it of items) {
    if (it.deleted) {
      if (await ctx.getFolder(it.id)) {
        await ctx.removeFolder(it.id);
        out.push({ action: 'delete-folder', itemId: it.id });
      } else {
        out.push({ action: 'delete-file', itemId: it.id });
      }
      continue;
    }
    if (it.id === ctx.rootFolderId) {
      out.push({ action: 'skip', itemId: it.id, reason: 'root-folder' });
      continue;
    }
    if (it.folder) {
      out.push({ action: 'skip', itemId: it.id, reason: 'folder-entry' });
      continue;
    }

    const base = await resolveBaseFolder(it.parentReference?.id, ctx);
    if (base === undefined) {
      out.push({ action: 'skip', itemId: it.id, reason: 'unresolvable' });
      continue;
    }

    const prev = await ctx.getPrevFile(it.id);
    if (!prev) {
      out.push({ action: 'process-file', item: it, baseFolderId: base, reason: 'new', previousBaseFolderId: null });
    } else if (prev.cTag !== it.cTag) {
      out.push({ action: 'process-file', item: it, baseFolderId: base, reason: 'content-changed', previousBaseFolderId: prev.baseFolderId });
    } else if (prev.baseFolderId !== base) {
      out.push({ action: 'process-file', item: it, baseFolderId: base, reason: 're-route', previousBaseFolderId: prev.baseFolderId });
    } else {
      out.push({ action: 'update-metadata', item: it, baseFolderId: base });
    }
  }
  return out;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd worker && yarn vitest run tests/graph/classify.test.ts`
Expected: 7 passed.

- [ ] **Step 5: Commit**

```bash
git add worker/src/graph/classify.ts worker/tests/graph/classify.test.ts
git commit -m "feat(graph): pure change classifier — base-folder routing, re-route, deletes, unresolvable guard"
```

---

### Task 13: `yarn dev:delta` dry run

**Files:**
- Create: `worker/src/cli/devDelta.ts`

**Interfaces:**
- Consumes: everything from Tasks 1–12. Composition only — logic already unit-tested; verification is the manual acceptance run below.
- Produces: the P1 deliverable — classified change listing, deltaLink persistence, `--reset` flag. P2's `sync` job will reuse exactly this composition (walk → classify → act), swapping "print" for "enqueue".

- [ ] **Step 1: Implement**

`worker/src/cli/devDelta.ts`:

```ts
import 'dotenv/config';
import { loadConfig } from '../config.js';
import { createLogger } from '../ops/logger.js';
import { createRedis } from '../state/redis.js';
import { deltaState } from '../state/deltaState.js';
import { folderMap } from '../state/folderMap.js';
import { fileState } from '../state/fileState.js';
import { createMsalApp, getGraphToken, scopesFrom } from '../auth/msal.js';
import { createGraphClient, graphGet, DeltaGoneError } from '../graph/client.js';
import { resolveRoot } from '../graph/resolveRoot.js';
import { walkDelta, initialDeltaUrl } from '../graph/delta.js';
import { classifyPage, type Classified } from '../graph/classify.js';
import type { DeltaPage, DriveItem } from '../graph/types.js';

const cfg = loadConfig();
const log = createLogger(cfg.LOG_LEVEL);
const redis = createRedis(cfg.REDIS_URL);
const app = createMsalApp(cfg, log);
const client = createGraphClient(() => getGraphToken(app, scopesFrom(cfg)));
const get = <T,>(url: string) => graphGet<T>(client, url);

const ds = deltaState(redis);
const fm = folderMap(redis);
const fs = fileState(redis);

if (process.argv.includes('--reset')) {
  await ds.clearDeltaLink();
  log.info('deltaLink cleared — next walk is a full enumeration');
}

const root = (await ds.getRoot()) ?? await (async () => {
  const r = await resolveRoot(get, cfg.ONEDRIVE_FOLDER_PATH);
  await ds.setRoot(r);
  log.info({ root: r }, `resolved ${cfg.ONEDRIVE_FOLDER_PATH}`);
  return r;
})();

const counts: Record<string, number> = {};
const bump = (k: string) => { counts[k] = (counts[k] ?? 0) + 1; };

async function handlePage(items: DriveItem[]): Promise<void> {
  const classified: Classified[] = await classifyPage(items, {
    rootFolderId: root.folderId,
    getFolder: fm.get,
    upsertFolder: fm.upsert,
    removeFolder: fm.remove,
    getPrevFile: fs.get,
    fetchItem: async id => {
      try { return await get<DriveItem>(`/drives/${root.driveId}/items/${id}?$select=id,name,folder,parentReference`); }
      catch { return null; }
    },
  });
  for (const c of classified) {
    bump(c.action === 'process-file' ? `process-file:${c.reason}` : c.action);
    if (c.action === 'process-file') {
      log.info({ base: c.baseFolderId ?? '_root', reason: c.reason }, `WOULD index  ${c.item.name}`);
    } else if (c.action === 'update-metadata') {
      log.info({ base: c.baseFolderId ?? '_root' }, `WOULD patch  ${c.item.name}`);
    } else if (c.action === 'delete-file' || c.action === 'delete-folder') {
      log.info(`WOULD delete ${c.action === 'delete-folder' ? 'folder subtree' : 'file'} ${c.itemId}`);
    } else if (c.reason === 'unresolvable') {
      log.warn({ itemId: c.itemId }, 'unresolvable ancestry — would be counted, not indexed');
    }
  }
}

const startUrl = (await ds.getDeltaLink()) ?? initialDeltaUrl(root);
try {
  const newLink = await walkDelta(url => get<DeltaPage>(url), startUrl, handlePage);
  await ds.setDeltaLink(newLink);
  log.info({ counts }, 'delta walk complete — deltaLink persisted');
} catch (err) {
  if (err instanceof DeltaGoneError) {
    log.warn('410 Gone — restarting with a fresh enumeration');
    const newLink = await walkDelta(url => get<DeltaPage>(url), err.location ?? initialDeltaUrl(root), handlePage);
    await ds.setDeltaLink(newLink);
    log.info({ counts }, 'resync walk complete — deltaLink persisted');
  } else {
    throw err;
  }
} finally {
  await redis.quit();
}
```

- [ ] **Step 2: MANUAL GATE — acceptance run against the real drive**

Prerequisite: Task 7's `yarn auth` done; compose Redis up (`docker compose up -d redis`) with `REDIS_URL=redis://localhost:6379`... note compose maps no Redis port — add one temporarily for host-side dev: in `docker-compose.yml` under `redis:` add `ports: ["127.0.0.1:6379:6379"]` (loopback only, keep it — host-side `yarn dev` needs it in every later phase too).

```bash
cd worker
yarn dev:delta            # 1st run: full enumeration — every existing file logs WOULD index, counts summary
yarn dev:delta            # 2nd run: counts {} (no changes) — proves deltaLink persistence
# now edit one file in the OneDrive folder via the web UI, wait ~30s
yarn dev:delta            # logs exactly one WOULD index (content-changed)
```

Expected additionally: files under `HR/…` (or any top-level subfolder) log `base: hr-folder-id`; a file directly in the root logs `base: _root`.

- [ ] **Step 3: Commit (including the compose port line)**

```bash
git add worker/src/cli/devDelta.ts docker-compose.yml
git commit -m "feat(cli): yarn dev:delta — classified dry run with persistent deltaLink and 410 restart"
```

---

### Task 14: Phase-1 spike — verify the delta assumptions, encode the findings

**Files:**
- Modify: `docs/design/04-onedrive-graph-integration.md` (add a "Spike findings" subsection under §4)
- Modify: `worker/tests/graph/classify.test.ts` (add regression fixtures for any behavior that differed)

**Interfaces:**
- Consumes: `yarn dev:delta` (Task 13).
- Produces: verified facts replacing the two recorded assumptions (spec §10 risk 2), as doc text + pinned tests. The P2 plan is written **after** this lands.

- [ ] **Step 1: Run the experiment matrix (real OneDrive web UI + `yarn dev:delta` after each action)**

| # | Action in OneDrive | Record: what does the delta emit? |
|---|---|---|
| 1 | Move a file **out** of the synced folder (to a sibling of the root) | A `deleted` entry? An item whose new parent is out-of-scope? Nothing? |
| 2 | Rename a top-level base folder (`HR` → `People`) | Just the folder item, or every descendant re-emitted? |
| 3 | Rename a nested (non-base) folder | Same question one level down |
| 4 | Move a file between two base folders | One item with the new parent (expected: `re-route`)? |
| 5 | Inspect one `WOULD index` item's raw JSON (temporarily `log.info({ item })` in `handlePage`) | Does `file.hashes.quickXorHash` exist on this drive? |

- [ ] **Step 2: Write the findings into `docs/design/04-onedrive-graph-integration.md` §4**

Add under §4 (replace the "assumptions to verify" caveats where they're now settled):

```markdown
#### Spike findings (measured on the real tenant, YYYY-MM-DD)

| Scenario | Observed delta behavior | Consequence for the classifier |
|---|---|---|
| Move out of scope | <observed> | <e.g. "arrives as deleted → delete-file path already correct" / "arrives with out-of-scope parent → unresolvable path handles it; reconcile cleans up"> |
| Base-folder rename | <observed> | <…> |
| Nested-folder rename | <observed> | <…> |
| Cross-base-folder move | <observed> | <…> |
| quickXorHash present? | <yes/no> | <hash-skip viability for P4> |
```

- [ ] **Step 3: Pin any surprising behavior as a classifier regression test**

For each scenario whose emitted items differ from the existing fixtures, add one test to `worker/tests/graph/classify.test.ts` reproducing the *actual* item shapes observed (copy the logged JSON, trim to the `DriveItem` fields). If a finding requires a classifier change, make it in `worker/src/graph/classify.ts` the TDD way: failing regression test → fix → pass.

Run: `cd worker && yarn test`
Expected: all green, including new regression tests.

- [ ] **Step 4: Commit**

```bash
git add docs/design/04-onedrive-graph-integration.md worker/tests/graph/classify.test.ts worker/src/graph/classify.ts
git commit -m "docs(spike): delta edge-case findings from the real tenant + classifier regression fixtures"
```

**P1 acceptance reached** (per `docs/design/09-roadmap.md`): restart-safe incremental change listing; token refresh survives restarts; base folders resolved per file; spike questions answered and encoded.

---

## After this plan

Write the **P2 plan** (queues & scheduling) with the spike findings in hand — same process, next boundary. Then P3 (conversion), P4 (indexing = MVP), P5 (hardening).
