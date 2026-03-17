---
name: compiler-type-system
description: Type system implementation expert — covers Hindley-Milner type inference, trait/typeclass solving, subtyping, variance, C implicit conversions, and runtime type checking patterns. Use when implementing a type checker, type inferencer, or type system for any language.
---

# Compiler Type System Implementation

## Reference Material

**Deep reference (primary):** `references/type-system-reference.md` — Full algorithm W, Rémy generalization, Rust trait solving, variance inference, type representation, error messages, decision guide.

Key sections in the type-system reference:
- **§I HM Type Inference** — Algorithm W pseudocode, unification with occurs check, Rémy level-based generalization (avoids env scanning), value restriction
- **§II Trait/Typeclass Constraint Solving** — Obligations, candidate assembly, winnowing, Chalk logic-programming approach, coherence/orphan rules, HRTB
- **§III Subtyping and Variance** — Subtype checking algorithm, variance inference (rustc algorithm), PhantomData, function type variance
- **§IV Type Representation** — Interning (pointer equality), normalization, alias expansion strategies
- **§V Error Messages** — Constraint tracking, cascading error suppression, type variable naming
- **§VI Swift Generics** — Protocol witness tables, existential boxes, `some` vs `any`
- **§VII Decision Guide** — Which type system to implement for your use case
- **§X Pitfalls** — Occurs check, value restriction, covariant mutation, orphan rules

**Secondary reference (EaC + CI):** `../compiler-name-resolution/references/analysis-reference.md`

Key sections in the secondary reference:
- **§3 Type Checking Approaches** — static vs. dynamic checking, inference strategies, type equivalence, implicit conversions
- **§2 Attribute Grammars** — synthesized vs. inherited attributes for type propagation

---

## Core Concepts

### Static vs. Dynamic Type Checking

**Static (compile-time)**:
- The compiler assigns a type to each name and expression from declarations and inference rules
- No runtime tags or case logic for type-correct code
- Enables selection of efficient, specialized operations (integer vs. float multiply)
- Prerequisites: mandatory declarations or strong inference rules; all called procedures visible with type signatures

**Dynamic (runtime)**:
- Each value carries a runtime tag encoding its type
- Every operation must check tags, select the variant, perform the operation, update the tag
- Cost paid on every execution, not once at compile time
- Necessary for dynamically typed languages (Python, Lisp, JavaScript, Lox)

**Key asymmetry**: the compile-time cost of static checking is paid once; the runtime cost of dynamic checking is paid on every execution.

**The static/dynamic spectrum is not binary**:
- Java casts: static system presumes success; JVM inserts runtime check → `ClassCastException`
- Covariant arrays in Java/C#: compiles cleanly but throws `ArrayStoreException` at runtime

A language designer can grant flexibility by deferring specific checks to runtime without sacrificing overall soundness — at the cost of some runtime overhead.

---

### Dynamic Type Checking Implementation (Tree-Walk)

Type checking occurs at the point of each operation:
1. Evaluate both operands
2. Use `instanceof` (or equivalent runtime type test) to determine actual types
3. Select the appropriate operation or throw a `RuntimeError` with source location

```java
if (left instanceof Double && right instanceof Double)   return (double)left + (double)right;
if (left instanceof String && right instanceof String)   return (String)left + (String)right;
throw new RuntimeError(operator, "Operands must be two numbers or two strings.");
```

Runtime errors must: (a) report the source location, (b) unwind the evaluator without crashing the host, (c) allow the REPL to continue.

**Truthiness as implicit boolean coercion**: design choice embedded in language spec. Common patterns:
- Ruby / Lox: `false` and `nil` are falsey; everything else truthy
- JavaScript: empty string, `0`, `null`, `undefined`, `NaN` are falsey
- Python: empty collections, `0`, `None` are falsey

---

### Type Inference Strategies

**Declarations-first (declare-before-use)**:
- Parse declarations first; populate symbol table; process executable statements with full type info
- Enables direct IR emission in a single pass
- Languages: Pascal, C, FORTRAN

**Declarations optional — consistent-use inference**:
- Assign a general type; narrow by examining usage contexts
- Iterative fixed-point algorithm over a type lattice
- Requires multiple passes: abstract IR first, refine types, then lower
- Languages: Python (partial), early FORTRAN (first-letter implicit typing)

**Parametric polymorphism (Hindley-Milner)**:
- Functions like `filter: (α→β) × list(α) → list(β)` have return types that are functions of argument types
- A simple fixed-point over types is insufficient; requires solving equations over the space of types
- Classical approach: **unification** (Robinson, 1965)
- Languages: ML, Haskell, modern Scheme, Rust (partially)

**Dynamically changing types**:
- Variable's type can change during execution (APL)
- Compiler approach: fall back on interpretation with tagged values
- JIT compilers can specialize on observed types at hot call sites

---

### Hindley-Milner Type Inference

Full pseudocode and implementation details: **reference §I** (`references/type-system-reference.md`).

**Algorithm W — four constraint rules**:
1. **Var**: instantiate the type scheme from the environment (replace `∀α` with fresh `TVar`)
2. **Lam**: allocate fresh `TVar` for parameter; type body with it in scope; return `param_ty → body_ty`
3. **App**: type both subexpressions; allocate fresh `TVar` for result; unify `fun_ty` with `arg_ty → result_ty`
4. **Let**: increment level; type RHS; decrement level; generalize RHS type (Rémy: quantify vars whose level > current_level); type body with generalized type in scope

**Unification**: find substitution `σ` such that `σ(t1) = σ(t2)`. Fails if:
- Two different base types (e.g., `Int` vs. `Bool`)
- Occurs check fails: `α` cannot unify with `f(α)` — would create infinite type and cause infinite loops

**Let-generalization**: type variables free in the type but NOT in the environment are universally quantified → polymorphism. Rémy's trick: instead of scanning the environment, attach a *level* (let-nesting depth) to each type variable. Generalize only vars whose level exceeds the current level — O(size of type), not O(size of env). See reference §I.5–§I.6.

**Value restriction**: generalize only when the let-RHS is syntactically a value (lambda, literal, constructor). Prevents unsound polymorphism for mutable references: `let r = ref []` must have monomorphic type. See reference §I.7.

**Principal types**: HM produces the most general type. Any other valid type is an instance.

---

### Trait / Typeclass Solving

Full details: **reference §II** (`references/type-system-reference.md`).

**Core idea**: traits/typeclasses add predicate constraints to the type system. `fn sort<T: Ord>(v: Vec<T>)` requires `T: Ord`. The type checker generates *obligations* (predicates needing proof) and the solver discharges them by finding matching impls.

**Three-phase resolution** (rustc model, reference §II.2):
- **Selection**: for one obligation, find and confirm a candidate impl (or where-clause assumption)
- **Fulfillment**: worklist that tracks pending obligations; enqueues nested obligations from impl where-clauses
- **Evaluation**: checks whether a candidate could work without committing inference variables (used during winnowing)

**Candidate assembly and winnowing** (reference §II.3): collect all impls that structurally match; then winnow using `evaluate_candidate` to eliminate ones whose where-clauses cannot be satisfied; require exactly one remaining candidate.

**Chalk/logic-programming view** (reference §II.5): each impl is a Horn clause. `impl<T: Clone> Clone for Vec<T>` becomes `Clone(Vec(?T)) :- Clone(?T)`. Trait resolution is SLD proof search (Prolog-style). Rust needs extensions beyond pure Horn clauses for HRTB, associated types, and negation.

**Coherence / orphan rules** (reference §II.6): at most one impl per (Trait, Type) pair globally. Enforced by orphan rule: impl must be in crate of `Trait` or `Type`. Without this, two crates could provide conflicting impls with no way to choose.

**Blanket implementations and overlap**: Rust forbids overlapping impls (no specialization by default). Haskell allows overlapping instances with extensions + priority rules.

**Runtime implementation choices**:
- **Dictionary passing** (Haskell default, Rust `dyn Trait`): constraint → implicit dict argument; one compiled function; runtime dispatch overhead
- **Monomorphization** (Rust generics default): separate compiled version per instantiation; zero runtime overhead; binary size grows

---

### Type Equivalence

Language designer's choice — compiler must implement whichever is specified:

- **Name equivalence**: two types are equivalent iff the programmer uses the same name. `Tree` and `Bush` (structurally identical structs) are distinct types.
- **Structural equivalence**: two types are equivalent iff they have the same structure. `Tree` and `Bush` are equivalent.

Both have been used in production languages. Name equivalence is more common in modern languages (prevents accidental type aliasing).

---

### Implicit Conversions (Coercion)

Two strategies:
1. **Insert explicit conversions early**: apply the language's conversion table during translation; expect optimization to reduce redundant conversions. IR is cluttered but complete.
2. **Keep conversions implicit until late**: reduce IR clutter; elaborate during a later pass after types are fully resolved.

**C widening rule**: convert to the narrowest type that can represent both operands. Encode as a lookup table per operator. Full implicit conversion hierarchy: `char → short → int → long → float → double`.

---

### Subtyping and Variance

Full algorithms: **reference §III** (`references/type-system-reference.md`).

**Subtyping relation**: `S <: T` means a value of type `S` can be used wherever `T` is expected.

**Variance** describes how the subtyping relation changes with type constructors:
- **Covariant** `F<S> <: F<T>` when `S <: T` — safe for producers (read-only containers, return positions)
- **Contravariant** `F<T> <: F<S>` when `S <: T` — safe for consumers (function argument positions)
- **Invariant** — neither; `F<S>` and `F<T>` are unrelated (mutable containers, `Cell<T>`)
- **Bivariant** — both (rare; usually unsound — occurs in unsafe code)

**Function type rule**: `(τ₁ → τ₂)` is contravariant in `τ₁` and covariant in `τ₂`. An `Animal → String` function is a subtype of `Cat → String` (accepts more inputs). See reference §III.4.

**Variance inference** (rustc algorithm, reference §III.2): analyze field types using a constraint lattice (`* > +/- > o`). Iterate to fixed point. `V(X) <= +` from covariant field; `V(X) <= -` from function-argument field; `V(X) <= o` from `&mut X` / `Cell<X>`. Composed positions multiply: `- × - = +`.

**PhantomData** (reference §III.3): raw pointer fields (`*mut T`) don't contribute to variance inference. Use `PhantomData<T>` to explicitly declare covariance, `PhantomData<fn(T)>` for contravariance.

Java's covariant arrays: `String[] <: Object[]` — covariant but allows `ArrayStoreException` at runtime. The canonical example of why mutable containers must be invariant.

---

### Attribute Grammar: Type Propagation

**Synthesized attributes** for types (preferred):
- Type of an expression: `F_+(type(left_operand), type(right_operand))` synthesized up to the `Expr` node
- Flows bottom-up; natural in LR parsing

**Inherited attributes** for type context (when needed):
- Propagate declared type to all names in a declaration list
- Work-around in bottom-up parsers: use a global variable `CurType` when `TypeSpec` reduces
- Better alternative: restructure grammar to encode type information structurally

---

## Implementation Checklist

- [ ] Define type representation (algebraic data type for types: base, arrow, tuple, polymorphic, etc.)
- [ ] Choose static vs. dynamic checking strategy (or hybrid)
- [ ] For static: choose inference algorithm (declarations-first, HM unification, etc.)
- [ ] Implement type environment (extend symbol table with type info)
- [ ] Implement type checking rules for each expression/statement form
- [ ] Implement implicit coercions (insert conversion nodes in AST/IR)
- [ ] Define type equivalence rule (name or structural)
- [ ] For HM: implement unification with occurs check
- [ ] For HM: implement let-generalization
- [ ] For traits/typeclasses: implement constraint solving and dictionary passing or monomorphization
- [ ] For subtyping: define the subtyping relation and variance rules
- [ ] Produce useful type error messages with source locations

---

## Additional Sources (already integrated into reference)

- okmij.org — Rémy's generalization algorithm (levels/ranks), detailed OCaml type checker walkthrough
- rustc-dev-guide — Trait resolution (old solver), Chalk logic-programming approach, variance inference algorithm, coherence, HRTB
- Swift Book — Generics, protocol witness tables, `some` vs `any` (existentials)
- Pierce — *Types and Programming Languages* (TAPL): authoritative type theory foundations
- Jones — *Typing Haskell in Haskell*: typeclass constraint solving in detail
- OCaml: modules and functors as a type-level abstraction mechanism (not yet in reference)

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-architect` — routes here as the fourth front-end phase
- `compiler-name-resolution` — type checking follows name resolution
- `compiler-expert` — routes here on type inference/checking/trait questions

**Trace Down ↓** — next skills after this phase
- `compiler-hir` — the typed HIR carries type annotations from this phase
- `compiler-elaboration` — for dependently-typed languages, type checking IS elaboration

**Related →** — peers at the front-end layer
- `compiler-name-resolution` — names must be resolved before types can be checked
- `compiler-comptime` — generics and compile-time type evaluation intersect here
- `compiler-diagnostics` — type error messages require good span tracking
