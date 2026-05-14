# Flows

## Pull flow

Fetches all documents from CouchDB and writes them to the local vault. CouchDB is treated as the source of truth.

```
obgo pull
  |
  1. Load .env file (if present); read COUCHDB_URL, E2EE_PASSWORD, OBGO_DATA
  |
  2. Create CouchDB HTTPClient (parses URL, extracts credentials and db name)
  |
  3. Create crypto.Service (enabled if E2EE_PASSWORD is non-empty)
  |
  4. [E2EE only] GET _local/obsidian_livesync_sync_parameters
     |  Extract base64 "salt" field → crypto.SetSalt(saltBytes)
     |  (Needed before any decryption can happen)
  |
  5. GET /{db}/_all_docs?include_docs=true
     |  Returns all non-chunk documents, including tombstones
     |  → []MetaDoc  (each has Path, Children []chunkID, ctime/mtime/size,
     |                and IsDeleted() true for tombstones)
  |
  6. For each MetaDoc:
     |
     6a. if doc.IsDeleted():
         |  absPath = resolveCase(dataDir, doc.Path or DecodeDocID(doc.ID))
         |  suppress.Add(absPath)
         |  os.Remove(absPath)   ← clean up local file
         |  continue
     |
     6b. POST /{db}/_bulk_get  with doc.Children IDs
         → []ChunkDoc  (each has ID, Data string)
     |
     6c. Build chunkMap[id] = data
     |
     6d. For each chunkID in doc.Children (in order):
         |  [E2EE]  crypto.DecryptContent(data)  →  []byte
         |  [plain] base64.Decode(data)           →  []byte
         |  append to content
     |
     6e. suppress.Add(absPath)   ← prevent local watcher feedback
         os.MkdirAll(dir, 0755)
         os.WriteFile(absPath, content, 0644)
  |
  7. Return (exit 0 on success)
```

If `--watch` is passed, control continues to [Watch mode](#watch-mode) after step 7.

---

## Push flow

Reads all files from the local vault and upserts them to CouchDB. The local vault is treated as the source of truth.

```
obgo push
  |
  1. Load .env file (if present); read COUCHDB_URL, E2EE_PASSWORD, OBGO_DATA
  |
  2. Create CouchDB HTTPClient
  |
  3. Create crypto.Service
  |
  4. [E2EE only] Ensure HKDF salt exists (ensureSalt):
     |
     4a. GET _local/obsidian_livesync_sync_parameters
         |  If salt field found → crypto.SetSalt(saltBytes); done
     |
     4b. If not found or no salt:
         |  Generate 32 random bytes
         |  crypto.SetSalt(salt)
         |  PUT _local/obsidian_livesync_sync_parameters  { "salt": base64(salt) }
  |
  5. Enumerate local files: filepath.WalkDir(OBGO_DATA) → localPaths set
     |
     Enumerate remote docs: AllMetaDocs → remoteIDs set
     |
     For each remote doc whose path has no local file:
     |  PUT /{db}/{docID} with MetaDoc{ deleted:true, children:nil }
     |  (app-level tombstone — preserves path for Obsidian sync)
     |
     For each local file:
     |
     5a. os.ReadFile(absPath) → content []byte
     |
     5b. Compute relPath = path relative to OBGO_DATA (forward slashes)
     |
     5c. livesync.Split(content, 0) → []chunk (line-boundary aware for text)
     |
     5d. For each chunk:
         |  id = crypto.ChunkID(chunk)
         |             "h:<sha256>"       (plain)
         |             "h:+<sha256>"      (E2EE)
         |  [E2EE]  data = crypto.EncryptContent(chunk)  → "%=<base64>"
         |  [plain] data = base64.Encode(chunk)
         |  → ChunkDoc{ ID, Data, Type:"leaf", Encrypted }
     |
     5e. POST /{db}/_bulk_docs  with all ChunkDocs
         (409 conflicts are ignored — chunk is content-addressed, already exists)
     |
     5f. docID = livesync.EncodeDocID(relPath)
         GET /{db}/{docID}  → fetch existing rev + original ctime (if any)
     |
     5g. PUT /{db}/{docID}  with MetaDoc:
           { id, rev, type:"plain", path, ctime, mtime, size, children:chunkIDs }
         (on 409: fetch current rev, retry once)
  |
  6. Return (exit 0 on success)
```

If `--watch` is passed, control continues to [Watch mode](#watch-mode) after step 6.

---

## Watch mode

Runs after the initial pull or push. Starts two concurrent goroutines and blocks until the context is cancelled (SIGINT/SIGTERM) or one goroutine returns an error. The `--debug`/`-d` flag (implies `--verbose`/`-v`) prints every raw CouchDB change event as JSON before processing — useful for diagnosing how Obsidian represents deletions.

```
svc.Watch(ctx)
  |
  +---------- goroutine 1: RemoteWatcher ----------------+
  |                                                       |
  |  Load last seq from <OBGO_DATA>/.obgo_seq             |
  |  GET /{db}/_changes?feed=continuous                   |
  |       &style=all_docs&conflicts=true                  |
  |       &heartbeat=10000&include_docs=true              |
  |       &since=<lastSeq>                                |
  |                                                       |
  |  (style=all_docs fires for every leaf revision,       |
  |   matching PouchDB behaviour so Obsidian deletions    |
  |   that land as non-winning conflict branches are seen)|
  |                                                       |
  |  for each ChangeEvent:                                |
  |    if event.Doc.IsDeleted():                          |
  |      absPath = resolveCase(dataDir, path)             |
  |      suppress.Add(absPath)                            |
  |      os.Remove(absPath)                               |
  |    else:                                              |
  |      applyRemoteDoc(ctx, event.Doc)                   |
  |        → BulkGet chunks → decrypt → suppress → write  |
  |    saveSeq(<OBGO_DATA>/.obgo_seq)                     |
  |                                                       |
  |  On connection drop: exponential backoff (1s→30s),    |
  |  reconnect with updated since value                   |
  |                                                       |
  +-------------------------------------------------------+

  +---------- goroutine 2: LocalWatcher -----------------+
  |                                                       |
  |  fsnotify.NewWatcher()                                |
  |  Watch OBGO_DATA and all subdirs recursively          |
  |  (hidden dirs like .git are skipped)                  |
  |  Periodic rescan every 30s adds any new subdirs       |
  |  that were not yet watched (belt-and-suspenders)      |
  |                                                       |
  |  for each fsnotify event:                             |
  |    skip hidden files and .obgo_seq                    |
  |                                                       |
  |    Write/Create (file):                               |
  |      if suppress.IsSuppressed(path): skip  <-- loop  |
  |      else: pushFile(ctx, path)              prevention|
  |              → Split → Encrypt → BulkDocs             |
  |              → GetMeta (rev) → PutMeta                |
  |                                                       |
  |    Remove/Rename:                                     |
  |      if suppress.IsSuppressed(path): skip             |
  |      else: start 500ms debounce timer for path        |
  |        if Write/Create arrives before timer fires:    |
  |          cancel timer → treat as update (not delete)  |
  |        else on timer fire:                            |
  |          GetMeta → doc.deleted=true → PutMeta         |
  |          (app-level tombstone, path field preserved)  |
  |                                                       |
  |    Create (directory):                                |
  |      recursively add newDir and all its children      |
  |      to fsnotify watcher                              |
  |                                                       |
  +-------------------------------------------------------+

  select {
    case err = <-remoteErrCh: return err
    case err = <-localErrCh:  return err
  }
```

### SuppressSet loop prevention

Without `SuppressSet`, the following feedback loop would occur:

```
RemoteWatcher receives change
  → writes file to disk
    → LocalWatcher sees Write event
      → pushes file back to CouchDB
        → CouchDB emits another change
          → (repeat forever)
```

`SuppressSet` breaks the loop: before writing a file to disk (in `applyRemoteDoc`), the sync service calls `suppress.Add(absPath)`. When `LocalWatcher` receives the subsequent fsnotify event, `suppress.IsSuppressed(absPath)` returns `true` and the event is dropped. Entries expire after 2 seconds.

```
applyRemoteDoc:
  suppress.Add(absPath)    ← mark as app-written
  os.WriteFile(absPath)

LocalWatcher (fsnotify Write event for same path):
  suppress.IsSuppressed(absPath)  → true  → skip push
```
