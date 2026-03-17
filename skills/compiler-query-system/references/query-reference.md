# Compiler Query System: Deep Reference

Sources researched: rustc-dev-guide (query.html, incremental-compilation-in-detail.html, query-evaluation-model-in-detail.html) and Salsa framework documentation (salsa-rs.github.io/salsa).

---

## 1. Linear Passes vs Query Architecture

### Traditional Linear Pass Compiler

A conventional compiler runs a fixed sequence of passes over the entire program:

```
parse → name resolution → type check → borrow check → MIR lower → codegen
```

Each pass runs to completion before the next begins. Every pass reads from the previous pass's output and writes to the next pass's input. When the user edits one source file, the entire pipeline reruns from scratch. For large programs (millions of lines), this is unacceptably slow.

**Problems with linear passes:**
- No incremental reuse: even if only a comment changed, every pass reruns
- Pass ordering is fixed at compile time — you cannot "demand" only what you need
- The dependency structure between computations is implicit and not tracked at runtime
- No natural way to parallelize across different parts of the program

### Query-Based (Demand-Driven) Architecture

A query system inverts the model. Instead of "run all passes," the compiler exposes a set of **queries** — named functions that compute a specific piece of information on demand:

```
type_of(def_id)      → Ty<'tcx>
mir_of(fn_id)        → &Mir
parse_file(path)     → &Ast
codegen_unit(id)     → ObjectFile
```

**Why queries enable incrementality:**

1. **Queries are lazy**: they execute only when their result is demanded. If nothing demands the MIR of an unreachable function, it is never computed.

2. **Queries are memoized**: once computed, the result is stored in a cache. Subsequent calls with the same key return the cached result immediately.

3. **Queries track their dependencies automatically**: when query A executes and calls query B, an edge A→B is recorded in the dependency graph. This happens at runtime without the programmer manually specifying what depends on what.

4. **Re-execution is targeted**: on an incremental rebuild, only queries whose transitive inputs have changed are re-executed. All other queries return their cached results instantly.

[rustc-dev-guide]: "Instead of entirely independent passes (parsing, type-checking, etc.), a set of function-like queries compute information about the input source."

---

## 2. The Query Model

### What a Query Is

[rustc-dev-guide]: "A query thus consists of the following things:
- A name that identifies the query
- A 'key' that specifies what we want to look up
- A result type that specifies what kind of result it yields
- A 'provider' which is a function that specifies how the result is to be computed if it isn't already present in the database."

Concretely, a query is a **pure function from a key to a value**, backed by a memoization cache:

```
query_name : Key → Value
```

The key type must implement `QueryKey` (in rustc) or be a "Salsa struct" (in Salsa). The value type must be cheaply cloneable (use `Rc`, `Arc`, or interned values for large data).

**Constraints for soundness:**
- The key and result must be immutable values
- The provider function must be a pure function: same key always yields same result
- The only parameters a provider takes are the key and a reference to the query context (the "database")
- Provider functions may only access other queries and pre-loaded input data — not arbitrary mutable global state

### The Dependency Graph

[rustc-dev-guide]: "Since every access from one query to another has to go through the query context, we can record these accesses and thus actually build this dependency graph in memory."

When query A's provider runs and calls `tcx.type_of(bar_def_id)`, the query engine records the edge:

```
A  →  type_of(bar_def_id)
```

By the time a top-level query completes, the engine has built a directed acyclic graph (DAG) of all the dependencies for that computation:

```
type_check_crate()
    ├── list_of_all_hir_items   (input)
    ├── type_check_item(foo)
    │     ├── type_of(foo)
    │     │     └── Hir(foo)   (input)
    │     └── type_of(bar)
    │           └── Hir(bar)   (input)
    └── type_check_item(bar)
          └── type_of(bar)     (shared — same node reused)
```

This graph is the core data structure for incremental compilation. It is built at runtime, not at compile time, so it accurately reflects what was actually accessed (not what could have been accessed).

**Nodes in the graph** correspond to query invocations (identified by query name + key fingerprint).

**Edges** represent "A read from B": if B changes, A may need to be re-executed.

**Leaves** are inputs (source files, command-line flags, upstream crate metadata).

### The Red/Green Algorithm

[rustc-dev-guide]: "We call this algorithm the red-green algorithm because nodes in the dependency graph are assigned the color green if we were able to prove that its cached result is still valid and the color red if the result has turned out to be different after re-evaluating it."

**The naive algorithm (and its flaw):**

A naive approach walks the dependency graph from a changed input outward, invalidating everything transitively. This is correct but has severe false positives. Example:

```
IntValue(x) = 1000  →  sign_of(x) = "+"  →  some_other_query(x)
```

If `IntValue(x)` changes to 2000, `sign_of(x)` still returns `"+"`. The naive algorithm would force `some_other_query(x)` to recompute even though nothing it cares about changed.

**The red/green algorithm (used in rustc and Salsa):**

Instead of propagating invalidation eagerly, the algorithm interleaves change-detection with re-evaluation:

```
fn try_mark_green(tcx, current_node) -> bool {
    let dependencies = tcx.dep_graph.get_dependencies_of(current_node);

    for dependency in dependencies {
        match tcx.dep_graph.get_node_color(dependency) {
            Green => { /* already verified, continue */ }
            Red => {
                // A changed input was found; cannot mark green without re-running
                return false
            }
            Unknown => {
                // Try to mark this dependency green recursively
                if try_mark_green(tcx, dependency) {
                    // dependency is green; continue
                } else {
                    // Could not mark green; re-run the query for this dependency
                    tcx.run_query_for(dependency);
                    match tcx.dep_graph.get_node_color(dependency) {
                        Red => return false,   // result changed
                        Green => { /* result same; continue */ }
                        Unknown => panic!("unreachable"),
                    }
                }
            }
        }
    }

    // All inputs are green → this node's cached result is still valid
    tcx.dep_graph.mark_green(current_node);
    true
}
```

[rustc-dev-guide]: "By using red-green marking we can avoid the devastating cumulative effect of having false positives during change detection."

**The key insight**: when we re-execute a query and its result is **identical** to the cached result (after hashing), we mark the node **green**. Downstream nodes that depend only on green nodes do not need to re-execute, even if some of their transitive inputs changed.

**Practical flow for incremental builds:**
1. Load the previous dependency graph and fingerprints from disk
2. For each query demanded, call `try_mark_green`
3. If the node cannot be marked green, re-execute the query provider
4. After re-execution, compare the result's fingerprint to the stored fingerprint
5. If fingerprints match: mark green (result unchanged, do not propagate)
6. If fingerprints differ: mark red (result changed, downstream must re-execute)

### Cycle Detection and How rustc Handles It

[rustc-dev-guide]: "The query engine includes a check for cyclic invocations of queries with the same input arguments. And, because cycles are an irrecoverable error, will abort execution with a 'cycle error' message that tries to be human readable."

**How cycles arise:** Invalid user code can trigger cycles. Example: a type alias that refers to itself:

```rust
type Foo = Vec<Foo>;  // triggers a cycle in type_of(Foo)
```

The query engine detects this by maintaining a "currently executing" set per thread. When a query Q(k) begins:
1. Check if Q(k) is already in the in-progress set
2. If yes: cycle detected — report a user-facing error
3. If no: add Q(k) to the set, execute, remove on completion

**Historical note on cycle recovery:** [rustc-dev-guide] "At some point the compiler had a notion of 'cycle recovery', that is, one could 'try' to execute a query and if it ended up causing a cycle, proceed in some other fashion. However, this was later removed because it is not entirely clear what the theoretical consequences of this are, especially regarding incremental compilation."

**Cross-thread cycles (Salsa):** [Salsa] When thread T1 wants to block on a query Q being executed by thread T2, the runtime checks whether T2 is (transitively) blocked on T1 before adding the blocking edge. If so, a cycle is detected. The runtime maintains a dependency DAG between threads (identified by RuntimeId) to make this check efficiently.

---

## 3. rustc's Query System

### How `tcx.type_of(def_id)` Works Under the Hood

[rustc-dev-guide]: "The TyCtxt ('type context') struct offers a method for each defined query. For example, to invoke the type_of query, you would just do this: `let ty = tcx.type_of(some_def_id);`"

When you call `tcx.type_of(def_id)`, the following happens:

1. **Cache lookup**: the query engine looks up `(type_of, def_id)` in the in-memory query cache
2. **Cache hit (non-incremental)**: return the cached `Ty<'tcx>` immediately
3. **Cache hit (incremental)**: call `try_mark_green` on the node; if green, return cached value; if red, proceed to step 4
4. **Cache miss**: look up the provider function in the `Providers` struct
5. **Local vs external**: check if `def_id.krate == LOCAL_CRATE`; if local, use `Providers`; if external, use `ExternProviders` (which routes through `rustc_metadata` to read the `.rmeta` file)
6. **Execute provider**: call `type_of_provider(tcx, def_id)` with dependency tracking active
7. **Record dependencies**: all queries invoked during the provider's execution are recorded as edges in the dep graph
8. **Cache result**: store the result with a fingerprint
9. **Return**: clone the result out of the cache

### The `rustc_queries!` Macro and What It Generates

[rustc-dev-guide]: "To declare the query name and arguments, you simply add an entry to the big macro invocation in `compiler/rustc_middle/src/query/mod.rs`."

Query declarations look like:

```rust
rustc_queries! {
    /// Records the type of every item.
    query type_of(key: DefId) -> Ty<'tcx> {
        cache_on_disk_if { key.is_local() }
        desc { |tcx| "computing the type of `{}`", tcx.def_path_str(key) }
    }
}
```

The macro generates:
- A method on `TyCtxt` named `type_of` that invokes the query engine
- A struct `ty::queries::type_of` for use in provider registration
- Entries in the `Providers` and `ExternProviders` structs (function pointer tables)
- Dependency graph node types
- Cache infrastructure

**Query modifiers** (inline attributes inside the query definition):

| Modifier | Effect |
|---|---|
| `cache_on_disk_if { expr }` | Persist result to disk between sessions if `expr` is true |
| `eval_always` | Re-execute unconditionally on incremental builds (no dep tracking overhead) |
| `no_hash` | Do not compute a fingerprint for the result; always treat as potentially changed |
| `anon` | Use "anonymous" dep-nodes identified by input fingerprints rather than query key |
| `desc { ... }` | User-facing description shown in cycle error messages |

### Query Providers: Where Queries Get Their Answers

[rustc-dev-guide]: "A provider is a function implemented in a specific module and manually registered into either the `Providers` struct (for local crate queries) or the `ExternProviders` struct (for external crate queries) during compiler initialization."

Provider signature:

```rust
fn provider<'tcx>(
    tcx: TyCtxt<'tcx>,
    key: QUERY_KEY,
) -> QUERY_RESULT {
    // implementation
}
```

Providers are registered during compiler initialization:

```rust
pub fn provide(providers: &mut query::Providers) {
    *providers = query::Providers {
        type_of,
        fubar,
        ..*providers
    };
}
```

Each `rustc_*` crate exposes a `provide` function. The `rustc_interface` crate aggregates these into `DEFAULT_QUERY_PROVIDERS`. External crate queries route through `rustc_metadata::provide_extern`, which decodes `.rmeta` files.

**Adding a new query** requires:
1. Add an entry to `rustc_queries!` in `rustc_middle/src/query/mod.rs`
2. Implement the provider function in the owning crate
3. Register it via that crate's `provide` function
4. If cross-crate: add an external provider in `rustc_metadata` and ensure encoding/decoding in metadata

### The Query Cache: What Gets Cached, When It Is Invalidated

**In-memory cache**: a hash map from `(QueryName, Key)` to `(Value, Fingerprint, DepNodeIndex)`. Lives for the duration of the compilation session.

**On-disk cache**: persisted between sessions. Only queries tagged with `cache_on_disk_if { true }` are written. The cache stores results as serialized bytes alongside fingerprints.

**Cache invalidation** happens through the red/green algorithm:
- A node is invalidated (marked red) when it is re-executed and its fingerprint changes
- A node is kept (marked green) when all its dependencies are green, or when it is re-executed but produces the same fingerprint

**Cache promotion** [rustc-dev-guide]: "Before emitting the new result cache it will walk all green dep-nodes and make sure that their query result is loaded into memory. That way the result cache doesn't unnecessarily shrink." This prevents intermediate results that were never loaded in session N from being missing from the cache in session N+1.

---

## 4. Salsa Framework

### How Salsa Abstracts the Query Pattern

[Salsa]: "The goal of Salsa is to support efficient incremental recomputation. Salsa is used in rust-analyzer, for example, to help it recompile your program quickly as you type."

Salsa takes the patterns developed in rustc's query system and packages them as a reusable Rust library. The core model is:

```rust
let mut input = initial_input();
loop {
    let output = your_program(&input);  // Salsa memoizes this
    modify(&mut input);                 // triggers revision bump
}
```

Salsa enforces:
- The `your_program` function is deterministic (pure)
- Mutation of inputs happens outside tracked functions
- The database is the single source of truth for all computed values

The algorithm Salsa uses for change propagation is explicitly named the **red-green algorithm** (same as rustc).

### `#[salsa::input]` — Defining Inputs

[Salsa]:
```rust
#[salsa::input]
pub struct ProgramFile {
    pub path: PathBuf,
    pub contents: String,
}
```

The `#[salsa::input]` macro generates a struct that is just a newtyped integer ID:

```rust
// Generated:
#[derive(Copy, Clone, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct ProgramFile(salsa::Id);
```

The actual field data lives in the database. This means `ProgramFile` is trivially copyable and hashable — it can be used as a query key. Reading fields requires a database reference:

```rust
let contents: String = file.contents(&db);  // clones from database
```

Mutation requires `&mut db` and bumps the revision counter:

```rust
file.set_contents(&mut db).to(String::from("new content"));
```

### `#[salsa::tracked]` — Tracked Functions and Structs

**Tracked functions** are the primary computation unit in Salsa:

```rust
#[salsa::tracked]
fn parse_file(db: &dyn crate::Db, file: ProgramFile) -> Ast {
    let contents: &str = file.contents(db);
    // ... parse contents ...
}
```

When called, Salsa:
1. Records which input fields were accessed (e.g., `file.contents`)
2. Memoizes the return value
3. On subsequent calls, checks if tracked inputs have changed; if not, returns cached value

**Tracked structs** are intermediate values created during computation. Like inputs, they are newtyped IDs with data stored in the database. Unlike inputs, they are immutable after creation and can only be created inside tracked functions:

```rust
#[salsa::tracked]
struct Ast<'db> {
    #[returns(ref)]
    top_level_items: Vec<Item>,
}
```

**`#[id]` fields** on tracked structs allow Salsa to match up structs across re-executions by identity (e.g., a function named `foo` in the old execution matches `foo` in the new execution, even if creation order changed):

```rust
#[salsa::tracked]
struct Item {
    #[id]
    name: Word,  // used for identity matching
    body: Expr,
}
```

**`#[salsa::tracked(specify)]`** allows overriding a tracked function's result for a specific struct instance — useful for built-in items or lazily initializing fields:

```rust
#[salsa::tracked(specify)]
fn representation(db: &dyn crate::Db, item: Item) -> Representation { ... }

// Override for a specific item:
representation::specify(db, builtin_item, hardcoded_repr);
```

### `#[salsa::interned]` — Interned Structs

[Salsa]:
```rust
#[salsa::interned]
struct Word {
    #[returns(ref)]
    pub text: String,
}
```

Interned structs provide canonical identity: two `Word::new(db, "foo")` calls return the same integer ID. This enables O(1) equality comparison (just compare integers) instead of O(n) string comparison.

Use cases:
- String identifiers / symbols
- Any value that is compared for equality frequently and where the actual data is rarely needed
- Deduplication of repeated values (e.g., the same type appearing in many places)

### Accumulated Diagnostics Pattern

[Salsa]: Accumulators provide a side-channel for emitting diagnostics without polluting return values:

```rust
#[salsa::accumulator]
pub struct Diagnostics(String);

#[salsa::tracked]
fn type_check(db: &dyn Db, item: Item) {
    // ... type checking ...
    Diagnostics::push(db, "type error: ...".to_string());
}

// Collect all diagnostics emitted during type_check and its callees:
let errors: Vec<String> = type_check::accumulated::<Diagnostics>(db);
```

**Why this matters for incrementality**: if diagnostics were part of the return value of `type_check`, any change to the set of errors would make the result "changed" and propagate upward. By using accumulators, diagnostics are orthogonal to the function's primary output — the function can be marked green (its main computation unchanged) even if the diagnostics were collected differently.

### The Database Concept

[Salsa]:
```rust
#[salsa::db]
struct MyDatabase {
    storage: Storage<Self>,
    // optional custom fields:
    source_map: HashMap<PathBuf, String>,
}
```

The database is the central runtime object. It contains:

- `Storage<Self>`: Salsa-managed storage for all memoized values, input data, interned values, and dependency graph information
- User-defined fields: arbitrary data accessible to tracked functions (e.g., file I/O handles, configuration)

**Parallel access**: the database supports snapshotting. A `Snapshot<DB>` can be shared across threads and only allows `&`-dereferencing (read-only). Mutable operations (setting inputs) require `&mut DB` and first cancel all snapshots via cancellation tokens.

**The revision counter**: every time an input is mutated, Salsa bumps a global revision number. Memoized results store the revision at which they were last verified. The `maybe_changed_after(R)` check compares stored revisions before recursing into dependency checks.

**Durability**: inputs can be marked with a durability level (low, medium, high). Tracked functions record the minimum durability of their dependencies. If no input with durability ≤ D has changed since last verification, a function with minimum durability D can skip its dependency checks entirely — a significant optimization for stable inputs like standard library definitions.

---

## 5. Incremental Compilation

### What Gets Serialized Between Sessions

At the end of a compilation session, rustc serializes to disk:

1. **The dependency graph**: all dep-nodes and their edges, identified by stable fingerprints
2. **Query result fingerprints**: 128-bit hashes of each query result (for change detection)
3. **The query result cache**: actual serialized query results for queries tagged `cache_on_disk_if`
4. **LLVM bitcode files**: for codegen units that were unchanged
5. **Object files**: for codegen units that were unchanged

The dep graph and fingerprints together enable `try_mark_green` to work across sessions.

### The Fingerprint/Hash Approach for Change Detection

[rustc-dev-guide]: "Each time a new query result is computed, the query engine will compute a 128 bit hash value of the result. We call this hash value 'the Fingerprint of the query result'."

**The challenge**: hashing must be "stable" across compilation sessions. A `DefId` (a numeric ID assigned during compilation) may refer to different items in different sessions (e.g., adding a function in the middle of a file shifts all subsequent `DefId`s). Plain hashing of `DefId` would be unstable.

**The solution — `HashStable` and stable identifiers**:
- `DefId` → hashed as its `DefPathHash` (a hash of the item's full path, e.g., `std::collections::HashMap`)
- `DefPath`: the human-readable path to an item; stable across unrelated changes
- `DefPathHash`: a 128-bit hash of the `DefPath`; used as the stable identifier in the dep graph
- `HirId`: a pair of `(DefPath, LocalId)`; stable if the owning item doesn't move

The `HashStable` infrastructure ensures that wherever a potentially-unstable ID appears during hashing, its stable equivalent is used instead.

**Fingerprint collision risk**: [rustc-dev-guide] "There is a small possibility of hash collisions. That is, two different results could have the same fingerprint and the system would erroneously assume that the result hasn't changed, leading to a missed update. We mitigate this risk by using a high-quality hash function and a 128 bit wide hash value."

**Performance cost**: [rustc-dev-guide] "Computing fingerprints is quite costly. It is the main reason why incremental compilation can be slower than non-incremental compilation. We are forced to use a good and thus expensive hash function, and we have to map things to their stable equivalents while doing the hashing."

### What Can and Cannot Be Incremental

**Can be incremental:**
- Type checking, borrow checking, MIR construction — all query-based
- Name resolution — query-based
- Trait solving — query-based
- Anything expressed as a query with `cache_on_disk_if`

**Cannot be fully incremental:**
- LLVM codegen: LLVM modules are opaque C++ objects; no `HashStable` implementation; instead, rustc tracks which queries were accessed when generating each CGU and re-emits the CGU if those queries changed
- Macro expansion with side effects: `eval_always` queries handle this, but at the cost of always re-running
- On-disk cache updates: the current implementation rewrites entire cache files each session (not in-place updates) — a few percent overhead per build

**The two dep-graph problem** [rustc-dev-guide]: During an incremental session, two dep graphs exist simultaneously:
- **Old graph** (loaded from disk, immutable): used to look up previous dependencies and fingerprints
- **New graph** (built in-memory): populated as queries execute

When a node is marked green, its node and edges are **copied** from the old graph to the new graph (because the regular dependency tracking mechanism only records edges during actual execution — but for green nodes, we avoided execution). At session end, the new graph is serialized to disk.

### The Red/Green Change Propagation Algorithm in Detail

Full algorithm for an incremental build session:

1. **Load**: deserialize old dep graph + fingerprints + query result cache from disk
2. **Hash inputs**: compute fingerprints for all input files (source, command-line flags)
3. **Compare inputs**: nodes corresponding to changed inputs are immediately marked **red**; unchanged inputs are **green**
4. **Demand-driven propagation**: as the compiler demands query results:
   a. Look up the dep-node in the old graph
   b. If not found (new query key): execute the provider, record as **red** (new result)
   c. If found: call `try_mark_green`
      - Recursively verify all dependencies
      - If all dependencies are green: mark this node **green**, load result from cache
      - If any dependency is **red**: re-execute the provider, compare fingerprint
        - Same fingerprint: mark **green** (result unchanged despite input change)
        - Different fingerprint: mark **red** (result changed, propagate upward)
5. **Cache promotion**: walk all green nodes, ensure their results are loaded into memory
6. **Serialize**: write new dep graph, fingerprints, and updated query cache to disk

---

## 6. Cycle Handling

### How Cycles Arise

Cycles arise from invalid user code where the compiler's computation of some fact A depends on fact B which depends on A:

```rust
// Recursive type (triggers cycle in type_of):
type Foo = Vec<Foo>;

// Mutually recursive trait bounds:
trait A: B {}
trait B: A {}
```

In a query system, this manifests as: while computing `type_of(Foo)`, the provider calls `type_of(inner_type)`, which eventually calls `type_of(Foo)` again.

### rustc's Cycle Error Approach

[rustc-dev-guide]: "Because cycles are an irrecoverable error, [the query engine] will abort execution with a 'cycle error' message that tries to be human readable."

Detection mechanism:
- Each thread maintains a stack of currently-executing queries
- When a query Q(k) is about to execute, check if Q(k) is already on the stack
- If yes: emit a cycle error with a description of the cycle chain

The `desc { ... }` attribute on query declarations provides the user-facing description shown in the cycle error message.

### Provisional Values and Fixpoint Iteration (Salsa / Chalk)

[Salsa] supports a more sophisticated cycle handling strategy for cases where cycles can be resolved via fixpoint iteration (e.g., recursive type inference, recursive trait solving):

**Provisional value strategy:**
1. When a cycle is detected for query Q(k), instead of immediately erroring, provide a "provisional" initial value (e.g., `false` for a boolean query)
2. Continue executing the computation using the provisional value
3. After the cycle completes, compare the result to the provisional value
4. If they differ: the fixpoint has not been reached; re-execute with the new result as the provisional value
5. Repeat until the result stabilizes (fixpoint)

This enables correct handling of:
- Recursive type definitions (under controlled conditions)
- Mutually recursive trait implementations
- Dataflow analyses that need multiple passes

**Cross-thread cycle detection in Salsa** [Salsa]: The runtime maintains a DAG of inter-thread blocking relationships. Before T1 blocks on T2, the runtime checks whether a T2→...→T1 path already exists in the DAG. If so, adding T1→T2 would create a cycle among threads, which is detected and reported as a query cycle error.

### The Cycle Error Fallback

When a cycle cannot be resolved via provisional values, rustc:
1. Reports a user-facing error ("cycle detected when computing X")
2. Returns a "cycle error sentinel" value from the query
3. Continues compilation (to find other errors) but uses the sentinel value wherever the result would have been used
4. Suppresses downstream errors that are only present because of the cycle sentinel

---

## 7. Performance Characteristics

### Query Overhead vs Pass Overhead

**Overhead of the query system:**
- Cache lookup on every query invocation (hash map lookup: ~10–50 ns)
- Dependency edge recording (push to a thread-local Vec: ~5–10 ns per edge)
- Fingerprint computation on every new result (expensive: 100 ns – 1 ms depending on result size)
- Dependency graph serialization at session end (a few percent of total build time)
- On incremental builds: `try_mark_green` traversal instead of immediate return

**Benefits that offset the overhead:**
- On incremental rebuilds, query results from the previous session are reused without re-execution
- Queries for unchanged code return from the in-memory cache in nanoseconds
- Lazy evaluation: queries for unreachable code are never executed

[rustc-dev-guide]: "Computing fingerprints is quite costly. It is the main reason why incremental compilation can be slower than non-incremental compilation."

### When to Use Queries vs a Simple Function

**Use a query (with memoization and dep tracking) when:**
- The result is expensive to compute (> 1 µs)
- The result is likely to be requested multiple times (e.g., `type_of` is called thousands of times per crate)
- The result needs to participate in incremental compilation (be cached across sessions)
- The result's dependencies need to be tracked for correct invalidation

**Use a plain function when:**
- The computation is trivial (a few arithmetic operations, a lookup in a local data structure)
- The result is always consumed immediately and never reused
- The inputs are not tracked queries (e.g., a helper that processes already-computed data)
- You are inside a query provider and computing an intermediate value that will be subsumed by the query's result

[Salsa]: "They can take additional arguments, but it's faster and better if they don't." — extra arguments on tracked functions mean finer-grained dependency tracking but higher overhead per call.

### Memory Usage of the Dependency Graph

The dependency graph is the primary memory cost of the query system:

- **Nodes**: one per unique `(QueryName, Key)` pair executed. For a large crate this can be millions of nodes.
- **Edges**: on average, each node has several dependencies. Total edges can be tens of millions.
- **Fingerprints**: 128 bits per node. For 1 million nodes: ~16 MB.
- **Serialized dep graph**: typically 10–100 MB for a large crate.

**Memory optimization strategies used in rustc:**
- `no_hash` on "projection queries" (see below) avoids double-hashing
- `eval_always` on queries that are certainly going to change avoids tracking their dependencies
- The "projection query pattern" limits the blast radius of changes to large data structures

**The projection query pattern** [rustc-dev-guide]:

```
                  +---------------+           +--------+
monolithic_query  | projection(x) | ←-------- | foo(a) |
(no_hash +        +---------------+           +--------+
eval_always)      +---------------+           +--------+
                  | projection(y) | ←-------- | bar(b) |
                  +---------------+           +--------+
```

When `monolithic_query` (e.g., the entire indexed HIR) changes, only the projections that actually changed become red. `foo(a)` and `bar(b)` can remain green if their respective projections are unchanged. Without the projection layer, `foo`, `bar`, and everything else depending on the monolithic query would all be forced to re-execute.

---

## 8. Implementation Decision Guide

### When to Adopt a Query Architecture

**Strong signals that a query system is worth the investment:**
- You are building an IDE or language server that needs sub-second response times
- Your compiler has > 50k lines of Rust/C++ source as the target language
- You need to support "watch mode" (continuous recompilation on file changes)
- Your pipeline has significant cross-module dependencies (type information flowing between files)
- You have evidence that incremental rebuilds dominate developer time

**Signals that a simpler approach is sufficient:**
- Your compiler targets a scripting language or small DSL where full compilation < 500 ms
- You are in early prototype stage and the query topology is not yet stable
- Your "incremental" story is just "don't relink if object files haven't changed" (standard `make`)

### Minimal Implementation Path

**Phase 1: Memoized functions (no dep tracking)**

Start with a simple `HashMap<Key, Arc<Value>>` per query type. This eliminates redundant recomputation within a single session but provides no cross-session incrementality.

```rust
fn type_of(cache: &mut HashMap<DefId, Arc<Ty>>, key: DefId) -> Arc<Ty> {
    if let Some(v) = cache.get(&key) {
        return v.clone();
    }
    let result = Arc::new(compute_type_of(cache, key));
    cache.insert(key, result.clone());
    result
}
```

**Phase 2: Add a dependency tracker**

Introduce a `DepGraph` alongside the cache. Each query records which other queries it called. At the end of a session, serialize the graph.

**Phase 3: Add fingerprinting for results**

Implement a `HashStable` trait for your result types. After computing a result, store a fingerprint alongside it. Between sessions, use fingerprint comparison instead of result comparison.

**Phase 4: Stable identifiers**

Define a stable identity for every entity in your language (analogous to `DefPath` in rustc). Ensure fingerprinting uses stable identities, not session-local numeric IDs.

**Phase 5: Adopt Salsa (optional)**

If implementing in Rust, consider adopting Salsa directly rather than implementing the query infrastructure from scratch. Salsa handles memoization, dep tracking, fingerprinting, revision management, parallel execution, and cycle detection. The main cost is fitting your domain model into Salsa's input/tracked/interned taxonomy.

### Salsa Minimum Viable Setup

```rust
// 1. Define the database trait
trait Db: salsa::Database {
    fn source_text(&self, file: SourceFile) -> &str;
}

// 2. Define inputs
#[salsa::input]
struct SourceFile {
    #[returns(ref)]
    path: PathBuf,
    #[returns(ref)]
    text: String,
}

// 3. Define tracked structs for intermediate results
#[salsa::tracked]
struct ParsedFile<'db> {
    #[returns(ref)]
    ast: Ast,
}

// 4. Define tracked functions for computations
#[salsa::tracked]
fn parse(db: &dyn Db, file: SourceFile) -> ParsedFile<'_> {
    let text = file.text(db);
    let ast = do_parse(text);
    ParsedFile::new(db, ast)
}

// 5. Define the concrete database struct
#[salsa::db]
struct Database {
    storage: salsa::Storage<Self>,
}
impl salsa::Database for Database {}
impl Db for Database { ... }

// 6. Use it
let mut db = Database::default();
let file = SourceFile::new(&db, path, initial_text);
let parsed = parse(&db, file);

// Incremental update:
file.set_text(&mut db).to(new_text);
let parsed2 = parse(&db, file);  // only re-parses if text changed
```

### Key Design Decisions

| Decision | Tradeoff |
|---|---|
| Granularity of query keys | Fine-grained (per item) = more incremental, more overhead; coarse-grained (per file) = less overhead, less incremental |
| What to cache on disk | More = faster incremental start, larger disk usage; less = faster serialization, slower cold start |
| `eval_always` usage | Use for queries that always change (global state, file I/O); avoid over-using or you lose dep tracking benefit |
| `no_hash` usage | Use for large monolithic queries that feed projection queries; do not use where result stability matters |
| Stable identifiers | Must be path-based, not ordinal-based; essential for correctness across sessions |
| Cycle handling | Simple error-out for compilers; provisional values only if you need recursive type/trait support |

---

## Quick Reference: rustc Query Declaration Anatomy

```
rustc_queries! {
    /// Doc comment visible in source
    query type_of(key: DefId) -> Ty<'tcx> {
    //    ^^^^^^^  ^^^  ^^^^^     ^^^^^^^^
    //    |        |    |         result type (cheaply cloneable)
    //    |        |    key type (implements QueryKey)
    //    |        query name → method on TyCtxt: tcx.type_of(def_id)
    //    keyword

        cache_on_disk_if { key.is_local() }   // persist to disk for local items
        desc { |tcx| "computing type of `{}`", tcx.def_path_str(key) }  // error messages
    }
}
```

## Quick Reference: Salsa Annotations

| Annotation | Used on | Purpose |
|---|---|---|
| `#[salsa::input]` | struct | Defines a mutable program input; fields stored in DB |
| `#[salsa::tracked]` | fn or struct | Tracked function (memoized, dep-tracked) or intermediate struct |
| `#[salsa::interned]` | struct | Interned value (canonical identity by content) |
| `#[salsa::accumulator]` | newtype struct | Side-channel diagnostic accumulation |
| `#[salsa::db]` | struct | Marks the concrete database struct |
| `#[id]` | struct field | Identity field for matching tracked structs across revisions |
| `#[returns(ref)]` | field or fn | Return a reference into the DB instead of cloning |
| `#[salsa::tracked(specify)]` | fn | Allow overriding the result for specific struct instances |

---

*Sources: [rustc-dev-guide] https://rustc-dev-guide.rust-lang.org/query.html, https://rustc-dev-guide.rust-lang.org/queries/incremental-compilation-in-detail.html, https://rustc-dev-guide.rust-lang.org/queries/query-evaluation-model-in-detail.html; [Salsa] https://salsa-rs.github.io/salsa/overview.html, https://salsa-rs.github.io/salsa/plumbing.html, https://salsa-rs.github.io/salsa/plumbing/cycles.html, https://salsa-rs.github.io/salsa/plumbing/maybe_changed_after.html, https://salsa-rs.github.io/salsa/plumbing/database_and_runtime.html*
