---
name: compiler-hir
description: High-level IR (HIR) design expert — covers AST desugaring, HIR design principles, span preservation during lowering, the boundary between AST and HIR, and when a separate HIR layer is warranted. Use when designing the AST-to-IR lowering pass or deciding whether to introduce a HIR layer.
---

# Compiler HIR (High-Level IR) Design

## Overview

A HIR (High-Level Intermediate Representation) sits between the AST and lower-level IRs (MIR, LLVM IR). Its purpose is to **simplify** the AST by desugaring surface syntax into a smaller, more uniform set of core constructs — without losing source-level information needed for type checking and IDE tooling.

The canonical reference implementation is `rustc`'s HIR, produced by `rustc_ast_lowering`. The HIR is what the type checker (`rustc_typeck`/`rustc_hir_analysis`) and trait solver operate on.

---

## When to Introduce a HIR Layer

A HIR layer is warranted when:

1. **The AST is too rich for analysis**: the AST preserves every surface detail (sugar, multiple equivalent forms, optional syntax). Analysis passes need a single canonical form.

2. **You need span-annotated nodes separate from type-annotated nodes**: the AST is the source of truth for positions; the HIR is the source of truth for semantic structure. Mixing them in one data structure creates tension.

3. **Type checking should not see syntax choices**: whether the user wrote `1 + 1` or `{1 + 1}` should not affect type checking.

4. **You need stable node IDs**: the HIR assigns `HirId` to every node, enabling cross-pass references (the type table, the diagnostic engine, and the borrow checker all refer to nodes by `HirId`).

**When NOT to introduce a HIR**: for simple languages, a well-designed AST may be sufficient. The added complexity of a separate HIR is only justified when the analysis passes are significantly simpler to write against the desugared form.

---

## Desugaring Catalog

*(Placeholder — to be expanded with language-specific sugar)*

### For-Loop Desugaring

```
// Surface:
for x in iter { body }

// HIR equivalent:
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

### `?` Operator Desugaring

```
// Surface:
expr?

// HIR equivalent:
match expr {
    Ok(val)  => val,
    Err(e)   => return Err(From::from(e)),
}
```

### Method Call Desugaring

```
// Surface:
obj.method(arg1, arg2)

// HIR equivalent (after receiver adjustment):
Trait::method(&mut obj, arg1, arg2)   // or by value / by ref depending on receiver type
```

### `while` / `while let` Desugaring

```
// Surface:
while cond { body }

// HIR equivalent:
loop { if cond { body } else { break } }
```

### `if let` / `let else` Desugaring

```
// Surface:
if let Pat = expr { then } else { else_ }

// HIR equivalent:
match expr { Pat => { then }, _ => { else_ } }
```

### Struct Update Syntax

```
// Surface:
Foo { field1: val1, ..base }

// HIR equivalent:
Foo { field1: val1, field2: base.field2, ... }  // all fields filled in explicitly
```

---

## Span Preservation Strategy

**The fundamental tension**: desugaring introduces synthetic nodes that have no direct source span. These nodes must still carry a span for diagnostics.

**Strategy**:
1. **Assign the parent's span** to synthetic nodes created during desugaring. For example, a loop generated from a `for` desugaring gets the span of the `for` keyword.
2. **Mark spans as synthetic/expansion spans** so the diagnostic engine can distinguish them from user-written spans.
3. **Preserve the original AST node's span** as the "outermost" span of the desugared construct.

**rustc's approach**: `rustc_span::Span` carries a `SyntaxContext` that encodes the expansion history. Macro-generated spans have a different context than user-written spans. The HIR lowering sets `SyntaxContext::root()` for syntactically desugared nodes and preserves the original context for macro-expanded nodes.

**Key invariant**: every HIR node must have a valid, non-dummy span. Dummy spans cause poor diagnostic output.

---

## HIR vs. AST Node Design

**AST node** priorities:
- Faithful representation of the source text
- Round-tripping (AST → source → AST should be lossless)
- Parser-friendly: nodes correspond to grammar productions
- Contains optional syntax elements (trailing commas, redundant parens, etc.)

**HIR node** priorities:
- Uniform: all sugar is desugared; there is one canonical form per construct
- Cheaply traversable: analysis passes iterate over nodes; the structure should facilitate that
- Stable node identity: `HirId` allows cross-pass references
- Separated concerns: spans preserved, but type annotations are a separate table keyed by `HirId`

**Concrete differences**:

| Aspect | AST | HIR |
|--------|-----|-----|
| `for` loop | `ForExpr` node | desugared to `loop { match ... }` |
| `?` operator | `TryExpr` node | desugared to `match ... return` |
| Method calls | `MethodCallExpr` | desugared to function call with explicit receiver |
| Type annotations | part of the node | separate type table (keyed by `HirId`) |
| Macros | `MacroCall` nodes | fully expanded (macros don't survive to HIR) |
| Visibility | `PubCrate`, `PubSuper`, etc. | resolved to a `Visibility` with a `DefId` |

---

## HIR Node Identity: HirId

`HirId` = `(DefId, ItemLocalId)` — identifies a node both globally (which item it belongs to) and locally (which node within that item).

**Uses**:
- Type table: `HashMap<HirId, Ty>` stores the inferred type for each expression
- Diagnostic engine: points users to specific AST nodes by ID
- Borrow checker: queries the HIR for node structure by ID
- IDE integration: maps source positions to HIR nodes and back

---

## AST → HIR Lowering Pass Design

**Input**: the fully macro-expanded AST (after `rustc_expand`)
**Output**: the HIR, a `Map<HirId, Node>` with:
- The `Crate` root
- All items, bodies, and expressions with `HirId`s assigned
- All spans preserved from the AST

**Lowering is not type-directed**: the lowerer does not know types. It only performs syntactic desugaring based on the surface structure. Type-dependent transforms (method resolution, operator overloading) happen later.

**Name resolution must precede lowering**: the lowerer uses resolved `DefId`s for paths. All paths must be resolved before lowering begins.

---

## Implementation Checklist

- [ ] Decide whether a HIR layer is warranted (complexity vs. benefit analysis)
- [ ] Define the HIR node types (one per core construct after desugaring)
- [ ] Assign stable node IDs (e.g., `HirId = (DefId, ItemLocalId)`)
- [ ] Implement the AST → HIR lowering pass
- [ ] Implement for-loop desugaring
- [ ] Implement `?` / try operator desugaring
- [ ] Implement method call lowering (receiver adjustment)
- [ ] Implement `while` / `while let` / `if let` desugaring
- [ ] Implement span assignment for synthetic nodes
- [ ] Implement the HIR map (fast lookup by `HirId`)
- [ ] Verify all analysis passes (type checker, borrow checker) use HIR, not AST

---

## Reference Material

### Deep Reference (rustc-dev-guide)

See [`references/hir-reference.md`](references/hir-reference.md) for comprehensive notes sourced directly from the rustc-dev-guide. Covers:

- What HIR is and why it exists (the problems it solves vs. keeping the AST)
- HIR structure: `HirId`, `BodyId`, owner vs. non-owner nodes, out-of-band storage, the `Crate` map
- AST validation: what gets checked before lowering (structural, not semantic)
- Complete desugaring catalog: `for`, `while let`, `?`, method calls, `impl Trait`, `async fn`, `await`, struct update `..base`, parentheses
- Span preservation: `SyntaxContext`, synthetic span assignment, the hygiene mechanism
- HIR vs AST design decisions: what stays the same, what changes, and why
- When to introduce a HIR layer: decision guide with concrete conditions
- Practical implementation: arena allocation, interning, the Owner model, `HirId` invariants, `LoweringContext`
- Name resolution and its relationship to HIR: ribs, namespaces, two-phase resolution, speculative crate loading

Sources:
- https://rustc-dev-guide.rust-lang.org/hir.html
- https://rustc-dev-guide.rust-lang.org/hir/lowering.html
- https://rustc-dev-guide.rust-lang.org/ast-validation.html
- https://rustc-dev-guide.rust-lang.org/name-resolution.html

### Other References (for future research)

- `rustc_ast_lowering` crate source — the primary reference implementation
- GHC Core — Haskell's HIR (System FC): a dependent-core-typed HIR
- Swift SIL — high-level IR between AST and LLVM IR, with ownership annotations

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-architect` — HIR layer is the architect's recommended Option 2/3 choice
- `compiler-parser` — HIR lowering consumes the AST
- `compiler-type-system` — type annotations are attached during or after HIR lowering
- `compiler-expert` — routes here on AST lowering / HIR design questions

**Trace Down ↓** — next skills after this phase
- `compiler-mir` — HIR lowers further to MIR (control-flow graph, borrow checking)
- `compiler-optimizer` — if skipping MIR, HIR feeds directly into optimization

**Related →** — sibling IR design skills
- `compiler-mir` — the next IR layer down; HIR and MIR design are coupled
- `compiler-query-system` — incremental compilation caches HIR as a query result
