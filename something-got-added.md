Good, that gives a complete picture of what still reads from `config.json` at runtime. Now I have everything needed to write the architecture document.

---

## RED Engine — New Architecture & Testing Guide

### Overview of the data model after these changes

There are now **two categories of configuration**:

---

### Category 1 — Node identity (`config.json`, read-only at boot)

These fields are loaded once when the process starts and never written back by the application:

|Field|Used by|
|---|---|
|`addr`|`http.ListenAndServe` in `main.go`|
|`dataDir`|store, registry path, file serving|
|`adminToken`|`adminOnly()` middleware — SHA-256 hashes the stored value and compares against incoming `X-Admin-Token` header|
|`webhookSecret`|`webhookSync` — HMAC-SHA256 verifies incoming GitHub payloads|
|`nodeName` / `siteName`|`/-/nodeinfo` response, page titles|
|`startupSync`|**Migration only** — read once on first boot, inserted into `startup_sync` DB table if that table is empty, then ignored forever|

The admin has no HTTP endpoint to change these. To change them, you edit `config.json` and restart the process. This is intentional — they are node-level secrets and identifiers, not operational data.

---

### Category 2 — Operational data (`registry.db`, read/write at runtime)

Everything that the admin manages through the UI lives here:

#### `startup_sync` table

The list of remote sources that are pulled on startup and re-pulled when a webhook fires.

**How data gets in:**

```
Admin UI / API
  POST /-/import
    body: { "url": "...", "filename": "...", "saveToStartup": true }
      → registry.AddStartupSync(url, filename)
        → INSERT OR REPLACE INTO startup_sync
```

**How data gets read:**

```
GET /-/admin/config
  → registry.ListStartupSync()
    → SELECT * FROM startup_sync

webhookSync goroutine (triggered by POST /-/webhook/sync)
  → registry.ListStartupSync()
    → matches incoming repo URL against each row
    → fetch.PullDelta() for each match
    → store.UpdateFiles() or store.Reload()

main.go startup
  → registry.ListStartupSync()
    → fetch.Pull() for each row
```

**How data gets removed:**

```
POST /-/admin/remove
  body: { "filename": "...", "deleteLocalFiles": true/false }
    → registry.RemoveStartupSync(filename)
      → DELETE FROM startup_sync WHERE filename = ?
```

---

#### `peers` table

Remote nodes this engine knows about.

**Write path:** `POST /-/admin/peers/add` → `FetchNodeInfo(url)` → `registry.AddPeer()`  
**Refresh:** `POST /-/admin/peers/refresh` → re-fetches `/-/nodeinfo` from the peer → upsert  
**Delete:** `POST /-/admin/peers/delete` → `registry.DeletePeer()`  
**Read:** `GET /-/admin/peers` → `registry.ListPeers()`

The `verified` column was removed from the struct and all queries. Existing databases keep the column (SQLite doesn't error on extra columns) and it's silently unused.

---

#### `trusted_authors` table

Public keys whose Ed25519 signatures the store will accept when verifying markdown files.

**Write path:** `POST /-/admin/contributors/add` → direct SQL insert (via `contributors.go`)  
**Revoke:** `POST /-/admin/contributors/delete` → sets `revoked = 1` (soft delete)  
**Read:** `GET /-/admin/contributors` returns non-revoked rows; `store.loadSecurityData()` reads them at every `Reload()` and `UpdateFiles()` call

---

#### `signer.db` (content-side, not node-side)

This file lives **inside the content repository** — the git repo that gets pulled by startup sync or webhook. The engine never writes to it. The flow is:

```
GitHub push
  → webhook fires POST /-/webhook/sync
    → verifies HMAC-SHA256 with config.json webhookSecret
    → registry.ListStartupSync() → finds matching repo
    → fetch.PullDelta() → git pull, returns changed file paths
    → signer.db is updated as part of the git pull (it's committed in the repo)
    → store.UpdateFiles(changedPaths) or store.Reload()
      → loadSecurityData() walks dataDir for signer.db files
        → opens each one: SELECT path, file_hash, public_key, signature FROM files
        → builds allSignatures map
      → processArticle() verifies each .md against allSignatures
        → checks file hash matches signer.db record
        → checks public_key is in trusted_authors (registry.db)
        → verifies Ed25519 signature
        → sets Article.Verified = true / false
```

The `signer.db` schema (from the uploaded file):

```sql
CREATE TABLE files (
    path TEXT PRIMARY KEY,         -- e.g. "Prince.md"
    file_hash TEXT NOT NULL,       -- SHA-256 of file content
    public_key TEXT NOT NULL,      -- Ed25519 public key (hex)
    signature TEXT NOT NULL,       -- Ed25519 signature (hex)
    updated_at INTEGER             -- Unix timestamp
);
CREATE TABLE metadata (
    key TEXT PRIMARY KEY,
    value TEXT NOT NULL            -- branch_author, branch, sub_branch
);
```

For a file to display as **Verified**, all three must be true simultaneously:

1. The file's SHA-256 hash matches `signer.db files.file_hash`
2. `signer.db files.public_key` is present in `registry.db trusted_authors` (not revoked)
3. `ed25519.Verify(pubKey, fileContent, signature)` returns true

---

### Full data flow diagram

```
config.json (boot only)
    │
    ├─ addr, dataDir ──────────────────► main.go (ListenAndServe)
    ├─ adminToken ─────────────────────► adminOnly() middleware
    ├─ webhookSecret ──────────────────► webhookSync() HMAC check
    ├─ nodeName/siteName ──────────────► /-/nodeinfo, page renders
    └─ startupSync[] ──────────────────► (migration only, first boot)
                                              │
                                              ▼
                                    registry.db / startup_sync
                                              │
                          ┌───────────────────┼───────────────────────┐
                          ▼                   ▼                       ▼
                  /-/admin/config     webhookSync goroutine      main.go boot
                  (list)              (delta pull on push)       (full pull)
                                              │
                                              ▼
                                    data/{filename}/ (git checkout)
                                       contains signer.db
                                              │
                                              ▼
                                    store.loadSecurityData()
                                    reads signer.db + trusted_authors
                                              │
                                              ▼
                                    processArticle() → Article.Verified
```

---

### Testing methodology

#### Test 1 — config.json migration to startup_sync

```bash
# Ensure registry.db does not exist yet (or startup_sync table is empty)
rm -f data/registry.db

# config.json has startupSync entries
cat config.json | jq '.startupSync'

# Start the engine
./red-engine -config config.json

# Check that entries were migrated
# Should log: "Migrated N startup sync entries from config.json to database"
# Verify with a read of the admin endpoint:
curl -H "X-Admin-Token: <your-token>" http://localhost:8080/-/admin/config
```

Expected: The response body is a JSON array matching the original `startupSync` from `config.json`.

---

#### Test 2 — Adding a sync source via admin API

```bash
curl -X POST http://localhost:8080/-/import \
  -H "X-Admin-Token: <your-token>" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://github.com/org/repo.git","filename":"my-guide","saveToStartup":true}'

# Verify it appears in the DB
curl -H "X-Admin-Token: <your-token>" http://localhost:8080/-/admin/config
```

Expected: `my-guide` appears in the list with the correct URL. Content is present in `data/my-guide/`.

---

#### Test 3 — signer.db verification

Place a `signer.db` file (using the schema above) inside `data/some-section/`. Add the matching public key to `trusted_authors` via the contributors API:

```bash
curl -X POST http://localhost:8080/-/admin/contributors/add \
  -H "X-Admin-Token: <your-token>" \
  -H "Content-Type: application/json" \
  -d '{"public_key":"<64-char-hex-pubkey>","name":"Test Author"}'
```

Then trigger a reload:

```bash
curl -X POST http://localhost:8080/-/reload \
  -H "X-Admin-Token: <your-token>"
```

Navigate to the article in the browser. The verification badge should show **Verified** with the contributor name. To confirm failure modes work, either modify the `.md` file (hash mismatch) or revoke the contributor (untrusted key) and reload again.

---

#### Test 4 — Webhook triggers signer.db re-read

```bash
# Simulate a GitHub webhook push event
curl -X POST http://localhost:8080/-/webhook/sync \
  -H "Content-Type: application/json" \
  -H "X-Hub-Signature-256: sha256=<computed-hmac>" \
  -d '{"repository":{"clone_url":"https://github.com/org/repo.git","html_url":"https://github.com/org/repo"}}'
```

Expected: The engine pulls the latest git state (including any updated `signer.db`), hot-patches changed files, and re-verifies them. Check the server logs for `"Webhook triggering delta pull for: ..."`.

---

#### Test 5 — Removing a sync source

```bash
curl -X POST http://localhost:8080/-/admin/remove \
  -H "X-Admin-Token: <your-token>" \
  -H "Content-Type: application/json" \
  -d '{"filename":"my-guide","deleteLocalFiles":false}'

# Verify it is gone
curl -H "X-Admin-Token: <your-token>" http://localhost:8080/-/admin/config
```

Expected: `my-guide` no longer appears. The next webhook push for that repo will not trigger a pull.

---

### What is NOT yet wired to the database

These still require manual `config.json` edits and a restart:

|Setting|Location|Path to DB|
|---|---|---|
|`adminToken`|`config.json`|Phase 2: `node_settings` table + admin UI form|
|`webhookSecret`|`config.json`|Phase 2: `node_settings` table|
|`nodeName` / `siteName`|`config.json`|Phase 2: `node_settings` table|
|`addr` / `dataDir`|`config.json`|Always CLI/env — needed before DB opens|

I need to push the committed changes. Let me use the GitHub MCP tool since the git remote requires it.