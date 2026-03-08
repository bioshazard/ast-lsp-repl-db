# AST-SQLite Agent Scratchpad

AI agent swarms perform large-scale, transactional refactors across hundreds of files by
operating on a SQLite database of parsed AST nodes instead of raw text. Git stays the
durable source of truth. SQLite is the structured working layer for the duration of a
swarm session. Text files are a derived artifact, not a canonical representation.

---

## Why

Text-based editing is the wrong primitive for agents. Agents produce syntactically
malformed code, create merge conflicts on concurrent edits, and have no native way to
express semantic changes like "rename this function everywhere it's called." SQLite over
AST nodes solves all three: the schema enforces valid structure, OCC versioning detects
semantic conflicts before commit, and cross-file queries ("find every caller of
processPayment") are plain SQL.

The system closes the full agent feedback loop across four layers:
- **CST/AST** — valid structure enforced at the schema level; lossless roundtrips
- **LSP** — type and semantic errors caught before commit
- **REPL** — runtime behavior validated during development
- **OTel traces** — execution history queryable as data, enabling session replay

Together these cover everything an agent needs during a swarm session. Integration and
e2e tests require a fully materialized environment and belong to the CI pipeline at deploy
time, not the swarm session itself.

---

## Core structure

One SQLite database holds the full working state of the project. Four tables carry the
load:

- `files(id, path, language)` — one row per source file
- `nodes(id, file_id, kind, parent_id, start, end, properties)` — every AST node,
  normalized and relational
- `symbols(id, name, kind, definition_node_id, version)` — cross-file semantic index,
  populated and maintained by the LSP server
- `traces(id, span_id, parent_span_id, name, start_time, end_time, attributes JSON)`
  — OTel spans emitted by the REPL pool, queryable alongside code structure

The `symbols.version` column is the concurrency primitive. Any mutation that changes a
symbol's name, signature, or type increments its version. Two agents mutating the same
symbol concurrently produce a version mismatch at commit time; the second transaction
aborts. Reference insertions (new call sites) are non-conflicting inserts and compose
freely.

The `traces` table turns the DB into a unified development intelligence layer — code
structure and runtime behavior in one place, one query interface.

---

## AST vs CST

The system uses CST (Concrete Syntax Tree) not AST (Abstract Syntax Tree) where
possible. AST strips trivia — comments, whitespace, formatting. CST preserves it. Lossless
roundtrips require CST; without it comments are lost on the first materialize cycle.

- **TypeScript** — ts-morph and the compiler API model trivia natively, CST by default
- **Python** — libcst required; stdlib `ast` strips comments entirely and cannot be used

This is a hard library constraint, not a preference. Each additional language must provide
a CST-capable parser to be supported.

---

## Lifecycle

```
git clone
    ↓
hydrate: tree-sitter parses source files → populate SQLite (nodes + symbols)
         fast enough that a 50k-line repo completes in low seconds
         runs once on clone, again on pull
    ↓
agents work: query symbols, mutate AST, run LSP diagnostics, commit transactions
             REPL evals emit OTel spans → traces table
    ↓
materialize: deterministic tree walk → formatted source text
    ↓
git commit → DB discarded
```

Git handles history, branching, and merging. SQLite handles structured edits during the
session. The DB is ephemeral, per-session, and never committed to git. When the swarm
completes and you materialize for git commit, the DB is discarded entirely.

---

## Editing flow

An agent starts a session by calling `session_init`, which returns the file tree and
top-level symbol graph as context. It then queries `symbols` for specific definitions and
references before planning any mutation.

To make a change, the agent mutates the AST in-memory — rename a function, add a
parameter, insert a node — then runs LSP diagnostics on the updated tree. If checks pass,
it commits inside one database transaction.

LSP coherence requires materialization on the hot path. Every structural edit must:
1. Mutate the AST in the DB
2. Materialize a fresh text buffer for the affected file (deterministic tree walk,
   low single-digit ms)
3. Send that buffer as a `textDocument/didChange` payload to keep the LSP server's
   virtual document consistent

For editor-integrated LSP (OpenCode, Cursor), skip the virtual document approach entirely
— write the materialized file to the actual worktree path on commit and let the editor's
file watcher handle LSP sync naturally.

**Conflict behavior:** abort returns a conflict error; retry policy is caller-defined.
Agents may retry with backoff or escalate to MCP arbitration.

**Known V1 gap:** write-write conflicts on the same symbol are detected. Read-set
staleness (agent reads symbol at version N, another agent mutates to N+1 before the
first commits) is not detected at the DB layer. Mitigated by mandatory pre-commit LSP
re-validation, which surfaces most invalid semantic changes. Full read-set OCC via
transaction manifest is deferred to V2.

---

## Execution and diagnostics

The DB never executes. On any eval request:
1. Materialize affected files to a temporary filesystem sandbox
2. Apply the canonical formatter (Black for Python, Prettier for TypeScript)
3. Feed formatted source to a warm, persistent runtime (Python REPL, Bun process)
4. OTel SDK — initialized once in the REPL process — emits spans automatically for every
   call, exception, and slow operation
5. Spans route to the `traces` table in the same SQLite DB
6. Stream stdout, return values, and errors back to the agent
7. DB remains unchanged unless the agent explicitly commits a new AST edit

**REPL staleness:** each commit emits a list of affected module paths. The REPL pool
marks sessions dirty for those paths. Next eval on a dirty session forces a re-import
before executing.

**Session replay over step debugging:** agents query the `traces` table to reconstruct
execution history rather than stepping through a debugger interactively. "Show me all
spans where processPayment exceeded 100ms" is a SQL query. This is the agent-native
diagnostic primitive — read the trace, reason over it, form a hypothesis, mutate, verify.
DAP-based step debugging is available for human handoff but is not the primary agent
workflow. rr (deterministic record/replay at syscall level) is deferred to V2 for the
rare class of bugs that require it.

---

## Worktrees

The worktree analog is a reflink copy of the DB file (`cp --reflink` on btrfs/APFS —
near-instant). Each branch gets an independent DB file and a corresponding git worktree.
Agents work in full isolation with no WAL contention between branches.

**Requires a CoW filesystem** (btrfs, APFS, ZFS). Falls back to a full copy on ext4 —
functional but slower. Document this constraint explicitly for contributors.

```bash
# bootstrap once per repo
bunx ast-sqlite hydrate

# every session is a worktree
bunx ast-sqlite worktree add --branch <name>   # git worktree + reflink db + start MCP
bunx ast-sqlite worktree materialize           # write files → git commit
bunx ast-sqlite worktree discard               # git worktree remove + delete db
```

`worktree discard` calls `git worktree remove` internally — git and db lifecycle stay
strictly coupled and cannot diverge. The main db is the hydrated base; agents never
mutate it directly.

Agents can also explore inside an open uncommitted transaction without creating a
worktree. Mutate, eval, inspect — then commit or rollback for free. The open transaction
is the scratchpad for short explorations.

---

## Snapshots

`snapshot_create(name)` tags the current DB savepoint and returns a snapshot ID.
Savepoints are SQLite-native and don't persist beyond the session — no GC required.

`snapshot_restore(id)` executes a full three-system reset in this order:
1. Quiesce REPL pool — drain or cancel in-flight evals; callers receive a
   `snapshot-restore` error code
2. `ROLLBACK TO savepoint_{id}`
3. `textDocument/didClose` all affected files (batch — close all before opening any)
4. Materialize T1 source for all affected files from the restored AST
5. `textDocument/didOpen` all affected files with T1 source (batch)
6. Mark affected REPL sessions dirty
7. Return `{snapshot_id, affected_files: [...]}`

The two-phase close/open batch (step 3 before step 5) ensures the LSP rebuilds its
semantic graph against a fully consistent T1 snapshot rather than a partially-restored
mix. This is the escape hatch when a swarm goes sideways across 50 files.

---

## Filesystem interop

Some operations are fundamentally filesystem-native: package managers, compilers, test
runners. The system handles these with an explicit materialize → run → hydrate cycle:

1. **Materialize** — write relevant files to a temp sandbox
2. **Run the tool** — `bun install`, `tsc`, `pytest`, etc.
3. **Hydrate** — pull results back into the DB explicitly

**Hydrate:** source files, config files, lockfiles, generated type stubs.
**Don't hydrate:** `node_modules`, `__pycache__`, `dist`, `.next` — anything
deterministically derivable from DB state. These reconstruct JIT from the lockfile on
each sandbox spin-up. The sandbox caches between evals when the lockfile hash is
unchanged.

---

## Language scope (V1)

**TypeScript** — CST via ts-morph or the compiler API. LSP via tsserver.

**Python** — CST via libcst (not stdlib `ast`). LSP via pyright.

Each additional language requires: CST-capable parser, AST schema, materializer, LSP
integration. Scope stays narrow in V1.

---

## MCP and CLI surface

MCP for agents. CLI for humans and CI.

```bash
# CLI — lifecycle boundaries
ast-sqlite hydrate
ast-sqlite worktree add --branch <name>
ast-sqlite worktree materialize
ast-sqlite worktree discard
```

```typescript
// MCP tools — agent session
session_init()                                    // file tree + symbol graph
symbol_query(name)                                // definition + references
node_mutate(fileId, mutation)                     // structural edit + LSP validation
eval(expression, language)                        // materialize + REPL + OTel
snapshot_create(name)                             // tag savepoint
snapshot_restore(id)                              // full three-system reset
materialize_to_sandbox(fileIds)                   // for fs-native tool runs
hydrate_from_sandbox(sandboxPath, paths)          // pull results back
trace_query(sql)                                  // query OTel spans directly
```

Add to `.mcp.json` for automatic agent discovery:
```json
{
  "mcpServers": {
    "ast-sqlite": {
      "command": "bunx",
      "args": ["ast-sqlite", "mcp", "--repo", "."]
    }
  }
}
```

---

## What this is not

This system does not replace Git for humans. Developers clone, branch, and commit
normally. The SQLite layer exists only for the duration of a swarm session. At the end,
the materialized output is a conventional project tree that any existing CI pipeline
consumes unchanged.
