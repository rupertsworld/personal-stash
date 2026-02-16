# Stash Filesystem Architecture - Developer Spec

Complete specification for the filesystem-based stash architecture.

---

## Overview

Transform stash from automerge-only storage to readable filesystem with automatic sync. Files live on disk as normal files; automerge handles CRDT sync in the background.

**Key changes:**
- Readable files at stash root, automerge data in `.stash/`
- StashReconciler watches files, diffs changes to automerge, syncs to remote
- MCP tools operate on files, not automerge directly
- Stashes can live anywhere on filesystem
- Symlink support for bringing stashes into project directories

---

## File Layout

### Global Config Location
```
~/.stash/
  config.json              # global config (actorId, stash registry, auth)
```

### Stash Layout (can be anywhere)
```
<stash-path>/              # e.g., ~/.stash/notes or ~/Repos/docs
  readme.md                # readable files at root
  subdir/
    file.md
  .stash/                  # automerge internals (gitignored equivalent)
    meta.json              # stash metadata
    structure.automerge    # file tree CRDT
    docs/
      <docId>.automerge    # file content CRDTs
    blobs/
      <sha256>.bin         # binary file content (content-addressed)
```

---

## Data Structures

### Global Config (`~/.stash/config.json`)

```typescript
interface GlobalConfig {
  actorId: string;                                    // ULID, generated on first run
  providers?: {
    github?: { token: string };                       // GitHub auth
    // future: other providers
  };
  stashes: Record<string, string>;                    // name → absolute path
}
```

Example:
```json
{
  "actorId": "01ARZ3NDEKTSV4RRFFQ69G5FAV",
  "providers": {
    "github": { "token": "ghp_..." }
  },
  "stashes": {
    "notes": "/Users/rupert/.stash/notes",
    "work": "/Users/rupert/Repos/work-docs"
  }
}
```

### Stash Meta (`.stash/meta.json`)

```typescript
interface StashMeta {
  name: string;                       // stash name
  description?: string;               // optional, for models
  remote?: string;                    // e.g., "github:user/repo" or null
}
```

Example:
```json
{
  "name": "notes",
  "description": "Shared context and notes for the project",
  "remote": "github:user/notes"
}
```

### Project Config (`.stash.json`)

Optional file in any directory for symlink setup:

```typescript
interface ProjectConfig {
  links: Record<string, string>;      // stash name → relative path
}
```

Example:
```json
{
  "links": {
    "company-docs": "./docs",
    "shared-notes": "./notes"
  }
}
```

### FileDoc Structure

Use a discriminated union for type safety:

```typescript
// Text files: character-level CRDT
interface TextFileDoc {
  type: 'text';
  content: Automerge.Text;
}

// Binary files: we don't store content in automerge
// Just track the file exists, actual bytes live on disk only
// On sync, binary files are transferred as blobs, not diffed
interface BinaryFileDoc {
  type: 'binary';
  hash: string;                       // SHA-256 of content, for change detection
  size: number;                       // file size in bytes
}

type FileDoc = TextFileDoc | BinaryFileDoc;
```

**Binary file approach:** Don't store binary content in automerge (wasteful for large files). Instead:
- Track hash + size in automerge for change detection
- Actual bytes live on disk and sync as opaque blobs
- On remote sync, if hash differs, transfer the whole file
- LWW: if two sides have different hashes, pick one (latest timestamp or deterministic tiebreaker)

---

## Initialization

On first run of any command requiring config:

```typescript
// src/core/config.ts
async function ensureConfig(baseDir = DEFAULT_STASH_DIR): Promise<GlobalConfig> {
  await fs.mkdir(baseDir, { recursive: true });

  const configFile = path.join(baseDir, "config.json");
  let config: GlobalConfig;
  let needsWrite = false;

  try {
    config = JSON.parse(await fs.readFile(configFile, "utf-8"));
  } catch {
    config = { actorId: ulid(), stashes: {} };
    needsWrite = true;
  }

  // Migration: add actorId if missing
  if (!config.actorId) {
    config.actorId = ulid();
    needsWrite = true;
  }
  if (!config.stashes) {
    config.stashes = {};
    needsWrite = true;
  }

  if (needsWrite) {
    await fs.writeFile(configFile, JSON.stringify(config, null, 2) + "\n");
  }

  return config;
}
```

---

## CLI Commands

### CLI UX Package

Use `@inquirer/prompts` for better interactive UX:
- Confirmations for destructive actions
- Selection menus for provider choice
- Password input for tokens

### Shared Options (`src/cli/options.ts`)

```typescript
import { Option } from "commander";

export const pathOption = new Option(
  "--path <path>",
  "where to create the stash (default: ~/.stash/<name>)"
);

export const descriptionOption = new Option(
  "--description <desc>",
  "description to help models understand the stash"
);

export const forceOption = new Option(
  "--force",
  "skip confirmation prompts"
);
```

### Command Reference

#### `stash` (no subcommand)
```
stash
```
- Start daemon in background if not running (same as `stash start`)
- Read `.stash.json` in current directory, create symlinks if present
- Convenience for project setup

#### `stash create <name>`
```
stash create <name> [--path <path>] [--description <desc>]
```
- Create new local stash
- `--path`: location (default: `~/.stash/<name>`)
- `--description`: optional description
- Prompts for remote provider (github/local)
- Registers in global config

#### `stash connect <remote>`
```
stash connect <remote> --name <name> [--path <path>]
```
- Connect to existing remote stash
- `<remote>`: e.g., `github:user/repo`
- `--name`: required, local name
- `--path`: location (default: `~/.stash/<name>`)
- Pulls remote content, registers in global config

#### `stash list`
```
stash list
```
- List all stashes
- Shows: name, description, path, remote status

Output format:
```
notes       "Project notes"           ~/.stash/notes        github:user/notes
work        "Work documents"          ~/Repos/work-docs     (local only)
```

#### `stash edit <name>`
```
stash edit <name> [--description <desc>] [--remote <remote>]
```
- Update stash metadata
- `--description`: set/update description
- `--remote`: change remote (use `none` to disconnect)

#### `stash delete <name>`
```
stash delete <name> [--remote] [--force]
```
- Delete stash locally
- `--remote`: also delete from remote provider
- `--force`: skip confirmation
- Confirms before deletion (unless --force)
- Extra confirmation for --remote (destructive)

#### `stash link`
```
stash link [<stash>] [<path>]
```
- `stash link`: read `.stash.json`, create all symlinks
- `stash link notes`: create `./notes` → stash location
- `stash link notes ./docs`: create `./docs` → stash location
- Like `ln -s <target> <link>`
- **Errors if target path already exists** (file, directory, or symlink)

#### `stash unlink`
```
stash unlink [<path>]
```
- `stash unlink`: read `.stash.json`, remove all symlinks listed there
- `stash unlink ./docs`: remove symlink at `./docs`
- Only removes symlinks, not regular files/directories
- Errors if path is not a symlink

#### `stash start`
```
stash start
```
- Start background daemon
- Daemon watches all stashes for file changes
- Syncs changes to remotes

#### `stash stop`
```
stash stop
```
- Stop background daemon

#### `stash status`
```
stash status
```
- Show daemon status (running/stopped)
- Show sync status per stash

#### `stash sync [<name>]`
```
stash sync [<name>]
```
- Manual sync
- No args: sync all stashes
- With name: sync specific stash

#### `stash auth github`
```
stash auth github
```
- Set GitHub personal access token
- Uses password input (hidden)

---

## MCP Tools

All tools operate on filesystem, not automerge directly. Watcher handles sync.

### `stash_list`

List stashes or directory contents.

```typescript
// List all stashes
stash_list({ })
→ {
  stashes: [
    { name: "notes", description: "Project notes", path: "/Users/.../notes" },
    ...
  ]
}

// List stash root
stash_list({ stash: "notes" })
→ { items: ["readme.md", "docs/"] }

// List subdirectory
stash_list({ stash: "notes", path: "docs/" })
→ { items: ["guide.md", "api.md"] }
```

Parameters:
- `stash?: string` - stash name
- `path?: string` - directory within stash

### `stash_glob`

Find files matching glob pattern.

```typescript
stash_glob({ stash: "notes", glob: "**/*.md" })
→ { files: ["readme.md", "docs/guide.md", "docs/api.md"] }
```

Parameters:
- `stash: string` - required
- `glob: string` - glob pattern

### `stash_read`

Read file content.

```typescript
stash_read({ stash: "notes", path: "readme.md" })
→ { content: "# My Notes\n\nThis is..." }
```

Parameters:
- `stash: string` - required
- `path: string` - required

### `stash_write`

Write file (full content replacement).

```typescript
stash_write({ stash: "notes", path: "readme.md", content: "# Updated\n\nNew content" })
→ { success: true }
```

Parameters:
- `stash: string` - required
- `path: string` - required
- `content: string` - required, full file content

Creates file if doesn't exist.

### `stash_edit`

Edit file with text replacement.

```typescript
stash_edit({
  stash: "notes",
  path: "readme.md",
  old_string: "# My Notes",
  new_string: "# Project Notes"
})
→ { success: true }
```

Parameters:
- `stash: string` - required
- `path: string` - required
- `old_string: string` - required, must be unique in file
- `new_string: string` - required

Errors if `old_string` not found or not unique.

### `stash_delete`

Delete file.

```typescript
stash_delete({ stash: "notes", path: "old-file.md" })
→ { success: true }
```

Parameters:
- `stash: string` - required
- `path: string` - required

### `stash_move`

Move or rename file.

```typescript
stash_move({ stash: "notes", from: "old.md", to: "new.md" })
→ { success: true }
```

Parameters:
- `stash: string` - required
- `from: string` - required
- `to: string` - required

### `stash_grep`

Search file contents with regex.

```typescript
stash_grep({ stash: "notes", pattern: "TODO" })
→ {
  matches: [
    { path: "readme.md", line: 12, content: "// TODO: fix this" },
    { path: "docs/api.md", line: 45, content: "TODO: document params" }
  ]
}

// With glob filter
stash_grep({ stash: "notes", pattern: "function \\w+", glob: "**/*.ts" })
→ {
  matches: [
    { path: "src/index.ts", line: 5, content: "function init() {" },
    ...
  ]
}
```

Parameters:
- `stash: string` - required
- `pattern: string` - required, regex pattern
- `glob?: string` - optional, filter files by glob pattern

---

## StashReconciler

### Overview

The reconciler bridges filesystem ↔ automerge. Detects file changes, applies as CRDT operations, triggers sync.

### Technology

Use `chokidar` for cross-platform file watching:
```
npm install chokidar
```

### Reconciler Class

```typescript
// src/core/reconciler.ts
import chokidar from 'chokidar';
import * as Automerge from '@automerge/automerge';

// Snapshot of automerge doc at time of last disk write
interface DiskSnapshot {
  doc: Automerge.Doc<FileDoc>;
  content: string;  // text content (for text files)
}

class StashReconciler {
  private fsWatcher: chokidar.FSWatcher;
  private stash: Stash;
  private writing = false;  // loop prevention flag

  // Track what we last wrote to disk, for correct diffing during concurrent edits
  private diskSnapshots: Map<string, DiskSnapshot> = new Map();

  constructor(stash: Stash) {
    this.stash = stash;
    this.fsWatcher = chokidar.watch(stash.path, {
      ignored: [/\.stash\//, /[\/\\]\./],  // ignore .stash directory and symlinks
      ignoreInitial: true,            // don't fire for existing files
      followSymlinks: false,          // don't follow symlinks
      awaitWriteFinish: {             // debounce rapid writes
        stabilityThreshold: 200,
        pollInterval: 50
      }
    });

    this.fsWatcher.on('add', (path) => this.onFileCreated(path));
    this.fsWatcher.on('change', (path) => this.onFileModified(path));
    this.fsWatcher.on('unlink', (path) => this.onFileDeleted(path));

    // Initialize snapshots from current state
    this.initializeSnapshots();
  }

  private async initializeSnapshots(): Promise<void> {
    for (const [relativePath, docId] of this.stash.listAllFiles()) {
      const doc = this.stash.getFileDoc(docId);
      if (doc && doc.type === 'text') {
        this.diskSnapshots.set(relativePath, {
          doc: Automerge.clone(doc),
          content: doc.content.toString(),
        });
      }
    }
  }
}
```

### Event Handling

#### File Modified (Snapshot-Based)

To handle concurrent local and remote edits correctly, we diff against the snapshot (what we last wrote to disk), apply patches to a fork of that snapshot, then merge with current automerge state.

```typescript
async onFileModified(filePath: string): Promise<void> {
  if (this.writing) return;  // ignore our own writes

  const relativePath = this.getRelativePath(filePath);
  const diskContent = await fs.readFile(filePath, 'utf-8');

  const snapshot = this.diskSnapshots.get(relativePath);
  if (!snapshot) {
    // File not tracked yet, treat as create
    return this.onFileCreated(filePath);
  }

  if (diskContent === snapshot.content) return;  // no-op

  // Fork from snapshot (the state user was editing from)
  const userDoc = Automerge.clone(snapshot.doc);

  // Compute user's changes relative to snapshot (indices are correct)
  const patches = this.computeDiff(snapshot.content, diskContent);
  this.applyPatchesToDoc(userDoc, patches);

  // Merge user's branch with current automerge (which may have remote changes)
  const currentDoc = this.stash.getFileDoc(relativePath);
  const mergedDoc = Automerge.merge(currentDoc, userDoc);

  // Update stash with merged doc
  this.stash.setFileDoc(relativePath, mergedDoc);

  // Write merged result to disk and update snapshot
  const mergedContent = mergedDoc.content.toString();
  this.writing = true;
  await fs.writeFile(filePath, mergedContent);
  this.writing = false;

  this.diskSnapshots.set(relativePath, {
    doc: Automerge.clone(mergedDoc),
    content: mergedContent,
  });

  await this.stash.save();
  this.stash.scheduleSync();
}
```

#### File Created (with Rename Detection)

```typescript
async onFileCreated(filePath: string): Promise<void> {
  if (this.writing) return;

  const relativePath = this.getRelativePath(filePath);
  const content = await fs.readFile(filePath);
  const isText = this.isUtf8(content);
  const contentHash = isText
    ? this.hashContent(content.toString('utf-8'))
    : this.hashBuffer(content);

  // Check if this matches a pending delete (rename detection)
  // Require BOTH content match AND same basename to avoid false positives
  const pending = this.pendingDeletes.get(contentHash);
  const sameBasename = pending && path.basename(pending.path) === path.basename(relativePath);

  if (pending && sameBasename) {
    // It's a rename! Cancel the delete timer, reuse docId
    clearTimeout(pending.timer);
    this.pendingDeletes.delete(contentHash);

    // Update structure: remove old path, add new path with same docId
    this.stash.renameFile(pending.path, relativePath);

    // Update snapshot for new path
    const snapshot = this.diskSnapshots.get(pending.path);
    if (snapshot) {
      this.diskSnapshots.delete(pending.path);
      this.diskSnapshots.set(relativePath, snapshot);
    }

    await this.stash.save();
    this.stash.scheduleSync();
    return;
  }

  // Not a rename, create new file
  if (isText) {
    const textContent = content.toString('utf-8');
    this.stash.write(relativePath, textContent);

    // Initialize snapshot for new file
    const doc = this.stash.getFileDoc(relativePath);
    this.diskSnapshots.set(relativePath, {
      doc: Automerge.clone(doc),
      content: textContent,
    });
  } else {
    this.stash.writeBinary(relativePath, content);
  }

  await this.stash.save();
  this.stash.scheduleSync();
}
```

#### File Deleted (with Rename Detection)

The filesystem watcher sees renames as separate `unlink` + `add` events. To detect renames and preserve docIds (important for concurrent remote edits), we buffer deletes for 500ms and match against subsequent adds.

```typescript
// Pending deletes waiting for potential rename match
private pendingDeletes = new Map<string, {
  path: string;
  docId: string;
  contentHash: string;
  timer: NodeJS.Timeout;
}>();

async onFileDeleted(filePath: string): Promise<void> {
  if (this.writing) return;

  const relativePath = this.getRelativePath(filePath);
  const docId = this.stash.getDocId(relativePath);
  if (!docId) return;  // File wasn't tracked

  const fileDoc = this.stash.getFileDoc(docId);
  const contentHash = fileDoc.type === 'binary'
    ? fileDoc.hash
    : this.hashContent(this.stash.read(relativePath));

  // Buffer the delete for 500ms to detect renames
  const timer = setTimeout(() => {
    this.finalizeDelete(relativePath, docId, fileDoc);
    this.pendingDeletes.delete(contentHash);
  }, 500);

  this.pendingDeletes.set(contentHash, { path: relativePath, docId, contentHash, timer });
}

private async finalizeDelete(relativePath: string, docId: string, fileDoc: FileDoc): Promise<void> {
  // Actually remove from structure
  this.stash.delete(relativePath);

  // GC binary blobs immediately if unreferenced
  if (fileDoc.type === 'binary') {
    const stillReferenced = this.stash.isHashReferenced(fileDoc.hash);
    if (!stillReferenced) {
      await fs.unlink(path.join(this.stash.path, '.stash', 'blobs', `${fileDoc.hash}.bin`));
    }
  }
  // Text docs: keep orphaned in .stash/docs/ (no GC)

  await this.stash.save();
  this.stash.scheduleSync();
}
```

### Diffing Algorithm

Use `fast-diff` for character-level diffing:

```
npm install fast-diff
```

```typescript
import diff from 'fast-diff';

computeDiff(oldText: string, newText: string): Patch[] {
  const changes = diff(oldText, newText);
  const patches: Patch[] = [];
  let index = 0;

  for (const [op, text] of changes) {
    if (op === diff.EQUAL) {
      index += text.length;
    } else if (op === diff.DELETE) {
      patches.push({ type: 'delete', index, count: text.length });
      // Don't advance index - deleted text is gone
    } else if (op === diff.INSERT) {
      patches.push({ type: 'insert', index, text });
      index += text.length;
    }
  }

  return patches;
}
```

### Applying Patches to Automerge

```typescript
// Apply patches to a specific doc (used in snapshot-based merge)
applyPatchesToDoc(doc: Automerge.Doc<TextFileDoc>, patches: Patch[]): void {
  Automerge.change(doc, (d) => {
    let offset = 0;

    for (const patch of patches) {
      const adjustedIndex = patch.index + offset;

      if (patch.type === 'delete') {
        d.content.deleteAt(adjustedIndex, patch.count);
        offset -= patch.count;
      } else if (patch.type === 'insert') {
        d.content.insertAt(adjustedIndex, ...patch.text.split(''));
        offset += patch.text.length;
      }
    }
  });
}
```

### Reverse Flow (Remote → Files)

When remote sync brings in changes, flush automerge state to disk:

```typescript
async flush(): Promise<void> {
  this.writing = true;  // suppress filesystem watcher

  try {
    const files = this.stash.listAllFiles();

    for (const [filePath, docId] of files) {
      const doc = this.stash.getFileDoc(docId);
      if (!doc) continue;

      const diskPath = path.join(this.stash.path, filePath);
      await fs.mkdir(path.dirname(diskPath), { recursive: true });

      if (doc.type === 'text') {
        const automergeContent = doc.content.toString();

        // Only write if different from current disk
        let diskContent: string | null = null;
        try {
          diskContent = await fs.readFile(diskPath, 'utf-8');
        } catch {
          // File doesn't exist on disk
        }

        if (diskContent !== automergeContent) {
          await fs.writeFile(diskPath, automergeContent);
        }

        // Always update snapshot to current automerge state
        this.diskSnapshots.set(filePath, {
          doc: Automerge.clone(doc),
          content: automergeContent,
        });

      } else if (doc.type === 'binary') {
        // Binary: copy from blobs if hash changed
        const blobPath = path.join(this.stash.path, '.stash', 'blobs', `${doc.hash}.bin`);
        const currentHash = await this.hashFile(diskPath).catch(() => null);
        if (currentHash !== doc.hash) {
          await fs.copyFile(blobPath, diskPath);
        }
      }
    }

    // Handle deletions: files on disk not in automerge
    await this.cleanupOrphanedFiles();

  } finally {
    this.writing = false;
  }

  // Re-check for changes made during write window
  await this.reconcile();
}
```

### Binary File Detection and Hashing

```typescript
isUtf8(buffer: Buffer): boolean {
  try {
    const text = buffer.toString('utf-8');
    // Check for replacement character indicating invalid UTF-8
    return !text.includes('\uFFFD');
  } catch {
    return false;
  }
}

hashBuffer(buffer: Buffer): string {
  return crypto.createHash('sha256').update(buffer).digest('hex');
}

hashContent(content: string): string {
  return crypto.createHash('sha256').update(content, 'utf-8').digest('hex');
}

async hashFile(filePath: string): Promise<string> {
  const content = await fs.readFile(filePath);
  return this.hashBuffer(content);
}
```

### Binary File Handling

Binary files don't store content in automerge - just metadata:
- Compute SHA-256 hash of file content
- Store `{ type: 'binary', hash, size }` in automerge
- Actual bytes stored in `.stash/blobs/<hash>.bin` (content-addressed)
- Readable file at stash root is a copy

**Storage:**
```typescript
async writeBinary(relativePath: string, content: Buffer): Promise<void> {
  const hash = this.hashBuffer(content);
  const size = content.length;

  // Store blob content-addressed
  const blobPath = path.join(this.path, '.stash', 'blobs', `${hash}.bin`);
  await fs.mkdir(path.dirname(blobPath), { recursive: true });
  await fs.writeFile(blobPath, content);

  // Update automerge with metadata
  this.stash.setBinaryMeta(relativePath, { type: 'binary', hash, size });
}
```

**Sync:**
- On sync, use `sync()` with `blob:<hash>` keys for binary content
- If hash differs from remote, transfer whole blob
- LWW: conflicting hashes resolved by timestamp/actor tiebreaker

**Garbage Collection:**
- **Text docs**: No GC. Orphaned docs stay in `.stash/docs/` indefinitely.
- **Binary blobs**: Immediate GC. When a binary file is deleted and no other file references the same hash, delete from `.stash/blobs/`.

This avoids bloating automerge with large binary data while ensuring binary blobs don't accumulate.

### Type Changes (Binary ↔ Text)

When a file changes type (e.g., text file replaced with binary content):

```typescript
async onFileModified(filePath: string): Promise<void> {
  // ... existing code ...

  const content = await fs.readFile(filePath);
  const isText = this.isUtf8(content);
  const currentDoc = this.stash.getFileDoc(relativePath);

  // Check for type change
  if (currentDoc && currentDoc.type === 'text' && !isText) {
    // Text → Binary: replace doc entirely
    this.diskSnapshots.delete(relativePath);
    this.stash.writeBinary(relativePath, content);
    await this.stash.save();
    this.stash.scheduleSync();
    return;
  }

  if (currentDoc && currentDoc.type === 'binary' && isText) {
    // Binary → Text: replace doc entirely
    const textContent = content.toString('utf-8');
    this.stash.write(relativePath, textContent);
    const newDoc = this.stash.getFileDoc(relativePath);
    this.diskSnapshots.set(relativePath, {
      doc: Automerge.clone(newDoc),
      content: textContent,
    });
    await this.stash.save();
    this.stash.scheduleSync();
    return;
  }

  // ... continue with normal diff logic for same-type changes ...
}
```

### Case Sensitivity

On case-insensitive filesystems (macOS default, Windows), multiple remote files with case-differing names (e.g., `README.md` and `readme.md`) will map to the same local path.

**Behavior:** Last writer wins on disk. Log a warning when detected:

```typescript
async writeFilesToDisk(): Promise<void> {
  const pathsLower = new Map<string, string>();  // lowercase → actual path

  for (const [filePath, docId] of files) {
    const lower = filePath.toLowerCase();
    const existing = pathsLower.get(lower);

    if (existing && existing !== filePath) {
      console.warn(`Case conflict: "${existing}" and "${filePath}" map to same path on this filesystem`);
    }
    pathsLower.set(lower, filePath);

    // ... write file ...
  }
}
```

### Loop Prevention

Two mechanisms:
1. **Write flag**: Set `this.writing = true` during flush, ignore filesystem events
2. **Content comparison**: Always compare before applying changes (safety net)

### Ignored Patterns

Only ignore `.stash/` directory. Everything else syncs.

Future: optional `.stashignore` file if needed.

---

## Architecture: StashReconciler ↔ Stash Interface

### Separation of Concerns

**Stash** is a pure data container - manages automerge docs, structure, persistence.
**StashReconciler** bridges filesystem ↔ Stash - watches files, computes diffs, applies operations.
**Daemon** owns reconcilers - lifecycle management, coordinates sync across stashes.

```
┌───────────────────────────────────────────────────────────────┐
│                           Daemon                               │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐      │
│  │StashReconciler│  │StashReconciler│  │StashReconciler│      │
│  │    (notes)    │  │    (work)     │  │   (shared)    │      │
│  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘      │
│          │                  │                  │               │
│  ┌───────▼───────┐  ┌───────▼───────┐  ┌───────▼───────┐      │
│  │     Stash     │  │     Stash     │  │     Stash     │      │
│  │    (notes)    │  │    (work)     │  │   (shared)    │      │
│  └───────────────┘  └───────────────┘  └───────────────┘      │
└───────────────────────────────────────────────────────────────┘
```

### StashReconciler Public API

```typescript
// src/core/reconciler.ts

class StashReconciler {
  constructor(stash: Stash);

  /** Start watching filesystem. Call after construction. */
  start(): Promise<void>;

  /** Stop watching. Call before disposing. */
  close(): Promise<void>;

  /**
   * Flush automerge state to disk.
   * Called by daemon after remote sync brings in changes.
   * Updates snapshots to match current automerge state.
   */
  flush(): Promise<void>;
}
```

### Stash Interface (required by StashReconciler)

```typescript
// src/core/stash.ts

interface IStash {
  readonly name: string;
  readonly path: string;  // absolute path to stash root

  // --- File enumeration ---
  listAllFiles(): Iterable<[relativePath: string, docId: string]>;

  // --- Doc access ---
  getDocId(relativePath: string): string | null;
  getFileDoc(docId: string): FileDoc | null;
  getFileDocByPath(relativePath: string): FileDoc | null;

  // --- Read content ---
  read(relativePath: string): string;  // throws if not found or binary

  // --- Write operations ---
  write(relativePath: string, content: string): void;
  writeBinary(relativePath: string, content: Buffer): void;
  delete(relativePath: string): void;
  renameFile(oldPath: string, newPath: string): void;

  // --- Doc manipulation (for snapshot-based merge) ---
  setFileDoc(relativePath: string, doc: Automerge.Doc<FileDoc>): void;
  cloneFileDoc(docId: string): Automerge.Doc<FileDoc>;

  // --- Binary support ---
  isHashReferenced(hash: string): boolean;
  setBinaryMeta(relativePath: string, meta: BinaryFileDoc): void;

  // --- Persistence ---
  save(): Promise<void>;
  scheduleSync(): void;
}
```

### Data Flow

**Local edit detected:**
```
1. chokidar emits 'change' event
2. StashReconciler.onFileModified(path)
3. Read disk content
4. Get snapshot (what we last wrote)
5. Compute diff(snapshot.content, diskContent)
6. Fork snapshot.doc, apply patches
7. Merge forked doc with current Stash doc
8. Write merged content to disk
9. Update snapshot
10. Stash.save() + Stash.scheduleSync()
```

**Remote sync arrives:**
```
1. Daemon's sync loop calls stash.sync() → pulls from remote
2. Stash merges remote changes into automerge
3. Daemon calls reconciler.flush()
4. StashReconciler writes automerge state to disk
5. StashReconciler updates all snapshots to current state
```

### Critical Invariants

These must ALWAYS hold to prevent data corruption:

1. **Snapshot = disk**: `diskSnapshots[path].content` always equals what's on disk at `path`
2. **Snapshot ≤ automerge**: snapshot doc is an ancestor of (or equal to) current automerge doc
3. **No orphan snapshots**: if a file is deleted, its snapshot is removed
4. **Atomic writes**: disk writes and snapshot updates happen together
5. **Single writer**: only StashReconciler writes to disk (MCP tools → disk → reconciler detects)

---

## Core Module Changes

### Stash Class (`src/core/stash.ts`)

#### Constructor Changes

```typescript
class Stash {
  readonly name: string;
  readonly path: string;  // absolute path to stash root
  // Note: Stash does NOT own reconciler - Daemon does

  constructor(
    name: string,
    stashPath: string,  // full path, not baseDir
    structureDoc: Automerge.Doc<StructureDoc>,
    fileDocs: Map<string, Automerge.Doc<FileDoc>>,
    meta: StashMeta,
    provider: SyncProvider | null = null,
  ) {
    this.name = name;
    this.path = stashPath;
    // ...
  }
}
```

#### save() Changes

Write readable files to root, automerge to `.stash/`:

```typescript
async save(): Promise<void> {
  const stashDir = path.join(this.path, '.stash');
  const docsDir = path.join(stashDir, 'docs');
  await fs.mkdir(docsDir, { recursive: true });

  // Write meta.json
  await atomicWrite(
    path.join(stashDir, 'meta.json'),
    JSON.stringify(this.meta, null, 2) + '\n'
  );

  // Write structure doc
  await atomicWriteBinary(
    path.join(stashDir, 'structure.automerge'),
    Automerge.save(this.structureDoc)
  );

  // Write file docs
  for (const [docId, doc] of this.fileDocs) {
    await atomicWriteBinary(
      path.join(docsDir, `${docId}.automerge`),
      Automerge.save(doc)
    );
  }

  // Write readable files (delegated to reconciler.flush())
}
```

#### load() Changes

Read automerge from `.stash/`:

```typescript
static async load(
  name: string,
  stashPath: string,
  provider: SyncProvider | null = null,
): Promise<Stash> {
  const stashDir = path.join(stashPath, '.stash');

  const metaData = await fs.readFile(path.join(stashDir, 'meta.json'), 'utf-8');
  const meta: StashMeta = JSON.parse(metaData);

  const structureBin = await fs.readFile(path.join(stashDir, 'structure.automerge'));
  const structureDoc = Automerge.load<StructureDoc>(new Uint8Array(structureBin));

  // ... load file docs from .stash/docs/
}
```

#### New Methods for Watcher

```typescript
// Get docId for a file path (from structure doc)
getDocId(relativePath: string): string | null;

// Get file doc by docId
getFileDoc(docId: string): FileDoc | null;

// Rename file: update structure to point old path's docId to new path
renameFile(oldPath: string, newPath: string): void;

// Check if any file in structure references this hash
isHashReferenced(hash: string): boolean;

// Set binary metadata in structure
setBinaryMeta(relativePath: string, meta: BinaryFileDoc): void;
```

### Manager Class (`src/core/manager.ts`)

Read stash paths from global config:

```typescript
static async load(): Promise<StashManager> {
  const config = await ensureConfig();
  const stashes = new Map<string, Stash>();

  for (const [name, stashPath] of Object.entries(config.stashes)) {
    try {
      const stash = await Stash.load(name, stashPath, /* provider */);
      stashes.set(name, stash);
    } catch (err) {
      console.warn(`Failed to load stash ${name}: ${err}`);
    }
  }

  return new StashManager(stashes, config);
}
```

---

## Daemon Changes

### Reconciler Integration

```typescript
// src/daemon.ts
class Daemon {
  private reconcilers: Map<string, StashReconciler> = new Map();
  private syncInterval: NodeJS.Timeout;

  async start(): Promise<void> {
    const manager = await StashManager.load();

    for (const [name, stash] of manager.stashes) {
      const reconciler = new StashReconciler(stash);
      await reconciler.start();
      this.reconcilers.set(name, reconciler);
    }

    // Periodic sync as fallback
    this.syncInterval = setInterval(() => this.syncAll(), 30000);
  }

  async stop(): Promise<void> {
    for (const reconciler of this.reconcilers.values()) {
      await reconciler.close();
    }
    this.reconcilers.clear();
    clearInterval(this.syncInterval);
  }

  // Called after remote sync completes for a stash
  async onSyncComplete(name: string): Promise<void> {
    const reconciler = this.reconcilers.get(name);
    if (reconciler) {
      await reconciler.flush();
    }
  }

  // Called when a stash is deleted while daemon is running
  async removeStash(name: string): Promise<void> {
    const reconciler = this.reconcilers.get(name);
    if (reconciler) {
      await reconciler.close();
      this.reconcilers.delete(name);
    }
  }

  // Called when a new stash is created while daemon is running
  async addStash(stash: Stash): Promise<void> {
    if (this.reconcilers.has(stash.name)) {
      await this.removeStash(stash.name);
    }
    const reconciler = new StashReconciler(stash);
    await reconciler.start();
    this.reconcilers.set(stash.name, reconciler);
  }
}
```

### Reconciler Lifecycle

```typescript
// src/core/reconciler.ts
class StashReconciler {
  async start(): Promise<void> {
    await this.initializeSnapshots();
    // chokidar already started in constructor, but could defer to here
  }

  async close(): Promise<void> {
    await this.fsWatcher.close();
    this.diskSnapshots.clear();
    this.pendingDeletes.clear();
  }
}
```

When `stash delete` is called, it must notify the daemon:
- If daemon is running (check PID file), send IPC message
- Daemon calls `removeStash(name)` to clean up

---

## Migration

### Automatic Migration

On load, detect old format and migrate:

```typescript
async function migrateIfNeeded(stashPath: string): Promise<void> {
  const newFormat = path.join(stashPath, '.stash', 'meta.json');
  const oldFormat = path.join(stashPath, 'meta.json');

  // Check if already new format
  if (await exists(newFormat)) return;

  // Check if old format
  if (!await exists(oldFormat)) return;

  console.log(`Migrating stash at ${stashPath}...`);

  // Create .stash directory
  const stashDir = path.join(stashPath, '.stash');
  await fs.mkdir(path.join(stashDir, 'docs'), { recursive: true });

  // Move automerge files
  await fs.rename(
    path.join(stashPath, 'structure.automerge'),
    path.join(stashDir, 'structure.automerge')
  );

  // Move docs
  const oldDocs = path.join(stashPath, 'docs');
  if (await exists(oldDocs)) {
    const files = await fs.readdir(oldDocs);
    for (const file of files) {
      await fs.rename(
        path.join(oldDocs, file),
        path.join(stashDir, 'docs', file)
      );
    }
    await fs.rmdir(oldDocs);
  }

  // Migrate meta.json format
  const oldMeta = JSON.parse(await fs.readFile(oldFormat, 'utf-8'));
  const newMeta: StashMeta = {
    name: oldMeta.localName,
    description: oldMeta.description,
    remote: oldMeta.key,
  };
  await fs.writeFile(
    path.join(stashDir, 'meta.json'),
    JSON.stringify(newMeta, null, 2) + '\n'
  );
  await fs.unlink(oldFormat);

  // Write readable files from automerge state
  // (done after loading)
}
```

### Global Config Migration

Migrate stashes to global config:

```typescript
async function migrateGlobalConfig(): Promise<void> {
  const config = await readConfig();

  // If stashes already populated, skip
  if (Object.keys(config.stashes ?? {}).length > 0) return;

  // Find stashes in default location
  const defaultDir = DEFAULT_STASH_DIR;
  const entries = await fs.readdir(defaultDir, { withFileTypes: true });

  const stashes: Record<string, string> = {};
  for (const entry of entries) {
    if (!entry.isDirectory()) continue;
    if (entry.name === 'config.json') continue;

    const stashPath = path.join(defaultDir, entry.name);
    // Check if it's a valid stash
    if (await exists(path.join(stashPath, '.stash', 'meta.json')) ||
        await exists(path.join(stashPath, 'meta.json'))) {
      stashes[entry.name] = stashPath;
    }
  }

  config.stashes = stashes;

  // Migrate actorId from first stash if needed
  if (!config.actorId) {
    for (const stashPath of Object.values(stashes)) {
      const metaPath = path.join(stashPath, 'meta.json');
      try {
        const meta = JSON.parse(await fs.readFile(metaPath, 'utf-8'));
        if (meta.actorId) {
          config.actorId = meta.actorId;
          break;
        }
      } catch {}
    }
    config.actorId = config.actorId ?? ulid();
  }

  await writeConfig(config);
}
```

---

## Tests

### New Test Files

#### `test/core/reconciler.test.ts`

```typescript
describe('StashReconciler', () => {
  describe('file change detection', () => {
    it('should detect file creation');
    it('should detect file modification');
    it('should detect file deletion');
    it('should detect file rename/move');
    it('should ignore .stash directory');
  });

  describe('rename detection', () => {
    it('should detect rename via content hash match within 500ms');
    it('should require same basename for rename detection');
    it('should treat as delete+create if basenames differ');
    it('should preserve docId on rename');
    it('should handle rename with concurrent remote edit');
    it('should treat as delete+create if content changed during rename');
    it('should treat as delete+create if add comes after 500ms');
    it('should update diskSnapshots on rename');
  });

  describe('garbage collection', () => {
    it('should keep orphaned text docs in .stash/docs/');
    it('should delete unreferenced binary blobs from .stash/blobs/');
    it('should not delete binary blob if still referenced by another file');
  });

  describe('diffing', () => {
    it('should compute character-level diff');
    it('should handle insertion');
    it('should handle deletion');
    it('should handle replacement');
    it('should handle multiple changes');
    it('should handle empty file');
    it('should handle large files');
    it('should correctly track offset across multiple patches');
    // Critical: verify index adjustment when applying patches
    // e.g., "abc" → "aXXc" (delete 'b' at 1, insert 'XX' at 1)
    // Patch 1: delete at index 1, offset becomes -1
    // Patch 2: insert at index 1, adjusted to 1 + (-1) = 0? No...
    // Need to verify the math is correct for all combinations
  });

  describe('automerge integration', () => {
    it('should apply diff patches to automerge doc');
    it('should preserve automerge history');
    it('should handle concurrent changes from filesystem and remote');
  });

  describe('snapshot-based merge', () => {
    it('should diff against snapshot, not current automerge');
    it('should fork from snapshot and merge with current');
    it('should update snapshot after writing to disk');
    it('should initialize snapshots on reconciler start');
    it('should handle concurrent local edit + remote sync');
    // Critical: verify user edit "ABC"→"ABCD" + remote "ABC"→"ABXC" = "ABXCD"
  });

  describe('type changes', () => {
    it('should detect text → binary change');
    it('should detect binary → text change');
    it('should replace doc entirely on type change');
    it('should update snapshot on text → binary');
    it('should create snapshot on binary → text');
  });

  describe('case sensitivity', () => {
    it('should warn on case-conflicting paths');
    it('should let filesystem decide on case-insensitive systems');
  });

  describe('loop prevention', () => {
    it('should not trigger on own writes');
    it('should handle changes during write window');
    it('should reconcile after write completes');
  });

  describe('binary files', () => {
    it('should detect binary files');
    it('should store in .stash/blobs/ content-addressed');
    it('should use full replacement for binary');
    it('should sync binary via blob:<hash> keys');
  });

  describe('symlinks', () => {
    it('should ignore symlinks in stash directory');
    it('should not follow symlinks');
  });

  describe('debouncing', () => {
    it('should debounce rapid file changes');
    it('should process after stabilization');
  });

  // CRITICAL: Data integrity tests - these MUST pass
  describe('data integrity', () => {
    describe('no data loss scenarios', () => {
      it('should not lose local edits when remote sync arrives');
      it('should not lose remote edits when local edit happens');
      it('should preserve both edits in concurrent edit scenario');
      it('should not corrupt file on crash mid-write');
      it('should recover gracefully from incomplete sync');
      it('should not lose data on rename + concurrent remote edit');
    });

    describe('merge correctness', () => {
      // Test: local "ABC"→"ABCD", remote "ABC"→"ABXC", result "ABXCD"
      it('should merge concurrent insertions at different positions');
      // Test: local "ABC"→"AC", remote "ABC"→"ABX", result "AXC" or similar
      it('should merge concurrent deletion + insertion');
      // Test: both insert at same position
      it('should handle concurrent insertions at same position deterministically');
      // Test: both delete same text
      it('should handle concurrent deletions of same text');
      // Test: one deletes text other edits
      it('should handle delete of text being edited');
    });

    describe('snapshot consistency', () => {
      it('should never have snapshot diverge from what is on disk');
      it('should update snapshot atomically with disk write');
      it('should handle snapshot for new files');
      it('should clean up snapshot on file delete');
    });

    describe('atomic operations', () => {
      it('should use atomic writes for automerge docs');
      it('should use atomic writes for config files');
      it('should not leave partial state on interrupt');
    });
  });
});
```

#### `test/core/config.test.ts` (additions)

```typescript
describe('Global Config', () => {
  describe('ensureConfig', () => {
    it('should create config on first run');
    it('should generate actorId');
    it('should initialize empty stashes map');
    it('should preserve existing config');
    it('should add actorId to existing config without one');
  });

  describe('stash registry', () => {
    it('should add stash to registry');
    it('should remove stash from registry');
    it('should handle stashes at custom paths');
  });
});
```

#### `test/cli/commands.test.ts`

```typescript
describe('CLI Commands', () => {
  describe('stash create', () => {
    it('should create stash at default path');
    it('should create stash at custom path');
    it('should set description');
    it('should register in global config');
    it('should prompt for remote provider');
  });

  describe('stash connect', () => {
    it('should connect to github remote');
    it('should pull initial content');
    it('should register in global config');
  });

  describe('stash edit', () => {
    it('should update description');
    it('should update remote');
    it('should disconnect with --remote none');
  });

  describe('stash delete', () => {
    it('should prompt for confirmation');
    it('should skip prompt with --force');
    it('should delete local files');
    it('should remove from global config');
    it('should delete remote with --remote flag');
  });

  describe('stash link', () => {
    it('should create symlink with explicit path');
    it('should create symlink with stash name as default');
    it('should read .stash.json and create all links');
    it('should handle missing stash');
    it('should error if path already exists as file');
    it('should error if path already exists as directory');
    it('should error if path already exists as symlink');
  });

  describe('stash unlink', () => {
    it('should remove symlink at explicit path');
    it('should read .stash.json and remove all links');
    it('should error if path is not a symlink');
    it('should error if path does not exist');
  });

  describe('stash (bare command)', () => {
    it('should start daemon');
    it('should create symlinks from .stash.json');
    it('should handle missing .stash.json');
  });
});
```

#### `test/mcp/tools.test.ts` (updates)

```typescript
describe('MCP Tools', () => {
  describe('stash_list', () => {
    it('should list stashes with name, description, path');
    it('should list files in stash root');
    it('should list files in subdirectory');
  });

  describe('stash_read', () => {
    it('should read file from disk');
    it('should error for missing file');
  });

  describe('stash_write', () => {
    it('should write file to disk');
    it('should create new file');
    it('should overwrite existing file');
  });

  describe('stash_edit', () => {
    it('should replace matching text');
    it('should error if old_string not found');
    it('should error if old_string not unique');
  });

  describe('stash_delete', () => {
    it('should delete file from disk');
    it('should error for missing file');
  });

  describe('stash_move', () => {
    it('should move file on disk');
    it('should rename file');
    it('should error for missing source');
  });

  describe('stash_glob', () => {
    it('should find files matching pattern');
    it('should return empty array for no matches');
  });

  describe('stash_grep', () => {
    it('should search file contents with regex');
    it('should return matching lines with path and line number');
    it('should filter by glob pattern');
    it('should return empty array for no matches');
    it('should skip binary files');
  });
});
```

#### `test/migration.test.ts`

```typescript
describe('Migration', () => {
  describe('stash format migration', () => {
    it('should migrate automerge files to .stash/');
    it('should migrate meta.json format');
    it('should write readable files');
    it('should preserve file content');
    it('should handle already-migrated stash');
  });

  describe('global config migration', () => {
    it('should discover existing stashes');
    it('should populate stashes map');
    it('should migrate actorId from stash meta');
    it('should generate actorId if none exists');
  });
});
```

### Integration Tests

#### `test/integration/sync.test.ts`

```typescript
describe('End-to-end sync', () => {
  it('should sync file creation to remote');
  it('should sync file edit to remote');
  it('should sync file deletion to remote');
  it('should pull remote changes to disk');
  it('should merge concurrent edits');
  it('should handle offline edits');
});
```

#### `test/integration/concurrent.test.ts`

Critical tests for concurrent edit scenarios. Run with real timers to simulate actual race conditions.

```typescript
describe('Concurrent edit scenarios', () => {
  // These tests simulate real-world timing issues

  describe('local edit during remote sync', () => {
    it('should not lose local edit when remote sync writes to disk');
    it('should not lose remote edit when local edit is being processed');
    it('should correctly merge overlapping edits');
  });

  describe('rapid editing', () => {
    it('should handle rapid consecutive edits');
    it('should coalesce debounced edits correctly');
    it('should not drop edits during high-frequency changes');
  });

  describe('rename during sync', () => {
    it('should handle rename while remote sync in progress');
    it('should preserve content on rename + concurrent edit');
  });

  describe('multi-file operations', () => {
    it('should handle multiple files edited simultaneously');
    it('should sync files in consistent order');
  });
});
```

#### `test/integration/recovery.test.ts`

Tests for crash recovery and data integrity after failures.

```typescript
describe('Crash recovery', () => {
  describe('incomplete writes', () => {
    it('should recover from crash during automerge save');
    it('should recover from crash during disk write');
    it('should detect and repair inconsistent state');
  });

  describe('snapshot recovery', () => {
    it('should rebuild snapshots from disk on restart');
    it('should handle missing snapshot gracefully');
    it('should detect snapshot/disk mismatch');
  });

  describe('remote sync recovery', () => {
    it('should resume interrupted sync');
    it('should not duplicate changes after retry');
  });
});
```

#### `test/integration/stress.test.ts`

Stress tests to ensure stability under load.

```typescript
describe('Stress tests', () => {
  it('should handle 1000 rapid edits without data loss');
  it('should handle 100 concurrent file creates');
  it('should handle large files (10MB+) correctly');
  it('should handle deep directory structures (50+ levels)');
  it('should handle many files (10000+) in single stash');
  it('should not leak memory during extended operation');
});
```

---

## Dependencies

### New Dependencies

```json
{
  "dependencies": {
    "chokidar": "^3.6.0",
    "fast-diff": "^1.3.0",
    "@inquirer/prompts": "^5.0.0"
  }
}
```

### Existing Dependencies (no changes)

- `@automerge/automerge`
- `commander`
- `minimatch`
- `ulid`
- `@modelcontextprotocol/sdk`
- `crypto` (Node.js built-in, for SHA-256 hashing)

---

## Open Decisions

None - all decisions finalized:

- ✅ Use `.stash/` for automerge data (not `.sync/`)
- ✅ Keep full MCP tool set
- ✅ Use `stash_edit` with old_string/new_string
- ✅ Use `stash_delete`/`stash_move` (verbose names)
- ✅ Binary files: hash+size in automerge, bytes in `.stash/blobs/`, LWW on conflict
- ✅ Only ignore `.stash/` directory and symlinks
- ✅ Use chokidar for file watching (followSymlinks: false)
- ✅ Use fast-diff for diffing
- ✅ Use @inquirer/prompts for CLI UX
- ✅ Use `providers: { github: ... }` config structure
- ✅ `stash link` errors if path exists
- ✅ Include `stash unlink` command
- ✅ Rename detection: buffer deletes 500ms, match adds by content hash + same basename
- ✅ Text doc GC: none (keep orphaned docs)
- ✅ Binary blob GC: immediate when unreferenced
- ✅ Snapshot-based diffing for correct concurrent edit handling
- ✅ Type changes (binary ↔ text): replace doc entirely
- ✅ Case conflicts: warn and let filesystem decide (LWW on disk)
- ✅ StashReconciler (not StashWatcher) - bridges filesystem ↔ automerge
- ✅ Stash is pure data, Daemon owns reconcilers
