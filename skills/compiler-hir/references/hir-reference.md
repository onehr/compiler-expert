# HIR (High-Level Intermediate Representation) — Deep Reference

> All content sourced from the Rust Compiler Development Guide (rustc-dev-guide).
> Each section is tagged [rustc-dev-guide: §chapter-title] for traceability.

---

## Table of Contents

1. [What HIR Is and Why](#1-what-hir-is-and-why)
2. [HIR Structure](#2-hir-structure)
3. [AST Validation](#3-ast-validation)
4. [Desugaring Catalog](#4-desugaring-catalog)
5. [Span Preservation](#5-span-preservation)
6. [HIR vs AST Design Decisions](#6-hir-vs-ast-design-decisions)
7. [When to Introduce a HIR Layer](#7-when-to-introduce-a-hir-layer)
8. [Practical Implementation Notes](#8-practical-implementation-notes)
9. [Name Resolution and Its Relationship to HIR](#9-name-resolution-and-its-relationship-to-hir)

---

## 1. What HIR Is and Why

[rustc-dev-guide: §41 "The HIR (High-level IR)"]

### Definition

The HIR — "High-Level Intermediate Representation" — is the **primary IR used in most of rustc**. It is a compiler-friendly representation of the abstract syntax tree (AST) that is generated after parsing, macro expansion, and name resolution.

### What Problem HIR Solves

Keeping the raw AST for all downstream analysis passes has several compounding problems:

**Problem 1: The AST is too syntactically rich.**
The AST preserves every source-level detail — redundant parentheses, multiple equivalent expression forms, optional commas, etc. Analysis passes would need to handle every syntactic variant. HIR desugar's these away, giving passes a single canonical form per construct.

**Problem 2: Macro expansion makes the AST non-homogeneous.**
After macro expansion the AST contains nodes that may look like surface syntax but originate from different macro call sites. HIR is the representation after all macros have been fully expanded. No `MacroCall` nodes survive into the HIR.

**Problem 3: Analysis should not depend on syntax choices.**
Whether the user wrote a `for` loop or the equivalent `loop { match iter.next() }` should not produce different behavior in the type checker or borrow checker. HIR normalizes both to the same structure.

**Problem 4: Cross-pass node identity requires stable IDs.**
The type checker, borrow checker, linter, and IDE tooling all need to refer to the same node. HIR assigns every node a `HirId` that is stable across analysis passes.

**Problem 5: The AST does not integrate well with incremental compilation.**
HIR's out-of-band storage model (items stored in maps rather than inline in parents) lets the compiler track access to individual items, enabling fine-grained incrementality.

### The Desugaring Rationale

The HIR guide states directly:

> "Many parts of HIR resemble Rust surface syntax quite closely, with the exception that some of Rust's expression forms have been desugared away. For example, `for` loops are converted into a `loop` and do not appear in the HIR. This makes HIR more amenable to analysis than a normal AST."

[rustc-dev-guide: §41]

Desugaring at the HIR boundary means:
- The type checker only has to understand `loop { match ... }`, not both `for` and `loop { match }`.
- The borrow checker only sees explicit function calls, not method call sugar.
- Diagnostic messages can still be attributed back to the original source span because lowering preserves spans from the AST.

### Pipeline Position

```
Source text
    |
    v
Lexing + Parsing  (produces AST)
    |
    v
Macro Expansion   (AST with macros expanded)
    |
    v
Name Resolution   (AST with all names resolved to DefIds)
    |
    v
AST Validation    (structural checks before lowering)
    |
    v
AST Lowering      (produces HIR)  <-- this boundary is the HIR
    |
    v
HIR Analysis Passes (type checking, trait solving, borrow checking, ...)
    |
    v
THIR / MIR lowering
```

---

## 2. HIR Structure

[rustc-dev-guide: §41 "The HIR (High-level IR)" — subsections: "Out-of-band storage and the Crate type", "Identifiers in the HIR", "HIR Operations", "HIR Bodies"]

### The Crate Type and Out-of-Band Storage

The top-level data structure in the HIR is the `Crate`, which stores the contents of the crate currently being compiled. The HIR is only ever constructed for the **current crate**.

The critical design choice: **HIR uses out-of-band storage**. Unlike the AST — where the crate structure basically just contains the root module, with items nested inside items — the HIR `Crate` structure contains a number of maps that organize the content for easier access.

Concretely: if there is a module item `foo` containing a function `bar()`, the HIR representation of module `foo` (the `Mod` struct) would **only have the `ItemId` of `bar()`**. To get the details of the function `bar()`, you look up that `ItemId` in the `items` map.

```
HIR Crate
 ├── items: Map<ItemId, Item>       // functions, structs, traits, impls, ...
 ├── trait_items: Map<TraitItemId, TraitItem>
 ├── impl_items: Map<ImplItemId, ImplItem>
 └── bodies: Map<BodyId, Body>
```

**Why out-of-band storage?**

1. **Iteration**: you can iterate over all items in the crate by iterating over the key-value pairs in these maps — without traversing the entire HIR tree.

2. **Incremental compilation**: when you gain access to an `&rustc_hir::Item` for `mod foo`, you do **not** immediately gain access to the contents of function `bar()`. You only get the ID for `bar()`, and must invoke a function to look up `bar()`'s contents given its ID. This gives the compiler a chance to observe that you accessed `bar()`'s data and record the dependency.

### Identifiers in the HIR

The HIR uses several coexisting identifier types, each with a distinct purpose:

#### DefId

- Identifies a particular definition (top-level item) in a given crate.
- Composed of: `CrateNum` (which crate) + `DefIndex` (which definition within that crate).
- Unlike `HirId`, there is **not** a `DefId` for every expression — only for items and similar top-level constructs.
- More stable across compilations than `HirId`.
- Items from other crates are referenced by `DefId`.

#### LocalDefId

- Essentially a `DefId` known to come from the **current crate**.
- Drops the `CrateNum` part.
- The type system enforces that only local definitions are passed to functions expecting a local definition.
- Used wherever you need to convert between the local item and its HIR node.

#### HirId

- Uniquely identifies **any node** in the HIR of the current crate — including fine-grained entities like individual expressions, not just top-level items.
- Composed of two parts:
  - An **owner** (`LocalDefId` of the owning item)
  - A **local_id** that is unique within that owner
- This combination makes values stable for incremental compilation.
- Unlike `DefId`, a `HirId` stays local to the current crate.
- The type table, diagnostic engine, borrow checker, and IDE tooling all refer to nodes by `HirId`.

#### BodyId

- Identifies a HIR `Body` in the current crate.
- Currently a wrapper around a `HirId`.
- See HIR Bodies section below.

#### Conversion Between IDs

These identifiers can be converted into one another through `TyCtxt`. Key conversions:
- `tcx.local_def_id_to_hir_id(def_id)` — `LocalDefId` → `HirId`
- `tcx.hir_node(hir_id)` — `HirId` → `Option<Node<'hir>>`
- `tcx.hir_expect_expr(hir_id)` — `HirId` → `&hir::Expr` (panics if not an expression)
- `tcx.parent_hir_node(hir_id)` — traverse parent chain

### HIR Operations via TyCtxt

Most work with the HIR goes through `TyCtxt`. The `hir::map` module contains methods mostly prefixed with `hir_` that:
- Convert between ID kinds
- Look up data associated with a HIR node

The `Node` enum returned by `tcx.hir_node(n)` lets you match on the kind of node and get a pointer to the data itself.

### Owner vs. Non-Owner Nodes

The `HirId` model distinguishes between **owner nodes** and **non-owner nodes**:

- **Owner nodes**: correspond to a `LocalDefId` — items, trait items, impl items, and foreign items. Each owner defines a new scope for `local_id` allocation.
- **Non-owner nodes**: everything else within an item body — individual expressions, statements, patterns, types. These get a `local_id` relative to their owning item.

This distinction matters for incremental compilation: you can re-query the compiler for an owner's HIR subtree independently.

### HIR Bodies

A `rustc_hir::Body` represents some kind of executable code:
- The body of a function or closure
- The definition of a constant
- Any other item that has an associated value to be computed

Bodies are associated with an **owner**, which is typically some kind of item (e.g., an `fn()` or `const`), but could also be a closure expression (e.g., `|x, y| x + y`).

Key `TyCtxt` methods for bodies:
- `tcx.hir_maybe_body_owned_by(def_id)` — find the body associated with a given `DefId`
- `tcx.hir_body_owner_def_id(body_id)` — find the owner of a body

### Inspecting the HIR

You can view the HIR representation of code using:
```
cargo rustc -- -Z unpretty=hir-tree   # full internal HIR tree
cargo rustc -- -Z unpretty=hir        # HIR closer to original source
```

---

## 3. AST Validation

[rustc-dev-guide: §40.7 "AST validation"]

### What AST Validation Is

AST validation is a **separate AST pass** that visits each item in the tree and performs simple structural checks. It runs:

1. **After** macro expansion (macros are fully expanded first)
2. **Before** name resolution / crate resolution
3. As a distinct pass that does not perform complex analysis, type checking, or name resolution

The goal is to catch **structural invalidity** in the AST before the more expensive passes run.

### What Gets Checked

Validations are defined in the `AstValidator` type, located in the `rustc_ast_passes` crate. This type implements the `Visitor` trait that defines how to visit AST items.

For each item, the visitor performs specific checks. For example, when visiting a function declaration, `AstValidator` checks that the function has:
- No more than `u16::MAX` parameters
- The c-variadic argument (if any) goes last in the declaration
- Documentation comments are not applied to function parameters
- Various other structural constraints

These are **not semantic checks** — they are purely about structural validity of the AST representation. Semantic correctness (types, name existence, trait impl validity) is checked later.

### Separation of Concerns

The design explicitly separates:
- **Structural validity** (AST validation pass) — can the construct exist at all?
- **Semantic validity** (type checking, name resolution) — does the construct make sense?

This separation ensures the later passes receive a well-formed AST and do not need to handle structurally invalid inputs.

---

## 4. Desugaring Catalog

[rustc-dev-guide: §41.1 "Lowering AST to HIR" (hir/lowering.html)]

The AST lowering step converts AST to HIR. Many structures are removed if they are irrelevant for type analysis or similar syntax-agnostic analyses.

### 4.1 `for` Loop

[rustc-dev-guide: §41.1]

**Surface syntax:**
```rust
for x in iter {
    body
}
```

**HIR equivalent:**
```rust
{
    let mut __iter = IntoIterator::into_iter(iter);
    loop {
        match Iterator::next(&mut __iter) {
            Some(x) => { body }
            None    => break,
        }
    }
}
```

The `for` loop is explicitly called out in the guide as one of the first examples of desugaring: "for loops are converted into a loop and do not appear in the HIR."

### 4.2 `while let` (and `while`)

[rustc-dev-guide: §41.1 — implied by for-loop desugaring approach]

**Surface syntax (`while let`):**
```rust
while let Pat = expr {
    body
}
```

**HIR equivalent:**
```rust
loop {
    match expr {
        Pat => { body }
        _   => break,
    }
}
```

**Surface syntax (`while`):**
```rust
while cond {
    body
}
```

**HIR equivalent:**
```rust
loop {
    if cond { body } else { break }
}
```

`while` is syntactic sugar for `loop { match ... }` — neither `while` nor `while let` appears in the HIR.

### 4.3 `?` (Try) Operator

**Surface syntax:**
```rust
expr?
```

**HIR equivalent:**
```rust
match expr {
    Ok(val)  => val,
    Err(e)   => return Err(From::from(e)),
}
```

The `?` operator (also called the "try" operator) is desugared into an explicit `match` that either extracts the success value or converts-and-returns the error. This desugaring makes explicit:
- The early return path
- The `From::from` conversion (which implements error type coercion)

### 4.4 Method Calls

**Surface syntax:**
```rust
obj.method(arg1, arg2)
```

**HIR equivalent (after receiver adjustment):**
```rust
Trait::method(&obj, arg1, arg2)
// or &mut obj, or obj by value — depending on the receiver type
```

The guide lists "Removed without replacement, the tree structure makes order explicit for loops" and implies method calls are explicit function calls in HIR. The receiver adjustment (auto-ref, auto-deref) is recorded, and the call becomes a plain function call with an explicit receiver argument.

Note: full method resolution (which impl to use) happens later during type checking, not during lowering. The HIR records that a method call occurred, with the receiver explicitly passed.

### 4.5 Universal `impl Trait`

[rustc-dev-guide: §41.1 — explicitly listed]

**Surface syntax (argument position):**
```rust
fn foo(x: impl Trait) -> i32 { ... }
```

**HIR equivalent:**
```rust
fn foo<T: Trait>(x: T) -> i32 { ... }
// (with flags indicating the user didn't write the type parameter explicitly)
```

Universal `impl Trait` (in argument position) is converted to generic arguments. The HIR records flags indicating that the user didn't write the explicit type parameter — this preserves the information needed for diagnostics and IDE tooling.

### 4.6 Existential `impl Trait` (Return Position)

[rustc-dev-guide: §41.1 — explicitly listed]

**Surface syntax (return position):**
```rust
fn foo() -> impl Trait { ... }
```

**HIR equivalent:**
```rust
// Converted to a virtual existential type declaration
// The return type becomes an opaque type alias
type __OpaqueType: Trait;
fn foo() -> __OpaqueType { ... }
```

Existential `impl Trait` (return position) is converted to a virtual existential type declaration — an opaque type. The guide notes this as a distinct desugaring from universal `impl Trait`.

### 4.7 `async fn` and `async` Blocks

**Surface syntax:**
```rust
async fn foo(x: i32) -> String { ... }
```

**HIR equivalent:**
```rust
fn foo(x: i32) -> impl Future<Output = String> {
    async move {
        // body, with captures of parameters
    }
}
```

`async fn` is desugared to a regular function returning `impl Future`. The function body becomes an `async` block that captures the parameters. This makes the future type explicit in the HIR.

### 4.8 `await` Expression

**Surface syntax:**
```rust
future.await
```

**HIR equivalent (conceptual poll loop):**
```rust
loop {
    match Future::poll(Pin::new(&mut future), cx) {
        Poll::Ready(val) => break val,
        Poll::Pending    => yield,   // suspends the coroutine
    }
}
```

`await` is desugared into a poll loop. In the HIR, `async`/`await` is represented in terms of coroutines (generators), where `yield` suspends execution. The HIR does not retain `await` as a distinct node.

### 4.9 Struct Update Syntax `..base`

[rustc-dev-guide: §41 — Desugaring Catalog, implied]

**Surface syntax:**
```rust
Foo { field1: val1, ..base }
```

**HIR equivalent:**
```rust
Foo {
    field1: val1,
    field2: base.field2,
    field3: base.field3,
    // ... all remaining fields copied from base explicitly
}
```

Struct update syntax is expanded to explicit field-by-field copies of all fields not otherwise specified. The HIR struct literal contains no `..base` shorthand.

### 4.10 Parentheses

[rustc-dev-guide: §41.1 — explicitly listed as first example]

**Surface syntax:**
```rust
(expr)
```

**HIR equivalent:**
```rust
expr
```

Parentheses are **removed without replacement**. The guide states: "the tree structure makes order explicit." Precedence is already encoded in the AST structure, so parentheses are purely syntactic and carry no semantic information.

### 4.11 `if let` / `let else`

**Surface syntax (`if let`):**
```rust
if let Pat = expr { then } else { else_ }
```

**HIR equivalent:**
```rust
match expr {
    Pat => { then },
    _   => { else_ },
}
```

`if let` is desugared to `match`. `let else` is similarly desugared.

---

## 5. Span Preservation

[rustc-dev-guide: §41.1 "Lowering AST to HIR"; §41 general]

### Why Spans Matter for Diagnostics

Every HIR node must carry a `Span` — a reference to the range of source text the node came from. Spans are the foundation of Rust's precise error messages:

- `"error[E0308]: mismatched types"` points to exactly the subexpression with the wrong type
- `"note: ...declared here"` points to the declaration site
- Fix-it suggestions ("consider adding `&` here") require a precise span to insert into

Losing or corrupting spans produces unhelpful errors like "dummy span" diagnostics that don't point anywhere useful.

### The Challenge: Synthetic Nodes

Desugaring creates HIR nodes that have **no direct corresponding source text**. For example:
- The `loop { match Iterator::next(...) }` generated from a `for` loop
- The `__iter` binding introduced during `for` desugaring
- The opaque type created from `impl Trait`

These nodes must still carry valid, non-dummy spans.

### How Lowering Assigns Synthetic Spans

The lowering pass assigns spans to synthetic nodes using the following strategies:

1. **Assign the parent's span**: synthetic nodes generated by desugaring a construct get the span of the original construct. The `loop` generated from `for x in iter { body }` gets the span covering the entire `for` expression.

2. **Assign the keyword's span**: specific synthetic sub-nodes get the span of the relevant keyword. The `break` generated in a `while` desugaring gets the span of the `while` keyword.

3. **Use `DUMMY_SP` sparingly, never in practice**: the lowering invariant states that every node must have a valid, non-dummy span. Dummy spans are a bug.

### The SyntaxContext Mechanism

[rustc-dev-guide: §41; §40.2 "Macro expansion"]

`rustc_span::Span` carries not just byte-range information but also a `SyntaxContext` that encodes the **expansion history** of the span:

- **`SyntaxContext::root()`** (context 0): the span comes from user-written source code, with no macro involvement.
- **Non-root contexts**: the span was introduced by macro expansion or desugaring.

During HIR lowering:
- **Syntactically desugared nodes** (for-loops, `?`, etc.) are set to `SyntaxContext::root()` because the desugaring is part of the language definition, not a user-written macro.
- **Macro-expanded nodes** preserve the original `SyntaxContext` from macro expansion.

This distinction lets the diagnostic engine and IDE tooling distinguish between "this node came from user source" and "this node came from a macro." The `Span::is_desugaring` predicate lets code check whether a span originates from a specific desugaring kind.

### Span Hygiene and Macros

Name resolution uses spans to implement **hygienic macros** — macros that cannot accidentally capture bindings from the call site. The `SyntaxContext` records which macro introduced each identifier, ensuring that identifiers from different syntactic contexts don't accidentally resolve to each other.

The HIR lowering pass must preserve this hygiene information. This means that spans copied from AST nodes into HIR nodes must carry their original `SyntaxContext`, not be stripped to `SyntaxContext::root()`.

---

## 6. HIR vs AST Design Decisions

[rustc-dev-guide: §41, §41.1, §40.7]

### What Stays the Same

- **Item structure**: functions, structs, enums, traits, impls all have close analogs in both AST and HIR.
- **Type syntax**: most type expressions (references, generics, paths) look similar.
- **Patterns**: most patterns (tuple, struct, enum variant, wildcard) are preserved.
- **Literal values**: integer literals, string literals, etc., are preserved.
- **Spans**: every node in both AST and HIR carries a span.

### What Changes

| Aspect | AST | HIR |
|--------|-----|-----|
| `for` loop | `ExprKind::ForLoop` node | desugared to `loop { match iter.next() ... }` |
| `while` / `while let` | `ExprKind::While` | desugared to `loop { match ... }` or `loop { if ... }` |
| `?` operator | `ExprKind::Try` | desugared to `match ... { Ok(v) => v, Err(e) => return ... }` |
| Method calls | `ExprKind::MethodCall` | explicit function call with receiver |
| `impl Trait` (arg) | inline in the function signature | converted to type parameter with flags |
| `impl Trait` (return) | inline in return type | virtual opaque type declaration |
| `async fn` | `FnHeader { asyncness: Async }` | fn returning `impl Future` + async block body |
| `await` | `ExprKind::Await` | desugared to coroutine poll loop |
| Struct update `..base` | `StructRest::Base(expr)` | expanded to explicit field copies |
| Parentheses | `ExprKind::Paren` | removed entirely |
| Macros | `ExprKind::MacCall` | fully expanded; no macro nodes in HIR |
| Visibility | keyword syntax | resolved to `Visibility` with `DefId` |
| Item nesting | items inline in parent | stored in maps, parents hold IDs only |
| Node identity | `NodeId` | `HirId = (owner: LocalDefId, local_id: ItemLocalId)` |

### Why These Specific Choices

**For-loop desugaring**: The borrow checker and type checker need to reason about the lifetime of `__iter`, the calls to `Iterator::next`, and the match arm bindings. Keeping `for` as a primitive would require special-casing all of these in multiple passes.

**Method call lowering**: Type checking needs to see an explicit `self` argument with its adjusted type (auto-ref, auto-deref). Converting to an explicit function call makes this explicit.

**impl Trait desugaring**: Both universal and existential `impl Trait` are fundamentally about type parameters and opaque types respectively. Desugaring makes these explicit before type checking, which then handles them uniformly.

**Out-of-band item storage**: Enables incremental compilation by making item access observable. Without this, accessing `mod foo` would implicitly access all its contents.

---

## 7. When to Introduce a HIR Layer

[rustc-dev-guide: §41 — design rationale extracted]

### Concrete Conditions That Warrant a HIR

A HIR layer is warranted when **all of the following conditions hold** or when **most hold strongly**:

1. **The AST is too syntactically rich for analysis.**
   The AST preserves multiple surface forms for the same semantic construct (`for` loop vs. equivalent `loop`+`match`). Analysis passes would need to handle each form separately without gaining anything semantic.

2. **You need a stable node identity scheme.**
   Multiple passes need to refer to the same nodes by a stable ID — for a type table, a lint table, a borrow checker variable table, IDE hover info, etc. A HIR assigns `HirId` to every node, enabling this.

3. **Macros must be fully expanded before analysis.**
   If your language has macros, analysis should operate on fully expanded code. The AST/HIR boundary is the natural point where macro expansion completes.

4. **Incremental compilation matters.**
   HIR's out-of-band storage makes item access observable to the compiler's dependency tracking. If you need incremental recompilation, this model lets you track which items were accessed and re-check only affected items.

5. **Diagnostics need span-attributed IR nodes.**
   HIR preserves spans for all nodes, including synthetic desugared nodes. This enables precise error messages that point to the original source location even when the IR is far from source form.

6. **You want to separate syntactic concerns from semantic concerns.**
   The AST is about what the user wrote; the HIR is about what the compiler analyzes. This separation makes both layers easier to reason about independently.

### When NOT to Introduce a HIR

- For a simple language where analysis is straightforward on the AST.
- When the added maintenance burden of two representations outweighs the benefit.
- When your language has little or no syntactic sugar to desugar.
- When you only have one analysis pass and don't need cross-pass node identity.

### Decision Guide

```
Does your language have significant syntactic sugar?
  No  → Probably don't need a HIR layer. Use a well-designed AST.
  Yes →
    Do you have multiple analysis passes that all need to handle the sugar?
      No  → Consider desugaring within a single pass without a new IR.
      Yes →
        Do you need cross-pass node identity?
          No  → Consider a simple desugaring pass that produces an annotated AST.
          Yes → A separate HIR with stable node IDs is warranted.
```

---

## 8. Practical Implementation Notes

[rustc-dev-guide: §41.1 "Lowering AST to HIR" (hir/lowering.html); §41]

### Arena Allocation

The HIR is allocated in a long-lived arena. HIR nodes are allocated once and never freed during a compilation. This means:
- `&'hir hir::Expr` is a valid reference for the lifetime of the compilation
- No reference counting or heap allocation per node
- The `'hir` lifetime parameter on many HIR types refers to this arena lifetime

### Interning

Repeated values (type paths, generic argument lists, etc.) are interned to save memory and enable pointer-equality comparisons. The `TyCtxt` holds the interner.

### The Owner Model and HirId Invariants

[rustc-dev-guide: §41.1 — "Lowering needs to uphold several invariants"]

The lowering pass must uphold these invariants to avoid triggering sanity checks in `compiler/rustc_passes/src/hir_id_validator.rs`:

1. **A `HirId` must be used if created.** If you call `lower_node_id`, you must use the resulting `NodeId` or `HirId`. Any `NodeId` in HIR is checked for a corresponding `HirId`.

2. **Lowering a `HirId` must be done in the scope of the owning item.** If you are creating parts of an item other than the one currently being lowered, you must use `with_hir_id_owner`. This happens during existential `impl Trait` lowering.

3. **A `NodeId` placed into a HIR structure must be lowered**, even if its `HirId` goes unused. Calling `let _ = self.lower_node_id(node_id);` is legitimate.

4. **New nodes not in the AST require new IDs.** Call `next_id` to produce a new `NodeId` and automatically lower it to get the `HirId`.

5. **New `DefId`s should have `NodeId`s in the AST.** Since each `DefId` needs a corresponding `NodeId`, it is advisable to add these `NodeId`s to the AST before lowering, rather than generating them on the fly. Benefits:
   - Provides a path from `NodeId` to `DefId`
   - If a `DefId` is needed in multiple places during lowering, the same `NodeId` (and thus same `DefId`) can be reused — generating a new `NodeId` each time would create a different `DefId` each time
   - The `DefCollector` can generate `DefId`s in one centralized place rather than during lowering

### The `LoweringContext` and `with_hir_id_owner`

The lowering pass maintains a `LoweringContext` that tracks:
- The current `HirId` owner (the item currently being lowered)
- The `local_id` counter for the current owner
- The `NodeId`-to-`HirId` mapping being built

When lowering needs to create HIR for a different item (e.g., creating a fresh opaque type for `impl Trait`), it calls `with_hir_id_owner(new_owner, |cx| { ... })` to temporarily switch the owner context.

### Querying the HIR After Lowering

After lowering, the HIR is stored in the `TyCtxt` and accessed through the query system. The main entry points:

- `tcx.hir()` — returns the `Map` which provides HIR navigation
- `tcx.hir_node(hir_id)` — look up any node by `HirId`
- `tcx.hir_expect_expr(hir_id)` — look up an expression, panics if not found
- `tcx.local_def_id_to_hir_id(def_id)` — convert a `LocalDefId` to its `HirId`
- `tcx.hir_maybe_body_owned_by(def_id)` — get the body for an item

### HIR Debugging Tools

```bash
# Print the full HIR tree (internal representation)
cargo rustc -- -Z unpretty=hir-tree

# Print HIR in source-like form
cargo rustc -- -Z unpretty=hir
```

---

## 9. Name Resolution and Its Relationship to HIR

[rustc-dev-guide: §40.3 "Name resolution"]

### Overview

Name resolution is a **prerequisite for HIR lowering**. The HIR lowerer uses resolved `DefId`s for paths — all paths must be resolved before lowering begins.

Name resolution in Rust is a **two-phase process**:

**Phase 1 (during macro expansion):** builds a tree of modules and resolves imports. Macro expansion and name resolution communicate via the `ResolverAstLoweringExt` trait.

**Phase 2 (after the full AST is available):** performs full name resolution — resolves all names in the crate. This happens in `rustc_resolve::late`. Since no new names can be added after macro expansion, each name is resolved exactly once.

### Namespaces

Name resolution uses separate namespaces:
- **Types**: structs, enums, traits, type aliases
- **Values**: variables, functions, constants
- **Macros**: macro definitions

The same identifier can coexist in different namespaces:
```rust
type x = u32;    // type namespace
let x: x = 1;   // value namespace, but type annotation uses type namespace
```

### Ribs (Scopes)

The compiler uses the concept of **Ribs** to represent scopes. Every time the set of visible names potentially changes, a new Rib is pushed onto a stack.

Rib boundaries include:
- Curly braces enclosing a block
- Function boundaries
- Module boundaries
- `let` binding introductions (shadowing)
- Macro expansion borders (for hygiene)

When resolving a name, the rib stack is traversed from innermost outward to find the closest binding (the one not shadowed by anything else).

**Critical rib rule for nested functions**: inner functions (not closures) cannot access the parameters and local bindings of outer functions, even though scoping rules would otherwise allow it. Closures can access outer locals; functions cannot. This is implemented by marking function-boundary ribs as opaque to local name lookups.

### What the Resolver Produces

A successful run of phase 2 (`Resolver::resolve_crate`) creates an **index** that maps names in the source to the places where those names were introduced. This index is then used by HIR lowering via the `hir::lowering::Resolver` interface.

The resolver also produces helpful diagnostic information:
- Typo suggestions
- Suggestions for traits to import
- Lint warnings about unused items

### Speculative Crate Loading

To give useful "did you mean to import..." suggestions, rustc looks through every module of every crate — **including crates not yet loaded** — for possible matches. This speculative loading is handled in `rustc_resolve::diagnostics` by `lookup_import_candidates`.

The key distinction: when loading is speculative, the `record_used` parameter is set to `false`, so errors encountered during speculative loading are not reported to the user.

### How Name Resolution Feeds HIR Lowering

After name resolution, the HIR lowering pass receives:
- A mapping from every name reference in the AST to the `DefId` (or `LocalDefId`) of the definition it refers to
- Resolution results for all imports
- Information about which identifiers are from which namespace

This means HIR lowering does **not** need to resolve names — it only needs to look up previously resolved results and embed the `DefId`s into the HIR nodes.

---

## Quick Reference: Pipeline Summary

```
1. Parse                    → AST (raw syntax tree)
2. Macro expansion          → AST (all macros expanded)
   + Phase 1 name resolution (imports, macro names)
3. AST Validation           → same AST, validated structurally
   (rustc_ast_passes: AstValidator)
4. Phase 2 Name Resolution  → AST with resolved names (DefIds)
   (rustc_resolve: Resolver::resolve_crate)
5. Feature gate checking    → validates feature usage
6. AST Lowering (HIR)       → HIR (desugared, out-of-band storage)
   (rustc_ast_lowering: LoweringContext)
   - for/while/while-let → loop+match
   - ? → match+return
   - impl Trait → type param / opaque type
   - async fn → fn returning impl Future
   - await → poll loop
   - method calls → explicit receiver
   - struct update → field copies
   - parens → removed
7. HIR Analysis             → type checking, trait solving, borrow checking
   (rustc_hir_analysis, rustc_trait_selection, rustc_borrowck)
```

---

*Sources:*
- https://rustc-dev-guide.rust-lang.org/hir.html — §41 "The HIR (High-level IR)"
- https://rustc-dev-guide.rust-lang.org/hir/lowering.html — §41.1 "Lowering AST to HIR"
- https://rustc-dev-guide.rust-lang.org/ast-validation.html — §40.7 "AST validation"
- https://rustc-dev-guide.rust-lang.org/name-resolution.html — §40.3 "Name resolution"
