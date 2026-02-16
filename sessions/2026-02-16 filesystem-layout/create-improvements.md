# Stash Create Improvements

Two improvements to `stash create`:
1. Import existing files when targeting an existing folder
2. Accept `--remote` flag to skip interactive prompt

---

## 1. Import Existing Files

### Current Behavior

```bash
stash create notes --path ~/Documents/notes
```

- Creates empty stash at path
- Existing files in folder are ignored
- Only new files (created after) are tracked via reconciler

### Desired Behavior

```bash
stash create notes --path ~/Documents/notes
```

- If folder exists with files, scan and import them
- Creates `.stash/` metadata
- All existing files added to Automerge
- Existing files become tracked immediately

---

## 2. `--remote` Flag

### Current Behavior

```bash
stash create notes
# Interactive prompt: "Sync provider? None / GitHub"
# If GitHub: "GitHub repo (owner/repo):"
```

### Desired Behavior

```bash
# Local-only (no sync)
stash create notes --remote none

# With GitHub remote (repo must exist and be empty)
stash create notes --remote github:user/repo

# No flag = interactive prompt (unchanged)
stash create notes
```

**GitHub remote validation:**
- Repo must exist → error if not found
- Repo must be empty → error if has commits (use `stash connect` instead)

### CLI Option

```typescript
// src/cli/options.ts
export const remoteOption = new Option(
  "--remote <remote>",
  "sync remote (none, github:owner/repo)"
);
```

### Create Command Changes

```typescript
// src/cli/commands/create.ts
interface CreateOptions {
  path?: string;
  description?: string;
  remote?: string;  // NEW
}

export async function createStash(
  name: string,
  opts: CreateOptions,
): Promise<void> {
  const manager = await StashManager.load();

  let provider = null;
  let remote: string | null = null;

  if (opts.remote !== undefined) {
    // --remote flag provided, skip prompt
    if (opts.remote === "none") {
      // Local only, no provider
    } else if (opts.remote.startsWith("github:")) {
      const parsed = parseGitHubRemote(opts.remote);
      if (!parsed) {
        console.error("Invalid remote format. Use github:owner/repo");
        process.exit(1);
      }

      const token = await getGitHubToken();
      if (!token) {
        console.error("No GitHub token found. Run `stash auth github` first.");
        process.exit(1);
      }

      provider = new GitHubProvider(token, parsed.owner, parsed.repo, parsed.pathPrefix);

      // Validate: repo must exist
      const remoteExists = await provider.exists();
      if (!remoteExists) {
        console.error(`Repository ${parsed.owner}/${parsed.repo} does not exist.`);
        process.exit(1);
      }

      // Validate: repo must be empty
      const isEmpty = await provider.isEmpty();
      if (!isEmpty) {
        console.error(`Repository ${parsed.owner}/${parsed.repo} is not empty.`);
        console.error("Use 'stash connect' to join an existing stash.");
        process.exit(1);
      }

      remote = opts.remote;
    } else {
      console.error("Unknown remote format. Use 'none' or 'github:owner/repo'");
      process.exit(1);
    }
  } else {
    // No --remote flag, interactive prompt (same validation as above)
  }

  // ... rest unchanged ...
}
```

---

## Implementation

### Changes to `StashManager.create()`

After creating the stash, scan for existing files:

```typescript
async create(
  name: string,
  stashPath?: string,
  provider: SyncProvider | null = null,
  remote: string | null = null,
  description?: string,
): Promise<Stash> {
  // ... existing validation ...

  const resolvedPath = stashPath
    ? path.resolve(stashPath)
    : path.join(this.baseDir, name);

  const stash = Stash.create(
    name,
    resolvedPath,
    this.config.actorId,
    provider,
    remote,
    description,
  );

  // NEW: Import existing files before saving
  await this.importExistingFiles(stash, resolvedPath);

  await stash.save();

  // ... rest unchanged ...
}
```

### New Method: `importExistingFiles()`

```typescript
private async importExistingFiles(stash: Stash, rootPath: string): Promise<void> {
  try {
    await fs.access(rootPath);
  } catch {
    // Folder doesn't exist yet, nothing to import
    return;
  }

  const files = await this.walkDirectory(rootPath);

  for (const relativePath of files) {
    const fullPath = path.join(rootPath, relativePath);
    const content = await fs.readFile(fullPath);

    if (this.isUtf8(content)) {
      stash.write(relativePath, content.toString("utf-8"));
    } else {
      // Binary file - store hash and blob
      const hash = this.hashBuffer(content);
      const size = content.length;
      await this.storeBinaryBlob(rootPath, hash, content);
      stash.writeBinary(relativePath, hash, size);
    }
  }
}

private async walkDirectory(
  rootPath: string,
  relativePath: string = "",
): Promise<string[]> {
  const results: string[] = [];
  const fullPath = relativePath
    ? path.join(rootPath, relativePath)
    : rootPath;

  const entries = await fs.readdir(fullPath, { withFileTypes: true });

  for (const entry of entries) {
    // Skip .stash directory and hidden files
    if (entry.name === ".stash" || entry.name.startsWith(".")) continue;

    const childRelative = relativePath
      ? `${relativePath}/${entry.name}`
      : entry.name;

    if (entry.isDirectory()) {
      const subFiles = await this.walkDirectory(rootPath, childRelative);
      results.push(...subFiles);
    } else if (entry.isFile()) {
      results.push(childRelative);
    }
    // Skip symlinks
  }

  return results;
}

private isUtf8(buffer: Buffer): boolean {
  try {
    const text = buffer.toString("utf-8");
    return !text.includes("\uFFFD");
  } catch {
    return false;
  }
}

private hashBuffer(buffer: Buffer): string {
  return crypto.createHash("sha256").update(buffer).digest("hex");
}

private async storeBinaryBlob(
  rootPath: string,
  hash: string,
  content: Buffer,
): Promise<void> {
  const blobDir = path.join(rootPath, ".stash", "blobs");
  await fs.mkdir(blobDir, { recursive: true });
  const blobPath = path.join(blobDir, `${hash}.bin`);
  await fs.writeFile(blobPath, content);
}
```

### CLI Output

Show import progress:

```typescript
// In create.ts
const stash = await manager.create(name, opts.path, provider, remote, opts.description);

const fileCount = stash.listAllFiles().length;
if (fileCount > 0) {
  console.log(`Imported ${fileCount} existing file(s).`);
}
console.log(`Stash "${name}" created.`);
```

---

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| Empty folder | Create stash, no files imported |
| Folder doesn't exist | Create folder and stash |
| Folder has `.stash/` already | Error: "Folder is already a stash" |
| Hidden files (`.gitignore`, etc.) | Skipped |
| Symlinks in folder | Skipped |
| Very large files | Import normally (no size limit for now) |

### Already a Stash Check

Add validation at start of create:

```typescript
const existingStashDir = path.join(resolvedPath, ".stash");
try {
  await fs.access(existingStashDir);
  throw new Error(`Folder is already a stash: ${resolvedPath}`);
} catch (err) {
  if ((err as NodeJS.ErrnoException).code !== "ENOENT") throw err;
  // ENOENT means .stash doesn't exist, which is good
}
```

---

## Tests

```typescript
describe('stash create --remote flag', () => {
  it('should create local-only stash with --remote none');
  it('should create stash with GitHub remote');
  it('should create GitHub repo if it does not exist');
  it('should connect to existing GitHub repo');
  it('should error on invalid remote format');
  it('should error if no GitHub token configured');
  it('should prompt interactively when --remote not provided');
});

describe('stash create with existing files', () => {
  it('should import existing text files');
  it('should import existing binary files');
  it('should handle nested directories');
  it('should skip hidden files');
  it('should skip .stash directory');
  it('should skip symlinks');
  it('should error if folder is already a stash');
  it('should work with empty existing folder');
  it('should work when folder does not exist');
  it('should report number of imported files');
});
```

---

## No New Dependencies

Uses existing imports:
- `crypto` (Node.js built-in)
- `fs/promises`
- `path`
