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

The system closes the full agent feedback loop:
- **LSP** catches structural and type errors before commit
- **REPL** catches runtime errors during development
- **SQLite** provides the transactional substrate and audit trail

Together these cover everything an agent needs during a swarm session. Integration and
e2e tests require a fully materialized environment and belong to the CI pipeline at deploy
time, not the swarm session itself.

---

## Core structure

One SQLite database holds the full working state of the project. Three tables carry the
load:

- `files(id, path, language)` — one row per source file
- `nodes(id, file_id, kind, parent_id, start, end, properties)` — every AST node,
  normalized and relational
- `symbols(id, name, kind, definition_node_id, version)` — cross-file semantic index,
  populated and maintained by the LSP server

The `symbols.version` column is the concurrency primitive. Any mutation that changes a
symbol's name, signature, or type increments its version. Two agents mutating the same
symbol concurrently produce a version mismatch at commit time; the second transaction
aborts. Reference insertions (new call sites) are non-conflicting inserts and compose
freely.

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

Without step 3, the LSP's semantic graph drifts from the DB state and cross-file
references go stale.

**Conflict behavior:** abort returns a conflict error; retry policy is caller-defined.
Agents may retry with backoff or escalate to MCP arbitration.

**Known V1 gap:** write-write conflicts on the same symbol are detected. Read-set
staleness (agent reads symbol at version N, another agent mutates to N+1 before the
first commits) is not detected at the DB layer. Mitigated by mandatory pre-commit LSP
re-validation, which surfaces most invalid semantic changes. Full read-set OCC via
transaction manifest is deferred to V2.

---

## Execution flow

The DB never executes. On any eval request:
1. Materialize affected files to a temporary filesystem sandbox
2. Apply the canonical formatter (Black for Python, Prettier for TypeScript) — enforces
   deterministic roundtrips across whitespace-sensitive languages
3. Feed formatted source to a warm, persistent runtime (Python REPL, Bun process)
4. Stream stdout, return values, and errors back to the agent
5. DB remains unchanged unless the agent explicitly commits a new AST edit

**REPL staleness:** each commit emits a list of affected module paths. The REPL pool
marks sessions dirty for those paths. Next eval on a dirty session forces a re-import
before executing.

---

## Worktrees

The worktree analog is a reflink copy of the DB file (`cp --reflink` on btrfs/APFS —
near-instant). Each branch gets an independent DB file and a corresponding git worktree.
Agents work in full isolation with no WAL contention between branches.

When two worktrees finish, they materialize, git commit, and reconcile through a normal
git merge. No new merge problem is introduced — git handles it exactly as it would any
two branches.

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

**TypeScript** — AST via ts-morph or the compiler API, which models trivia including
comments natively. LSP via tsserver.

**Python** — AST via libcst, not stdlib `ast`, which strips comments entirely. libcst
preserves comments as concrete syntax nodes and enables lossless roundtrips. LSP via
pyright.

Each additional language requires: AST schema, materializer, LSP integration. Scope stays
narrow in V1.

---

## MCP surface

MCP exposes structural-edit and eval tools so any MCP-aware client (Claude Code,
OpenCode, Cursor) discovers them automatically. Core tools:

- `session_init()` — return file tree + top-level symbol graph for agent orientation
- `symbol_query(name)` — return definition + all references
- `node_mutate(file_id, mutation)` — structural edit with pre-commit LSP validation
- `eval(expression)` — materialize + run in REPL pool + stream output
- `snapshot_create(name)` — tag current savepoint
- `snapshot_restore(id)` — full three-system reset (see above)

---

## What this is not

This system does not replace Git for humans. Developers clone, branch, and commit
normally. The SQLite layer exists only for the duration of a swarm session. At the end,
the materialized output is a conventional project tree that any existing CI pipeline
consumes unchanged.
