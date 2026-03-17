# Compiler Optimizer Reference

**Sources:**
- **[EaC Ch8]** Engineering a Compiler, Chapter 8 — Introduction to Optimization (Cooper & Torczon, 3rd ed., 2023)
- **[EaC Ch9]** Engineering a Compiler, Chapter 9 — Data-Flow Analysis (Cooper & Torczon, 3rd ed., 2023)

---

## Quick Decision Guide

| Task | Recommended Approach |
|------|----------------------|
| First pass, simple compiler | Constant folding + local value numbering + DCE — no dataflow needed |
| Need global optimization | Build CFG, compute LIVEOUT (backward), run global DCE and copy propagation |
| Want CSE across blocks | Compute AVAILIN (forward), use as initial info for LVN in each block |
| Build modern optimizer | Use SSA form as the IR; GVN and SSCP become simple worklist algorithms |
| Optimize loops | Identify natural loops via back edges + dominators; apply LICM, then strength reduction |
| Memory disambiguation | Add TBAA metadata; use `restrict` annotations where programmer guarantees no aliasing |
| Pass ordering | instcombine → inline → instcombine → GVN/SCCP/LICM → DCE |
| Constant propagation in loops | Use SCCP/SSCP with optimistic initialization (⊤ for unknowns), not pessimistic (⊥) |

---

## 1. Optimization Taxonomy

Optimizations are classified by **scope** — the amount of the program they can see and transform. [EaC Ch8]

**Local optimizations** — within a single basic block:
- Only see straight-line code between two branch points
- Cannot reason about values arriving from different predecessors
- Fastest to compute; no CFG or dataflow analysis required
- Examples: constant folding, local value numbering (LVN), algebraic simplification, peephole optimization

**Global (intraprocedural) optimizations** — within a single function:
- See the entire CFG of one function
- Can reason about values flowing across basic block boundaries
- Require dataflow analysis: liveness, reaching definitions, dominance
- Examples: global DCE, global CSE (via GVN), constant propagation (SCCP), LICM, inlining

**Regional optimizations** — subsets of the CFG larger than a block but not the whole procedure:
- Extended basic blocks (EBBs): trees of blocks with a single entry; superlocal value numbering
- Loop bodies: loop-specific analyses and transformations
- Examples: loop-invariant code motion (scoped to loop body), superlocal value numbering

**Interprocedural (IPA) / whole-program optimizations**:
- See multiple functions or the entire program
- Can propagate constants across call sites, inline across module boundaries, devirtualize
- Require points-to / alias analysis across the call graph
- Much more expensive; typically run at link time (LTO)
- Examples: interprocedural constant propagation, devirtualization, dead function elimination, IPO inlining [EaC Ch8]

**Key principle** [EaC Ch8]: A transformation needs analytical information that covers *at least as large a scope* as the transformation itself — local optimizations need local info; global optimizations need global info.

---

## 2. Classical Local Optimizations

### 2.1 Local Value Numbering (LVN)

**Purpose**: Detect and eliminate redundant expressions within a single basic block; simultaneously perform constant folding, copy propagation, and CSE. [EaC Ch8]

**Mechanism**: Assign a value number (VN) to each computation. Two expressions have the same VN if and only if they are provably equal. Maintain a hash table mapping `(op, VN(arg1), VN(arg2)) → VN(result)`.

**Algorithm** (forward pass over the block):
1. For each operation `x ← y op z`:
   a. Look up (or assign) VN for each operand: `vn_y = VN(y)`, `vn_z = VN(z)`
   b. If `y` or `z` have known constant values, fold: evaluate `y op z` at compile time
   c. Check hash table for the key `(op, vn_y, vn_z)`
   d. If found: `x` gets the same VN as the existing result; replace the operation with a copy
   e. If not found: assign a new VN to `x`; insert `(op, vn_y, vn_z) → VN(x)` in the table

**Handles**: constant folding, CSE within a block, copy propagation, algebraic identities (e.g., `x + 0 = x`, `x × 1 = x`, `x - x = 0`)

**Limitation**: Cannot look beyond the block boundary. Any definition of a variable in a predecessor that has multiple versions (join point) kills LVN's ability to track that variable.

**Extension — Superlocal Value Numbering (SVN)**: Extends LVN to extended basic blocks (EBBs). An EBB is a maximal set of blocks forming a tree (single-entry, tree shape). SVN processes EBBs by propagating the value table down each tree path, then restoring it when backtracking. Values from dominating blocks remain valid. [EaC Ch8]

**Uselessness criterion for DCE during LVN**: A definition `x ← ...` is useless if `x` is not in LIVEOUT and no subsequent use in the block references `x` before another definition kills it.

### 2.2 Dead Code Elimination (DCE)

**Goal**: Remove definitions (assignments, computations) whose results are never used. [EaC Ch8]

**Local DCE**: Within a basic block, a definition is dead if no subsequent instruction in the block uses it and the variable is not live on exit. Perform a backward pass:
- Maintain a live set, initialized to LIVEOUT of the block
- For each instruction `x ← y op z` (in reverse):
  - If `x` is not in the live set and `x` has no side effects: mark the instruction dead
  - Otherwise: remove `x` from live set; add `y`, `z` to live set

**Global DCE** (liveness-based): Compute LIVEOUT sets (backward dataflow over the CFG). A definition of `x` is dead at a point if `x` is not live (used on any path forward from that point). Remove dead definitions; iterate to a fixed point. [EaC Ch9]

**SSA-based DCE**: In SSA form, a definition is dead if its use count is zero. DCE becomes a simple worklist algorithm:
1. Initialize the worklist with all definitions whose use count is zero
2. Remove a definition from the worklist; delete it
3. Decrement use counts of its operands; if any operand's definition now has use count zero, add it to the worklist
4. Repeat until worklist is empty

**Critical constraint** [EaC Ch8]: Side-effecting instructions (stores, calls, I/O, volatile reads) cannot be eliminated even if their result is unused.

### 2.3 Common Subexpression Elimination (CSE)

**Goal**: If the same expression is computed more than once and none of its inputs changed between computations, replace the second computation with a use of the first result.

**Local CSE**: Handled directly by LVN — the hash table catches expressions computed earlier in the same block.

**Global CSE** via Available Expressions (AVAILIN): [EaC Ch9]
- An expression `e` is **available** at point `p` if on every path from the procedure's entry to `p`, `e` is evaluated and none of its operands is redefined
- Compute AVAILIN(n) as a forward dataflow problem (see Section 3.4 below)
- At block entry, treat AVAILIN as the initial value table for LVN; this pre-populates the hash table with expressions already computed on all incoming paths

**GVN on SSA** (see Section 4.2): In SSA form, GVN is cleaner — use hash-consing over SSA values.

### 2.4 Constant Folding and Constant Propagation

**Constant folding** [EaC Ch8]: If all operands of an operation are compile-time constants, evaluate at compile time:
```
_1 = 3 * 4      →   _1 = 12
_2 = _1 + 0     →   _2 = 12   (if _1 is known to be 12)
```
This is typically performed inline during IR construction or as part of LVN — it does not require a separate pass.

**Simple constant propagation**: Track which variables hold constant values using reaching definitions. At each use of a variable, if all definitions that reach that use assign the same constant value, replace the use with the constant literal.

**SCCP — Sparse Conditional Constant Propagation** [EaC Ch9]: The most powerful form. Simultaneously:
1. Propagates constants through the CFG
2. Folds unreachable branches: if a conditional branch has a constant condition, the false branch is marked unreachable; constants do not flow through unreachable edges

**SCCP lattice** (per SSA name):
```
       ⊤   (unanalyzed / top — might become constant)
      / \
   c1   c2   c3 ...   (known constant values)
      \ /
       ⊥   (proven non-constant / bottom)
```
**Meet rules** (in order of precedence):
- `⊤ ∧ x = x` (for any x)
- `⊥ ∧ x = ⊥` (for any x)
- `ci ∧ ci = ci` (same constant meets itself)
- `ci ∧ cj = ⊥` (if ci ≠ cj — two different constants, proven non-constant)

**SSCP algorithm** (Sparse Simple Constant Propagation, the SSA-based version) [EaC Ch9]:
```
// Initialization
for each SSA name n:
    if n's definition is a known constant ci: Value(n) = ci; add n to WorkList
    if n's value is unknowable (external input): Value(n) = ⊥; add n to WorkList
    otherwise: Value(n) = ⊤  (optimistic — do NOT add to WorkList yet)

// Propagation
while WorkList ≠ ∅:
    remove n from WorkList
    for each operation op that uses n:
        let m = SSA name defined by op
        if Value(m) = ⊥: skip (already at bottom)
        t = Value(m)
        Value(m) = evaluate op over lattice values of operands
        if Value(m) ≠ t: add m to WorkList
```

**Optimism vs. pessimism** [EaC Ch9]: SSCP initializes unknowns to `⊤` (optimistic), not `⊥` (pessimistic). This allows constant values to propagate into loops:
- **Pessimistic** initialization: `x1 ← φ(17, x2)` would compute `17 ∧ ⊥ = ⊥` immediately, blocking propagation into the loop body
- **Optimistic** initialization: `Value(x1) = ⊤` initially; `17 ∧ ⊤ = 17`; the constant 17 propagates through the loop; if the loop increments it, subsequent iterations lower `x1` to `⊥`

**Complexity** [EaC Ch9]: Each SSA name's lattice value can change at most twice (⊤ → constant → ⊥). Each name appears on the worklist at most twice. Total work is O(2 × number of uses in the code) — linear.

### 2.5 Copy Propagation

**Goal**: If `y = x` (a copy), replace later uses of `y` with `x` as long as neither `x` nor `y` is redefined between the copy and the use. [EaC Ch8]

**In SSA form**: Trivially simple. If `y` is defined by `y ← x`, substitute `x` for all uses of `y` and remove the definition. SSA guarantees no intervening redefinitions.

**Out-of-SSA (non-SSA IR)**: Requires reaching definitions analysis to confirm that `x` is not redefined on any path between the copy and each use of `y`.

**Interaction with DCE**: Copy propagation often exposes dead copies, which DCE then removes. Run them together or alternate them.

---

## 3. Dataflow Analysis Framework

### 3.1 The General Framework [EaC Ch9]

**Definition**: A collection of techniques for compile-time reasoning about the runtime flow of values. Solves simultaneous equations over sets associated with CFG nodes and edges.

**Key insight** [EaC Ch9]: A transformation needs analytical information that covers at least as large a scope as the transformation itself.

**Standard problem structure**:
- Each CFG node `n` has an associated set (e.g., LIVEOUT(n), AVAILIN(n))
- Transfer function (per block): `f(x) = c1 op1 (x op2 c2)` where c1, c2 are constants derived from the block's code and op1, op2 are set operations (union or intersection)
- Meet operator: combines information from multiple predecessors (or successors) — either union or intersection depending on the problem

**Termination guarantee** (Finite Descending Chain Property) [EaC Ch9]: The iterative algorithm halts because:
- Sets grow or shrink monotonically (never both)
- Sets are bounded in size (maximum is the universe set)
- Therefore the algorithm must reach a fixed point

**Correctness guarantee** [EaC Ch9]: Most classic dataflow problems have a *unique fixed point* — the meet-over-all-paths (MOP) solution. Uniqueness guarantees that the iterative algorithm's result equals the MOP solution regardless of evaluation order.

**Practical implementation**: Implement a single parameterized solver with:
- Functions to compute c1 and c2 (the local gen/kill sets)
- The operators op1 and op2
- A direction flag (forward/backward)
This enables code reuse across all client analyses. [EaC Ch9]

### 3.2 Forward vs. Backward Analyses [EaC Ch9]

**Forward problem**: Facts at a node `n` are computed from `n`'s CFG *predecessors*. Information flows in the same direction as execution.
- Evaluation order: Reverse Postorder (RPO) on the CFG
- Examples: Dominance, Available Expressions, Reaching Definitions, AVAILIN

**Backward problem**: Facts at a node `n` are computed from `n`'s CFG *successors*. Information flows against the direction of execution.
- Evaluation order: RPO on the *reverse* CFG (equivalently: postorder on the CFG)
- Examples: Liveness (LIVEOUT), Anticipable Expressions (ANTOUT)

**Why RPO matters** [EaC Ch9]: RPO visits as many of a node's predecessors as possible before visiting the node. This means most information is already computed when a node is visited, minimizing the number of iterations needed. On a reducible CFG with no irreducible loops, many problems converge in 1-2 passes.

### 3.3 Liveness Analysis (Backward) [EaC Ch8, Ch9]

**Definition**: A variable `v` is **live** at point `p` if and only if there is a path from `p` to a use of `v` along which `v` is not redefined (a "v-clear path"). [EaC Ch9]

**Purpose**: Used for dead code elimination, register allocation, uninitialized variable detection, SSA construction.

**Local sets** (computed per block, no iteration needed):
- **UEVAR(n)**: Set of upward-exposed variables in block `n` — variables used in `n` before any definition of them in `n`
- **VARKILL(n)**: Set of variables defined in `n` (kills their prior values)

**Equations**:
```
LIVEOUT(n) = ∪_{m ∈ succ(n)} (UEVAR(m) ∪ (LIVEOUT(m) ∩ ¬VARKILL(m)))
```
Equivalently: a variable `v` is in LIVEOUT(n) if, among `n`'s successors, there is some `m` where either `v` is used before being killed in `m` (UEVAR), or `v` is live on exit from `m` and not killed in `m`.

**Initial values**:
```
LIVEOUT(n) = ∅, for all n
```

**Solution procedure** [EaC Ch8]:
1. Build the CFG
2. Compute UEVAR and VARKILL for each block (single pass through each block)
3. Apply the iterative solver (backward direction — use RPO on the reverse CFG)

**Termination** [EaC Ch9]: Sets grow monotonically (start at ∅, can only add elements). Each set is bounded by the size of the variable set V. Terminates in at most `n × |V|` iterations; in practice much faster with RPO ordering.

**Set naming convention** [EaC Ch9]: VARKILL (variables) vs. EXPRKILL (expressions) — these are distinct sets used in different analyses. The naming encodes both domain and meaning to avoid confusion.

### 3.4 Available Expressions (Forward) [EaC Ch9]

**Definition**: An expression `e` is **available** at point `p` if and only if on every path from the procedure's entry to `p`, `e` is evaluated and none of its operands is redefined.

**Purpose**: Enables global CSE — an available expression can be replaced with the previously computed result.

**Local sets** (per block):
- **DEEXPR(n)**: Downward-exposed expressions — those evaluated in `n` where none of their operands is redefined between the evaluation and the end of `n`
- **EXPRKILL(n)**: Expressions killed in `n` — any expression in which at least one operand is redefined in `n`

**Equations**:
```
AVAILIN(n) = ∩_{m ∈ preds(n)} (DEEXPR(m) ∪ (AVAILIN(m) ∩ ¬EXPRKILL(m)))
```
An expression is available on entry to `n` if it is available on exit from *every* predecessor of `n` — hence intersection, not union.

**Initial values**:
```
AVAILIN(n0) = ∅               (entry node: nothing available yet)
AVAILIN(n) = {all expressions}, for all n ≠ n0   (optimistic start for iteration)
```

**Direction**: Forward. Use RPO on the CFG.

**Sets shrink monotonically** during iteration (start at "everything available" for non-entry nodes, narrow down). [EaC Ch9]

### 3.5 Reaching Definitions (Forward) [EaC Ch9]

**Definition**: A definition `d` of variable `x` **reaches** operation `u` if `u` uses the value of `x` and there exists a path from `d` to `u` along which `x` is not redefined.

**Purpose**: Used to identify which definition of a variable reaches a use; prerequisite for constructing SSA form (naive method) and for non-SSA copy propagation.

**Domain**: The set of *definition points* in the procedure (all assignment statements), not just variables.

**Local sets** (per block):
- **DEDEF(n)**: Downward-exposed definitions — those in `n` for which the defined name is not subsequently redefined in `n`
- **DEFKILL(n)**: Definition points killed by `n` — `d ∈ DEFKILL(n)` if `d` defines some name `v` and `n` contains a definition that also defines `v`

**Equations**:
```
REACHES(n) = ∪_{m ∈ preds(n)} (DEDEF(m) ∪ (REACHES(m) ∩ ¬DEFKILL(m)))
```
A definition reaches `n` if it reaches *some* predecessor of `n` — hence union, not intersection.

**Initial values**:
```
REACHES(n) = ∅, for all n
```

**Implementation cost** [EaC Ch9]: More expensive than liveness to initialize — computing DEDEF and DEFKILL requires a mapping from variable names to their definition points (not just from names to kill flags).

### 3.6 Anticipable Expressions (Backward) [EaC Ch9]

**Definition**: An expression `e` is **anticipable** at point `p` if (1) every path leaving `p` evaluates `e`, and (2) evaluating `e` at `p` would produce the same result as the first evaluation along each of those paths.

**Purpose**: Enables *code hoisting* — moving an expression to an earlier point where all paths will compute it, reducing redundancy. Also used in lazy code motion.

**Equations**:
```
ANTOUT(n) = ∩_{m ∈ succ(n)} (UEEXPR(m) ∪ (ANTOUT(m) ∩ ¬EXPRKILL(m)))
```

**Initial values**:
```
ANTOUT(nf) = ∅               (exit node)
ANTOUT(n) = {all expressions}, for all n ≠ nf
```

**Direction**: Backward.

### 3.7 The Worklist Algorithm (Efficient Iterative Solver) [EaC Ch9]

**Round-robin solver** (naive):
```
changed = true
while changed:
    changed = false
    for each node n (in RPO):
        temp = evaluate equation at n
        if temp ≠ current_set(n):
            current_set(n) = temp
            changed = true
```

**Worklist improvement**: Only re-evaluate a node when one of its inputs changes:
```
WorkList = all nodes
while WorkList ≠ ∅:
    remove n from WorkList
    temp = evaluate equation at n using current sets
    if temp ≠ current_set(n):
        current_set(n) = temp
        for each node m affected by n (successor for forward, predecessor for backward):
            add m to WorkList if not already there
```

**Why worklist is faster**: Nodes that have reached their fixed point are not re-evaluated. On reducible CFGs, only nodes in or after a loop need re-evaluation after back edges are processed. [EaC Ch9]

**For SSA-based analyses** (like SSCP): The worklist operates on *SSA names*, not blocks. When a name's lattice value changes, all operations that *use* that name are added to the worklist. This is sparser and more efficient than block-granularity analysis.

---

## 4. SSA-Based Optimizations

### 4.1 Why SSA Simplifies Optimizations [EaC Ch9]

SSA (Static Single-Assignment) form encodes data flow and control flow directly in the IR. Two rules:
1. Each computation in the procedure defines a unique name
2. Each use refers to a single name

**Benefits**:
- Kills are implicit: once defined, a name's value never changes, so availability is trivial
- Use-def chains are explicit: following from a use to its definition is O(1)
- Many algorithms that required iterative dataflow become simple worklist algorithms
- The φ-function at join points explicitly encodes where values merge

**SSA construction** requires:
1. Compute dominance frontiers (DF sets)
2. Insert φ-functions: for each variable `x` defined in block `n`, insert a φ-function at each block in `DF(n)` (transitive closure — φ-functions are also definitions)
3. Rename: preorder walk of dominator tree, maintaining a stack of current SSA names per base name

**Flavors** [EaC Ch9]:
- **Maximal SSA**: φ-function for every variable at every join point — too many φ-functions, wastes memory
- **Minimal SSA**: φ-function only where two distinct definitions converge — may include dead φ-functions
- **Pruned SSA**: Minimal + live at insertion point — fewest φ-functions, but requires LIVEOUT computation
- **Semipruned SSA** (practical choice): Only insert φ-functions for *global names* (variables live across block boundaries) — avoids most dead φ-functions without the full cost of pruned SSA

**Out-of-SSA translation** [EaC Ch9]: Replace each φ-function with copies inserted in predecessor blocks. Complications:
- **Critical edges**: A φ-function's predecessor with multiple successors requires splitting the edge to insert copies safely
- **Lost-copy problem**: If an optimization extends an SSA name's live range past a copy insertion point, naive copy insertion can overwrite a live value. Fix: check whether the copy target is live at the insertion point; if so, introduce a new name
- **Swap problem**: φ-functions have concurrent semantics; naive sequential copy insertion can change program behavior when φ-functions in the same block feed each other. Fix: use a two-stage parallel copy protocol, or build a dependence graph for the copy group and serialize it (breaking cycles with temporary names)

### 4.2 GVN — Global Value Numbering on SSA [EaC Ch9]

On SSA, GVN is a hash-consing operation over SSA values:
- Two SSA values are equal if they have the same opcode and the same SSA name operands
- Implement as a hash table: `(op, VN(arg1), VN(arg2)) → VN(result)`, same as LVN but global
- In SSA, there are no kills to worry about — a name's value is fixed at its single definition point
- Process the dominator tree in preorder; at each block, look up or create value numbers for each instruction

Because SSA guarantees one definition per name, the value table built at a dominating block is always valid at dominated blocks — no need to check for intervening redefinitions.

### 4.3 SSCP — Sparse Simple Constant Propagation [EaC Ch9]

See Section 2.4 above for the full algorithm. Key advantages over non-SSA constant propagation:
- Operates on individual SSA names, not blocks → sparser graph
- Shallow semilattice (height 2 per name) → each name appears on worklist at most twice
- Linear time complexity O(number of uses) vs. potentially O(n²) or worse for dense block-level analysis [EaC Ch9]
- Naturally handles loops via optimistic initialization (⊤), propagating constants into loop bodies

**Full SCCP** (Sparse *Conditional* Constant Propagation — Wegman & Zadeck): Extends SSCP by simultaneously tracking reachability of CFG edges. When a branch condition evaluates to a constant, the false edge is marked unreachable and constants do not propagate through it. This discovers more constants (values that are constant only in reachable code paths). The implementation maintains two worklists: one for SSA names and one for CFG edges.

---

## 5. Loop Optimizations

Loops are the primary source of runtime cost in most programs. Loop optimizations have disproportionate impact. [EaC Ch8]

### 5.1 Loop Identification: Natural Loops, Dominators, Back Edges [EaC Ch9]

**Back edge**: A CFG edge `(n, h)` where `h` dominates `n` (h appears in DOM(n)). Back edges indicate loops.

**Natural loop** of a back edge `(n, h)`:
- The *header* is `h` (the only entry to the loop; `h` dominates all nodes in the loop body)
- The *loop body* consists of all nodes from which `n` is reachable without going through `h`
- Finding the natural loop: start with `{n, h}`; add all predecessors of non-header nodes not already in the set (backwards BFS from `n` stopping at `h`)

**Dominance** [EaC Ch9]: Node `bi` dominates node `bj` (`bi ≫ bj`) if every path from the entry node to `bj` contains `bi`.

**Computing DOM sets** (iterative forward dataflow):
```
DOM(n0) = {n0}
DOM(n) = N, for all n ≠ n0      (initialize to "everything")

repeat until stable:
    for each n ≠ n0:
        DOM(n) = {n} ∪ ∩_{m ∈ preds(n)} DOM(m)
```
Sets shrink monotonically; terminates in at most 2 iterations with RPO ordering on most reducible CFGs. [EaC Ch9]

**Immediate dominator (IDOM)**: The unique closest dominator of `n`. Encoded in the dominator tree: `n` is a child of `IDOM(n)`.

**Dominance frontier DF(n)** [EaC Ch9]: The set of nodes just beyond the region `n` dominates:
- `q ∈ DF(n)` iff (1) `n` dominates a CFG predecessor of `q`, and (2) `n` does not strictly dominate `q`

**Algorithm for DF sets** (Fig. 9.10 in EaC):
```
for all n: DF(n) = ∅
for all n with multiple predecessors:
    for each predecessor p of n:
        runner = p
        while runner ≠ IDOM(n):
            DF(runner) = DF(runner) ∪ {n}
            runner = IDOM(runner)
```

**Irreducible CFGs**: Graphs with loops that have multiple entry points (no single header that dominates the rest). The iterative DOM algorithm still works but may require more iterations. Some loop optimizations (LICM) require reducible loops. [EaC Ch9]

### 5.2 LICM — Loop-Invariant Code Motion [EaC Ch8]

**Goal**: Move computations whose value does not change across loop iterations out of the loop, into the loop pre-header (a block that dominates the header and has the header as its only successor).

**Definition of loop invariance**: An instruction `x ← f(a, b)` is loop-invariant if all of its operands `a`, `b` are defined outside the loop or have already been identified as loop-invariant.

**Conditions for safe hoisting** [EaC Ch8]:
1. `f` is side-effect-free (or can be safely moved)
2. All operands are loop-invariant
3. The block containing the instruction **dominates all loop exits** — this ensures the instruction would have been executed on every iteration anyway; hoisting it to the pre-header does not change whether it executes

**Algorithm**:
1. Mark values defined outside the loop as invariant
2. Iteratively: if all operands of an instruction are invariant, mark the instruction as invariant
3. Hoist all marked invariant instructions to the pre-header

**The domination hazard**: If an instruction does NOT dominate all loop exits (e.g., it is inside a conditional that may not execute), hoisting it introduces a new execution that may not have occurred in the original program. This is incorrect if the instruction can have side effects or if the loop might execute zero times.

**Interaction with exception semantics**: In languages with exceptions, hoisting a load or call out of a loop may cause an exception on a code path that would not have triggered it. Compilers must be conservative here.

### 5.3 Strength Reduction [EaC Ch8]

**Goal**: Replace expensive operations (multiply, divide) with cheaper equivalents (add, shift) by exploiting the inductive structure of loops.

**Induction variable**: A variable whose value is a linear function of the loop iteration count `i`: `v = a * i + b` for constants `a`, `b`.

**Classic strength reduction**:
```
// Before: multiply per iteration
for i in 0..n:
    x = a * i      // multiply

// After: introduce induction variable
x = 0
for i in 0..n:
    use x instead of a * i
    x += a         // add per iteration
```

**Linear Function Test Replacement (LFTR)**: When the loop exit test compares an induction variable to a bound (e.g., `i < n`), and `i` has been replaced by a derived induction variable `t` (e.g., `t = a * i`), the test can also be transformed: `i < n` becomes `t < a * n`. If `a * n` is loop-invariant, this eliminates the original induction variable entirely.

**Operator strength hierarchy**: multiply > divide > add/subtract > shift

### 5.4 Loop Unrolling [EaC Ch8]

**Goal**: Reduce loop control overhead (branch, counter update) by duplicating the loop body multiple times per iteration.

```
// Unroll factor 4 (conceptual transformation):
// Before:
for i in 0..n:
    body(i)

// After:
for i in 0..n step 4:
    body(i); body(i+1); body(i+2); body(i+3)
// Handle remainder: body(n - n%4), ...
```

**Benefits**:
- Reduces branch and counter-update overhead
- Exposes instruction-level parallelism within the unrolled body
- Enables better register allocation within the larger body
- Can eliminate copy operations between loop-carried variables (when the unroll factor matches the copy cycle length) [EaC Ch8]

**Full unrolling**: For loops with a *constant* trip count, the loop can be completely unrolled and eliminated.

**Costs** [EaC Ch8]:
- Code size increase — proportional to unroll factor
- Over-unrolling increases instruction cache (I-cache) pressure; if the unrolled body no longer fits in the I-cache, performance degrades
- Increases register pressure within the unrolled body

**Heuristic rule**: Unroll when the loop body is small relative to the I-cache line; choose an unroll factor that balances overhead reduction against code size growth. Profile-guided selection is more reliable than static heuristics.

---

## 6. Alias Analysis

### 6.1 Why Alias Analysis Matters for Optimization Safety [EaC Ch9]

Two pointers **alias** if they might refer to the same memory location. Without alias information, the compiler must assume that any store through a pointer can modify any load target — this blocks most optimizations that move or eliminate memory operations.

**Concrete example** [EaC Ch9]:
```c
x = y * z + 12;
*p = 0;
q = y * z + 13;    // Is y * z still available here?
```
`y * z` is a redundant computation, but only if `p` does not hold the address of `y` or `z`. Without aliasing information, the compiler must conservatively recompute `y * z`.

**Impact on LICM**: A load cannot be hoisted out of a loop if a store inside the loop might modify the loaded location. Alias analysis that proves the store and load access disjoint locations allows the hoist.

**Impact on register allocation**: Without alias analysis, a value enregistered from a memory location cannot safely be kept in a register across any pointer-based store. [EaC Ch9]

### 6.2 May-Alias vs. Must-Alias [EaC Ch9]

**May-alias**: Two pointers *might* refer to the same location. The compiler must assume they do when optimizing conservatively. Most alias analysis produces may-alias information.

**Must-alias**: Two pointers *always* refer to the same location. Can be used to propagate values more aggressively; less commonly computed.

**Conservative treatment of arrays** [EaC Ch9]: Without analysis of index values, `A[i,j]` is treated as a reference to all of `A`. A *use* of `A[i,j]` counts as a use of all of `A`; a *definition* of `A[m,n]` might kill any element, so for analysis purposes:
- For *must-kill* (e.g., removing a definition): `A[i,j]` does NOT kill `A` (can't be sure which element)
- For *may-kill* (e.g., preserving a definition): `A[i,j]` MIGHT kill any element of `A`

**Pointer imprecision** [EaC Ch9]: Without points-to analysis, an assignment `*p = v` must be treated as a potential definition of every variable that `p` might reach. Variables local to the current scope whose addresses have never been taken can be exempted from this conservative treatment.

**Procedure calls** [EaC Ch9]: Without summary information, a call site must be assumed to use and modify every reachable variable. Interprocedural MAYMOD and MAYREF sets (Section 9.2.4 of EaC) provide precise summary information that limits this over-approximation.

### 6.3 TBAA — Type-Based Alias Analysis [EaC Ch9]

**Key insight**: The C standard (§6.5) specifies that an object may only be accessed through an lvalue of a compatible type or a `char` type. Therefore, if two pointers have incompatible types, they cannot alias.

**Practical rule**: If `int *p` and `float *q` have incompatible types, the compiler may assume `*p` and `*q` do not alias, even if their addresses might numerically be equal. This is the *strict aliasing rule*.

**What TBAA enables**: Loads of one type can be freely hoisted across stores of an incompatible type. This is important for numerical code with mixed-type arrays.

**LLVM's TBAA metadata**: LLVM attaches TBAA metadata nodes to load/store instructions describing a type hierarchy. Two accesses cannot alias if their types are in unrelated branches of the hierarchy. [EaC Ch8 skill context]

**`restrict` keyword (C99)**: The programmer asserts that a pointer is the *only* pointer to a given memory region in the current scope. The compiler can then treat all aliasing through other pointers as impossible for that region — the strongest alias annotation available.

**Warning**: TBAA only applies when the code respects the strict aliasing rule. Type-punning via unions (in C) or `memcpy`-based punning is defined behavior; direct pointer casting to an incompatible type violates strict aliasing and produces undefined behavior that the compiler may exploit aggressively.

---

## 7. Pass Ordering and Interactions

### 7.1 Why Pass Ordering Matters [EaC Ch8]

Optimization passes create opportunities for each other. Running them in the wrong order can miss transformations or require more iterations.

**Classic interaction example 1: Inlining + Constant Propagation**
- After inlining a callee with a constant argument, the inlined body often contains computations on that constant
- Running constant propagation *after* inlining discovers these constant values and further simplifies the code
- Running constant propagation *before* inlining does not see inside the callee

**Classic interaction example 2: Constant Propagation + DCE**
- Constant propagation discovers that a conditional branch always takes one direction
- DCE (or SCCP's unreachable-branch elimination) then removes the dead branch
- This removes code that might have blocked other optimizations

**Classic interaction example 3: CSE + DCE**
- CSE replaces redundant computations with copies
- DCE removes the now-dead original computations (whose result was copied but the original expression is no longer needed)

**Classic interaction example 4: LICM + Strength Reduction**
- LICM hoists loop-invariant expressions
- Strength reduction transforms loop-dependent multiplications to additions
- LICM should run first to shrink the set of expressions that strength reduction needs to handle

### 7.2 Fixed-Point Iteration for Dependent Passes [EaC Ch8]

When two passes create opportunities for each other, they should be run in a fixed-point loop:

```
repeat:
    changed = false
    changed |= run_pass_A()
    changed |= run_pass_B()
until not changed
```

Common example: `instcombine` (peephole algebraic simplification + local DCE) and `copy propagation`:
- Copy propagation exposes new simplification opportunities
- Simplification can create new copies
- Run together until neither makes progress

### 7.3 Classic Pass Order [EaC Ch8, SKILL.md]

**Production pipeline order** (e.g., LLVM's `-O2`):
1. **instcombine / algebraic simplification** — cheap, high payoff, creates opportunities; cleans up IR from frontend
2. **Inlining** — exposes constants, removes call overhead, creates larger optimization contexts
3. **instcombine again** — clean up after inlining (inlined code often has obvious simplifications)
4. **SCCP / constant propagation** — propagate constants created by inlining
5. **GVN** — global CSE on the simplified IR
6. **LICM** — hoist loop-invariant expressions (do after inlining so inner loops are visible)
7. **DCE** — clean up dead code left by propagation and constant folding
8. **Register allocation and code generation**

**For simpler compilers** (fixed pipeline) [EaC SKILL.md]:
1. Constant folding + copy propagation
2. DCE
3. Inlining (optional)
4. LICM (optional, if loops are present)
5. DCE again

**Inlining order** [EaC Ch8]: Inline callees before callers (bottom-up call graph traversal). This ensures the callee has been optimized in isolation before it is inlined, avoiding re-optimization of the same callee body at each call site.

### 7.4 Analysis Caching and Invalidation

Analyses are expensive. Cache them and invalidate only when the IR changes:
- **Transform passes** declare which analyses they require (will trigger computation if absent) and which they invalidate (will mark cached results as stale)
- **Analysis passes** compute information and store it; subsequent passes can read cached results
- When a pass modifies the IR in a way that preserves a particular property (e.g., a DCE pass that only removes dead instructions cannot change liveness), it can preserve the analysis result instead of invalidating it

---

## 8. Dominance Frontiers and SSA Construction (Detailed)

### 8.1 Dominance Frontier Definition [EaC Ch9]

`q ∈ DF(p)` if and only if:
1. `p` dominates a CFG predecessor of `q` (there exists `x` such that `(x, q)` is a CFG edge and `p ∈ DOM(x)`)
2. `p` does NOT strictly dominate `q` (`p ∉ DOM(q) - {q}`)

Informally: `q` is one CFG edge beyond the region that `p` dominates.

**Significance for φ-insertion**: A definition of `a` in block `n` forces a φ-function for `a` at the head of each block in `DF(n)`. The inserted φ-function is itself a new definition, so this process may cascade (the φ-function's block also has a DF).

### 8.2 Semipruned SSA Construction [EaC Ch9]

**Step 1**: Find global names (names live in multiple blocks):
```
Globals = ∅
for each block b:
    VARKILL = ∅
    for each operation "x ← y op z" in b:
        if y ∉ VARKILL: Globals = Globals ∪ {y}
        if z ∉ VARKILL: Globals = Globals ∪ {z}
        VARKILL = VARKILL ∪ {x}
        Blocks(x) = Blocks(x) ∪ {b}
```

**Step 2**: Insert φ-functions using the worklist + DF:
```
for each name x ∈ Globals:
    WorkList = Blocks(x)
    for each block b ∈ WorkList:
        remove b from WorkList
        for each block d ∈ DF(b):
            if d has no φ-function for x:
                insert φ-function for x in d
                WorkList = WorkList ∪ {d}
```

**Step 3**: Rename using preorder dominator-tree walk:
- Maintain a counter and a stack per global name
- At each definition: generate new SSA name (counter++, push to stack)
- At each use: rewrite with top of stack
- At each φ-function argument: fill in the SSA name from the appropriate predecessor's stack
- On exit from a block: pop all names defined in that block from their stacks

---

## 9. Dataflow Analysis Limitations [EaC Ch9]

Understanding what dataflow analysis *cannot* do is as important as understanding what it can:

1. **All paths assumed feasible**: Standard dataflow analysis takes union/intersection over ALL successors/predecessors, assuming every CFG edge can be taken. If a branch condition is never true, the analysis still includes its effects. SCCP's conditional reachability tracking addresses this but requires more machinery.

2. **Arrays**: `A[i,j]` is treated as a reference to all of `A` because the compiler cannot determine specific indices without value analysis. This introduces imprecision — kills of specific array elements are not tracked.

3. **Pointers**: Without points-to analysis, a store `*p = v` may define any variable reachable by `p`. This can force the compiler to assume every variable has been modified, blocking propagation across pointer writes.

4. **Procedure calls**: Without summary information (MAYMOD, MAYREF sets), every call must be treated as using and modifying every reachable variable. [EaC Ch9]

5. **Precision bound**: The information is precise "up to symbolic execution" — the analysis is correct for all possible executions but may be overly conservative if the code has infeasible paths.

---

## 10. Summary of Key Equations

| Analysis | Direction | Meet Op | Initial Values | Local Sets |
|----------|-----------|---------|----------------|------------|
| Dominance (DOM) | Forward | Intersection | DOM(n0)={n0}; DOM(n)=N | — (structural only) |
| Liveness (LIVEOUT) | Backward | Union | LIVEOUT(n)=∅ | UEVAR, VARKILL |
| Available Exprs (AVAILIN) | Forward | Intersection | AVAILIN(n0)=∅; AVAILIN(n)=all | DEEXPR, EXPRKILL |
| Reaching Defs (REACHES) | Forward | Union | REACHES(n)=∅ | DEDEF, DEFKILL |
| Anticipable Exprs (ANTOUT) | Backward | Intersection | ANTOUT(nf)=∅; ANTOUT(n)=all | UEEXPR, EXPRKILL |

**Pattern**: When the analysis asks "is a property true on *all* paths?", use **intersection** and initialize to "everything true". When it asks "is a property true on *some* path?", use **union** and initialize to "nothing true".
