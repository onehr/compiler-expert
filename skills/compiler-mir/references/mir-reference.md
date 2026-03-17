# Rustc MIR: Comprehensive Reference for Compiler Implementers

> Source attribution: all content tagged [rustc-dev-guide] is derived from the
> Rust Compiler Development Guide (https://rustc-dev-guide.rust-lang.org/).
> Section references are included inline.

---

## 1. Why MIR? The Design Rationale

### What Problem Does MIR Solve That HIR Cannot?

[rustc-dev-guide §44, borrow-check.html]

The HIR (High-level IR) is a tree-structured representation that still carries
the full complexity of Rust's surface syntax: nested expressions, syntactic
sugar, implicit coercions, pattern matching at every level, complex lifetime
relationships, etc. This complexity makes it very difficult to implement
flow-sensitive analyses like the borrow checker correctly.

Key advantages of doing borrow checking on MIR rather than HIR:

1. **Radical desugaring reduces bug surface.** MIR is a drastically simplified
   form. By the time code reaches MIR, complex constructs (closures, async/await,
   match arms, compound operators) have been transformed into a small, uniform
   IR. The MIR-based borrow checker fixed a measurable number of bugs that the
   HIR-based checker had (the guide explicitly links a list of these).

2. **Non-lexical lifetimes (NLL) become possible.** Lexical lifetimes were an
   artifact of HIR's tree structure, where a lifetime was tied to a syntactic
   scope (block, expression). MIR is built on a control-flow graph (CFG), so
   regions can be derived from CFG points. A lifetime can end in the middle of
   a block, as soon as the value is no longer used — this is NLL. HIR cannot
   express this because its structure does not model control flow explicitly.

3. **All types are explicit.** MIR has no implicit coercions. Every type
   annotation is present. This makes type-checking and region inference
   tractable: there is nothing to infer from context other than region
   variables.

### The 15-Statement Language Philosophy

[rustc-dev-guide §44, mir/index.html]

MIR is described as a "radically simplified" form. The key properties that
characterize this philosophy:

- **No nested expressions.** Every computation is broken into individual
  assignment statements. The right-hand side of an assignment is an Rvalue,
  which can only reference places (memory locations) and constants — never
  other operations. Operations that would be nested in HIR are sequenced
  into multiple statements with temporary variables.

  Example: `x = a + b + c` becomes:
  ```
  tmp1 = a + b
  x = tmp1 + c
  ```

- **Control flow is explicit.** There are no `if` expressions in the middle
  of other expressions. All branching happens at basic block terminators.

- **A small, fixed set of statement and terminator kinds.** Rather than dozens
  of AST node kinds, MIR has a handful of statement types and a small set of
  terminator types. New language features do not add new MIR constructs;
  they are instead desugared into the existing vocabulary.

- **Moves and copies are explicit.** The grammar of operands distinguishes
  `copy Place` from `move Place`. There is no implicit copy.

- **Types are fully resolved.** No type inference remains in MIR. All generic
  parameters are present (MIR is not yet monomorphized, but the types are
  fully specified in terms of generic parameters).

### MIR vs SSA: Similarities and Differences

[rustc-dev-guide §44, mir/index.html; appendix/background]

MIR is **not** SSA (Static Single Assignment) form, but it shares some goals.

**Similarities:**
- Both use temporary variables extensively.
- Both make data flow explicit.
- Both facilitate flow-sensitive optimizations.
- Both eliminate nested expressions.

**Differences:**
- MIR variables (locals) can be assigned multiple times. There is no phi-node
  mechanism; locals are mutable. SSA requires each variable to be defined
  exactly once, with phi nodes at join points.
- MIR retains a direct concept of *places* (memory locations, including fields
  and dereferences), which SSA typically does not. SSA is value-oriented;
  MIR is memory-location-oriented.
- MIR explicitly models moves and borrows, which is not part of SSA.
- SSA is primarily an optimization-facilitating form; MIR serves both as an
  analysis target (borrow checker) and an optimization/codegen target.
- Drop semantics, initialization tracking, and region inference are native
  concepts in MIR but absent from SSA.

---

## 2. MIR Structure

[rustc-dev-guide §44, mir/index.html "Key MIR vocabulary" and "MIR data types"]

### Basic Blocks

The fundamental unit of MIR control flow is the **basic block** (`BasicBlock`,
a newtype index into `Body::basic_blocks: IndexVec<BasicBlock, BasicBlockData>`).

A `BasicBlockData` contains:
- `statements: Vec<Statement>` — a sequence of straight-line operations.
- `terminator: Option<Terminator>` — the last operation in the block, which
  transitions to one or more successor blocks.

Basic blocks have no branches internally — all control flow happens in the
terminator.

### Statements

A `Statement` has a `source_info` (span + scope) and a `kind: StatementKind`.

Complete taxonomy of `StatementKind`:

| Variant | Description |
|---------|-------------|
| `Assign(Place, Rvalue)` | The core statement. Assigns an rvalue to a place. |
| `FakeRead(FakeReadCause, Place)` | Forces a read for diagnostics (e.g., match ergonomics); no code generated. |
| `SetDiscriminant(Place, VariantIdx)` | Sets the discriminant of an enum place. |
| `Deinit(Place)` | Marks a place as uninitialized (used in const eval). |
| `StorageLive(Local)` | Begins the storage lifetime of a local (hint to allocate stack space). |
| `StorageDead(Local)` | Ends the storage lifetime of a local (hint to free stack space). |
| `Retag(RetagKind, Place)` | Stacked borrows retag (used by Miri). |
| `PlaceMention(Place)` | Like `FakeRead` but does not require the place to be initialized. |
| `AscribeUserType(Place, UserTypeProjection, Variance)` | Ties a place to a user-written type annotation for diagnostics. |
| `Coverage(CoverageKind)` | Instrumentation for code coverage. |
| `Intrinsic(NonDivergingIntrinsic)` | Non-diverging intrinsic call (e.g., `assume`, `copy_nonoverlapping`). |
| `ConstEvalCounter` | Counts interpreter steps in const eval. |
| `Nop` | No operation; used as a placeholder after optimization. |

### Terminators

A `Terminator` has a `source_info` and `kind: TerminatorKind`.

Key terminator kinds:

| Variant | Successors | Description |
|---------|-----------|-------------|
| `Goto { target }` | 1 | Unconditional jump |
| `SwitchInt { discr, targets }` | N | Branch on integer value (covers `if`, `match`) |
| `Return` | 0 | Return from function |
| `Unreachable` | 0 | Marks unreachable code |
| `Call { func, args, destination, target, unwind }` | 1–2 | Function call; `target` is return block, `unwind` is unwind block |
| `Drop { place, target, unwind, replace }` | 1–2 | Drop a value (may unwind) |
| `Assert { cond, expected, msg, target, unwind }` | 1–2 | Runtime assertion (overflow checks, bounds checks) |
| `Yield { value, resume, resume_arg, drop }` | 2 | Coroutine yield |
| `CoroutineDrop` | 0 | Drop a coroutine when it's suspended |
| `FalseEdge { real_target, imaginary_target }` | 2 | Borrow checker hint; `imaginary_target` only for the checker |
| `FalseUnwind { real_target, unwind }` | 1–2 | Like `FalseEdge` for unwind paths |
| `InlineAsm { ... }` | variable | Inline assembly |

### Places (Lvalues)

A `Place` identifies a memory location. It consists of:
- A `Local` (newtype index into `Body::local_decls`)
- A list of `ProjectionElem` applied left to right

`ProjectionElem` variants:

| Variant | Type param | Description |
|---------|-----------|-------------|
| `Deref` | — | Dereference (`*p`) |
| `Field(FieldIdx, T)` | type of field | Access a struct/tuple/closure field |
| `Index(Local)` | — | Array index (`a[i]`) |
| `ConstantIndex { offset, min_length, from_end }` | — | Compile-time-known index, possibly from end |
| `Subslice { from, to, from_end }` | — | Slice of a slice |
| `Downcast(Option<Symbol>, VariantIdx)` | — | Cast to specific enum variant |
| `OpaqueCast(T)` | type | Cast through opaque type |
| `Subtype(T)` | type | Subtype coercion for borrow checking |

Special locals:
- `_0` is `RETURN_PLACE` — the return value.
- `_1`, `_2`, ... are arguments (in order), then user variables, then temporaries.

### Rvalues

An `Rvalue` is the right-hand side of an assignment. It can **not** be nested.

| Variant | Description |
|---------|-------------|
| `Use(Operand)` | Copy or move an operand |
| `Repeat(Operand, Const)` | Array construction `[x; N]` |
| `Ref(Region, BorrowKind, Place)` | Take a reference `&place` or `&mut place` |
| `ThreadLocalRef(DefId)` | Reference to a thread-local static |
| `RawPtr(Mutability, Place)` | Raw pointer (`&raw const p` / `&raw mut p`) |
| `Len(Place)` | Length of a slice or array |
| `Cast(CastKind, Operand, Ty)` | Type cast |
| `BinaryOp(BinOp, Operand, Operand)` | Arithmetic, logical, bitwise operations |
| `NullaryOp(NullOp, Ty)` | Zero-operand ops: `size_of`, `align_of`, `offset_of` |
| `UnaryOp(UnOp, Operand)` | Negation, bitwise not |
| `Discriminant(Place)` | Read enum discriminant |
| `Aggregate(AggregateKind, IndexVec<FieldIdx, Operand>)` | Construct struct, tuple, array, closure, coroutine |
| `ShallowInitBox(Operand, Ty)` | Internal: initialize a `Box` pointer |
| `CopyForDeref(Place)` | Copy through a deref, for borrow checker purposes |

`CastKind` variants include: `PointerExposeProvenance`, `PointerWithExposedProvenance`,
`PointerCoercion`, `DynStar`, `IntToInt`, `FloatToInt`, `IntToFloat`,
`FloatToFloat`, `PtrToPtr`, `FnPtrToPtr`, `Transmute`.

### Operands

An `Operand` is the leaf value in the expression tree:

```
Operand = Constant(ConstOperand)   // compile-time constant
        | Copy(Place)              // copy a place (requires T: Copy)
        | Move(Place)              // move out of a place
```

---

## 3. Building MIR from HIR

[rustc-dev-guide §44.1, mir/construction.html]

### Overview

MIR construction does **not** operate on HIR directly. It goes through THIR
(Typed High-level IR) as an intermediate step. The query chain is:
```
HIR -> THIR -> MIR  (via mir_built query)
```

The `mir_built` query triggers lowering. The MIR builder processes THIR
expressions recursively.

### Argument and Binding Lowering

For each function:
1. A local (`_0`) is created for the return place.
2. Locals are created for each argument as per the signature.
3. Locals are created for each binding in the argument patterns.
   Example: `(a, b): (i32, String)` creates 3 locals — one for the argument
   tuple and two for the destructured bindings — plus field-access statements
   that copy the fields into the binding locals.

### The `unpack!` Pattern

MIR-building functions are categorized by whether they produce new basic blocks:

```rust
// Appends statements to existing block, returns result:
fn generate_some_mir(&mut self, block: BasicBlock) -> ResultType

// May create new blocks; returns (new_cursor_block, result):
fn generate_more_mir(&mut self, block: BasicBlock) -> BlockAnd<ResultType>
```

The `unpack!` macro extracts the new block and updates a local cursor:
```rust
let v = unpack!(block = self.generate_more_mir(...));
```

### Expression Lowering Hierarchy

There are four representation targets when lowering an expression:

1. **Place** — a preexisting memory location (local, static, promoted)
2. **Rvalue** — can be assigned to a Place
3. **Operand** — argument to an operation; either `Constant` or `copy/move` from a Place
4. **Temporary** — a fresh local holding a computed value

The lowering cascade: function body -> Rvalue -> Operand -> Place -> (possibly creates Temporary via Rvalue).

### Pattern Matching Lowering

[rustc-dev-guide §44.1, mir/construction.html "Pattern matching"]

`match` on enums with fields is lowered to `TerminatorKind::SwitchInt` where
the `Operand` refers to a `Place` containing the discriminant value. This
typically involves:
1. Reading the discriminant into a temporary: `tmp = Discriminant(place)`
2. `SwitchInt { discr: move tmp, targets: [...] }`

`if` conditions and `match` on fieldless enum variants also lower to
`SwitchInt` (with values 0 and 1 for booleans).

### Match Exhaustiveness

Match exhaustiveness checking happens before MIR construction, during HIR
analysis (the `check_match` pass). By the time MIR is built, exhaustiveness
is guaranteed and does not need to be re-verified.

### Closure Lowering

[rustc-dev-guide §44.1, §70]

Closures are lowered to anonymous struct types. Captures are fields of this
struct. The debug annotations in MIR for captured variables are expressed as
place projections:
```
debug x => (*((*_1).0: &T));
```
In MIR, closure calls become regular function calls. The closure environment
(the captured-variable struct) is passed as the first argument.

### Operator Lowering

[rustc-dev-guide §44.1, mir/construction.html "Operator lowering"]

Operators on **builtin types** (integers, floats, booleans) are NOT lowered
to function calls — they become `Rvalue::BinaryOp` or `Rvalue::UnaryOp`.
This avoids infinite recursion through trait impls.

Operators on **user types** are lowered to `TerminatorKind::Call` targeting
the relevant trait impl (e.g., `Add::add`).

All operands to operators are first lowered to `Operand`s.

### Async/Await Desugaring

[rustc-dev-guide §71 (coroutine-closures)]

`async fn` bodies are desugared into coroutines (previously called generators).
A coroutine is a state machine represented in MIR with:
- `Rvalue::Aggregate(AggregateKind::Coroutine, ...)` to construct the initial state.
- `TerminatorKind::Yield` at each `await` point.
- `TerminatorKind::CoroutineDrop` to handle drop during suspension.
- The coroutine's state (all locals live across a `Yield`) is stored in the
  coroutine's fields. The MIR type checker assigns types to these fields.

`await expr` desugars roughly to:
```
loop {
    match Pin::new(&mut future).poll(cx) {
        Poll::Ready(v) => break v,
        Poll::Pending => yield (),
    }
}
```

---

## 4. The Borrow Checker (NLL)

[rustc-dev-guide §69, borrow-check.html; §69.4, borrow-check/region-inference.html]

### What the Borrow Checker Enforces

The borrow checker (`rustc_borrowck` crate, entry point `mir_borrowck` query)
enforces:
- All variables are initialized before use.
- No value is moved twice.
- No value is moved while borrowed.
- No place is accessed while mutably borrowed (except through the reference).
- No place is mutated while immutably borrowed.

### Major Phases

1. **Clone MIR and replace regions.** `replace_regions_in_mir` replaces all
   region annotations in the function body with fresh inference variables.
   It also identifies the "universal" (free) regions from the function signature.

2. **Dataflow analyses.** Compute what data is moved and when (move analysis,
   initialization analysis). This uses `MaybeInitializedPlaces` and
   `MaybeUninitializedPlaces` dataflow analyses.

3. **MIR type check.** A second type-check pass over MIR (distinct from the
   regular HIR type-checker) generates constraints between region variables.

4. **Region inference.** Compute values for all region variables — the sets
   of CFG points where each region must be valid.

5. **Compute borrows in scope** at each point.

6. **Error reporting walk.** Walk the MIR again and report errors based on
   all the computed information.

### Core Concepts: Loans, Regions, Liveness

**Regions** (also called lifetimes in this context) are sets of CFG points.
Formally, a region `'a`'s value `Values('a)` is a set of elements:
- CFG locations (basic block index + statement index)
- `end('b)` elements for universal regions `'b` (representing caller execution)
- `placeholder(n)` elements for universally-quantified placeholder regions

**Liveness constraints:** A region `R` must contain point `P` if a variable
of type containing `R` is live at `P`. Computed by the MIR type checker.

**Outlives constraints:** `'a: 'b` means `Values('a)` must be a superset of
`Values('b)`. Generated by subtyping relationships.

**Loans:** A borrow creates a loan — a record of what was borrowed, how (shared
or mutable), and at what CFG point. The borrow checker checks that no
conflicting access to the borrowed place occurs while the loan is live.

### Two-Phase Borrows

[rustc-dev-guide §69.5, borrow-check/two-phase-borrows.html]

**The Problem They Solve**

Without two-phase borrows, code like `vec.push(vec.len())` is rejected.
The naive reading is:
1. Take `&mut vec` for the `push` receiver.
2. Then read `vec.len()` — but `vec` is already mutably borrowed!

This is actually safe because `vec.len()` is computed *before* the mutable
borrow is activated (before `push` actually starts).

**What They Are**

Two-phase borrows are a special kind of mutable borrow with two phases:

- **Reservation phase**: The borrow is created and acts like a *shared* borrow.
  Other shared borrows of the same place are allowed.
- **Activation phase**: At the point where the borrow is first used (always a
  function call), it becomes a full mutable borrow.

Between reservation and activation, the borrow acts as shared. After activation,
it acts as mutable.

**When Two-Phase Borrows Are Generated**

Only *implicit* mutable borrows can be two-phase:
1. Autoref borrow for method call with `&mut self` receiver.
2. Mutable reborrow in function arguments.
3. Implicit mutable borrow in overloaded compound assignment (`+=`, etc.).

Explicit `&mut` or `ref mut` in source code is never two-phase.

**Implementation**

- Tracked via a flag on `AutoBorrow` after type-checking.
- Converted to `BorrowKind` during MIR construction.
- Each two-phase borrow is assigned to a temporary used exactly once.
- **Reservation point**: where the temporary is assigned.
- **Activation point**: where the temporary is used (always a function call),
  found by the `GatherBorrows` visitor. Stored in `BorrowData`.

**Checking Rules**
- At reservation: error if conflicting live *mutable* borrows exist; lint if
  conflicting *shared* borrows exist.
- Between reservation and activation: treat as shared borrow.
- Activation: check no conflicting borrows exist (as if mutable).
- `is_active` uses MIR dominator tree to determine the current phase.

### Region Inference: Constraint Generation and Solving

[rustc-dev-guide §69.4, borrow-check/region-inference.html; §69.4.1, constraint-propagation.html]

**Setup: Universal Regions**

`replace_regions_in_mir` identifies "universal" regions — those appearing in
the function signature (like `'a` in `fn foo<'a>(...)`). These are not
inference variables; they represent caller-controlled lifetimes. A
`UniversalRegions` struct holds indices for all free regions plus known
relationships (`where` clauses, implied bounds).

**Inference Variables**

Each region appearing in the MIR body (other than universal regions) is
replaced with a fresh inference variable. A variable's value is a set of
`RegionElement`s.

**Constraint Propagation as Fixed-Point**

Conceptually, region inference is a fixed-point computation:
```
Initially: Values(R) = {} for all inference variables R
Repeat until fixpoint:
  For each liveness constraint "R live at E":  Values(R) += {E}
  For each outlives constraint "R1: R2":       Values(R1) += Values(R2)
```

**Optimization via SCCs**

In practice, outlives constraints are converted to a directed graph where
nodes are region variables and each `'a: 'b` becomes an edge `'a -> 'b`.
Strongly Connected Components (SCCs) are computed. Regions in the same SCC
must be equal — their values are merged.

This gives a DAG of SCCs. Propagation processes SCCs in reverse topological
order, unioning the values of each SCC's successors into its own value.
This is `O(N)` rather than iterative.

**Error Checking**

After propagation:
1. **Type tests**: Check constraints like `T: 'a`.
2. **Universal region checks**: Each universal region `'a`'s computed value
   must only contain elements it is known to contain. If `Values('a)` contains
   `end('b)`, we must already know that `'a: 'b` holds. If not — error.

**Placeholders and Universes**

[rustc-dev-guide §69.4.4, placeholders-and-universes.html]

Higher-ranked types (e.g., `for<'a> fn(&'a u32)`) introduce placeholder
regions `!1`, `!2`, etc. Each placeholder belongs to a universe (U1, U2, ...).
An inference variable in universe `Uk` can only have its value contain
placeholders from universes `U0..Uk`.

This mechanism soundly handles subtyping of higher-ranked types.

### Polonius: The Datalog Formulation

[rustc-dev-guide §69.4; NLL RFC background]

Polonius is a more precise borrow checker formulation, developed as a research
project. Key aspects:

**Why More Precise**

The NLL borrow checker has a known limitation: it can reject some programs
that are actually safe, particularly those involving "conditional returns" of
borrows. The classic example:

```rust
fn get_or_insert<'a>(map: &'a mut HashMap<K, V>, key: K, default: V) -> &'a V {
    // NLL incorrectly rejects this, Polonius accepts it
    if let Some(v) = map.get(&key) { return v; }
    map.insert(key, default);
    map.get(&key).unwrap()
}
```

NLL conservatively keeps the loan from `map.get()` alive through the
conditional, causing a conflict with `map.insert()`.

**The Datalog Formulation**

Polonius reformulates borrow checking as a Datalog program. The core idea:
instead of asking "what regions must contain this CFG point?" (NLL's approach),
Polonius asks "which loans are live at each point?" using fact-based rules.

Key Polonius facts (inputs):
- `loan_issued_at(loan, point)` — a loan is created at a point
- `loan_killed_at(loan, point)` — a loan is invalidated at a point (by a move or end of borrow)
- `loan_invalidated_at(loan, point)` — a conflicting access occurs

Key Polonius rule:
```datalog
// A loan is live at a point if it was issued and not yet killed
loan_live_at(loan, point) :-
    loan_issued_at(loan, origin),
    origin_live_on_entry(origin, point),
    !loan_killed_at(loan, point).

// An error occurs if a live loan is invalidated
error_at(point) :-
    loan_invalidated_at(loan, point),
    loan_live_at(loan, point).
```

Polonius tracks loan liveness through the CFG using origin (region) liveness,
but uses the *loan* as the actual unit, not just the region. This lets it
distinguish the case where a loan has provably ended (even if the region
variable technically still "contains" that point in NLL's formulation).

Polonius is more precise but has worse worst-case complexity (`O(N^3)` in
pathological cases with NLL being `O(N^2)`). It is available on nightly via
`-Z polonius` and is being incrementally adopted.

---

## 5. Drop Elaboration

[rustc-dev-guide §68, mir/drop-elaboration.html]

### The Problem: Dynamic Drop Decisions

When MIR is first built, `Drop` and `DropAndReplace` terminators mark *potential*
drop points. Whether a destructor actually runs depends on whether the variable
is initialized at runtime — and this is often not statically known:

```rust
let mut y = vec![];
{
    let x = vec![1, 2, 3];
    if std::process::id() % 2 == 0 {
        y = x;  // conditionally move x into y
    }
}
// Does x need to be dropped here? Only if it wasn't moved.
```

### Drop Obligations

[rustc-dev-guide §68, RFC 320 "Non-zeroing dynamic drops"]

When a variable becomes initialized, it establishes **drop obligations**: a
set of structural paths that must be dropped.

Rules:
- If type `T` implements `Drop`, the variable itself is a drop obligation.
- If `T` does NOT implement `Drop`, the drop obligations are the union of
  obligations of all fields that have `Drop` implementations (recursively).
- When a structural path is *moved from*, its drop obligation is *released*.
- Types with `Drop` implementations do not permit moves from individual fields
  (so you cannot partially move a type with a custom drop).
- For enums, only the active variant's fields are drop obligations. The
  discriminant is checked at runtime.

### Drop Flags

The mechanism for tracking runtime initialization is **drop flags**: boolean
variables (typically stored in a byte) that are set when a path is initialized
and cleared when it is moved from.

Drop elaboration transforms:
1. Statements that initialize a variable → insert code to set the drop flag.
2. Statements that move from a variable → insert code to clear the drop flag.
3. `Drop` terminators → wrap in a conditional checking the drop flag.
4. `DropAndReplace` terminators → split into a conditional `Drop` followed
   by a write.

### Drop Order Rules

[rustc-dev-guide §68; Rust Reference]

- Local variables are dropped in **reverse order of declaration**.
- Struct fields are dropped in **declaration order** (not reverse).
- Tuple elements are dropped in **declaration order**.
- The drop order for temporaries follows a more complex rule related to
  statement and expression evaluation order.

### Optimization: Not Every Variable Needs a Drop Flag

[rustc-dev-guide §68, "Drop elaboration in rustc"]

The compiler runs `MaybeInitializedPlaces` and `MaybeUninitializedPlaces`
dataflow analyses. For each `Drop` terminator, the target is classified:

| Classification | Meaning | Action |
|----------------|---------|--------|
| `Static` | Always initialized | Keep `Drop` as-is (no flag needed) |
| `Dead` | Always uninitialized | Remove `Drop` entirely (no flag needed) |
| `Conditional` | Wholly initialized or wholly uninitialized | Add one drop flag |
| `Open` | May be *partially* initialized | Complex: recurse into fields |

**Open drops** require breaking down the `Drop` into field-level drops, since
calling the type's drop glue requires all fields to be initialized. Open drops
only occur for types without a custom `Drop` impl (since those prohibit
partial moves).

For enums in Open drops, each variant is treated like a "field" — at most one
can be initialized. May require checking the discriminant at runtime.

### Drop Glue vs. Drop::drop

The "drop glue" (or "drop shim") for a type:
1. Calls `Drop::drop` if the type has an explicit `Drop` impl.
2. Recursively calls drop glue for all fields.

`Drop` terminators in fully elaborated MIR represent calls to drop glue,
not direct calls to `Drop::drop`.

### Const Eval Interaction

Non-const `Drop` implementations cannot be called in const-eval contexts.
However, checking this on the *unelaborated* MIR would produce false positives
(a `Drop` terminator may be `Dead` after elaboration). Therefore, the check
for non-const drops runs **after** drop elaboration, which has already
eliminated `Dead` drop sites.

---

## 6. MIR Optimizations

[rustc-dev-guide §73, mir/optimizations.html; §44.3, mir/passes.html]

### Why Optimize on MIR?

MIR optimizations are run before codegen (before LLVM IR is generated).
Two reasons:
1. **Better final code**: LLVM sees simpler, already-optimized input.
2. **Faster compilation**: LLVM has less work to do.

Most importantly: **MIR is still generic** (not yet monomorphized). An
optimization applied to a generic function body applies to all monomorphizations
for free. This is a significant advantage over LLVM IR, which operates post-
monomorphization.

### The MIR Pass Pipeline

[rustc-dev-guide §44.3, mir/passes.html]

The MIR processing pipeline is structured as a series of queries, each
"stealing" the MIR from the previous query (to avoid cloning):

```
mir_built(D)
  -> mir_const(D)              [simple transforms; const qualification reads here]
  -> mir_promoted(D)           [extract promoted constants; borrow check reads here]
  -> mir_drops_elaborated_and_const_checked(D)
     [borrow checking, drop elaboration, major transforms]
  -> optimized_mir(D)          [all enabled optimizations]
```

Each query defines a list of `MirPass` implementations to run. Passes are
defined in `rustc_mir_transform`.

**Querying optimized MIR**: `optimized_mir(D)` for codegen;
`mir_for_ctfe(D)` for const evaluation (different pass configuration).

### Pass Implementation Pattern

Each pass is a zero-sized struct implementing `MirPass<'tcx>`:

```rust
pub struct RemoveStorageMarkers;

impl<'tcx> MirPass<'tcx> for RemoveStorageMarkers {
    fn run_pass(&self, tcx: TyCtxt<'tcx>, body: &mut Body<'tcx>) {
        // modify body in-place
    }
}
```

MIR is modified in place (efficient — no cloning needed per pass).

### Key Passes

| Pass | Phase | Description |
|------|-------|-------------|
| `CleanupPostBorrowck` | post-borrow-check | Removes info only needed for analysis, not codegen |
| `RemoveStorageMarkers` | optimization | Removes `StorageLive`/`StorageDead` if not needed |
| `ConstProp` | optimization | Constant propagation and folding |
| `SimplifyCfg` | optimization | Removes unreachable blocks, merges trivial chains |
| `SimplifyLocals` | optimization | Removes unused locals |
| `RemoveUnneededDrops` | optimization | Eliminates drops of types that don't need dropping |
| `CopyProp` | optimization | Copy propagation |
| `DeadStoreElimination` | optimization | Removes writes to places never read |
| `Inline` | optimization | Function inlining |
| `CheckPackedRef` | lint | Checks for references to packed struct fields |

### MIR Optimization Levels

Controlled by `-Z mir-opt-level`:
- Level 0: Only required passes (correctness).
- Level 1: Basic optimizations (default for debug builds).
- Level 2: More aggressive optimizations (default for release builds).
- Level 3+: Experimental optimizations.

### Why Some Optimizations Are Better on MIR Than LLVM IR

- **Generics**: Optimize once, benefit all monomorphizations.
- **Semantic knowledge**: MIR knows about moves, drops, borrows — LLVM does not.
  `RemoveUnneededDrops` can only work on MIR because LLVM doesn't understand
  Rust's drop semantics.
- **Faster feedback**: MIR optimizations run before codegen, giving the
  compiler faster overall compile times.
- **No SSA complications**: MIR's place-based representation makes some
  transforms (like copy propagation of non-`Copy` types via move tracking)
  easier to reason about than in SSA.

---

## 7. Implementation Decision Guide

### When to Use MIR-Style IR vs Simpler Approaches

**Use a MIR-style IR when you need:**
- Flow-sensitive analysis (liveness, initialization, borrow checking).
- Explicit modeling of control flow (for optimization or verification).
- Clear separation of expression evaluation and memory operations.
- Drop/destructor semantics that must be analyzed.
- Any form of region/lifetime analysis.

**A simpler tree IR suffices when:**
- You only need type checking (tree traversal is sufficient).
- You are targeting a VM with its own CFG (e.g., WASM, JVM).
- Control flow is not performance-critical and errors are easy to surface.
- You have no move semantics or destructors to track.

**The MIR transition point in rustc** is: HIR for type inference and trait
solving; THIR as an intermediate; MIR for everything that requires control
flow (borrow checking, const eval, optimizations, codegen).

### The Minimal MIR for Borrow Checking

To implement a borrow checker on a MIR-like IR, you need:

1. **Control-flow graph** with basic blocks and explicit branches.
2. **Explicit place model**: locals + projections (field, deref, index).
3. **Explicit move/copy distinction** in operands.
4. **Borrow rvalue** annotated with region variable + mutability.
5. **StorageLive/StorageDead** (or equivalent) for liveness scope.
6. **Drop terminators** at scope exits for every place that might be
   initialized.
7. **FalseEdge terminators** (or equivalent) to model the borrow checker's
   view of match arm scope (where a borrow appears to possibly flow to the
   arm's "imaginary" continuation for analysis purposes).
8. **Region inference infrastructure**: a set of region variables, constraint
   generation from type-checking, and a fixed-point or SCC-based solver.

You do NOT need:
- Full optimization passes (those are optional).
- Promoted constants.
- Const evaluation.
- Two-phase borrows (they are an ergonomics feature; safe to omit initially).
- Polonius (NLL is sufficient for most programs).

### Cost of Implementing a Full Borrow Checker

**Estimated complexity layers:**

| Component | Complexity | Notes |
|-----------|-----------|-------|
| MIR data types | Low | ~500 lines of data definitions |
| HIR/AST -> MIR lowering | High | Complex; match lowering alone is substantial |
| Move analysis (dataflow) | Medium | Standard monotone dataflow; ~1000 lines |
| Region variable plumbing | Medium | Replace regions with fresh vars; ~500 lines |
| MIR type checker (constraint gen) | High | Must handle all type rules; ~3000+ lines in rustc |
| Region inference (constraint solving) | Medium | SCC + fixed-point; ~1500 lines in rustc |
| Loan tracking + conflict detection | High | The "heart" of the checker; ~2000+ lines |
| Error diagnostics | High | Ongoing; rustc has invested heavily here |
| Two-phase borrows | Medium | Adds ~500 lines but improves ergonomics |
| Drop elaboration | High | Requires dataflow; ~2000 lines in rustc |
| Polonius | Very High | Requires Datalog engine or equivalent |

**Key insight from rustc's experience**: The borrow checker is not a single
monolithic component. It is the composition of: dataflow analyses (move/init
tracking), a second type checker (for constraint generation), a region
inference engine (for solving), and a conflict checker. Each piece is
independently testable and improvable.

**Practical advice for a new implementation:**
1. Start with lexical lifetimes (lifetime = syntactic scope). Much simpler;
   no region inference needed. Accept false positives.
2. Add NLL by replacing lexical lifetimes with CFG-point sets and building
   the constraint/inference engine.
3. Add two-phase borrows only after NLL works correctly.
4. Add Polonius only if NLL false-positive rate is unacceptable.

**The single hardest part** is the interaction between borrows and moves in
the presence of control flow. The move analysis must track partial
initialization (which fields of a struct are initialized), and the borrow
checker must reason about this in the face of conditional branches and loops.
Drop elaboration depends on both.

---

## Appendix: Quick Reference — MIR Rust Types

```
rustc_middle::mir::Body<'tcx>          // A function's MIR
  .basic_blocks: IndexVec<BasicBlock, BasicBlockData<'tcx>>
  .local_decls: IndexVec<Local, LocalDecl<'tcx>>
  .arg_count: usize
  .spread_arg: Option<Local>

BasicBlockData<'tcx>
  .statements: Vec<Statement<'tcx>>
  .terminator: Option<Terminator<'tcx>>

Statement<'tcx>
  .source_info: SourceInfo
  .kind: StatementKind<'tcx>

Terminator<'tcx>
  .source_info: SourceInfo
  .kind: TerminatorKind<'tcx>

Place<'tcx>
  .local: Local
  .projection: &'tcx [PlaceElem<'tcx>]  // PlaceElem = ProjectionElem<Local, Ty<'tcx>>

Rvalue<'tcx>  // right-hand side of Assign
Operand<'tcx> // leaf value: Constant | Copy(Place) | Move(Place)

// Key crates:
// rustc_middle::mir          — data types
// rustc_mir_build            — HIR/THIR -> MIR lowering
// rustc_mir_transform        — passes (optimizations, drop elaboration)
// rustc_mir_dataflow         — dataflow analysis infrastructure
// rustc_borrowck             — borrow checker
```

---

*Last updated: 2026-03-15. Based on rustc-dev-guide as of early 2026.*
*Sections: mir/index.html, borrow-check.html, borrow-check/two-phase-borrows.html,*
*borrow-check/region-inference.html, mir/drop-elaboration.html, mir/optimizations.html,*
*mir/construction.html, mir/passes.html, borrow-check/region-inference/constraint-propagation.html,*
*borrow-check/region-inference/placeholders-and-universes.html*
