---
name: compiler-query-system
description: Incremental compilation and query system expert — covers lazy demand-driven computation, query-based compiler architecture (rustc style), cache invalidation strategies, and how to make a compiler incremental. Use when designing an incremental compiler, build system integration, or a demand-driven computation framework.
---

# Incremental Compilation and Query Systems

---

## What a Query System Is

### Traditional Compiler: Linear Passes

A conventional compiler runs a sequence of passes over the entire program:
```
parse → name resolution → type check → borrow check → codegen
```
Every pass runs to completion before the next starts. When the user edits one file, the entire pipeline reruns from scratch. For large programs, this is unacceptably slow.

### Query-Based Compiler: Demand-Driven Computation

A query system inverts the model. Instead of "run all passes," the compiler exposes a set of **queries** — functions that compute a specific piece of information:
- `parse_file(path)` → AST
- `type_of(expr_id)` → Type
- `mir_of(fn_id)` → MIR
- `codegen_unit(crate_id)` → ObjectFile

Queries are **lazy**: they run only when their result is demanded. Queries are **memoized**: once computed, the result is cached and reused as long as inputs have not changed. Queries are **composable**: they can call other queries, which may themselves be cached or need recomputation.

The result: on an incremental rebuild, only the queries whose inputs have changed (directly or transitively) are re-executed.

---

## rustc's Query Architecture

### How Queries Compose

Every piece of compiler-computed information in rustc is a query. The dependency graph is built automatically at runtime: when query A calls query B, an edge A → B is recorded. This graph captures the full dependency structure without the programmer manually specifying what depends on what.

```
codegen_crate
  └── mir_of(fn_id)
        └── type_check(fn_id)
              ├── type_of(expr_id_1)
              │     └── parse_file("src/lib.rs")
              └── type_of(expr_id_2)
                    └── parse_file("src/lib.rs")  ← same node, reused
```

### Query Inputs

At the bottom of the dependency graph are **inputs** — source files, environment variables, command-line flags. Inputs are the only things that change between incremental builds. Everything else is derived.

### Result Caching

Query results are stored in a **query cache** keyed by (query name, query arguments). On a subsequent build:
1. Look up the cached result for the query.
2. Check if any input that contributed to this result has changed (using the dependency graph).
3. If unchanged: return cached result immediately (green node).
4. If changed: re-execute the query, update the cache (red node).

---

## Cache Invalidation: Red/Green Computation

### The Problem

Naive invalidation: when file F changes, invalidate everything that transitively depends on F. This is safe but over-invalidates — if only a comment changed in F, the AST is the same, and nothing downstream needs to rerun.

### Red/Green Strategy

rustc uses a finer-grained approach:
1. Re-execute only the queries directly over changed inputs (e.g., `parse_file` for edited files).
2. Compare the new result against the cached result.
3. If the result is **equal** to the cached result (green): mark the node green and do not propagate invalidation upward. Downstream queries can still use their cached results.
4. If the result **differs** (red): invalidate all downstream queries.

**Key insight**: changing a comment or whitespace in a function makes `parse_file` red, but if the resulting AST is identical (after normalization), `type_check` remains green. The cost of a "null rebuild" is checking equality of the changed nodes' outputs, not re-running everything.

### Change Detection

Inputs are compared using content hashing (not timestamps). A file's hash is stored; on rebuild, if the hash matches, the file is treated as unchanged even if its modification time changed (e.g., after `git checkout`).

---

## Cycle Detection

### The Problem

Queries can be mutually recursive. A naive query system will infinite-loop. Even a sound type system can have queries that look recursive (e.g., resolving a type alias that refers to itself by mistake).

### Detection Strategy

rustc uses a **cycle detection table** per query type. When a query Q(x) begins execution:
1. Check if Q(x) is already in progress (in the "currently executing" set).
2. If yes: a cycle has been detected. Report an error (or use a provisional value if the query system supports it).
3. If no: add Q(x) to the in-progress set, execute, remove on completion.

### Provisional Values

Some query systems (Chalk, Salsa) support **provisional values** for cyclic queries:
- Start with an "optimistic" provisional result (e.g., `true` for a subtyping check).
- Re-execute queries in the cycle using the provisional value.
- Repeat until a fixpoint is reached (result stabilizes).

This enables recursive types and mutually recursive trait impls to be handled correctly. The implementation is significantly more complex than simple cycle detection.

---

## Salsa Framework

Salsa is the query system abstracted from rustc into a reusable Rust library. It provides:

- **`#[salsa::query_group]`**: declare a trait of queries; Salsa generates the caching and dependency tracking infrastructure
- **`#[salsa::input]`**: mark a query as an input (no dependencies, set by the caller)
- **`#[salsa::tracked]`**: mark a struct as a tracked entity whose identity persists across revisions
- **Revisions**: each time an input changes, the global revision counter increments; queries re-execute if their inputs' last-changed revision is newer than their cached revision

Salsa handles:
- Memoization
- Dependency graph construction
- Red/green change propagation
- Cycle detection
- Parallel query execution (with careful locking)

Salsa is used by rust-analyzer (the Rust LSP server) and was the basis for rustc's own query system design.

---

## When to Use a Query System

### Use a query system when:

- **Large programs**: re-parsing + re-type-checking entire program on every keystroke is too slow
- **IDE integration**: language server needs to answer queries (go-to-definition, hover type, completion) on partially-edited, possibly-inconsistent source
- **Modular compilation**: independent modules can be checked in parallel; queries for one module do not block queries for another
- **Iterative development**: developers edit one file at a time; incremental rebuilds dominate total build time

### Prefer simpler strategies when:

- **Small programs**: a full rebuild takes < 1 second; incremental overhead (hashing, graph maintenance) adds latency without benefit
- **Batch compilation**: compiling for release; full rebuild is expected; no IDE integration needed
- **Simple pipeline**: the compiler has 2-3 passes with no cross-module dependencies; a dependency graph is overkill
- **Prototype stage**: query system adds significant architectural complexity; wait until the compiler architecture is stable

### Incremental alternatives that are simpler than a full query system:

- **File-level dependency tracking**: recompile only translation units whose source files changed. No intra-file incremental, but very simple to implement.
- **Module-level separate compilation**: compile modules to `.o` files; relink only. Standard in C/C++ with `make`.
- **Memoized passes**: cache the output of expensive passes (e.g., parsed AST) to disk using content-addressed hashing. Simpler than a full query graph.

---

## Deep Reference

A comprehensive reference document covering all aspects of the query system — from the abstract model to rustc implementation details to Salsa's API — is available at:

**`references/query-reference.md`**

It covers:
1. Linear passes vs query architecture and why queries enable incrementality
2. The query model: keys, values, providers, the dependency DAG, the red/green algorithm, cycle detection
3. rustc internals: how `tcx.type_of(def_id)` works, the `rustc_queries!` macro, provider registration, the query cache
4. Salsa framework: `#[salsa::input]`, `#[salsa::tracked]`, `#[salsa::interned]`, accumulators, the database struct, durability
5. Incremental compilation: serialization, fingerprints, `HashStable`, stable identifiers (`DefPath`, `DefPathHash`), what can/cannot be incremental, the two-dep-graph problem
6. Cycle handling: detection mechanism, provisional values and fixpoint iteration, cross-thread cycles in Salsa
7. Performance: query overhead, the projection query pattern, memory usage of the dep graph
8. Implementation decision guide: when to adopt queries, minimal implementation phases, Salsa minimum viable setup

Sources: rustc-dev-guide (query.html, incremental-compilation-in-detail.html, query-evaluation-model-in-detail.html) and Salsa framework documentation (salsa-rs.github.io/salsa). All content tagged [rustc-dev-guide] or [Salsa].

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-architect` — query system is an architectural layer choice (incremental vs. batch)
- `compiler-expert` — routes here on incremental compilation / cache invalidation questions

**Trace Down ↓** — the query system orchestrates other skills' outputs as cached results
- `compiler-hir` — HIR is typically a query result (type-check query → typed HIR)
- `compiler-optimizer` — optimization passes can be made incremental via the query system

**Related →** — closely related architectural skills
- `compiler-architect` — the query system is a cross-cutting architectural choice
- `compiler-mir` — MIR queries and borrow-check queries are common incremental units
