---
name: compiler-name-resolution
description: Name resolution and scope analysis expert — covers symbol table design, scope chains, module systems, path resolution, use/import semantics, and the resolver pass. Use when implementing name resolution, scope analysis, or symbol lookup for any language.
---

# Compiler Name Resolution & Scope Analysis

## Reference Material

Full reference: `references/analysis-reference.md` (EaC + CI distillation — PART I, Sections 1, 4, 5, 6)

Key sections in the reference:
- **§1 Symbol Table Design** — scoping strategies, sheaf-of-tables, data structure choices, repository design
- **§4 Two-Pass vs. Single-Pass Analysis** — when and why to use a separate resolver pass (CI pattern)
- **§5 Name Resolution — Forward References and Mutual Recursion** — forward refs, mutual recursion strategies, function eagerness vs. variable initializer safety
- **§6 Variable Resolution — Static Scope Depth Encoding** — the CI resolver technique, side table vs. AST annotation, global variable special case

---

## Core Concepts

### Scoping Strategies

**Lexical scoping** (dominant): names visible in the declaring scope and all nested scopes. The compiler resolves all bindings at compile time; names map to static coordinates `<lexical_level, offset>`.

CI's precise rule: "A variable usage refers to the preceding declaration with the same name in the innermost scope that encloses the expression where the variable is used."

**Dynamic scoping** (deprecated): a free variable binds to the most recently created instance at runtime. Avoid unless the source language requires it.

**Inheritance hierarchies** (OOL): a second scope structure layered on lexical scoping. For `obj.member`, resolve the object lexically, then search the inheritance chain upward.

**Closed vs. open class structures**: closed (C++) — all definitions present at compile time, prefer static dispatch; open (Java) — hierarchy can change at runtime, must support dynamic method lookup.

---

### The Sheaf-of-Tables Model

Build a distinct hash table for each scope. Link tables into search paths that reflect the language-defined lookup order. For a method `m` in class `c`, the search path combines lexical nesting tables with the inheritance chain.

**Runtime realization** (tree-walk interpreters): each environment object holds a `name → value` hashmap and a pointer to its enclosing environment — a linked list of hashmaps walked at runtime.

---

### Symbol Table Data Structures

| Structure | Lookup | Insert | When to Use |
|-----------|--------|--------|-------------|
| Linear list | O(n) | O(1) | Small scopes, prototypes |
| Balanced BST | O(log n) | O(log n) | Medium scopes, worst-case matters |
| Hash map | O(1) expected | O(1) expected | General case — the standard choice |
| Multiset discrimination | O(1) guaranteed | offline | When worst-case hash behavior is unacceptable |

**Repository design**: storage should be contiguous or block-contiguous for locality; keep the index scheme independent of the mapping scheme.

**IR reference discipline**: store handles (`<table, offset>` pairs) rather than lexemes (text) in the IR. Handles enable direct access and inexpensive equality tests.

---

### The Mutable Environment Bug

In tree-walk interpreters using mutable hashmaps, a closure captures a live reference to the mutable map. Variables declared later in the same block become visible to the closure — violating lexical scoping.

**Root cause**: the closure must capture a frozen snapshot, not a live reference.

**Resolutions**:
1. **Persistent environments**: each `var` produces a new immutable environment extending the previous one (Scheme approach)
2. **Static resolver pass**: pre-compute scope depths; at runtime walk exactly that many links (CI approach)

---

### The Separate Resolver Pass (CI Pattern)

A dedicated O(n) walk over the AST between parsing and execution that:
- Resolves all variable references to their declaring scope (computes "environment hop distance")
- Detects static errors: use-before-init, duplicate local declarations, `return` outside function
- Stores resolution data in a side table `Map<Expr, Integer>` keyed by AST node identity

**Why a separate pass?**
- During parsing: feasible but complicates the parser (clox approach)
- During execution: correct but slow (re-resolves on each loop iteration) and exposes the mutable-environment bug
- Separate pass: cleanest separation of concerns; runs once; natural insertion point for additional static analyses

**Static analysis properties** (key behavioral differences from execution):
- No side effects: visiting a print statement does not print
- No control flow: loops visited once; both branches of `if` always visited (conservative)
- No short-circuit: logical operators not short-circuited

---

### Static Scope Depth Encoding

At resolve time, for each variable use, compute the **scope depth** (environment hop distance to the declaring scope). At runtime:

```
// Instead of dynamic walk:
environment.get(name)        // walks until found

// Use static depth:
environment.getAt(depth, name)  // walks exactly `depth` hops
```

**Global variables special case**: intentionally excluded from the resolver's scope stack. Globals are looked up dynamically by name at runtime. This supports REPL-style incremental definition. Result: two-tier lookup — static depth-encoded for locals, dynamic name-based for globals.

---

### Forward References and Mutual Recursion

**Language-level solutions**:
- Pascal: forbid forward refs; permit `forward` declaration for mutual recursion
- C: require function prototypes for forward-called functions
- Fortran: implicit typing from first letter of name

**Compiler-level solutions**:
- Build abstract IR with symbolic references; resolve in a subsequent pass
- Separate compilation: require type signatures for externally-defined names

**Two-pass approach**: first pass scans all declarations and builds the symbol table; second pass translates procedure bodies with full visibility.

**Function vs. variable eagerness** (CI): function names are declared and defined before the body is resolved (enabling self-recursion). Variable names are declared, initializer resolved, then defined (enabling use-before-init detection).

---

### Resolver Error Detection

The resolver pass is the natural home for static errors beyond syntactic correctness:
- **Use in own initializer**: variable declared but not yet defined → compile error
- **Duplicate local declaration**: name already in current scope map → compile error (globals exempt for REPL)
- **Return outside function**: track `currentFunction` context through resolution walk

---

## Module Systems and Path Resolution

*(To be expanded with language-specific patterns)*

### Common Patterns

- **Absolute paths**: `use std::collections::HashMap` — resolved from a root module registry
- **Relative paths**: `use super::foo` — resolved relative to the current module's position in the tree
- **Glob imports**: `use foo::*` — pulls all public names into scope; creates shadowing questions
- **Re-exports**: `pub use foo::Bar` — names become part of the exporting module's public interface

### Resolution Order

Languages typically define a resolution priority order, e.g.:
1. Local scope (innermost first, walking outward)
2. Explicit imports
3. Glob imports (in declaration order)
4. Prelude / implicit imports

Ambiguity at the same priority level is a compile error.

### Module vs. Lexical Scope Interaction

Module paths are resolved at compile time against the module tree, which is separate from the lexical scope chain. The two systems interact at `use`/`import` statements, which inject module-path-resolved names into the current lexical scope.

---

## Implementation Checklist

- [ ] Define scope representation (linked tables, flat table + scope IDs, etc.)
- [ ] Choose symbol table data structure (hash map for general use)
- [ ] Define symbol record schema (name, kind, type ref, source location, etc.)
- [ ] Implement scope push/pop during AST traversal
- [ ] Implement lookup (innermost-first search)
- [ ] Implement declare/define two-step for variable declarations
- [ ] Handle function declarations eagerly (name visible in own body)
- [ ] Implement the resolver pass (separate from parsing and execution)
- [ ] Store resolution data (side table or AST annotation)
- [ ] Handle forward references (two-pass or abstract IR with symbolic refs)
- [ ] Handle mutual recursion
- [ ] Implement module path resolution (if applicable)
- [ ] Detect resolver errors (use-before-init, duplicate declaration, return-outside-function)

---

## Planned Additional Sources

- Rust reference: `rustc_resolve` crate — Rust's name resolver (handles use declarations, macro resolution, module system)
- Swift: name resolution across modules and type extensions
- Python: LEGB rule (Local, Enclosing, Global, Built-in) — a well-specified four-level scope chain
- Haskell: qualified imports, hiding, and the interaction with type class resolution

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-architect` — routes here as the third front-end phase
- `compiler-parser` — name resolution runs after the AST is built
- `compiler-expert` — routes here on scope/symbol/module questions

**Trace Down ↓** — next skills after this phase
- `compiler-type-system` — type checking requires all names to be resolved first
- `compiler-hir` — HIR lowering uses resolved names to build typed references

**Related →** — peers at the front-end layer
- `compiler-macro` — macro expansion must interleave carefully with name resolution (hygiene)
- `compiler-diagnostics` — "undefined variable" errors are produced here
