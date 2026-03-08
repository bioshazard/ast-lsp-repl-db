# Queryable Codebase

A codebase is a graph. This system makes it queryable.

Every hard problem in large-scale software — security audits, performance optimization,
API documentation, dependency management, service decomposition — is currently solved by
humans reading files. This system makes those problems queries. Agents operate on the
graph directly. Text files, API specs, service boundaries, and deployment topologies are
all projections of the same underlying structure, materialized on demand.

---

## The core idea

Code is not text that happens to have structure. Code is structure that happens to be
renderable as text. The file tree is one possible view over the data — the one humans and
CI pipelines expect. It is not the canonical representation.

The canonical representation is a graph of symbols, nodes, and their relationships. From
that graph you can materialize:

- A conventional file tree (for git, CI, humans)
- Module interfaces (for consumers and type checking)
- An OpenAPI spec (generated from live route symbols, always current)
- A flat bundle (for edge deployment)
- Independent services (partitioned at clean module boundary cut points)
- A semantic changelog (symbol-level diff between versions, not line diffs)

The graph doesn't know or care which projection you choose. Agents operate on the graph.
Materialization is a rendering decision made at deploy time.

**The file is not the atom. The symbol is.**

---

## What this unlocks

Every interesting agentic task is either a reachability query or a join between runtime
behavior and code structure:

**Reachability queries** (graph-native):
- Find every caller of this function across 300 files
- Trace untrusted input from HTTP entry point to database sink without passing a validator
- Find all circular dependencies that block microservice extraction
- Detect dead code with no reachable callers from any entry point

**Runtime + structure joins** (relational):
- Find DB queries averaging >100ms in traces, identify missing indexes, generate migrations
- Find all symbols that throw unhandled exceptions in production traces
- Add caching to expensive computations identified from OTel span duration data

These two query types cover most of what a senior engineer does day to day — not just
writing code but reasoning about a codebase at scale.

---

## Feedback loop

The system closes the full agent feedback loop across four layers:

- **CST** — valid structure enforced at schema level; lossless roundtrips via concrete
  syntax tree (comments, trivia preserved)
- **LSP** — type and semantic errors caught before commit
- **REPL** — runtime behavior validated during development
- **OTel traces** — execution history queryable as data, enabling session replay

Agents read traces to diagnose failures rather than stepping through a debugger. Root
cause analysis becomes a chained query across traces and the symbol graph — no
reproduction required.

---

## SDLC

The system adds a faster inner loop. It does not replace existing outer loops. Every
existing gate — linters, type checkers, test suites, CI pipelines — still fires against
real materialized files, unchanged. The graph layer means code is already well-validated
by the time it reaches those gates.

**The six-step flow:**

```
1. query    → understand reality (graph + traces)
2. REPL     → grow solution against real data (red → green)
3. commit   → persist to graph atomically (LSP clean, unit tests pass)
4. swarm    → scale across codebase in one transaction
5. pre-commit hook → existing gates fire against materialized files
6. CI       → integration, e2e, deploy (unchanged)
```

Steps 1–4 are agent-native and new. Steps 5–6 are existing tooling, parasitized cleanly.

**Red/green in this system:**

Traditional TDD specifies behavior before implementation using invented fixtures. Here
the OTel trace table is a library of real inputs that already caused real behavior in
production. Red/green stops being "does it pass my imagined inputs" and becomes "does it
pass what production actually threw at it."

```
write symbol: processPayment (interface only)
query traces: find all real inputs that hit this path
REPL: run test against symbol → red (not implemented)
      grow implementation until green against real inputs
commit: symbol + implementation + test as one atomic transaction
```

Unit tests live in the REPL loop, pre-commit, validated against real data. Integration
tests live post-commit in the materialized sandbox. That boundary is enforced by the
architecture, not by convention.

**Example: add input validation to all untrusted entry points**

```
query graph   → find every path from HTTP entry point to DB sink
                with no validator in between
REPL          → draft validator
              → red: unguarded input reaches DB
              → test against real traced production inputs
              → green
commit        → validator to graph, LSP confirms types hold
swarm         → insert validator at every unguarded path, one transaction
pre-commit    → eslint, tsc, existing test suite fire against materialized files
CI            → integration tests, docker build, deploy
```

The trace-as-fixtures step is the sharpest delta from today. You are not imagining
edge cases — you are replaying what production actually sent.

**Example: add DB indexes for slow queries**

```
trace_query   → find spans averaging >100ms
graph_query   → find emitting symbols, identify unindexed columns
REPL          → draft migration
              → materialize sandbox DB
              → replay slow traced queries against it
              → new spans confirm duration dropped
              → green
commit        → migration to graph
pre-commit    → existing migration linting fires
CI            → integration tests run against migrated schema, deploy
```

You ship an index you already measured fixing the problem. Not hoped.

**Example: generate OpenAPI spec from live symbol graph**

```
graph_query   → extract route symbols, input/output types, auth guards
REPL          → grow spec materializer interactively
              → red: missing nullable fields, wrong error responses
              → refine against real traced request/response pairs
              → green
commit        → materializer to graph
materialize   → spec file lands in repo alongside code on every commit
pre-commit    → openapi-lint fires against materialized spec
CI            → contract tests against generated spec
```

The spec is never stale because it materializes from the live graph. Changing a route
handler's return type updates the spec automatically on next materialization.

**Git compatibility:**

The pre-commit hook materializes files and runs whatever checks the project already has.
Git sees a conventional file tree. CI sees a conventional file tree. No changes required
to existing pipelines. Teams adopt the graph layer without touching their infrastructure.

---

## Live development

The REPL is not a validation step bolted onto the graph. It is a peer to the graph —
two views of the same program, one structural and persistent, one live and exploratory.

```
eval: define a new function → exists in runtime immediately
eval: call it with real traced inputs → see actual behavior
eval: refine → redefine in place
commit: when satisfied → write back to graph
```

The graph is the save state. The REPL is the workspace. Commit is the gesture that moves
work from the live runtime into the persistent graph. You grow the program from inside a
live runtime and persist when confident — the Clojure REPL model, applied to TypeScript
and Python, grounded in real production data.

---

## Storage

One database holds the full working state. Four tables:

- `files(id, path, language)` — materialization targets; import boundary markers
- `nodes(id, file_id, kind, parent_id, start, end, properties)` — every CST node
- `symbols(id, name, kind, definition_node_id, version)` — cross-file semantic index
- `traces(id, span_id, parent_span_id, name, start_time, end_time, attributes JSON)`
  — OTel spans from the REPL pool; runtime behavior alongside code structure

File boundaries matter only as import contracts and materialization paths. Inside the DB
they are foreign keys, not organizing principles. A cross-file refactor is a query across
`file_id`s. Moving a function between files is `UPDATE nodes SET file_id = ?`.

**Graph vs relational:** code structure and traversal queries are native graph operations.
Runtime behavior (traces, aggregates, joins) is native relational. V1 uses SQLite —
pragmatic, embedded, ships fast. V2 evaluates Kuzu (embedded graph DB, Cypher queries,
zero-infra) with DuckDB alongside for relational and trace queries. The materialization
layer is decoupled from storage and stays identical across the swap.

---

## Concurrency

`symbols.version` is the concurrency primitive. Any mutation to a symbol's name,
signature, or type increments its version. Second writer on the same symbol sees a
version mismatch and aborts. New reference insertions are non-conflicting and compose
freely.

**V1 gap:** write-write conflicts detected; read-set staleness is not. Mitigated by
mandatory pre-commit LSP re-validation. Full OCC via transaction manifest deferred to V2.

Abort returns a conflict error. Retry policy is caller-defined.

---

## Lifecycle

```
git clone
    ↓
hydrate: parse source → populate graph (tree-sitter; low seconds for 50k-line repo)
    ↓
agents work in graph: query, mutate, validate, eval
    ↓
materialize: graph → chosen projection (file tree, API spec, service split, etc.)
    ↓
git commit → DB discarded
```

Git is the durable save state. The DB is ephemeral, per-session, never committed.
Worktrees are reflink copies of the DB file plus a corresponding git worktree — branching
is a filesystem primitive on CoW filesystems (btrfs, APFS, ZFS).

---

## Editing flow

Every structural edit:
1. Mutates the graph in the DB
2. Materializes a fresh text buffer (deterministic tree walk, low single-digit ms)
3. Pushes buffer as `textDocument/didChange` to keep LSP virtual document consistent

For editor-integrated LSP (OpenCode, Cursor): write materialized file to actual worktree
path on commit; file watcher handles LSP sync naturally.

---

## Snapshots

`snapshot_restore` is a three-system reset: quiesce REPL pool → DB rollback → LSP
didClose all affected (batch) → materialize T1 source → LSP didOpen all affected (batch)
→ mark REPL sessions dirty. Two-phase batch ensures LSP rebuilds from consistent state.
Savepoints are SQLite-native; no GC required.

---

## CLI and MCP surface

```bash
# lifecycle (humans + CI)
ast-sqlite hydrate
ast-sqlite worktree add --branch <n>
ast-sqlite worktree materialize [--target openapi|files|services]
ast-sqlite worktree discard
```

```typescript
// agent tools (MCP)
session_init()                    // file tree + symbol graph
symbol_query(name)                // definition + references
node_mutate(fileId, mutation)     // structural edit + LSP validation
eval(expression, language)        // materialize + REPL + OTel
snapshot_create(name)
snapshot_restore(id)
materialize_to_sandbox(fileIds)   // for fs-native tool runs
hydrate_from_sandbox(path, paths)
trace_query(sql)                  // query OTel spans
graph_query(cypher)               // V2: native graph traversal
```

`.mcp.json` in the repo root enables automatic agent discovery via any MCP-aware client.

---

## Language scope (V1)

**TypeScript** — ts-morph / compiler API (trivia-preserving, CST by default). LSP via
tsserver.

**Python** — libcst required; stdlib `ast` strips comments and cannot be used. LSP via
pyright.

Each additional language requires a CST-capable parser, schema, materializer, and LSP
integration.

---

## What this is not

A replacement for Git, a new version control system, or a tool for humans editing code
directly. Git handles history. Humans work normally. The graph layer exists only for the
duration of a swarm session, then discards itself. The materialized output is a
conventional project tree any existing CI pipeline consumes unchanged.

The shift is not in how code is stored long-term. The shift is in what becomes possible
during the session when agents have a queryable graph instead of a pile of text files.
