# Implementation Plan

Implement Stash spec using test-driven development. Write tests first, verify they fail, then implement.

## Current State (v1)

The existing implementation uses:
- Per-file Automerge docs (`document.ts`)
- Dual storage: `files/` (plain text) + `automerge/` (binary)
- Per-file push/pull provider interface

## Target State (v2)

The v2 architecture uses:
- Single structure doc + separate file docs with ULIDs
- Single storage: only `.automerge` files (no plain text locally)
- Single `sync()` provider method returning all merged docs

## Migration Strategy

1. Keep existing `document.ts` logic (it's reusable for file docs)
2. Add new `structure.ts` for structure document
3. Rewrite `stash.ts` to use structure doc + ULID-based file docs
4. Rewrite provider interface to single `sync()` method
5. Update tests incrementally

## Phase 1: Core Data Model

### 1.1 Structure Document
- [ ] Test: Create structure doc, add/remove file entries
- [ ] Test: Rename preserves docId
- [ ] Test: Serialize/deserialize structure doc
- [ ] Implement: `src/core/structure.ts`

### 1.2 File Document (refactor existing)
- [ ] Test: Create file doc with content
- [ ] Test: Apply patch (start, end, text)
- [ ] Test: Get text content
- [ ] Test: Serialize/deserialize file doc
- [ ] Refactor: `src/core/file.ts` (rename from document.ts)

### 1.3 Stash Class
- [ ] Test: Create new stash (initializes structure doc)
- [ ] Test: Write file (creates entry in structure + file doc)
- [ ] Test: Read file
- [ ] Test: Patch file
- [ ] Test: Delete file (removes from structure, orphans file doc)
- [ ] Test: Move file (preserves docId)
- [ ] Test: List files in directory
- [ ] Test: Glob pattern matching
- [ ] Test: Save to disk (atomic writes)
- [ ] Test: Load from disk
- [ ] Test: Dangling ref creates empty doc on sync
- [ ] Implement: `src/core/stash.ts`

### 1.4 Stash Manager
Facade over all stashes. Used by daemon (sync loop) and MCP (tool handlers).
- [ ] Test: Load all stashes from ~/.stash/
- [ ] Test: Create stash
- [ ] Test: List stash names
- [ ] Test: Get stash by name
- [ ] Test: Delete stash
- [ ] Test: Sync all stashes
- [ ] Implement: `src/core/manager.ts`

### 1.5 Config
- [ ] Test: Read/write GitHub token
- [ ] Test: Config file creation
- [ ] Implement: `src/core/config.ts`

### 1.6 Errors
- [ ] Implement: `src/core/errors.ts` (SyncError class)

## Phase 2: Sync Provider

### 2.1 Provider Interface
- [ ] Define: `src/providers/types.ts`

### 2.2 GitHub Provider
- [ ] Test: fetch() returns all remote docs
- [ ] Test: push() creates atomic commit
- [ ] Test: sync() merges local + remote
- [ ] Test: sync() with empty local (join case)
- [ ] Test: renderPlainText() creates readable files
- [ ] Test: Auth error handling
- [ ] Test: Network error handling (retryable)
- [ ] Implement: `src/providers/github.ts`

### 2.3 Stash Sync Integration
- [ ] Test: Sync updates local from remote
- [ ] Test: Sync pushes local to remote
- [ ] Test: Concurrent edits merge correctly
- [ ] Test: Remote deletion creates orphan locally
- [ ] Test: Retry on network error

## Phase 3: MCP Server

### 3.1 MCP Tools
- [ ] Test: stash_list (no path = list stashes)
- [ ] Test: stash_list (with path = list files)
- [ ] Test: stash_glob
- [ ] Test: stash_read
- [ ] Test: stash_write (content)
- [ ] Test: stash_write (patch)
- [ ] Test: stash_delete
- [ ] Test: stash_move
- [ ] Test: Error responses
- [ ] Implement: `src/mcp.ts`

## Phase 4: CLI Commands

### 4.1 Auth
- [ ] Test: `stash auth github` stores token
- [ ] Implement: `src/cli/commands/auth.ts`

### 4.2 Stash Management
- [ ] Test: `stash create` (local-only)
- [ ] Test: `stash create` (with GitHub)
- [ ] Test: `stash join`
- [ ] Test: `stash list`
- [ ] Test: `stash delete`
- [ ] Test: `stash sync`
- [ ] Test: `stash status`
- [ ] Implement: `src/cli/commands/*.ts`

### 4.3 Daemon
- [ ] Test: `stash start` spawns background process
- [ ] Test: `stash stop` kills daemon
- [ ] Test: Daemon serves HTTP on port 32847
- [ ] Test: Daemon runs sync loop
- [ ] Test: MCP endpoint responds to POST /mcp
- [ ] Test: `stash install` configures MCP (writes url to config)
- [ ] Implement: `src/daemon.ts`, `src/cli/commands/start.ts`

### 4.4 CLI Entry Point
- [ ] Implement: `src/cli.ts`

## Phase 5: Documentation (FINAL STEP)

Update README.md after all implementation is complete:
- [ ] Overview, installation, quickstart
- [ ] CLI reference (all commands)
- [ ] MCP tools reference (updated for v2 - no stash_mkdir, stash_glob has separate args)
- [ ] Architecture overview (structure doc, file docs, sync)
- [ ] Remove outdated v1 references

## Test Infrastructure

- Test framework: Vitest (already in use)
- Test location: `test/` directory mirroring `src/`
- Mocking: GitHub API calls mocked in tests
- Fixtures: Sample Automerge docs for testing

## Implementation Order

1. Core data model (structure, file, stash) - foundation
2. Config and errors - needed by other modules
3. Manager - ties stashes together
4. Provider interface + GitHub provider - enables sync
5. MCP server - exposes functionality to agents
6. CLI commands - user interface
7. Daemon - background operation
8. Documentation - final polish

## Notes

- Each phase builds on previous
- Tests written before implementation
- Commit after each subsection passes tests
- Keep existing test infrastructure
