# 04 — OneDrive / Microsoft Graph integration (delegated)

> Status: draft · Last updated: 2026-07-26

## 1. Entra app registration (one-time, manual)

1. Entra admin center → **App registrations → New registration**.
   - Name: `vector-syncer`.
   - Supported account types: **single tenant** (Q1 answered 2026-07-26: work/school OneDrive for Business). A personal-account variant would use the `/consumers` authority instead — not our case.
2. **Authentication** → Advanced settings → **Allow public client flows: Yes** (required for device code flow). No redirect URI, no client secret — this is a public client; possession of the encrypted token cache is the credential that matters.
3. **API permissions** (Microsoft Graph → *Delegated*):

| Scope | Why | Notes |
|---|---|---|
| `Files.Read` | Read the signed-in user's own OneDrive | Sufficient if the folder is in the user's own drive — **preferred (least privilege)** |
| `Files.Read.All` | Read all files the user can access | Only if the target folder is *shared with* the user rather than owned |
| `offline_access` | Refresh token — the whole "runs unattended" story | Mandatory |
| `User.Read` | Basic sign-in | Standard |

These are user-consentable in most tenants; if the tenant restricts user consent, one admin approval of this app is needed (constraint noted in requirements).

4. Record **Directory (tenant) ID** and **Application (client) ID** → `.env`.

## 2. Delegated auth design (MSAL Node)

Delegated + unattended = *interactive once, silent forever after*:

```
bootstrap CLI (once, human present)          worker runtime (forever)
────────────────────────────────             ─────────────────────────
acquireTokenByDeviceCode()                   accounts = tokenCache.getAllAccounts()
  → prints URL + code, human signs in        acquireTokenSilent({ account, scopes })
  → MSAL caches access+refresh token   ──►     → cached access token, or
    via ICachePlugin (encrypted file)          → auto-refresh via refresh token
                                               → throws interaction_required → ALERT
```

### Bootstrap CLI (`yarn auth`)

```ts
import { PublicClientApplication } from '@azure/msal-node';

const pca = new PublicClientApplication({
  auth: {
    clientId: env.AZURE_CLIENT_ID,
    authority: `https://login.microsoftonline.com/${env.AZURE_TENANT_ID}`,
    // personal OneDrive instead: https://login.microsoftonline.com/consumers
  },
  cache: { cachePlugin: encryptedFileCachePlugin },
});

await pca.acquireTokenByDeviceCode({
  scopes: env.GRAPH_SCOPES,               // ['Files.Read', 'offline_access', 'User.Read']
  deviceCodeCallback: (info) => console.log(info.message), // "go to https://microsoft.com/devicelogin, enter ABC-123"
});
```

### Runtime acquisition (every job)

```ts
async function getGraphToken(): Promise<string> {
  const accounts = await pca.getTokenCache().getAllAccounts();
  if (accounts.length === 0) throw new AuthRequiredError('Run `yarn auth` first');
  try {
    const res = await pca.acquireTokenSilent({ account: accounts[0], scopes: env.GRAPH_SCOPES });
    return res.accessToken;
  } catch (e) {
    if (isInteractionRequired(e)) throw new AuthRequiredError(e.errorCode); // → health red + alert
    throw e; // transient → normal retry path
  }
}
```

### Encrypted persistent token cache

`msal-node-extensions` targets desktop keychains (DPAPI/libsecret) — not a fit for a headless container. Instead: a ~40-line custom `ICachePlugin`:

- `beforeCacheAccess`: read `data/msal-cache.bin`, decrypt (AES-256-GCM, key = `MSAL_CACHE_KEY` env, random IV per write, auth tag verified), `tokenCache.deserialize(json)`.
- `afterCacheAccess`: if `cacheContext.cacheHasChanged`, `serialize()` → encrypt → atomic write (tmp + rename), mode `0600`.

The cache file lives on a mounted volume so re-auth survives container rebuilds.

### Token lifetime reality check

Access tokens last ~1 h and are refreshed silently. Refresh tokens are long-lived **but revocable**: password reset, Conditional Access changes, admin revocation, or long inactivity kill them. Since the sync runs every few minutes, the token stays warm; the design still treats `interaction_required` as a first-class state (FR-18): `/health` flips to `auth_required`, every sync fails fast with the runbook message, nothing crash-loops.

## 3. Resolving the target folder (once)

Config gives a human path, e.g. `ONEDRIVE_FOLDER_PATH=/Documents/KnowledgeBase`:

```
GET /me/drive/root:/Documents/KnowledgeBase
→ { id: "01ABC...", parentReference: { driveId: "b!xyz..." } }
```

Store as `vs:root = { driveId, folderId }` in Redis (re-resolved on miss). All later calls use IDs, so renaming the folder itself doesn't break the sync (the path is only re-resolved if the ID disappears).

## 4. Delta queries — the heart of change detection

Scoped to the folder (non-root delta is GA since June 2021 for OneDrive for Business/SharePoint; consumer OneDrive also supports it — verify against the actual tenant in the Phase-1 spike):

```
Initial:  GET /drives/{driveId}/items/{folderId}/delta?$select=id,name,size,file,folder,deleted,cTag,eTag,parentReference,createdDateTime,createdBy,lastModifiedBy,lastModifiedDateTime,webUrl&$top=200
Pages:    follow @odata.nextLink until absent
Then:     store @odata.deltaLink   →   next run: GET {deltaLink}
```

Notes:

- **First run = full enumeration.** Every existing file arrives as a "change" — that *is* the initial indexing. (`token=latest` exists to skip history; only useful for a "start empty, index only future changes" mode.)
- **The walk is recursive by nature.** Delta on the root folder returns changes for the entire subtree at any depth; new subfolders are picked up automatically, and folder items appearing in the response feed the `vs:folder:{id}` map used for ancestor/base-folder resolution.
- `createdBy`/`lastModifiedBy` ride along in `$select` so author information lands in the vector payload (FR-14) without extra requests.
- `$select` keeps pages small; `$top=200` reduces round trips.
- Optional header `deltaExcludeParent` suppresses the ancestor/parent items in responses.
- The response can contain the same item multiple times and folders we don't care about — classification below decides.
- **Fallback flag** `DELTA_SCOPE=root`: if folder-scoped delta misbehaves on some drive type, walk `/root/delta` and filter client-side by `parentReference.path` prefix. Same downstream logic.

### Classifying a delta item

```
item.deleted?            → was a base folder ⇒ drop its collection + catalog entry (FR-15/FR-21)
                           was any folder    ⇒ delete-folder(id)  [Qdrant: ancestor_ids contains id]
                           was a file        ⇒ delete-file(id)
item.folder?             → upsert vs:folder:{id} = { parentId, name }  (folder map for ancestor resolution)
                           → new direct child of the sync root ⇒ base folder: ensure collection + catalog entry
                           → rename/move → path refresh (weekly reconcile in v1)
item.file?
  → resolve ancestor chain via the vs:folder map (Graph fetch on cache miss)
  → base folder = first ancestor under the sync root ⇒ target collection ("_root" catch-all if none)
  ├─ not under sync root (root-scope fallback mode)  → delete-file (moved out) or ignore if never indexed
  ├─ extension not in allowlist                      → record skipped_unsupported
  ├─ cTag == stored, same base folder                → update-metadata job (payload patch, no re-embed)
  ├─ cTag == stored, different base folder           → process-file re-route (index into new collection, purge old)
  └─ else                                            → process-file job (dedup id `${id}:${cTag}`)
```

Change-detection fields, in order of preference:

| Field | Semantics | Caveat |
|---|---|---|
| `file.hashes.quickXorHash` | content hash (OneDrive for Business) | not on every drive type |
| `file.hashes.sha256Hash` / `sha1Hash` | content hash (consumer OneDrive) | availability varies |
| `cTag` | changes when content changes | opaque; treat as equality token only |
| `eTag` | changes on *any* change incl. metadata | too noisy alone — used only to notice metadata-only updates |

Edge cases parked for the Phase-1 spike (behavior verified empirically, then encoded in tests): item moved *out* of a scoped folder (does scoped delta emit a delete-style entry?), folder rename (are descendants re-emitted?). The weekly reconcile job is the safety net for whichever way those fall.

### Resync: `410 Gone`

When a stored token is too old or server state changed, delta answers `410 Gone` with a `Location` header holding a **fresh nextLink**. Handling (per Graph docs, error code `resyncChangesApplyDifferences`):

1. Follow the `Location` link → full re-enumeration begins.
2. Process every returned item through the normal classifier (hash-skip makes unchanged files cheap).
3. Reconcile: any file id in local state that the enumeration did not return ⇒ delete from Qdrant.
4. Store the new deltaLink.

This is exactly the same code path as the manual **full resync** command (FR-5) — one implementation, two triggers.

## 5. Downloading content

Per file (at processing time, not enumeration time — download URLs are short-lived):

```
GET /drives/{driveId}/items/{itemId}/content
→ 302 Location: https://…pre-authenticated CDN URL…
→ stream to scratch file (no auth header needed on the redirect target)
```

- Stream to disk (`SCRATCH_DIR`), never buffer whole files in memory; the scratch file is **always removed after processing** (`finally`, success or failure) — downloaded content persists nowhere outside Qdrant payloads.
- Re-fetch item metadata first (`$select=id,name,size,cTag,file,parentReference`): the file may have changed or vanished since enqueueing — if `cTag` differs from the job's, complete as `superseded` (a newer job is already queued, courtesy of dedup id).
- Enforce `MAX_FILE_SIZE_MB` from metadata *before* downloading.

## 6. Throttling & efficiency

- Graph signals throttling with **429 + `Retry-After`** (occasionally 503/504). Worker behavior: `await worker.rateLimit(retryAfterMs); throw Worker.RateLimitError()` — pauses the whole `files` queue without burning a retry attempt ([07-bullmq-design.md](07-bullmq-design.md) §5).
- Static safety net: BullMQ `limiter` (e.g. max 8 jobs/sec) + `sync` concurrency 1.
- The Graph JS client's built-in retry middleware additionally honors `Retry-After` for transient in-call retries.
- `$batch` (up to 20 sub-requests) is an optimization for bulk metadata refresh during reconcile — not needed for v1 correctness.

## 7. Security notes

- Token cache: encrypted at rest, `0600`, on a volume excluded from any backup that leaves the host.
- Scopes: start with `Files.Read`; escalate to `Files.Read.All` only if the folder is shared (Q1).
- Downloaded bytes live only in `SCRATCH_DIR` for the duration of one job.
- Logs carry file *paths and ids*, never content.

## 8. Future: change-notification webhooks (v2)

Polling every 5 min is fine for v1. To cut latency later:

- Subscription on the **drive root** (`/me/drive/root`) — root only, `changeType: updated` only (both are documented Graph limitations for drive subscriptions).
- Max expiration ≈ 30 days ⇒ needs a renewal job (BullMQ scheduler again).
- Requires a public HTTPS `notificationUrl` — the reason it's not in v1.
- Notifications carry **no item details**; the handler just triggers the existing `sync` job immediately. Delta remains the source of what changed — nothing else in the design moves.
