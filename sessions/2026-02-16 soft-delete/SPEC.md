# Soft Delete: Tombstones + Local Path Tracking

## Problem

When `flush()` runs, it can't distinguish between:
- **New files** on disk (not yet imported) → should import
- **Deleted files** (removed from Automerge, still on disk) → should delete

This causes race conditions and incorrect behavior.

## Solution: Two-Part Approach

### Part 1: Tombstones in Automerge (synced)

Mark deleted files with `deleted: true` instead of removing them. This syncs to other machines.

```typescript
interface FileEntry {
  docId: string;
  created: number;
  deleted?: boolean;  // tombstone flag
}
```

### Part 2: Local Known-Paths Tracking (not synced)

Maintain `.stash/known-paths.json` locally to track paths this machine has seen and synced.

```typescript
// .stash/known-paths.json
{
  "paths": ["foo.md", "docs/bar.md", ...]
}
```

### Combined Logic

**On file import/write:**
- Add path to known-paths

**On delete (local):**
- Set `deleted: true` in structure
- Keep in known-paths (until we handle it)

**On sync:**
- Tombstones propagate via Automerge

**On cleanup after sync:**
```
For each file on disk:
  if path has tombstone (deleted: true):
    if path in known-paths:
      → Old file, remote deletion wins
      → Delete from disk
      → Remove from known-paths
    else:
      → New local file, local wins
      → Resurrect: clear tombstone, new docId
      → Add to known-paths
  else if path not in structure at all:
    → New file, import it
    → Add to known-paths
  else:
    → Normal tracked file, keep it
```

**On resurrection (clearing tombstone):**
- Set `deleted: false` (or remove flag)
- Create new docId
- Update created timestamp

---

## Edge Cases & Expected Behavior

### Basic Operations

| # | Scenario | Expected Behavior |
|---|----------|-------------------|
| 1 | Create file via MCP/disk | File imported to Automerge, added to known-paths |
| 2 | Delete file locally | `deleted: true` set, file removed from disk on flush |
| 3 | Delete file remotely, sync | Tombstone arrives, file deleted from disk, removed from known-paths |
| 4 | Read deleted file | Error: file not found |
| 5 | List files | Deleted files not shown |

### Race Conditions (Original Bug)

| # | Scenario | Expected Behavior |
|---|----------|-------------------|
| 6 | MCP writes file, flush runs before watcher imports | File imported during flush (not deleted) |
| 7 | File created while daemon off, daemon starts | File imported on startup scan |

### Recreate After Delete

| # | Scenario | Expected Behavior |
|---|----------|-------------------|
| 8 | Delete file, create new file at same path | New docId, `deleted: false`, file persists |
| 9 | Delete file, create file with same content | Same as #8 - new file, persists |
| 10 | Remote deletes file, local creates new file at same path (daemon running) | Local file imported first, survives sync (not in known-paths yet for that tombstone) |
| 11 | Remote deletes file, local creates new file (daemon was off) | On startup: scan imports file, sync sees tombstone but path not in known-paths → resurrect |

### Multi-Machine Sync

| # | Scenario | Expected Behavior |
|---|----------|-------------------|
| 12 | A deletes file, B syncs | B sees tombstone, deletes local copy |
| 13 | A deletes file, B edits file, both sync | Conflict: content wins - file survives with B's edits, tombstone cleared |
| 14 | A and B both delete same file | Both have tombstone, no conflict |
| 15 | A deletes file, B never had it, B syncs | B sees tombstone, no local file to delete, tombstone persists |

### Known-Paths Edge Cases

| # | Scenario | Expected Behavior |
|---|----------|-------------------|
| 16 | Fresh machine, first sync with existing stash | All remote files imported, added to known-paths |
| 17 | Fresh machine, sync with stash containing tombstones | Tombstoned paths ignored (not in known-paths, no local file) |
| 18 | Daemon crashes, restarts | known-paths.json persists, state recovered |
| 19 | known-paths.json deleted/corrupted | Treat all files as "new" - safe default (might resurrect deleted files) |

### Conflict Resolution

| # | Scenario | Expected Behavior |
|---|----------|-------------------|
| 20 | Tombstone exists + doc has content after merge | Content wins: clear tombstone |
| 21 | Tombstone exists + doc is empty | Deletion wins: keep tombstone |

---

## Implementation Plan

### Files to Modify

1. **`src/core/structure.ts`**
   - Add `deleted?: boolean` to `FileEntry`
   - `removeFile()` → sets `deleted: true` instead of deleting
   - `addFile()` → if path exists with `deleted: true`, create new docId, clear flag
   - `listPaths()` → filter out deleted entries
   - Add `listAllPathsIncludingDeleted()` for internal use
   - Add `isDeleted(path)` helper
   - Add `clearTombstone(path)` helper

2. **`src/core/stash.ts`**
   - Update `delete()` to use new tombstone behavior
   - Update `write()` to handle resurrection
   - Add known-paths management: `loadKnownPaths()`, `saveKnownPaths()`, `addKnownPath()`, `removeKnownPath()`
   - Update `listAllFiles()` to filter deleted

3. **`src/core/reconciler.ts`**
   - Update `cleanupOrphanedFiles()` to check tombstones + known-paths
   - Update `scan()` to handle tombstoned paths (resurrect if file exists)
   - Remove the `scan()` call I added earlier in `flush()` - will redo properly

4. **New file: `.stash/known-paths.json`**
   - Created per-stash
   - Contains array of paths
   - Loaded on stash init, saved on changes

### Test Files

1. **`test/core/structure.test.ts`** - tombstone behavior
2. **`test/core/reconciler.test.ts`** - flush/scan with tombstones + known-paths
3. **`test/core/stash.test.ts`** - delete/resurrect cycle
4. **`test/integration/sync-delete.test.ts`** - multi-machine scenarios (mocked)

### Implementation Order

1. Add tombstone support to `structure.ts`
2. Add known-paths to `stash.ts`
3. Update `reconciler.ts` cleanup logic
4. Write tests for each edge case
5. Integration test: full cycle
