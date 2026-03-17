---
name: compiler-optimizer
description: Compiler optimization expert — covers classical optimizations (DCE, CSE, constant propagation, inlining), loop optimizations, alias analysis, TBAA, dataflow analysis, SSA construction, and pass manager design. Use when implementing optimization passes or designing an optimization pipeline.
---

# Compiler Optimization

## Reference Files

Detailed technical content is in:
- **[optimizer-reference.md](references/optimizer-reference.md)** — Comprehensive reference drawn from EaC Chapters 8 and 9. Covers: optimization taxonomy, local value numbering, DCE/CSE/constant propagation/copy propagation with full algorithms, the complete dataflow analysis framework (liveness, available expressions, reaching definitions, anticipable expressions, worklist solver), SSA construction and use (SSCP, GVN, phi-function placement), loop optimizations (LICM, strength reduction, unrolling), alias analysis and TBAA, and pass ordering.

---

## Quick Decision Guide

| Situation | Recommended Approach |
|-----------|----------------------|
| Simple compiler, no dataflow | Constant folding + local value numbering (LVN) + local DCE — all within single blocks, no analysis required |
| Need global dead code elimination | Compute LIVEOUT (backward dataflow) → global DCE; or use SSA use-count worklist |
| Need cross-block CSE | Compute AVAILIN (forward dataflow, intersection) → seed LVN at block entry |
| Constant propagation in loops | Use SCCP / SSCP with optimistic initialization (⊤ for unknowns); pessimistic (⊥) fails to propagate into loops |
| Want efficient modern optimizer | Use SSA as IR: GVN and SSCP become simple worklist algorithms with O(n) complexity |
| Identify and optimize loops | Compute dominators → find back edges → identify natural loops → apply LICM and strength reduction |
| Safe code motion out of a loop | Instruction must dominate ALL loop exits; must be side-effect-free; all operands must be loop-invariant |
| Memory disambiguation | TBAA (type-incompatible pointers cannot alias); `restrict` (programmer asserts exclusive access); points-to analysis for full precision |
| Pass order for production pipeline | instcombine → inline → instcombine → SCCP → GVN → LICM → DCE |
| Pass order for simple compiler | constant folding + copy prop → DCE → [inline → LICM] → DCE |
| Two passes that enable each other | Run in a fixed-point loop until neither makes progress |
| Analysis result caching | Transform passes declare required analyses and which they invalidate; cache results between passes |

---

## Overview

Optimization transforms the program's IR to produce equivalent but more efficient code. The key constraint: **semantics must be preserved**. An optimization may reorder, eliminate, or reshape operations, but the observable behavior of the program must be unchanged.

This skill covers:
1. Optimization taxonomy (local, global, interprocedural)
2. Classical passes: DCE, CSE, constant folding/propagation, copy propagation, inlining
3. Dataflow analysis framework: liveness, available expressions, reaching definitions, worklist algorithm
4. SSA form: construction, SSCP, GVN, out-of-SSA translation
5. Loop optimizations: LICM, strength reduction, unrolling
6. Alias analysis and TBAA
7. Pass manager design and pass ordering

---

## Optimization Taxonomy

Optimizations are classified by their **scope** — the amount of the program they can see and transform.

**Local optimizations** — within a single basic block:
- Only see the straight-line code between two branch points
- Cannot reason about values that might arrive from different predecessors
- Fastest to compute; no CFG or dataflow analysis required
- Examples: constant folding, local value numbering, algebraic simplification, peephole

**Global optimizations** — within a single function:
- See the entire CFG of one function
- Can reason about values that flow across basic block boundaries
- Require dataflow analysis (liveness, reaching definitions, dominance)
- Examples: global DCE, global CSE (via GVN), constant propagation (SCCP), LICM, inlining

**Interprocedural (IPA) / whole-program optimizations**:
- See multiple functions or the entire program
- Can propagate constants across call sites, inline across module boundaries, devirtualize
- Require points-to / alias analysis across the call graph
- Much more expensive; typically run at link time (LTO)
- Examples: interprocedural constant propagation, devirtualization, dead function elimination, IPO inlining

---

## Classical Passes

### Dead Code Elimination (DCE)

**Goal**: remove definitions (assignments, computations) whose results are never used.

**Local DCE**: within a basic block, a definition is dead if no subsequent instruction in the block uses it and the variable is not live on exit. Remove dead definitions in a backward pass.

**Global DCE** (liveness-based): compute liveness (backward dataflow over the CFG). A definition of `x` is dead at a point if `x` is not live (used on any path forward from that point). Remove dead definitions; iterate to a fixed point.

**SSA-based DCE**: in SSA form, a definition is dead iff its use count is zero. DCE becomes a simple worklist algorithm: initialize the worklist with all definitions; when a definition's use count drops to zero, remove it and decrement the use counts of its operands; repeat.

**Note**: side-effecting instructions (stores, calls, I/O) cannot be eliminated even if their result is unused.

---

### Common Subexpression Elimination (CSE)

**Goal**: if the same expression is computed more than once and the values it reads have not changed, replace the second computation with a use of the first.

**Local CSE** (value numbering): assign a value number to each expression. Two expressions with the same value number are provably equal. Maintain a hash table mapping (op, vn(arg1), vn(arg2)) → vn(result). Before computing an expression, check if its value number already exists.

**Global CSE** (via GVNPRE or similar): extends value numbering to dominating expressions across basic blocks. An expression `e` can be replaced at a use site if `e` is available (computed on every path from entry to this point without any of its inputs being redefined).

**In SSA form**: GVN (Global Value Numbering) is simpler. Since each name has one definition, two names with the same definition (same opcode, same operand names) compute the same value. GVN can be implemented as a hash-cons over SSA values.

---

### Constant Folding and Constant Propagation

**Constant folding**: if all operands of an operation are compile-time constants, evaluate the operation at compile time.
```
_1 = 3 * 4     →   _1 = 12
_2 = _1 + 0   →   _2 = 12   (if _1 is known to be 12)
```

**Constant propagation**: track which variables hold constant values. At each use of a variable, if the variable is known to be a constant, replace the use with the constant literal.

**Sparse Conditional Constant Propagation (SCCP)**: more powerful. Simultaneously propagates constants and folds unreachable branches. If a conditional branch has a constant condition, the false branch is marked unreachable; constants from the true branch propagate further. Values use a three-point lattice: `⊤` (unknown), `constant(v)`, `⊥` (non-constant). The algorithm propagates along the lattice monotonically.

---

### Copy Propagation

**Goal**: if `y = x` (a copy), replace later uses of `y` with `x` as long as neither `x` nor `y` is redefined.

In SSA form, copy propagation is trivially simple: if `y` is defined by `y = x`, substitute `x` for all uses of `y` and remove the definition. There are no intervening redefinitions in SSA.

**Out-of-SSA copy propagation**: more complex; requires checking that `x` is not redefined between the copy assignment and each use of `y`.

---

### Function Inlining

**Goal**: replace a function call with the callee's body. Eliminates call overhead; enables further optimization of the combined body (constant propagation into the callee, CSE across the call boundary).

**When to inline**:
- Small functions (leaf functions, getters/setters, simple wrappers) — always beneficial
- Hot call sites (frequently executed) — high payoff
- Calls where arguments are constants — enables SCCP to fire in the callee

**Costs**:
- Code size growth (code bloat): inlining large callees many times increases binary size
- Increased register pressure: the combined function has more live values
- Compilation time: the inlined body must be re-analyzed and optimized in the new context

**Heuristics**: most compilers use a cost model (instruction count of callee, estimated benefit based on argument constants, call site hotness from profile data). LLVM uses an inlining cost threshold.

**Inlining order**: inline callees before callers (bottom-up call graph order). This ensures that the callee has been optimized before it is inlined.

---

## Loop Optimizations

Loops are the primary source of runtime cost in most programs. Loop optimizations have a disproportionate impact on performance.

### Loop-Invariant Code Motion (LICM)

**Goal**: move computations whose value does not change across loop iterations out of the loop (into the loop pre-header).

**Condition for hoisting `x = f(a, b)`**:
1. `f` is side-effect-free (or can be moved safely)
2. All operands `a`, `b` are loop-invariant (defined outside the loop or already hoisted)
3. The block containing the computation dominates all loop exits (ensures the computation would have been executed anyway)

**Algorithm**: mark values defined outside the loop as invariant; iteratively mark loop-body definitions as invariant if all their operands are invariant; hoist marked definitions to the pre-header.

### Strength Reduction

**Goal**: replace expensive operations (multiply, divide) with cheaper ones (add, shift) by exploiting the inductive structure of loops.

**Classic example**: replace `a * i` inside a loop (where `i` is a linear induction variable) with a new induction variable that is incremented by `a` each iteration.

```
// Before:
for i in 0..n:
    x = a * i    // multiply each iteration

// After:
x = 0
for i in 0..n:
    // use x instead of a * i
    x += a       // add each iteration
```

### Loop Unrolling

**Goal**: reduce loop control overhead (branch, counter update) by duplicating the loop body multiple times per iteration, reducing the number of iterations.

```
// Before (unroll factor 4):
for i in 0..n:
    body(i)

// After:
for i in 0..n step 4:
    body(i); body(i+1); body(i+2); body(i+3)
// handle remainder: body(n-n%4), ...
```

**Benefits**: exposes instruction-level parallelism; reduces branch overhead; enables better register allocation within the unrolled body.

**Cost**: code size increase. Over-unrolling can hurt performance by increasing I-cache pressure.

**Full unrolling**: for loops with a constant trip count, the loop can be completely unrolled (eliminated entirely).

---

## Alias Analysis and TBAA

### The Alias Problem

Two pointers **alias** if they might refer to the same memory location. Alias analysis determines which pairs of pointers cannot alias (are *distinct*) and which might alias.

**Why it matters for optimization**: the compiler cannot hoist a load out of a loop if a store inside the loop might modify the loaded location. If alias analysis proves the store and load access disjoint locations, the hoist is safe.

### Types of Alias Analysis

**Points-to analysis**: computes, for each pointer variable, the set of memory locations it might point to. Two pointers alias if their points-to sets overlap.

**Flow-sensitive vs. flow-insensitive**: flow-sensitive analysis considers the order of statements (more precise, more expensive); flow-insensitive analysis ignores order (less precise, cheaper; the standard for interprocedural analysis).

**Context-sensitive vs. context-insensitive**: context-sensitive analysis tracks the call site from which a function was called (more precise for function pointers); context-insensitive uses a single summary per function.

**Andersen's analysis**: inclusion-based points-to analysis. More precise than Steensgaard's (union-find-based) but more expensive. Andersen: O(n³); Steensgaard: nearly O(n).

### Type-Based Alias Analysis (TBAA)

**Key insight** (from the C standard, §6.5): an object may only be accessed through an lvalue of a compatible type or a `char` type. Therefore, if two pointers have incompatible types, they cannot alias.

**In practice**: if `int *p` and `float *q` point to different types, the compiler assumes `*p` and `*q` do not alias, even if their addresses happen to be equal. This enables significant optimization (e.g., hoisting loads across stores of a different type).

**LLVM's TBAA metadata**: LLVM attaches TBAA metadata to load/store instructions. The metadata describes a type hierarchy; two accesses cannot alias if their types are unrelated in the hierarchy.

**`restrict` keyword** (C99): the programmer asserts that a pointer is the only pointer to a given memory region in the current scope. This is the most powerful alias annotation — it tells the compiler to assume no aliasing without requiring type incompatibility.

---

## Pass Manager Design

### The Problem

A compiler has many optimization passes. They must be run in the right order, multiple times if needed (passes create opportunities for other passes), and efficiently (avoid redundant work).

### LLVM-style Pass Manager (New PM)

**Analysis passes**: compute information (liveness, dominator tree, alias analysis). Results are cached and invalidated when the IR changes.

**Transform passes**: modify the IR. They declare which analyses they require and which they invalidate.

**Pass pipeline**: a sequence of passes applied to a function (or module). The pipeline is specified as a string (e.g., `-passes=instcombine,gvn,licm,dce`) or composed programmatically.

**Pass composition**:
- `FunctionPassManager`: runs a sequence of function-level passes on each function
- `ModulePassManager`: runs module-level passes (IPA, LTO) on the entire module
- `LoopPassManager`: runs loop passes on each loop in a function (innermost loops first)
- Passes can be nested: a `ModulePass` can run a `FunctionPassManager` on each function

**Pass ordering heuristics**:
1. Run `instcombine` (algebraic simplification, local DCE) first — cheap, high payoff, creates opportunities
2. Run inlining — exposes more optimization opportunities in the combined body
3. Run `instcombine` again — clean up after inlining
4. Run GVN, SCCP, LICM — global analysis passes
5. Run DCE — clean up dead code left by propagation
6. Register allocation and code generation

### Fixed Pipeline (simpler compilers)

For simpler compilers that do not need the full flexibility of a pluggable pass manager, a fixed pipeline is acceptable:
1. Constant folding + copy propagation
2. DCE
3. Inlining (optional)
4. LICM (optional, if loops are present)
5. DCE again

---

## Implementation Checklist

- [ ] Choose IR form (SSA recommended for global optimizations; linear TAC for simple compilers)
- [ ] Implement CFG construction (basic blocks, edges)
- [ ] Implement dominator tree computation
- [ ] Implement constant folding (inline, during IR construction or as a pass)
- [ ] Implement local value numbering (CSE within a block)
- [ ] Implement DCE (SSA-based worklist or liveness-based)
- [ ] Implement copy propagation
- [ ] Implement constant propagation (simple reaching-def or SCCP)
- [ ] Implement inlining (define cost model and heuristics)
- [ ] Implement LICM (loop-invariant code motion)
- [ ] Implement GVN for global CSE (if SSA)
- [ ] Implement basic alias analysis (type-based at minimum)
- [ ] Design pass manager (fixed pipeline or pluggable)
- [ ] Implement analysis caching and invalidation

---

## Incorporated Sources

- **EaC ch. 8** — Introduction to Optimization (fully incorporated in optimizer-reference.md)
- **EaC ch. 9** — Data-Flow Analysis (fully incorporated in optimizer-reference.md)

## Planned Sources (not yet incorporated)

- EaC ch. 10 — Scalar Optimizations (DCE, CSE, constant propagation, LICM — deeper treatment)
- EaC ch. 11 — Redundancy Elimination (SSA-based GVN, PRE, lazy code motion)
- LLVM pass manager documentation and source (`llvm/include/llvm/IR/PassManager.h`)
- Cooper & Harvey & Kennedy — "A Simple, Fast Dominance Algorithm" (faster IDOM computation)
- Wegman & Zadeck — "Constant Propagation with Conditional Branches" (full SCCP with conditional reachability)

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-mir` — optimization runs on the MIR-level CFG
- `compiler-hir` — for compilers without MIR, optimization may run on HIR
- `compiler-expert` — routes here on optimization pass / DCE / CSE / SSA questions

**Trace Down ↓** — next skill after optimization
- `compiler-codegen` — optimized IR feeds into instruction selection and register allocation

**Related →** — closely related skills
- `compiler-query-system` — incremental compilation and query caching interact with the optimization pass manager
- `compiler-comptime` — constant propagation and partial evaluation overlap with comptime
