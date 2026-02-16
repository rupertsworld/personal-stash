# Stash Specification

A local-first collaborative folder service with MCP interface for AI agents and humans.

## Overview

Stash provides synced folders ("stashes") that multiple AI agents and humans can collaborate on. It uses Automerge CRDTs for automatic conflict-free merging of concurrent edits.

### Why Automerge?

Automerge is not optional - it's fundamental to how Stash works. The documents **must** be Automerge docs because:

1. **Conflict-free merging** - Multiple users/agents can edit concurrently. Automerge merges automatically, no conflicts.
2. **No coordination needed** - No locks, no "who edited last" logic. Just merge.
3. **Works with dumb storage** - GitHub is just blob storage. `Automerge.merge(local, remote)` handles everything.

Without Automerge, we'd need conflict detection, 3-way merge strategies, user intervention, or last-write-wins (losing data). Automerge gives us conflict-free sync with simple blob storage.

## Design Goals

- **Local-first**: Files live on your machine, work offline
- **Conflict-free**: Automerge CRDTs handle concurrent edits
- **Simple**: Clean, modular code; easy to understand
- **Extensible**: Swap sync providers (GitHub now, others later)
- **No infrastructure**: Users don't deploy anything (GitHub is the backend)

## Architecture

```
┌─────────────────────────────────────────┐
│          Daemon (stash start)           │
│  - Background process                   │
│  - HTTP server (Express)                │
│  - Sync loop (interval-based)           │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         MCP over HTTP                   │
│  POST /mcp (Streamable HTTP transport)  │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│        MCP Tools (mcp.ts)               │
│  (read, write, list, glob, delete, move)│
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│           Stash Core                    │
│  - Structure doc (file registry)        │
│  - File docs (content per file)         │
│  - Automerge sync protocol              │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│       SyncProvider (interface)          │
│  - Automerge sync messages              │
├─────────────────────────────────────────┤
│  GitHubProvider   │  Future providers   │
└─────────────────────────────────────────┘
```

## Data Model

Each stash consists of:

1. **Structure Document** - A single Automerge document tracking all files in the stash
2. **File Documents** - Separate Automerge documents for each file's content

### Structure Document Schema

```typescript
interface StructureDoc {
  files: {
    [path: string]: {
      docId: string;      // ULID referencing the file's Automerge document
      created: number;    // Unix timestamp (ms) when file was created
    }
  }
}
```

- **Paths** are flat strings (e.g., `"docs/readme.md"`)
- **Directories** are implicit - derived from file paths (no empty directories, like git)
- **docId** is a ULID (Universally Unique Lexicographically Sortable Identifier)
- **Renames** preserve identity: delete old path key, add new path key with same `docId`

### File Document Schema

```typescript
interface FileDoc {
  content: Automerge.Text;  // The file's text content
}
```

- Text files only (no binary support in v1)
- Uses Automerge.Text for character-level CRDT merging

### Document Identity

- Each file document has a ULID as its identifier
- The ULID is generated when the file is created
- Renames/moves change the path in the structure doc but preserve the `docId`
- This allows tracking file identity across renames

### File Deletion

When a file is deleted locally:
1. Remove the path entry from the structure doc (Automerge change)
2. The file doc becomes an orphan (kept on disk for recovery, not loaded or synced)
3. On next sync, the merged structure doc won't reference the docId
4. Remote cleanup happens when provider pushes (orphan doc not included)

When a file is deleted remotely:
1. After sync, the merged structure doc won't have the path
2. The local file doc becomes orphaned (kept on disk for recovery)
3. Orphan is not loaded into memory on subsequent loads

**Concurrent delete + edit:** If one client deletes a file while another edits it:
- Automerge merge may resurrect the path (depending on timing)
- If path survives in merged structure, the file doc with edits is preserved
- If path is deleted in merged structure, the file doc becomes orphan (kept for recovery)
- This is eventually consistent - no data is silently lost during the merge window

## Storage

### Local Storage Structure

```
~/.stash/
  config.json                          # Global config (auth tokens)
  <stash-name>/
    meta.json                          # Stash metadata (provider, key)
    structure.automerge                # Structure document (binary)
    docs/
      <docId>.automerge                # File documents (binary, named by ULID)
```

No plain text files are stored locally. The Automerge documents are the source of truth.

**Example:** A stash with two files (`hello.md` and `notes/todo.md`):

```
~/.stash/
  config.json
  my-project/
    meta.json
    structure.automerge                # Contains: { files: { "hello.md": { docId: "01HXK...", created: 1699000000000 }, "notes/todo.md": { docId: "01HXM...", created: 1699000001000 } } }
    docs/
      01HXKABCD1234567890123456.automerge    # Content of hello.md
      01HXMEFGH1234567890123456.automerge    # Content of notes/todo.md
```

### meta.json

```json
{
  "localName": "my-project",
  "provider": "github",
  "key": "github:owner/repo",
  "actorId": "01HXK..."
}
```

- **actorId**: ULID generated once when stash is created. Used as Automerge actor ID for deterministic conflict resolution. Persists across sessions.

Or for local-only:

```json
{
  "localName": "my-project",
  "provider": null,
  "key": null,
  "actorId": "01HXK..."
}
```

### config.json

Global configuration stored at `~/.stash/config.json`:

```json
{
  "github": {
    "token": "ghp_xxxxxxxxxxxx"
  }
}
```

### Remote Storage (GitHub)

```
repo/
  .stash/
    structure.automerge                # Same as local
    docs/
      01HXKABCD1234567890123456.automerge
      01HXMEFGH1234567890123456.automerge
  hello.md                             # Rendered plain text (read-only)
  notes/
    todo.md                            # Rendered plain text (read-only)
```

The `.stash/` directory mirrors local storage. The plain text files at repo root are rendered snapshots for human readability on GitHub - they are regenerated on every sync. The rendered file paths match the stash paths.

**Note:** This rendering is GitHub-specific. Other providers may render differently or not at all. The `.stash/` directory with Automerge docs is the only required storage - rendering is optional and provider-dependent.

## Sync

### Architecture

Sync is simple: Stash hands all documents to the provider, provider handles sync and returns merged state.

```typescript
interface SyncProvider {
  /**
   * Sync all documents with remote storage.
   *
   * @param docs - Map of document ID to Automerge binary data.
   *               "structure" key is the structure doc, others are file docs (ULID keys).
   * @returns Map of merged documents. Must include all docs from input,
   *          plus any remote-only docs discovered during sync.
   * @throws SyncError on network/auth failures (caller handles retry)
   */
  sync(docs: Map<string, Uint8Array>): Promise<Map<string, Uint8Array>>;

  /** Check if remote storage exists */
  exists(): Promise<boolean>;

  /** Create remote storage (e.g., GitHub repo) */
  create(): Promise<void>;

  /** Delete remote storage */
  delete(): Promise<void>;
}
```

The provider owns the entire sync process:
- Fetching remote state
- Merging (using Automerge)
- Pushing merged state
- Rendering (for GitHub)
- Choosing the sync strategy (simple merge vs incremental protocol)

### Stash Sync Flow

```typescript
async function sync() {
  // 1. Pre-sync consistency check: create empty docs for any dangling refs
  for (const [path, { docId }] of Object.entries(this.structureDoc.files)) {
    if (!this.fileDocs.has(docId)) {
      console.warn(`Creating empty doc for dangling ref: ${path}`);
      this.fileDocs.set(docId, Automerge.init<FileDoc>());
    }
  }

  // 2. Gather all local documents
  const localDocs = new Map<string, Uint8Array>();
  localDocs.set("structure", Automerge.save(this.structureDoc));
  for (const [docId, fileDoc] of this.fileDocs) {
    localDocs.set(docId, Automerge.save(fileDoc));
  }

  // 3. Provider syncs and returns merged state
  const mergedDocs = await this.provider.sync(localDocs);

  // 4. Load merged structure first
  this.structureDoc = Automerge.load(mergedDocs.get("structure")!);

  // 5. Load only file docs referenced in merged structure (orphans filtered out)
  const referencedDocIds = new Set(
    Object.values(this.structureDoc.files).map(f => f.docId)
  );

  this.fileDocs.clear();
  for (const [docId, data] of mergedDocs) {
    if (docId !== "structure" && referencedDocIds.has(docId)) {
      this.fileDocs.set(docId, Automerge.load(data));
    }
  }

  // 6. Persist merged state to disk (atomic writes, orphans left for recovery)
  await this.save();
}
```

### Consistency Rules

**Pre-sync (inline in sync flow):**
1. **Orphan file docs** (doc exists in `docs/` but not referenced in structure) → keep on disk (for potential recovery), but don't load into memory or sync
2. **Dangling references** (structure references a `docId` that doesn't exist) → create empty doc, log warning

**Post-sync (implicit in sync flow):**
- **Remote deletions**: Step 5 only loads docs referenced in merged structure - orphans are simply not loaded
- **Remote additions**: Merged docs from provider include new files - loaded in step 5
- **Disk writes**: `save()` writes only in-memory docs; orphans are left on disk

**Conflicts** → let Automerge resolve automatically (deterministic winner based on actor IDs)

### GitHub Provider

Uses atomic commit strategy for consistency:

```typescript
class GitHubProvider implements SyncProvider {
  async sync(localDocs: Map<string, Uint8Array>): Promise<Map<string, Uint8Array>> {
    // 1. Fetch all remote docs
    const remoteDocs = await this.fetch();

    // 2. Merge all locally
    const merged = new Map<string, Uint8Array>();
    for (const [docId, localData] of localDocs) {
      let doc = Automerge.load(localData);
      const remoteData = remoteDocs.get(docId);
      if (remoteData) {
        const remoteDoc = Automerge.load(remoteData);
        doc = Automerge.merge(doc, remoteDoc);
      }
      merged.set(docId, Automerge.save(doc));
    }

    // 3. Include remote-only docs (initial clone, or files deleted locally)
    for (const [docId, remoteData] of remoteDocs) {
      if (!localDocs.has(docId)) {
        merged.set(docId, remoteData);
      }
    }

    // 4. Push all changes in single commit (atomic write)
    await this.push(merged);

    // 5. Render plain text files for GitHub browsing
    await this.renderPlainText(merged);

    return merged;
  }

  private async fetch(): Promise<Map<string, Uint8Array>> {
    // Fetch all .automerge files from .stash/ directory
    const docs = new Map<string, Uint8Array>();
    const structureBlob = await this.getBlob(".stash/structure.automerge");
    if (structureBlob) docs.set("structure", structureBlob);

    const docFiles = await this.listDir(".stash/docs/");
    for (const file of docFiles) {
      const docId = file.replace(".automerge", "");
      const blob = await this.getBlob(`.stash/docs/${file}`);
      if (blob) docs.set(docId, blob);
    }
    return docs;
  }

  private async push(docs: Map<string, Uint8Array>): Promise<void> {
    // Use GitHub's Git Data API (trees + commits) for atomic updates
    // This ensures all files are updated in a single commit
    const tree = await this.buildTree(docs);
    const commit = await this.createCommit(tree, "sync: update stash");
    await this.updateRef(commit);
  }
}
```

**Why atomic commits matter:** If sync crashes mid-way through individual file pushes, the remote state becomes inconsistent - some files updated, others not. Using Git's tree API, either all files update or none do.

**Remote storage:**
```
repo/
  .stash/
    structure.automerge          # Structure document
    docs/
      <docId>.automerge          # File documents
  hello.md                       # Rendered plain text (read-only)
  notes/
    todo.md                      # Rendered plain text (read-only)
```

- Automerge state stored in `.stash/` directory
- Plain text files rendered to repo root for human readability
- Uses GitHub Git Data API for atomic commits (trees/commits/refs)

**Note on rendered files:** When multiple clients sync concurrently, rendered plain-text files may temporarily be inconsistent with the Automerge state. This is acceptable because rendered files are read-only snapshots for human viewing - the `.stash/` Automerge docs are the source of truth.

### Future Providers

Other providers implement the same interface with different strategies:

- **WebSocket provider**: Could use Automerge sync protocol internally for efficiency
- **S3 provider**: Simple merge like GitHub
- **Iroh provider**: P2P sync protocol

Stash doesn't know or care about the strategy. It just calls `sync()` and gets merged docs.

## CLI Commands

### Auth

```bash
stash auth github
# Prompts for GitHub personal access token
# Stores in ~/.stash/config.json
```

### Stash Management

```bash
stash create <name>
# Prompts: Sync provider? [None, GitHub]
# If GitHub: prompts for repo name, creates repo
# Creates local stash with structure doc
# Returns key (e.g., github:owner/repo)

stash join <key> --name <local-name>
# Key format: github:owner/repo
# 1. Creates empty local stash
# 2. Calls provider.sync() with empty doc set
# 3. Provider returns all remote docs (structure + files)
# 4. Loads docs into local stash
# 5. Saves to disk

stash list
# Lists all local stashes with their keys

stash delete <name>
# Prompts: also delete remote? (if synced)
# Removes local stash directory

stash sync [name]
# Manual sync (all stashes if name omitted)

stash status
# Shows daemon status and stash info
```

### Daemon

```bash
stash start
# Starts background daemon
# - HTTP server with MCP endpoint
# - Sync loop (default 30s interval)

stash stop
# Stops the daemon

stash install
# Configures MCP server for AI assistants (e.g., Claude Code)
```

## MCP Server

The daemon exposes MCP tools over HTTP using the Streamable HTTP transport. The implementation is modular: `mcp.ts` defines tools and creates the MCP server, while `daemon.ts` handles HTTP transport.

### Transport

Uses MCP's **Streamable HTTP** transport (spec 2025-03-26):

- Single endpoint: `POST /mcp`
- Stateless request/response (no session management needed)
- Supports SSE for streaming responses (optional, not used in v1)

### Client Configuration

```json
{
  "mcpServers": {
    "stash": {
      "url": "http://localhost:32847/mcp"
    }
  }
}
```

Port `32847` is the default (mnemonic: "STASH" on a phone keypad, truncated).

### Implementation

```typescript
// mcp.ts - Transport-agnostic MCP server
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StashManager } from "./core/manager.js";

export function createMcpServer(manager: StashManager): Server {
  const server = new Server(
    { name: "stash", version: "1.0.0" },
    { capabilities: { tools: {} } }
  );

  // Register tools
  server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: [
      { name: "stash_list", description: "...", inputSchema: { ... } },
      { name: "stash_read", description: "...", inputSchema: { ... } },
      // ... other tools
    ]
  }));

  server.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;
    switch (name) {
      case "stash_list": return handleList(manager, args);
      case "stash_read": return handleRead(manager, args);
      // ... other tools
    }
  });

  return server;
}
```

```typescript
// daemon.ts - HTTP transport + sync loop
import express from "express";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { createMcpServer } from "./mcp.js";
import { StashManager } from "./core/manager.js";

const PORT = 32847;

async function main() {
  const manager = await StashManager.load();
  const mcpServer = createMcpServer(manager);

  const app = express();
  app.use(express.json());

  // MCP endpoint
  app.post("/mcp", async (req, res) => {
    const transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: undefined, // Stateless
    });
    res.on("close", () => transport.close());
    await mcpServer.connect(transport);
    await transport.handleRequest(req, res, req.body);
  });

  // Health check
  app.get("/health", (req, res) => res.json({ status: "ok" }));

  app.listen(PORT, () => {
    console.log(`Stash daemon listening on http://localhost:${PORT}`);
  });

  // Sync loop
  setInterval(() => manager.sync(), 30_000);
}

main();
```

### Why This Design

1. **Modular**: `mcp.ts` knows nothing about HTTP; `daemon.ts` knows nothing about MCP tools
2. **Testable**: Can test MCP tools in isolation without HTTP
3. **Simple**: Single `/mcp` endpoint, stateless requests, no session management
4. **Standard**: Uses official MCP SDK transport, compatible with all MCP clients

## MCP Tools

Path format: `<stash>:<filepath>` or just `<stash>:` for root.

Each tool call syncs with remote storage before executing, so the daemon is optional for syncing. Write operations sync both before and after to ensure changes are pushed.

### stash_list

List stashes or immediate children within a path.

**Arguments:**
- `path` (optional): Empty = list stashes. `"stash:"` = list root. `"stash:dir/"` = list in dir.

**Returns:** `{ items: string[] }` - directories have trailing `/`

**Example:**
```
stash_list({}) → { items: ["my-project", "research"] }
stash_list({ path: "my-project:" }) → { items: ["readme.md", "docs/"] }
stash_list({ path: "my-project:docs/" }) → { items: ["guide.md", "api/"] }
```

### stash_glob

Find files matching a glob pattern.

**Arguments:**
- `stash`: Stash name (e.g., `"my-project"`)
- `pattern`: Glob pattern to match (e.g., `"**/*.md"`)

**Returns:** `{ files: string[] }` - all matching file paths

**Example:**
```
stash_glob({ stash: "my-project", pattern: "**/*.md" }) → { files: ["readme.md", "docs/guide.md", "docs/api/auth.md"] }
stash_glob({ stash: "my-project", pattern: "docs/*.md" }) → { files: ["docs/guide.md"] }
```

**Errors:**
- Stash not found: `{ error: "Stash not found: <name>" }`

### stash_read

Read file content.

**Arguments:**
- `path`: `"stash-name:filepath"`

**Returns:** `{ content: string }`

**Errors:**
- File not found: `{ error: "File not found: <path>" }`
- Stash not found: `{ error: "Stash not found: <name>" }`

### stash_write

Write or update a file. Creates the file if it doesn't exist. Parent directories are implicit.

**Arguments:**
- `path`: `"stash-name:filepath"`
- `content` (optional): Full content replacement (creates file if new)
- `patch` (optional): `{ start: number, end: number, text: string }` (file must exist)

Must provide either `content` or `patch`.

**Note:** Prefer `patch` for small edits - it preserves concurrent edits better. Use `content` only for new files or complete rewrites.

**Returns:** `{ success: true }`

**Errors:**
- Neither content nor patch: `{ error: "Must provide content or patch" }`
- Patch on non-existent file: `{ error: "File not found: <path>" }`
- Stash not found: `{ error: "Stash not found: <name>" }`

### stash_delete

Delete a file.

**Arguments:**
- `path`: `"stash-name:filepath"`

**Returns:** `{ success: true }`

**Errors:**
- File not found: `{ error: "File not found: <path>" }`
- Stash not found: `{ error: "Stash not found: <name>" }`

### stash_move

Move or rename a file within a stash.

**Arguments:**
- `from`: `"stash-name:filepath"`
- `to`: `"stash-name:filepath"` (must be same stash)

**Returns:** `{ success: true }`

**Errors:**
- Source not found: `{ error: "File not found: <path>" }`
- Stash not found: `{ error: "Stash not found: <name>" }`
- Cross-stash move: `{ error: "Cross-stash moves not supported" }`

**Note:** Moving preserves the file's document identity (`docId`), so edit history is maintained. For cross-stash moves, use read + write + delete instead.

## Module Structure

```
src/
├── cli.ts                   # CLI entry point (commander setup)
├── daemon.ts                # Background daemon (HTTP server + sync loop)
├── mcp.ts                   # MCP server setup (tools, transport-agnostic)
├── core/
│   ├── stash.ts             # Stash class (single stash operations)
│   ├── manager.ts           # StashManager (multi-stash operations)
│   ├── structure.ts         # Structure document helpers
│   ├── file.ts              # File document helpers
│   ├── config.ts            # Global config (~/.stash/config.json)
│   └── errors.ts            # Error types (SyncError, etc.)
├── providers/
│   ├── types.ts             # SyncProvider interface
│   └── github.ts            # GitHub implementation
└── cli/
    ├── commands/            # CLI command implementations
    │   ├── auth.ts
    │   ├── create.ts
    │   ├── delete.ts
    │   ├── install.ts
    │   ├── join.ts
    │   ├── list.ts
    │   ├── start.ts
    │   ├── status.ts
    │   ├── stop.ts
    │   └── sync.ts
    └── prompts.ts           # Interactive prompts (inquirer)
```

### Key Classes

```typescript
// core/stash.ts
class Stash {
  readonly name: string;
  private structureDoc: Automerge.Doc<StructureDoc>;
  private fileDocs: Map<string, Automerge.Doc<FileDoc>>;
  private provider: SyncProvider | null;

  // File operations (update structure doc + file docs)
  read(path: string): string;
  write(path: string, content: string): void;
  patch(path: string, start: number, end: number, text: string): void;
  delete(path: string): void;
  move(from: string, to: string): void;
  list(dir?: string): string[];
  glob(pattern: string): string[];

  // Sync
  async sync(): Promise<void>;

  // Persistence (atomic: write to .tmp, then rename)
  async save(): Promise<void>;
  static async load(name: string, stashBaseDir: string, provider?: SyncProvider): Promise<Stash>;
}

// core/manager.ts
// Facade over all stashes. Used by daemon (sync loop) and MCP (tool handlers).
// Loads all stashes on startup, provides access by name, handles cross-stash operations.
class StashManager {
  private stashes: Map<string, Stash>;

  static async load(): Promise<StashManager>;  // Load all stashes from ~/.stash/
  get(name: string): Stash | undefined;
  list(): string[];                            // Stash names
  async create(name: string, provider: SyncProvider | null): Promise<Stash>;
  async join(key: string, localName: string): Promise<Stash>;
  async delete(name: string, deleteRemote: boolean): Promise<void>;
  async sync(): Promise<void>;                 // Sync all stashes
}
```

## Dependencies

- **@automerge/automerge** - CRDT implementation
- **@modelcontextprotocol/sdk** - MCP server + Streamable HTTP transport
- **express** - HTTP server for daemon
- **octokit** - GitHub API
- **commander** - CLI framework
- **inquirer** - Interactive prompts
- **ulid** - ULID generation
- **minimatch** - Glob pattern matching for stash_glob

## Error Handling

### MCP Tools
- Return `{ error: string }` with descriptive message
- Don't throw exceptions

### Sync Errors

```typescript
class SyncError extends Error {
  constructor(
    message: string,
    public readonly retryable: boolean,
    public readonly cause?: Error
  ) {
    super(message);
  }
}
```

**Error categories:**
- **Network errors** (retryable): Connection timeout, DNS failure, 5xx responses
- **Auth errors** (not retryable): 401/403 responses → surface to user
- **Conflict errors**: Shouldn't happen (Automerge handles), but if provider detects → retry sync
- **Consistency errors**: Log warning, fix automatically, continue

**Retry strategy:**
```typescript
const RETRY_CONFIG = {
  maxAttempts: 3,
  baseDelayMs: 1000,
  maxDelayMs: 30000,
  backoffMultiplier: 2,
};

async function syncWithRetry() {
  for (let attempt = 1; attempt <= RETRY_CONFIG.maxAttempts; attempt++) {
    try {
      await this.sync();
      return;
    } catch (err) {
      if (err instanceof SyncError && !err.retryable) throw err;
      if (attempt === RETRY_CONFIG.maxAttempts) throw err;

      const delay = Math.min(
        RETRY_CONFIG.baseDelayMs * Math.pow(RETRY_CONFIG.backoffMultiplier, attempt - 1),
        RETRY_CONFIG.maxDelayMs
      );
      await sleep(delay);
    }
  }
}
```

### CLI
- Print error message to stderr
- Exit with non-zero code

## Future Considerations

- Real-time sync providers (WebSocket-based)
- Editor plugins for direct Automerge integration
- Conflict surfacing UI
- File watching for local edits (with diff-based patching)
- Binary file support
- File size limits
- Permissions / read-only sharing
