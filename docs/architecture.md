# Architecture

## Package dependency diagram

```
cmd/obgo
  |
  +---> internal/config         (load env vars)
  |
  +---> internal/couchdb        (HTTP client)
  |
  +---> internal/crypto         (E2EE encrypt/decrypt)
  |
  +---> internal/sync           (pull / push / watch orchestration)
           |
           +---> internal/couchdb   (via Client interface)
           +---> internal/crypto    (via *crypto.Service)
           +---> internal/watcher   (RemoteWatcher, LocalWatcher)
           +---> lib/livesync       (EncodeDocID, Split)
```

`internal/watcher` depends only on `internal/couchdb` (the `Client` interface and `ChangeEvent` type).  
`lib/livesync` has no internal dependencies.

---

## Package responsibilities

### `cmd/obgo`

CLI entry point built with [cobra](https://github.com/spf13/cobra). Registers `pull`, `push`, and `list` sub-commands. `pull` and `push` support watch flags: `--watch`/`-w` (bidirectional), `--wl` (local-only), `--wr` (remote-only), `--verbose`/`-v` (per-file logging), `--debug`/`-d` (dump raw change events as JSON, implies `-v`), and `--silence`/`-s` (suppress progress output). Loads the `.env` file (or `--env-file`), wires together the config, CouchDB client, crypto service, and sync service, then dispatches to `svc.Pull`, `svc.Push`, and optionally `svc.Watch`. Installs a signal handler (SIGINT/SIGTERM) for graceful shutdown via context cancellation.

### `internal/config`

Reads environment variables (`COUCHDB_URL`, `E2EE_PASSWORD`, `OBGO_DATA`, `OBGO_PATH_OBFUSCATION`) and returns a validated `Config` struct. Returns an error if required variables are missing.

### `internal/couchdb`

Defines the `Client` interface and its concrete implementation `HTTPClient`. All CouchDB interaction goes through this package. Key methods:

| Method | CouchDB endpoint | Purpose |
|--------|-----------------|---------|
| `AllMetaDocs` | `GET /{db}/_all_docs?include_docs=true` | List all non-chunk, non-deleted documents |
| `GetMeta` / `PutMeta` | `GET`/`PUT /{db}/{id}` | Single meta-document CRUD; PutMeta retries once on 409 Conflict |
| `GetChunk` / `PutChunk` | `GET`/`PUT /{db}/{id}` | Single chunk CRUD; PutChunk treats 409 as "already exists" (content-addressed) |
| `BulkGet` | `POST /{db}/_bulk_get` | Fetch many chunks in one request |
| `BulkDocs` | `POST /{db}/_bulk_docs` | Write many chunks in one request; ignores `conflict` errors |
| `Changes` | `GET /{db}/_changes?feed=continuous` | Long-poll change feed; goroutine reconnects with exponential backoff |
| `GetLocal` / `PutLocal` | `GET`/`PUT /{db}/_local/{id}` | Read/write non-replicated local documents (salt, seq) |
| `ServerInfo` | `GET /{db}` | Fetch database info |

`ErrNotFound` is a sentinel error returned on HTTP 404.

### `internal/crypto`

Handles all E2EE operations. When `E2EEPassword` is empty, the service is a transparent pass-through (base64 encode/decode only). When a password is set:

- `ChunkID(content)` — computes the content-addressed chunk `_id`: `h:<sha256>` (plain) or `h:+<sha256>` (encrypted).
- `EncryptContent(plaintext)` — AES-256-GCM using an HKDF-SHA256 derived key; returns a `%=`-prefixed base64 string (V2 format).
- `DecryptContent(ciphertext)` — detects format by prefix (`%=` = V2 HKDF, `%` = V1 PBKDF2) and decrypts accordingly. Reading V1 is supported for backward compatibility; writing always uses V2.
- `SetSalt([]byte)` — injects the HKDF salt fetched from `_local/obsidian_livesync_sync_parameters`.

### `internal/sync`

Orchestrates the three top-level operations. Holds references to the CouchDB client, crypto service, data directory path, and a `SuppressSet`.

- `Pull(ctx)` — see [`docs/flows.md`](flows.md). Removes local files whose remote document is deleted.
- `Push(ctx)` — see [`docs/flows.md`](flows.md). Tombstones remote docs for files absent locally, using app-level `deleted:true` (not CouchDB `_deleted:true`) so the path field is preserved for Obsidian sync.
- `Watch(ctx, watchLocal, watchRemote)` — starts `RemoteWatcher` and/or `LocalWatcher` as concurrent goroutines and blocks until one returns or the context is cancelled.
- `applyRemoteDoc` — shared helper used by both `Pull` and the remote watcher callback: fetches chunks via `BulkGet`, assembles, decrypts, and writes to disk; handles deletion (both lean CouchDB tombstones and app-level `deleted:true`) by removing the local file.
- `pushFile` — shared helper used by both `Push` and the local watcher callback: reads a file, splits, encrypts, uploads chunks via `BulkDocs`, upserts the meta document.
- `resolveCase` (`casepath.go`) — case-insensitive path resolution for deletions on case-sensitive Linux filesystems: walks each path component and falls back to a case-insensitive match so `os.Remove` finds the real path regardless of casing differences between the doc ID and the actual filename.

### `internal/watcher`

Contains two watcher types and the `SuppressSet`.

**`RemoteWatcher`** opens the CouchDB continuous `_changes` feed (via `Client.Changes`) with `style=all_docs&conflicts=true` so that deletions landing as non-winning conflict branches (as Obsidian/PouchDB sometimes produces) are visible. Calls a caller-supplied `onEvent` callback for each `ChangeEvent`. Persists the last seen sequence number to `<dataDir>/.obgo_seq` so that watch mode resumes from the correct position after a restart.

**`LocalWatcher`** uses [fsnotify](https://github.com/fsnotify/fsnotify) to watch the vault directory tree recursively. On `Write`/`Create` (file) events it calls an `onChange` callback. On `Remove`/`Rename` events it starts a 500 ms debounce timer; if a `Write`/`Create` for the same path arrives before the timer fires (atomic-save pattern used by Vim/Neovim), the timer is cancelled and the event is treated as an update rather than a delete. On `Create` (directory) events it recursively registers the new directory tree with fsnotify. A periodic 30 s rescan adds any subdirectories that may have been missed. Hidden files and directories (dot-prefixed) are skipped. Before invoking callbacks, the watcher checks `SuppressSet.IsSuppressed` to drop events caused by the app's own writes.

**`SuppressSet`** is a thread-safe set of recently-written absolute file paths with a 2-second TTL. `Add(path)` records a write; `IsSuppressed(path)` returns true and lazily evicts expired entries. Used to break the remote-write → fsnotify-event → push feedback loop.

### `lib/livesync`

Shared utilities that implement protocol-level details:

- `EncodeDocID(relPath)` — encodes a vault-relative path into a CouchDB document `_id`, prepending `/` when the path starts with `_` to avoid collisions with CouchDB design documents.
- `Split(content, chunkSize)` — splits file content into chunks following the Livesync chunking rules (line-boundary aware for text; fixed-size base64 blocks for binary). When `chunkSize` is 0 the default sizes from the protocol are used.

---

## Key interfaces

### `couchdb.Client`

```go
type Client interface {
    AllMetaDocs(ctx context.Context) ([]MetaDoc, error)
    GetMeta(ctx context.Context, id string) (*MetaDoc, error)
    PutMeta(ctx context.Context, doc *MetaDoc) (string, error)
    GetChunk(ctx context.Context, id string) (*ChunkDoc, error)
    PutChunk(ctx context.Context, doc *ChunkDoc) (string, error)
    BulkGet(ctx context.Context, ids []string) ([]ChunkDoc, error)
    BulkDocs(ctx context.Context, docs []interface{}) error
    Changes(ctx context.Context, since string) (<-chan ChangeEvent, error)
    GetLocal(ctx context.Context, id string) (map[string]interface{}, error)
    PutLocal(ctx context.Context, id string, doc map[string]interface{}) error
    ServerInfo(ctx context.Context) (map[string]interface{}, error)
}
```

Defined in `internal/couchdb`; implemented by `HTTPClient`. Accepted by `sync.New` so that tests can inject a mock.

### `sync.Service`

Not an interface but the central orchestrator. Created via:

```go
svc := sync.New(db couchdb.Client, cr *crypto.Service, dataDir string) *Service
```

Public methods: `Pull(ctx)`, `Push(ctx)`, `Watch(ctx)`.

### `watcher.SuppressSet`

```go
func NewSuppressSet() *SuppressSet
func (s *SuppressSet) Add(path string)
func (s *SuppressSet) IsSuppressed(path string) bool
```

Shared between `sync.Service` (which calls `Add` before writing to disk) and `LocalWatcher` (which calls `IsSuppressed` before pushing). The single instance is created in `sync.New` and passed to `watcher.NewLocalWatcher`.
