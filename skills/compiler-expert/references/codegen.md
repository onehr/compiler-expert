# Code Generation: Instruction Selection, Register Allocation, and Bytecode VMs
## Sources: Engineering a Compiler, Cooper & Torczon (3rd ed.), Chapters 11 & 13 [EaC]; Crafting Interpreters, Robert Nystrom [CI]

---

## Part I: Instruction Selection (Chapter 11) [EaC]

---

### 1. The Core Problem

Instruction selection rewrites IR code into target machine assembly. The difficulty comes from:

- **Multiple implementations** of every IR operation (e.g., a register copy can be done with `i2i`, `addI r,0`, `subI r,0`, `orI r,0`, etc.)
- **Address mode explosion**: ISAs encode common address computations (base+offset, base+index, immediate) as single operations, requiring the selector to recognize multi-operation IR patterns
- **Abstraction mismatches**: IR may be more abstract (need to elaborate) or less abstract (need to combine) than the ISA
- **Cost variation**: operations have different latencies, and the right choice depends on context

The instruction selector runs at compile time. Its output may still use virtual registers; the register allocator completes the mapping to physical registers later. [EaC]

**Contrast with bytecode VMs** [CI]: When targeting a bytecode VM rather than native code, instruction selection is dramatically simpler. The bytecode IS the target ISA — you design it to be easy to emit. There are no address modes, no physical registers, no condition codes to track. The tradeoffs shift: bytecode buys portability and simplicity at the cost of performance (interpreted execution adds overhead vs. native), but a well-designed VM recovers much of that through cache-friendly instruction encoding. See Part III for the CI bytecode VM design.

---

### 2. Ad-Hoc Matching [EaC]

**What it is**: A hand-coded selector that performs a simple treewalk (for tree IR) or linear scan (for linear IR), emitting a fixed template for each IR operation.

**When it is appropriate**:
- Target ISA is very small (e.g., the DEC PDP/11 in the BLISS-11 compiler era)
- Compile-time speed dominates code quality
- Prototype or educational compiler

**Problems**:
- Produces uniform, template-like code; ignores context
- Cannot exploit address modes without explicitly tracking address expressions
- Cannot use immediate-mode operations without tracking constant values
- Offers no support for retargeting: changing the target requires rewriting the matcher

**Verdict**: Avoid for any compiler where code quality or portability matters. Ad-hoc matching is the baseline, not the goal.

---

### 3. Selection via Peephole Optimization

#### 3.1 Architecture [EaC]

A peephole selector breaks instruction selection into three passes:

1. **Expander**: Translates each IR operation into a sequence of LLIR (Low-Level IR) operations that make all effects explicit. This is a template-driven pass; it ignores context. It produces more operations than the final output.

2. **Simplifier**: Slides a window over the LLIR and applies four transformations:
   - Forward substitution (fold a definition into its use)
   - Algebraic simplification (eliminate `x + 0`, `x * 1`, etc.)
   - Constant folding (evaluate constant expressions)
   - Dead-code elimination (remove operations whose results are unused)

   The simplifier is the heart of the approach. It removes inter-operation inefficiencies that earlier passes missed because they operated at a higher abstraction level.

3. **Matcher**: Maps the simplified LLIR onto target ISA operations. Technologies used include hand-coded matchers, LR parsers treating LLIR as a linear form, and tree-pattern matchers over the LLIR.

#### 3.2 Window Size [EaC]

**Physical window**: Contains adjacent LLIR operations. Simple to implement.
- A window of 3-4 operations is typical.
- Window size directly bounds algorithmic complexity: larger windows increase time cost.
- Small windows miss opportunities when related operations are separated by unrelated code.

**Logical window**: Contains operations linked by value flow (def-use chains), even if not adjacent. More powerful.
- Handles modern IRs that interleave independent computations for ILP.
- The simplifier follows use-def links to find operations that operate on the same value.
- Can be extended to span multiple blocks using reaching definitions analysis, with caution about multiple definitions reaching a single use.
- Restricting to extended basic blocks avoids complex multi-def/multi-use cases.

**Key insight**: The logical window is superior for modern parallel code. The physical window is simpler to implement and sufficient for straight-line code. [EaC]

**Single-pass compiler note** [CI]: In a single-pass compiler that emits bytecode directly during parsing (no AST), peephole-style optimizations can be applied inline during emission. The "window" is effectively the compiler's local emit buffer. CI's clox applies constant folding at parse time (constant expressions folded before emission) and simple backward-looking peephole rewrites (e.g., negation of the most recently emitted constant becomes a new constant rather than a negate instruction). This is a restricted but zero-overhead form of peephole optimization.

#### 3.3 What Patterns to Target [EaC]

Priority targets for peephole optimization:
- **Store followed by load from same address**: replace load with copy
- **Arithmetic identities**: `sub r, 0 => r`, `mult r, 1 => r`, `add r, 0 => r`
- **Address mode folding**: forward-substitute `r2 = @CP; r3 = @G; r4 = r2 + r3` into `loadAI r_CP, @G` or similar
- **Constant folding into immediate-mode ops**: fold `r1 = 2; r9 = r1 * r8` into `multI r8, 2 => r9`
- **Condition code dead values**: If `cc = f_mul(a,b)` is dead, eliminate it to enable a multiply-add fusion
- **Jump-to-jump**: `jmp L1; L1: jmp L2` → `jmp L2`

#### 3.4 Dead Value Recognition [EaC]

Dead-value detection is critical to the simplifier's effectiveness. Approaches:
- Compute `LIVEOUT` sets per block, then track live values in a backward pass; expensive but precise
- Simpler: track global names (used in more than one block) and treat all as live. Names introduced by the expander are local by construction, so this works well for them.
- Mark last uses in LLIR during bottom-up expansion to inform forward simplification

#### 3.5 Strengths and Weaknesses of Peephole Selection [EaC]

**Strengths**:
- Systematic late-stage cleanup removes inefficiencies introduced at any earlier stage
- Forward substitution exposes address-mode opportunities automatically
- Supported by tools (GCC uses RTL + peephole selector)

**Weaknesses**:
- No explicit cost model: no optimality claim can be made
- Opportunities visible only within the window are missed
- Generates code with the same *effects* as the IR, not a literal translation — harder to reason about

---

### 4. Selection via Tree-Pattern Matching [EaC]

#### 4.1 The Core Idea

Both the IR and the target ISA are expressed as trees. A **rewrite rule** maps an IR subtree onto a code template with a cost:
```
Reg -> +(Reg1, T3_2)   cost=1   emit: addI r1, T3 => rnew
```

The selector **tiles** the IR tree with these operation trees. A tiling is valid if:
- Every AST node is covered
- Pattern roots overlap with leaves of their parents
- Overlaps are type-compatible
- Each overlap involves exactly one node

Code is generated bottom-up from the tiling's templates.

#### 4.2 Rule Set Design [EaC]

**Start with minimal coverage**: Create rules that handle every possible node type. Then add specialized rules for complex operations.

**Key restriction — single-operator rules**: Each pattern should contain at most one operator. Multi-operator patterns can be decomposed into single-operator rules using auxiliary nonterminals (e.g., `T1`, `T2` for address computations). This simplifies the matcher and keeps it purely local.

**Cost structure with decomposed rules**: Assign costs to individual rules that sum to the total cost of the complex instruction they collectively describe.

**Leaf nodes in singleton rules**: Leaves (constants, labels, values) should appear only in rules of the form `Class -> Leaf`. This allows the matcher to annotate leaves before processing operators, keeping operator-matching code uniform.

**Ambiguity is expected and correct**: The same AST subtree legitimately matches multiple rules (e.g., `+(Reg, T3)` matches both an `addI` rule and an address-computation rule). The cost model drives the choice.

#### 4.3 The Tiling Algorithm (BURS / Tree Matching) [EaC]

The `Tile` algorithm performs a single bottom-up (postorder) pass over the AST:

1. For each leaf node, precompute the set of matching rules (one entry per LHS symbol class).
2. For each interior node, recurse on children first. Then for each rule whose operator matches the current node, check compatibility with the children's match sets. Add the rule to `Match(n, class(r))` if compatible.
3. Track costs: `NewCost = RuleCost(r) + sum of children's best costs for the required classes`. Keep the minimum-cost rule per class per node.

The result is **local optimality**: at each node, the selector makes the least-cost choice given the subtree below it. This is not global optimality, but it is fast and practical.

#### 4.4 Tools: Hand-coded vs. Generated [EaC]

| Approach | Mechanism | Best For |
|---|---|---|
| Hand-coded matcher | Explicit rule checks in postorder walk | Small ISAs, compact output |
| BURS (Bottom-Up Rewrite System) | Table-driven finite tree automaton | Performance, large rule sets |
| Parser-based (ambiguous grammar) | LR/Earley parser on linearized tree | When grammar tooling is available |
| String matching | Linearize tree to prefix form, apply string matcher | Alternative to automata-based |

All four produce fast, effective selectors. The principal difference is in the cost model: some tools require fixed costs per rule (enabling offline precomputation), others allow dynamic cost variation (more flexibility, higher compile-time cost).

#### 4.5 Peephole vs. Tree-Pattern Matching — When to Use Each [EaC]

| Criterion | Peephole | Tree-Pattern Matching |
|---|---|---|
| IR shape | Linear IR | Tree-shaped IR |
| Cost model | None (no optimality claim) | Explicit cost → local optimality |
| Context for constants | Limited by window size | Carries constant type in match sets |
| CISC / complex address modes | Good (forward substitution finds them) | Good (address nonterminals in rules) |
| Retargetability | Tool support (GCC/VPO) | Tool support (BURS tools, Twig) |
| Code produced | Same effects as IR (not literal) | Implements IR computation |

**Choose peephole** when: the IR is linear, you want to clean up inefficiencies from prior phases, or your toolchain (e.g., GCC-family) already provides RTL infrastructure.

**Choose tree-pattern matching** when: the IR is tree-shaped (AST or close), you need explicit cost-driven selection, or you need to handle constant-in-immediate-field decisions that depend on subtree type information that a physical window cannot see.

Both approaches are supported by specification-driven tools that reduce retargeting effort.

---

### 5. ISA Design Impact on Selection Strategy [EaC]

| ISA Feature | Effect on Selection |
|---|---|
| RISC (load-store, few address modes) | Simpler selection, fewer alternatives; register allocation becomes more critical |
| CISC (memory operands, many address modes) | More pattern alternatives; automated tree matching or peephole essential |
| Duplicate implementations (e.g., copy via add+0) | Selector must enumerate and cost all alternatives |
| Immediate-field size restrictions | Rules must distinguish NUM vs. CON vs. LAB; window must carry this forward |
| Condition code side effects | LLIR must represent CC explicitly; dead CC must be eliminated to unlock fused ops |
| High-level ops (multiply-add, string move, call) | Selector must synthesize or recognize multi-operation sequences |
| Register use restrictions (x86 AX/DX, ARM overlap) | Selector may need coroutine with allocator for partial pre-allocation |

**Key principle**: The more complex the ISA, the more critical automated (tool-based) instruction selection becomes. CISC complexity drove the development of both peephole-based and tree-matching tools.

---

### 6. Selection, Scheduling, and Allocation Interactions [EaC]

- Selection determines what operations exist and their latencies — directly shapes the scheduling problem.
- If two implementations of an IR operation use different functional units, the scheduler's needs may influence which to pick; this requires integration or iteration.
- With uniform register sets: selector assumes unlimited virtual registers; allocator inserts spill code later. This is the clean, decoupled design.
- With non-uniform / restricted register sets: selector may need to pre-assign specific physical registers (e.g., IA-32 multiply), which predetermines allocation decisions and complicates both phases.
- **Guideline**: Keep selection, scheduling, and allocation separate to the extent possible. Merge only when the register restrictions of the ISA force it.

---

## Part II: Register Allocation (Chapter 13) [EaC]

---

### 1. The Core Problem

The allocator maps virtual registers (VRs) onto physical registers (PRs). It must:
- Decide which values reside in registers vs. memory at each program point
- Insert loads and stores ("spill code") to enforce those decisions
- Minimize runtime cost of spill code

The problem is NP-complete in general. All practical allocators compute good approximate solutions in O(n) to O(n²) time. [EaC]

**Stack-based VM contrast** [CI]: A bytecode stack machine has no register allocation problem at all — values are implicitly managed on a runtime value stack. The compiler's job is to emit push- and pop-style instructions in the correct order. The stack slot for each value is determined by expression evaluation order; no coloring or spill analysis is needed. This is the key compile-time simplicity win of stack machines over register machines: the "register allocator" is trivially the expression evaluation order. See Part III for details.

---

### 2. Live Ranges [EaC]

A **live range (LR)** is the closed set of definitions and uses of a single value. The allocator operates on LRs rather than VR names because:
- A VR may be written multiple times, creating independent values that can each go to different PRs
- Two LRs with non-overlapping lifetimes can share a PR

**Local LR**: single definition, extends to last use within a block — an interval.

**Global LR**: all definitions that reach a use, plus all uses those definitions reach. Spans the CFG as a web. SSA form makes construction straightforward: merge all names at each φ-function using union-find, yielding maximal global LRs.

**SSA-name LRs**: Each SSA name is its own LR. Produces chordal interference graphs, enabling optimal coloring in polynomial time. Requires out-of-SSA translation afterward.

**Linear-scan interval approximation**: LR of value v is the interval [i, j] spanning all operations where v is live, ignoring control flow. May overestimate the true LR. Produces interval graphs colorable in linear time.

---

### 3. Local Register Allocation [EaC]

#### 3.1 When to Use

- Single basic blocks
- JIT compilers on a tight compile-time budget
- As a sub-problem within other allocators
- When MAXLIVE ≤ k (no spilling needed — local allocation is trivially optimal in this case)

#### 3.2 Best's Algorithm

**Principle**: When a PR is needed and all are occupied, spill the PR whose current value has the farthest next use.

**Implementation**:
1. Backward pass: assign VR names (one per live range), record next-use distances (`NU`) at each operand.
2. Forward pass: for each operation, allocate PRs to uses, free PRs whose last use just occurred, allocate PRs to definitions.
3. `GetAPR`: pop free PR from stack; if none, scan `PRNU` array for the PR with the largest next-use distance and spill it.

**Optimality**: If all spill and restore costs are uniform, Best's algorithm is optimal for a block.

**With non-uniform costs**: The problem becomes NP-hard when dirty, clean, and rematerializable values are mixed. Practical heuristic: prefer spilling lower-cost values (clean before dirty, rematerializable before clean) when next-use distances are similar.

---

### 4. Global Register Allocation via Graph Coloring [EaC]

#### 4.1 Pipeline Overview

```
Find Live Ranges
    ↓
Build Interference Graph
    ↓ (iterate until no more coalescable copies)
Coalesce Copies ──────────────┐
    ↓                         │ rebuild graph
Estimate Spill Costs          │
    ↓                         │
Color (Simplify + Select)     │
    ↓                         │
[k-coloring found] → rewrite code, done
    ↓ [uncolored nodes]
Insert Spills → restart
```

Typically converges in 1-3 iterations.

#### 4.2 Building the Interference Graph

- Create a node per LR.
- Walk each block backward; maintain LIVENOW.
- At each definition of LRi, add edge (LRi, LRj) for every LRj in LIVENOW — except if the operation is a copy.
- **Copy operations do not create interferences**: after a copy `LRi ← LRj`, both have the same value and could occupy the same PR.
- Use both a bit-matrix (O(1) interference test) and adjacency lists (efficient neighbor iteration).
- Build separate graphs for disjoint register classes (GPR, FPR) to reduce graph size.

**Building the graph dominates overall allocation cost.**

#### 4.3 Coalescing

**What it is**: If LRi and LRj are connected by a copy and do not otherwise interfere, combine them into one LR and eliminate the copy.

**Why it matters**:
- Eliminates copy operations (smaller, faster code)
- Reduces degree of nodes that interfered with both LRi and LRj
- Makes the coloring problem easier

**Conservative coalescing (Briggs)**: Only coalesce LRi and LRj if the resulting LRij satisfies:
- `LR°ij ≤ MAX(LR°i, LR°j)` (degree doesn't increase), OR
- LRij has fewer than k neighbors with degree ≥ k (the combined LR will still color)

**Benefit of conservative coalescing**: Cannot make the coloring problem worse. May leave some copies uncoalesced that aggressive coalescing would remove.

**Aggressive coalescing (Chaitin)**: Coalesce whenever LRi and LRj do not interfere. Can increase degree of the result and occasionally prevent colorings that would otherwise succeed. Faster convergence, riskier.

**Biased coloring (deferred coalescing)**: Do not coalesce during graph construction. Instead, during Select, when assigning a color to LRi, prefer colors already assigned to LRs connected to LRi by copies. Eliminates copies without changing the graph. Low overhead, good results.

**Iterated coalescing**: Interleave Simplify and coalescing. After all trivially-colored nodes are removed, attempt conservative coalescing again; copies that were unsafe to coalesce in the full graph may be safe in the reduced graph.

**Order of coalescing**: Coalesce more frequently executed copies first (innermost loop blocks first). The order affects which LRs get combined and can affect final code quality.

#### 4.4 Spill Cost Estimation

For each LR, compute the estimated runtime cost of spilling it everywhere:

```
spill_cost(LR) = Σ_defs [store_cost(d) × freq(block(d))]
               + Σ_uses [load_cost(u) × freq(block(u))]
```

**Execution frequency**: Use profile data if available; otherwise, assume each loop executes 10 times, giving weight `10^d` for nesting depth `d`. An unpredictable branch halves the frequency.

**Value classifications**:
- **Dirty value**: has been computed, not yet in memory. Spill costs one store; restore costs one load.
- **Clean value**: value already exists in memory (prior spill or known memory location). Spill costs zero; restore costs one load.
- **Rematerializable value**: cheaper to recompute than to load. Classic case: immediate load constant. Spill costs zero; restore is a `loadI` (cheaper than a memory load).

**Special cases**:
- **Negative spill cost**: LR containing only a load and store to the same address, or a copy that is more expensive than the spill. Should always spill.
- **Infinite spill cost**: LR so short that spilling it inserts two operations (store + load) without freeing any register in between. Do not spill; mark as `cost = ∞`.

#### 4.5 Coloring: Simplify and Select

**Simplify** (build ordering stack):
```
while graph not empty:
    if node n with n° < k exists:
        remove n, push onto stack   # trivially colorable
    else:
        pick node by spill metric, remove and push  # may not color
```

**Select** (assign colors):
```
while stack not empty:
    pop node n, reinsert n and its edges
    assign n a color not used by any current neighbor
    if no color available: n is uncolored (will be spilled)
```

**Why it works**: A node removed because `n° < k` was removed from a smaller graph; when reinserted, it still has fewer than k neighbors in the current (smaller) graph, so it will find a color. Nodes removed by the spill metric may or may not find a color in Select — if their neighbors use fewer than k distinct colors, they color; this often happens in practice.

**Spill metrics** (used to order constrained node removal in Simplify):
- `cost / degree`: balance spill cost against impact on neighboring nodes. The original PL.8 allocator.
- `cost / degree²`, `cost / area`, `cost / area²`: variants weighting local impact more heavily.
- Straight `cost`: focus purely on runtime performance.
- Total spill operations: minimize code-space growth.

No single metric dominates. Since coloring is fast relative to graph building, try multiple metrics and keep the best result.

#### 4.6 Spill Code Insertion

When LRs remain uncolored after Select:
- Insert a **store after every definition** of the LR
- Insert a **load before every use** of the LR
- This "spill everywhere" discipline breaks the uncolored LR into tiny LRs (one per reference), which are trivially colorable

These tiny LRs need their own PRs; the next coloring iteration handles them. Typically 1-2 additional iterations suffice.

**Clean spilling optimization**: In blocks where register pressure is low, avoid redundant restores. Keep the value in a PR for its local sub-range, restoring only once per block.

**Register scavenging**: A postpass that identifies unused registers in regions and promotes spilled values back into them.

---

### 5. Coalescing and Spilling Tradeoffs Summary [EaC]

| Decision | Aggressive | Conservative |
|---|---|---|
| Coalescing | Combine all non-interfering copies | Only combine if result does not increase colorability difficulty |
| Risk | May prevent colorings, increase spills | Leaves some copies uncoalesced |
| Typical outcome | More copies eliminated, occasionally more spills | Fewer copies eliminated, more predictable spill behavior |

| Spill Strategy | Granularity | Tradeoff |
|---|---|---|
| Spill everywhere | Entire LR | Simple; overspills in low-pressure regions |
| Interference-region spilling | Only in high-pressure region | Reduces total spill code; more complex |
| Live-range splitting | Split LR at chosen points | Can isolate pressure; split points introduce copies |
| Zero-cost splitting | Use schedule NOP slots | No added cost; depends on scheduling |
| Passive splitting | Directed interference graph analysis | Principled choice between split and spill |

---

### 6. Rematerialization [EaC]

**When**: The value can be recomputed more cheaply than loading it from memory.

**Canonical case**: Constants defined by `loadI`. Spill = free; restore = another `loadI`, which is typically 1 cycle vs. 3+ cycles for a cache-hit load.

**General case**: Any operation whose inputs are all available at each use point. The allocator propagates rematerialization tags using a variant of SSCP. Key rules:
- Only combine SSA names with identical rematerialization tags into one LR
- Use conservative coalescing to avoid merging LRs with different tags
- Spill cost estimate must reflect the actual recomputation cost, not a load cost
- Spill code insertion emits the recomputation expression, not a memory store/load

---

### 7. Linear Scan Allocation [EaC]

#### 7.1 How It Works

1. Compute live information; represent each value v as an interval `[i, j]` (overestimate is acceptable).
2. Sort intervals by start point.
3. Scan left to right; allocate free PRs greedily. If no free PR, spill the interval with the largest endpoint.

The algorithm is essentially Best's local algorithm applied to an interval approximation of the whole procedure.

#### 7.2 When to Prefer Linear Scan over Graph Coloring

- **JIT compilation** (Java HotSpot, early LLVM JIT): compile-time budget is tight; graph construction dominates coloring allocator cost; linear scan is O(n log n) vs. O(n²) for graph coloring.
- **Small procedures where MAXLIVE < k**: no spilling needed; linear scan is effectively optimal.
- **Simple targets**: when the ISA has few address modes and register restrictions, the approximation loss from interval overestimation is acceptable.

#### 7.3 Limitations

- Ignores control flow: intervals can overestimate true live ranges, causing unnecessary spills
- Coalescing is limited to the simple case where one interval ends exactly where another begins (copy at block boundary)
- Live-range splitting is complex to integrate correctly

#### 7.4 Linear Scan vs. Graph Coloring

| Criterion | Linear Scan | Graph Coloring |
|---|---|---|
| Compile time | O(n log n) | O(n²) for graph build; O(n) coloring |
| Code quality | Lower (interval overestimate) | Higher (precise interference) |
| Coalescing | Limited | Powerful |
| Spilling precision | Spill by interval end | Spill by cost metric |
| JIT suitability | Excellent | Marginal (too slow for hot paths) |
| Correctness of interference | Approximate (may over-constrain) | Precise |

---

### 8. SSA-Based Allocation [EaC]

#### 8.1 Key Property

If each SSA name is treated as its own live range, the resulting interference graph is a **chordal graph**. Chordal graphs can be optimally colored in O(|N| + |E|) time — no NP-hardness.

#### 8.2 Benefits

- Optimal coloring may use fewer registers than the greedy heuristic on global LRs
- SSA-name LRs are shorter than maximal global LRs; this gives the allocator finer-grained spill placement (spill over a loop rather than across the whole procedure)
- Live-range construction is trivial (no φ-unification needed)

#### 8.3 Costs and Complications

- **Out-of-SSA translation is required after allocation**: this step inserts copy operations and may need an extra register to break copy cycles
- The inserted copies may not be coalescable (coalescing SSA copies would destroy the chordal property)
- SSA-based allocators need a different coalescing strategy that does not rely on the interference graph
- Spill choice and spill placement are still NP-hard; SSA form does not help here
- The net benefit vs. global-LR coloring is context-dependent and empirically unclear

#### 8.4 When to Use SSA-Based Allocation

- When the compiler already works in SSA form throughout (LLVM-style pipelines)
- When optimal register count matters more than compile time (embedded, code-size-critical targets)
- When the procedure has many loop-local values that a global LR would smear across the whole procedure

---

### 9. Register Classes and Overlapping Registers [EaC]

#### 9.1 Disjoint Classes (GPR vs. FPR)

Handle by building **separate interference graphs** per class. Reduces graph size significantly. Spills of FPR values may require GPR address registers; allocate FPR class first.

#### 9.2 Overlapping Classes (ARM, x86)

**ARM A-64**: Q/D/S/H share one 128-bit physical register. X/W share one 64-bit GPR. Each alias excludes exactly one other register, so degree arithmetic works normally.

**IA-32**: EAX/AH/AL overlap asymmetrically. AH and AL can coexist; using EAX precludes both. An EAX neighbor reduces the 8-bit register pool by two. Degree-based colorability estimation breaks.

**Solution (Smith-Ramsey-Holloway estimator)**:
- Represent each class as a set of register names; define `alias(r)` = set of registers sharing physical space with r.
- Precompute the number of physical registers a given neighbor of a given class consumes.
- Replace `n° < k` in Simplify with a conservative colorability test that correctly counts aliased consumption.

**Assignment with overlapping classes**:
- Choosing EAX when AL or AH is already used may be "free" (no additional constraint). Choosing CL when AH is free would make ECX unavailable.
- First-in-first-out register assignment is incorrect; the allocator must account for aliased conflicts in assignment.

**Coalescing with overlapping classes**:
- Only coalesce LRi and LRj if `class(LRi) ∩ class(LRj)` is non-empty and the resulting LR has a feasible (non-empty) register class.

#### 9.3 Forced Register Placement

**Restrictions** (LR must go in specific PR): Create a singleton register class for that LR. The coloring mechanism handles it automatically.

**Exclusions** (LR cannot use specific PRs): Build an exclusion set per LR; Select tests candidates against it. Useful for caller-saves regions where caller-saves PRs must not hold live values.

**Linkage conventions**: Pass each actual parameter as a copy to the PR designated by the calling convention. The interference graph captures the constraint; coalescing can eliminate the copy.

---

### 10. Practical Decision Framework Summary [EaC]

#### Choosing an Allocation Strategy

```
Is compile-time budget very tight? (JIT, interpreter overhead)
  YES → Linear scan
  NO  → Is SSA already available and code quality critical?
          YES → SSA-based allocation (chordal coloring)
          NO  → Global graph-coloring allocator (Chaitin-Briggs)

Is the scope a single basic block?
  YES → Local allocator (Best's algorithm)
```

#### Choosing a Coalescing Strategy

```
Conservative coalescing: use when spill quality matters and you cannot
  afford unexpected increases in register pressure.

Aggressive coalescing: use when copies are the dominant concern and
  you are willing to risk slightly higher spill rates.

Biased coloring: lowest risk; good default when coalescing is hard to
  analyze statically.

Iterated coalescing: best quality; higher compile-time cost; use when
  building an optimizing compiler targeting high performance.
```

#### Choosing What to Spill

```
1. Never spill LRs with infinite cost (no register freed between def and use).
2. Always spill LRs with negative cost (spilling removes more ops than it adds).
3. Among remaining candidates: prefer lower spill cost relative to degree.
   Metric: cost / degree (standard), or cost / degree² (for high-pressure graphs).
4. Prefer spilling clean or rematerializable values over dirty values when
   next-use distances are close.
5. Try multiple spill metrics; coloring is cheap relative to graph construction.
```

#### Rematerialization Checklist

```
Tag as rematerializable if:
- Defined by a loadI (constant load): spill=free, restore=loadI
- Defined by an operation whose operands are all invariant and available at every use
- The recomputation cost is ≤ a load from memory (typically 1 cycle vs. 3+ cycles)

Propagate tags over SSA using SSCP.
Do not combine LRs with different tags.
Use conservative coalescing to protect tag purity.
```

---

## Part III: Bytecode VM Design and Implementation [CI]

*Source: Crafting Interpreters (clox chapters 14–30), Robert Nystrom*

---

### 1. Why Bytecode? The Three-Way Tradeoff [CI]

The three principal execution models for dynamic languages occupy a spectrum:

| Approach | Portability | Speed | Simplicity |
|---|---|---|---|
| AST tree-walk interpreter | High | Low (pointer chasing, bad cache behavior) | Very high |
| Bytecode VM (interpreted) | High | Medium (~50-100x faster than tree-walk) | High |
| Native code compilation | Low (per-ISA backend) | High | Low (register allocation, scheduling, ISA complexity) |

**Tree-walk problems** [CI]: An AST node for `1 + 2` allocates multiple heap objects with pointers between them. Walking them during interpretation thrashes the CPU cache. jlox (the tree-walk interpreter) runs Fibonacci(40) in approximately 72 seconds; clox (bytecode VM) runs the same benchmark roughly two orders of magnitude faster.

**Bytecode sweet spot** [CI]: Dense, linear instruction sequences keep cache lines full. The VM is a simple loop: fetch-decode-execute. No native instruction scheduling or register allocation needed. The tradeoff is an extra layer of interpretation overhead vs. native code.

**Decision guidance** [CI, EaC]: If targeting a specific high-performance ISA and willing to write one backend per platform, go native (EaC's domain). If targeting a portable, embeddable VM where development speed and portability matter, bytecode (CI's domain). LLVM provides a third path: compile to LLVM IR and let LLVM handle native backends.

---

### 2. Chunk Format: Bytecode Layout [CI]

A **chunk** is the bytecode container for one compilation unit (top-level script or function):

```c
typedef struct {
    int count;
    int capacity;
    uint8_t* code;        // bytecode instruction stream
    int* lines;           // source line numbers, parallel to code[]
    ValueArray constants; // constant pool
} Chunk;
```

**Key design points** [CI]:
- The `code` array is a flat, densely packed byte array. Instructions start at byte 0; operands follow immediately after the opcode byte.
- The `lines` array is parallel to `code`: `lines[i]` is the source line number of `code[i]`. This enables error messages to show source locations without storing debug info in the instruction stream itself.
- The `constants` array is the chunk's **constant pool**: literals (numbers, strings, later function objects) are stored here and referenced by index from `OP_CONSTANT` and similar instructions.
- Chunks grow dynamically using a doubling strategy (same as dynamic arrays): initial capacity 8, doubles on overflow.

**Constant pool indexing** [CI]: The one-byte `OP_CONSTANT` operand limits constant pools to 256 entries. For larger programs, `OP_CONSTANT_LONG` uses a 3-byte operand. This is a direct encoding size tradeoff: compact common case, escape hatch for outliers.

**Contrast with EaC** [EaC, CI]: EaC targets x86-64 native code where the "constant pool" does not exist — constants fold into immediate fields or are placed in `.rodata`. In a bytecode VM, all constants must be explicitly stored and loaded because the VM has no immediate-field encoding.

---

### 3. Opcode Design Principles [CI]

clox's opcode set is minimal and stack-oriented. Key design principles:

**One opcode per semantic operation, not per type** [CI]: There is no `OP_ADD_INT` vs. `OP_ADD_FLOAT`. A single `OP_ADD` checks operand types at runtime. This sacrifices some performance for simplicity; JIT compilers (HotSpot, V8) add type-specialized opcodes later as an optimization.

**Stack depth is implicit** [CI]: Every instruction knows how many values it pops and pushes. The compiler tracks the current stack depth statically; no operand needed. `OP_ADD` always pops 2, pushes 1.

**Local variables by slot index** [CI]: `OP_GET_LOCAL n` and `OP_SET_LOCAL n` address local variables by their slot offset within the current call frame's stack window. The compiler assigns slots at compile time based on declaration order.

**Globals by name** [CI]: `OP_GET_GLOBAL` and `OP_SET_GLOBAL` carry a constant pool index for the variable's name string. The VM does a hash table lookup at runtime. This is slower than locals but simpler to implement.

**Selected opcode list** [CI]:
```
OP_CONSTANT idx          // push constants[idx]
OP_NIL                   // push nil
OP_TRUE / OP_FALSE       // push boolean singleton
OP_POP                   // discard top of stack
OP_GET_LOCAL slot        // push stack[frame.base + slot]
OP_SET_LOCAL slot        // stack[frame.base + slot] = peek(0)
OP_GET_GLOBAL idx        // push globals[constants[idx]]
OP_SET_GLOBAL idx        // globals[constants[idx]] = peek(0)
OP_GET_UPVALUE slot      // push closure->upvalues[slot]->location
OP_SET_UPVALUE slot      // closure->upvalues[slot]->location = peek(0)
OP_EQUAL / OP_LESS / OP_GREATER
OP_ADD / OP_SUBTRACT / OP_MULTIPLY / OP_DIVIDE / OP_NEGATE
OP_NOT
OP_PRINT
OP_JUMP offset           // unconditional forward jump
OP_JUMP_IF_FALSE offset  // conditional jump, does not pop
OP_LOOP offset           // backward jump (for loops)
OP_CALL argCount         // function call
OP_CLOSURE idx           // create closure, capture upvalues
OP_CLOSE_UPVALUE         // hoist top-of-stack local to heap
OP_RETURN                // return from current call frame
```

---

### 4. The Value Stack: Stack-Based vs. Register-Based VMs [CI, EaC]

#### 4.1 Stack-Based VM (clox approach) [CI]

All operands and results pass through an explicit **value stack**:
- The stack is an array of `Value` with a `stackTop` pointer.
- `push(v)` writes to `*stackTop` and increments.
- `pop()` decrements `stackTop` and returns the old top.
- `peek(n)` reads `stackTop[-1-n]` without modifying the stack.

The VM interprets `print 3 - 2` as:
```
OP_CONSTANT 0   // push 3
OP_CONSTANT 1   // push 2
OP_SUBTRACT     // pop 3 and 2, push 1
OP_PRINT        // pop 1, print it
```

**Advantages** [CI]:
- Extremely simple instruction encoding: no register operands needed.
- Compiler trivially derives instruction sequence from expression AST (postorder traversal).
- No register allocation problem at compile time.

**Disadvantages** [CI]:
- More instructions than a register VM for the same computation (explicit push/pop of intermediates).
- Stack operations cause memory traffic that a register VM with physical registers avoids.

#### 4.2 Register-Based VMs [EaC, CI]

Lua 5.0+ and Dalvik (Android) use register-based VMs where instructions name their operand registers explicitly: `ADD R0, R1, R2`. Fewer instructions needed, but each instruction is wider (3 register operands vs. 0). The compiler must do some form of register allocation (simpler than native, but non-trivial).

**Benchmark data** [CI]: The Lua team's analysis found that register-based Lua 5 executes approximately 47% fewer VM instructions than stack-based Lua 4, but instruction dispatch overhead increased. Net result: approximately 12% faster. The tradeoff is real but modest.

**Decision** [CI, EaC]: Stack VMs are easier to implement and compile for. Register VMs can be faster but require a register allocator in the compiler front-end. For a first VM implementation, stack-based is strongly recommended.

---

### 5. Call Frames and Function Dispatch [CI]

Each function call creates a **CallFrame**:

```c
typedef struct {
    ObjClosure* closure;  // function being executed
    uint8_t* ip;          // instruction pointer for this frame
    Value* slots;         // pointer into vm.stack at base of this frame
} CallFrame;
```

**Frame stack** [CI]: The VM maintains a fixed-size array of `CallFrame` (max 64 frames deep). Each frame's `slots` pointer divides the shared value stack into windows — local variables and temporaries of each active function occupy non-overlapping regions of the same flat stack array.

**Slot addressing** [CI]: `OP_GET_LOCAL n` computes `frame->slots[n]`. The compiler assigns slot 0 to the function itself (for method dispatch), slots 1..arity to parameters, then slots arity+1.. to locals in declaration order.

**Dispatch mechanism** [CI]: The run loop caches a pointer to the current CallFrame (not an index). This avoids an array index computation on every instruction. The frame pointer is refreshed whenever a call or return modifies the call stack.

**IP as raw pointer** [CI]: The instruction pointer is a `uint8_t*` pointing directly into the bytecode array rather than an integer offset. Dereferencing a pointer is faster than an indexed array access. For maximum performance, the IP would be a C register variable to hint the compiler to keep it in a hardware register.

---

### 6. Value Representation [CI]

#### 6.1 Tagged Union (default representation)

clox uses a tagged union to represent all Lox values:

```c
typedef enum { VAL_BOOL, VAL_NIL, VAL_NUMBER, VAL_OBJ } ValueType;

typedef struct {
    ValueType type;
    union {
        bool boolean;
        double number;
        Obj* obj;    // pointer to heap-allocated object
    } as;
} Value;
```

**Size** [CI]: On a 64-bit system, this struct is 16 bytes — 4 bytes for the tag, 4 bytes padding, 8 bytes for the union payload. The double and Obj* fields both require 8-byte alignment.

**Heap allocation boundary** [CI]: Small fixed-size values (bool, nil, number) live entirely within Value. Variable-size or large objects (strings, functions, closures, instances) live on the heap; Value stores a pointer (`Obj*`). This two-level split avoids per-number heap allocation.

**Object type hierarchy** [CI]: All heap objects share a common `Obj` header:
```c
struct Obj {
    ObjType type;
    bool isMarked;       // GC mark bit
    struct Obj* next;    // intrusive linked list for GC traversal
};
```
Each specific object type (ObjString, ObjFunction, ObjClosure, ObjUpvalue, ObjClass, ObjInstance) begins with an `Obj` as its first field. The C standard guarantees that a pointer to a struct equals a pointer to its first field, making safe upcasting and downcasting via pointer casts possible — CI calls this "struct inheritance" in C.

#### 6.2 NaN Boxing (optimization) [CI]

NaN boxing reduces Value from 16 bytes to 8 bytes by exploiting IEEE 754 double representation.

**IEEE 754 background** [CI]: A 64-bit double has:
- 1 sign bit
- 11 exponent bits
- 52 mantissa bits

When all exponent bits are 1, the value is a NaN (Not a Number). Among NaNs, the quiet NaN bit (highest mantissa bit) is 1. With one additional bit reserved to avoid Intel's "QNaN Floating-Point Indefinite" value, there are 51 remaining mantissa bits available for arbitrary data — approximately 2.25 quadrillion distinct bit patterns.

**Key insight** [CI]: On 64-bit architectures, pointers only use the low 48 bits. So a 48-bit pointer fits in 51 bits, with 3 bits to spare for type tags.

**NaN-boxed encoding** [CI]:
```
Normal double:    any bit pattern where not all exponent bits are 1
Quiet NaN mask:   0x7ffc000000000000  (all exponent bits + quiet NaN bit + 1 extra)

nil:   QNAN | 0b01
false: QNAN | 0b10
true:  QNAN | 0b11
Obj*:  QNAN | SIGN_BIT | (48-bit pointer)
```

The sign bit (bit 63) is the Obj pointer flag — it is not used by IEEE 754 NaN values and is safe to claim.

**Value as uint64_t** [CI]: Under NaN boxing, `Value` is simply `typedef uint64_t Value`. Numbers are stored as raw doubles (zero conversion needed). Other types are quiet NaN patterns with payloads. The memcpy type-pun idiom is used to convert between double and uint64_t in a standards-compliant way; compilers optimize the memcpy call away entirely.

**Type checking** [CI]:
```c
#define IS_NUMBER(v)  (((v) & QNAN) != QNAN)  // any valid double
#define IS_NIL(v)     ((v) == NIL_VAL)
#define IS_BOOL(v)    (((v) | 1) == TRUE_VAL)
#define IS_OBJ(v)     (((v) & (QNAN | SIGN_BIT)) == (QNAN | SIGN_BIT))
```

**Performance** [CI]: Halves Value memory usage. More Values fit per cache line. Particularly beneficial on memory-bandwidth-constrained workloads (large arrays, closures with many upvalues).

**Portability caveat** [CI]: NaN boxing relies on hardware representation of pointers and IEEE 754 behavior. clox guards it with `#ifdef NAN_BOXING` and falls back to the tagged union on unsupported platforms.

**Comparison** [CI, EaC]:

| Representation | Size | Portability | Number access | Obj access |
|---|---|---|---|---|
| Tagged union | 16 bytes | Universal | Struct field read | Pointer dereference |
| NaN boxing | 8 bytes | 64-bit IEEE only | memcpy type pun (optimized away) | Extract low 48 bits |

---

### 7. String Representation and Interning [CI]

#### 7.1 ObjString Layout

Strings are heap-allocated objects:
```c
struct ObjString {
    Obj obj;         // type tag, GC mark, intrusive list link
    int length;
    char* chars;     // heap-allocated NUL-terminated character array
    uint32_t hash;   // precomputed FNV-1a hash of chars
};
```

**Ownership** [CI]: Every ObjString owns its `chars` array. String literals are copied from the source buffer into new heap allocations so all ObjStrings can be freed uniformly. Concatenation results similarly own their newly allocated character arrays.

**Hash caching** [CI]: The hash is computed once at creation time and stored. Hash table lookups reuse it. This is critical because string equality checks and hash table operations are on the hot path in a dynamic language (variable lookup, method dispatch).

**Hash function** [CI]: clox uses **FNV-1a** — a simple, well-distributed, fast non-cryptographic hash:
```c
uint32_t hash = 2166136261u;
for (int i = 0; i < length; i++) {
    hash ^= (uint8_t)chars[i];
    hash *= 16777619;
}
```

#### 7.2 String Interning [CI]

**Problem** [CI]: Without interning, two separate `"hello"` string literals produce two ObjString objects at different addresses. Comparing them for equality requires a character-by-character `memcmp` — O(n) per comparison.

**Solution: intern all strings** [CI]: The VM maintains a global hash table (`vm.strings`) as a set (keys are ObjStrings, values are nil). Every newly created string is looked up in this table before allocation. If a matching string already exists, the existing pointer is returned and no new allocation is made.

**Interning invariant** [CI]: After interning, any two ObjStrings with the same character sequence are guaranteed to be the same object (same pointer). String equality reduces to pointer comparison: O(1) instead of O(n).

**Implementation in copyString** [CI]:
```c
ObjString* copyString(const char* chars, int length) {
    uint32_t hash = hashString(chars, length);
    ObjString* interned = tableFindString(&vm.strings, chars, length, hash);
    if (interned != NULL) return interned;  // reuse existing
    char* heapChars = ALLOCATE(char, length + 1);
    memcpy(heapChars, chars, length);
    heapChars[length] = '\0';
    return allocateString(heapChars, length, hash);
}
```

**Interning cost** [CI]: O(n) hash computation at string creation time. This adds constant overhead per string creation but saves O(n) on every subsequent equality check and hash table lookup. For a dynamic language where string equality drives variable resolution and method dispatch, this is a decisive win.

**Impact on equality** [CI]: After interning, the `==` operator for strings becomes pointer equality only:
```c
case VAL_OBJ: return AS_OBJ(a) == AS_OBJ(b);
```

**Languages that intern all strings** [CI]: Lua (all strings), Python (small strings and identifiers), Java (string literals; `String.intern()` for others), Ruby symbols.

---

### 8. Hash Table Design for Dynamic Language Runtime [CI]

#### 8.1 Core Data Structure

```c
typedef struct {
    ObjString* key;    // NULL means empty bucket
    Value value;
} Entry;

typedef struct {
    int count;
    int capacity;      // always a power of two
    Entry* entries;
} Table;
```

**String-only keys** [CI]: clox's hash table hardcodes ObjString keys. This is sufficient for the use cases (global variables, instance fields, method tables, string interning). Adding other key types requires only a different key comparison and hashing function.

#### 8.2 Open Addressing with Linear Probing [CI]

clox uses **open addressing** (all entries in the bucket array) with **linear probing** (step one bucket at a time on collision). This is cache-friendly: probing walks the array linearly, which maps well to cache line loading.

**Separate chaining** [CI]: The alternative — linked list per bucket — is conceptually simpler but uses pointer-heavy nodes scattered across the heap. Poor cache behavior; not used in clox.

**Tombstones** [CI]: Deletion cannot simply clear a bucket — doing so would break the linear probe chain for later entries that had collided past this slot. Instead, deleted entries are replaced with a "tombstone" (NULL key, non-nil value). Probing skips tombstones but stops at truly empty buckets. Tombstones are reused for insertions.

#### 8.3 Load Factor and Resizing [CI]

**Load factor** = count / capacity. clox resizes (doubles capacity, rehashes) when load factor exceeds **0.75**. Below this threshold, collision chains stay short and lookup time stays close to O(1).

**Power-of-two capacity** [CI]: Always keeping capacity as a power of two allows replacing the modulo operation with a bitwise AND:
```c
// Slow:  index = hash % capacity;          // modulo: ~30-50x slower on x86
// Fast:  index = hash & (capacity - 1);    // AND: single clock cycle
```

Apply the same fix to the linear probe wrap-around (`index = (index + 1) & (capacity - 1)`).

**Measured impact** [CI]: This single optimization doubled hash table throughput on the Zoo benchmark (3,192 → 6,249 batches per 10 seconds). `tableGet()` execution share dropped from 72% to 35% of total runtime. Near-zero code change, dramatic result. Found via profiling — not predictable from first principles.

#### 8.4 FNV-1a as a Hash Function [CI]

Properties making FNV-1a suitable for this use case:
- **Deterministic**: same input always produces same hash.
- **Uniform**: distributes typical identifier strings evenly across buckets.
- **Fast**: single loop, no divisions, no special instructions.

Not cryptographically secure, but security is not a requirement for a language runtime's symbol table.

---

### 9. Single-Pass Compilation: No AST [CI]

#### 9.1 Architecture

CI's clox uses a **single-pass compiler**: it parses and emits bytecode simultaneously, without building an AST intermediate. The parser directly drives code emission.

```
source text
    ↓ [scanner: tokenize]
token stream
    ↓ [recursive descent parser + Pratt expression parser]
bytecode (emitted directly into current Chunk)
```

**Comparison with EaC's multi-pass model** [EaC, CI]:

| | EaC multi-pass | CI single-pass |
|---|---|---|
| Intermediate representation | AST, then IR, then machine code | Bytecode emitted during parse |
| Optimization opportunities | Many (whole-program analysis, iterative passes) | Limited (local only, no lookahead) |
| Memory for IR | Substantial (full AST maintained) | Minimal (no persistent AST) |
| Implementation complexity | High | Low |
| Suitable for | Optimizing compilers | Interpreters, scripting languages |

#### 9.2 Pratt Parser for Expressions [CI]

The compiler uses a **Pratt parser** (top-down operator precedence) for expressions. Each token type has:
- A **prefix parse function** (called when the token starts an expression)
- An **infix parse function** (called when the token appears in the middle of an expression)
- A **precedence level**

Parsing an expression calls `parsePrecedence(minPrec)`, which:
1. Calls the prefix function for the current token.
2. Loops while the next token's precedence >= minPrec, calling its infix function.

This elegantly handles operator precedence, associativity, and prefix/infix overloading (e.g., `-` as both negation and subtraction) without grammar transformations.

#### 9.3 Constraint: Upvalue Detection After Emission [CI]

The single-pass design creates a specific challenge for closures. Consider:
```
fun outer() {
    var x = 1;    // compiler emits OP_GET_LOCAL for x immediately
    x = 2;        // already emitted before...
    fun inner() { print x; }  // ...discovering x is closed over
}
```

By the time the compiler sees `inner()` and learns that `x` needs to be an upvalue, code for `x = 1` and `x = 2` has already been emitted using `OP_GET_LOCAL`. The compiler cannot go back and change those instructions.

**Solution** [CI]: All local variables start as stack locals (`OP_GET_LOCAL`). When a closure captures a variable, the *runtime* moves it from the stack to the heap at the moment the enclosing scope exits, using `OP_CLOSE_UPVALUE`. The compiled local access instructions remain correct until the variable is actually closed over.

---

### 10. Closures and Upvalues [CI]

#### 10.1 The Problem

A closure captures variables from enclosing scopes. Those variables may outlive the function that declared them (when the enclosing function returns but the closure persists). Captured variables cannot simply remain on the stack — stack frames are reclaimed on return.

#### 10.2 Upvalue Objects [CI]

An **upvalue** is an indirection object that represents a reference to a captured variable:

```c
typedef struct ObjUpvalue {
    Obj obj;
    Value* location;         // points to the variable (stack or closed field)
    Value closed;            // storage when variable moves off stack
    struct ObjUpvalue* next; // intrusive sorted list of all open upvalues
} ObjUpvalue;
```

**Open upvalue** [CI]: `location` points into the stack. The variable is still live on the stack frame of the enclosing function.

**Closed upvalue** [CI]: When the enclosing variable goes out of scope, `OP_CLOSE_UPVALUE` copies its value into the `closed` field and updates `location` to point to `&upvalue->closed`. The upvalue now carries the variable's storage with it on the heap. All closures that captured the same variable share one upvalue and therefore see the same mutable storage.

#### 10.3 ObjClosure and ObjFunction [CI]

```c
typedef struct {
    Obj obj;
    ObjFunction* function;  // static compiled representation
    ObjUpvalue** upvalues;  // array of captured upvalue pointers
    int upvalueCount;
} ObjClosure;
```

Every function declaration creates an ObjClosure wrapping the ObjFunction. The ObjFunction holds the bytecode chunk; the ObjClosure holds the runtime upvalue array.

**Why wrap in ObjClosure** [CI]: ObjFunction is the static, compile-time representation — compiled once, immutable. ObjClosure is the runtime representation — the same function compiled once but instantiated multiple times with different captured variable sets. Each execution of a function declaration creates a new ObjClosure.

#### 10.4 Compiler-Side: Upvalue Resolution [CI]

The compiler maintains a chain of `Compiler` structs (one per nesting level). When resolving a name:
1. Look in current function's locals → emit `OP_GET_LOCAL`.
2. Recursively check enclosing compiler's locals → emit `OP_GET_UPVALUE`, create upvalue entry in compiler's upvalue array.
3. Recursively check enclosing compiler's upvalues (for multi-level closures: `outer > middle > inner`) → chain upvalues.
4. Not found anywhere → assume global, emit `OP_GET_GLOBAL`.

**Flattening upvalues** [CI]: If `inner` captures `x` from `outer` but `x` is not in `middle` (which sits between them), `middle` creates a "pass-through" upvalue that captures from its enclosing upvalue rather than from a local. The `isLocal` flag in the compiler's Upvalue struct distinguishes "capture from local" (isLocal=true) vs. "capture from enclosing upvalue" (isLocal=false).

**Deduplication** [CI]: `addUpvalue()` checks if the function already has an upvalue for the same slot before creating a new one, preventing multiple upvalues for the same variable within one closure.

#### 10.5 Runtime: Tracking Open Upvalues [CI]

The VM maintains a sorted linked list (`vm.openUpvalues`) of all open upvalues ordered by stack slot. When a new closure needs an upvalue for slot S:
- Walk the list looking for an existing upvalue pointing to slot S.
- If found: reuse it (ensures two closures capturing the same variable share one upvalue and observe each other's mutations).
- If not found: allocate a new ObjUpvalue and insert it in sorted order.

**Closing upvalues on scope exit** [CI]: `OP_CLOSE_UPVALUE` is emitted when a captured local goes out of scope. The runtime calls `closeUpvalues(Value* last)` which walks `vm.openUpvalues` and closes all upvalues pointing to slots at or above `last`.

#### 10.6 Key Semantic Point: Variables vs. Values [CI]

Closures capture **variables** (mutable storage locations), not **values** (snapshots). Two closures capturing the same variable share one upvalue and observe each other's mutations. This matches the semantics of JavaScript, Python, and most other languages with closures.

---

### 11. Garbage Collection: Mark-and-Sweep [CI]

#### 11.1 Motivation and Reachability [CI]

clox uses **precise mark-and-sweep GC**. Memory is reclaimed when it becomes unreachable — when no chain of live references connects it to any root.

**Roots** [CI]: Directly reachable objects requiring no traversal:
- All values on the VM stack (locals, temporaries)
- Global variables (in `vm.globals` hash table)
- Open upvalues (`vm.openUpvalues` list)
- Compiler's current function and any function being compiled (needed because GC can run during compilation)

**Indirect reachability** [CI]: An object O is reachable if any reachable object holds a reference to O. This includes: closed upvalues referenced by closures, strings referenced as hash table keys, functions referenced by closures, fields of class instances.

#### 11.2 Mark Phase: Tri-Color Marking [CI]

All heap objects carry a 1-bit `isMarked` flag in the `Obj` header. Initially false on allocation.

**Three states**:
- **White** (unmarked): not yet seen by the traversal. May be collected.
- **Gray**: discovered but not yet fully traced (children not yet marked). clox uses a worklist (`vm.grayStack`) for gray objects.
- **Black** (marked, children traced): fully processed; will survive this collection cycle.

**Mark roots** [CI]:
```c
static void markRoots() {
    for (Value* slot = vm.stack; slot < vm.stackTop; slot++)
        markValue(*slot);
    for (int i = 0; i < vm.frameCount; i++)
        markObject((Obj*)vm.frames[i].closure);
    for (ObjUpvalue* uv = vm.openUpvalues; uv != NULL; uv = uv->next)
        markObject((Obj*)uv);
    markTable(&vm.globals);
    markCompilerRoots();
}
```

**Trace gray objects** [CI]:
```c
static void traceReferences() {
    while (vm.grayCount > 0) {
        Obj* object = vm.grayStack[--vm.grayCount];
        blackenObject(object);  // marks all objects this one references
    }
}
```
`blackenObject` dispatches on object type to mark all contained references (e.g., for ObjClosure: mark the ObjFunction and all upvalues in the upvalue array).

#### 11.3 Sweep Phase [CI]

After marking, the sweep walks the VM's intrusive linked list of all heap objects (`vm.objects`). Unmarked objects are freed; marked objects have their `isMarked` flag reset for the next cycle.

**String table weak reference** [CI]: The interned string table (`vm.strings`) must be handled specially. It holds references to all ObjStrings, but those references should not prevent collection — they are weak references. Before sweeping, clox calls `tableRemoveWhite(&vm.strings)` to remove entries for strings that were not marked. This prevents the string table from keeping dead strings alive while preserving the interning invariant for reachable strings.

#### 11.4 When to Collect [CI]

**Trigger** [CI]: GC runs when a heap allocation request would exceed a threshold (`vm.nextGC`, initially 1 MB). After each collection, `nextGC` is reset to `vm.bytesAllocated * GC_HEAP_GROW_FACTOR` (factor of 2), creating a self-tuning heap that runs GC less often as the live set grows.

**Collect on allocation** [CI]: Collecting right before allocation is the classic approach — it is already calling into the memory manager, and allocation is the only time you actually need freed memory. clox collects in `reallocate()` when `newSize > oldSize`.

**Stress test mode** [CI]: `#define DEBUG_STRESS_GC` forces a GC on every allocation. Extremely slow but invaluable for exposing GC bugs — if GC can happen at any moment, any unsafe assumption that a value will not be collected will fail immediately.

**Alternative GC timing** [EaC, CI]: More sophisticated collectors run on a separate thread, interleave at function call boundaries, or use generational strategies that collect young objects more frequently. The clox approach is the simplest correct strategy.

#### 11.5 Mark-and-Sweep Properties [CI]

| Property | Value |
|---|---|
| Type | Precise (knows exact pointer locations) |
| Algorithm | Tri-color mark-sweep with worklist |
| Pause | Stop-the-world (user program halts during collection) |
| Fragmentation | Yes (does not compact; mitigated by OS allocator) |
| Cycle handling | Yes (mark-sweep handles cycles by design) |
| Reference counting comparison | Reference counting cannot collect cycles; mark-sweep can |

**Contrast with reference counting** [CI]: Reference counting (Swift, CPython for most objects) frees objects immediately on last reference release — simple, no stop-the-world pause. But cycles (A → B → A) are never freed without a separate cycle detector. Mark-sweep handles cycles naturally.

---

### 12. Optimization in a Bytecode VM [CI]

#### 12.1 Profiling First [CI]

CI emphasizes empirical performance work: measure before optimizing, use a profiler, and validate that changes actually improve performance on representative benchmarks. Never guess at hotspots; let the profiler show them.

**Benchmark design** [CI]: Benchmarks should stress the code paths you care about, use their results (to prevent dead-code elimination), and measure enough iterations for statistical significance. The SunSpider → Octane → JetStream progression in JavaScript VMs illustrates how benchmark suites must evolve as usage patterns change.

#### 12.2 Hash Table Probing: Modulo to Bitmasking [CI]

**Problem found by profiling** [CI]: In a typical Lox program exercising field access and method calls, 72% of execution time was spent in `tableGet()`, with the bottleneck being:
```c
uint32_t index = key->hash % capacity;  // modulo: ~30-50x slower than & on x86
```

**Fix** [CI]: Maintain table capacity as always a power of two:
```c
uint32_t index = key->hash & (capacity - 1);  // AND: single fast operation
```

Apply the same fix to the linear probe wrap-around.

**Result** [CI]: Zoo benchmark throughput doubled. `tableGet()` dropped from 72% to 35% of execution time. This demonstrates the value of profiling: the bottleneck was not in the obvious algorithmic parts of hash table lookup, but in a single arithmetic operator.

#### 12.3 NaN Boxing as a Value Representation Optimization [CI]

Described in Section 6.2. Halving Value size improves cache utilization across all operations that push and pop values on the stack or store them in arrays.

#### 12.4 Inline Caching (Beyond clox) [CI]

CI notes but does not implement inline caching, which is the dominant optimization technique in production VMs (V8, SpiderMonkey, HotSpot):

**Problem**: `OP_GET_PROPERTY` looks up a field name in a hash table on every execution. In a tight loop accessing the same field millions of times, this is redundant.

**Solution (monomorphic inline cache)**: The first time `OP_GET_PROPERTY` executes at a given call site, record the object's "shape" (class/structure layout) and the field's slot offset. On subsequent executions, check if the shape matches; if so, skip the hash table lookup and go directly to the cached slot.

**Impact in production VMs**: Inline caching is responsible for much of the 10-100x performance difference between a naive bytecode VM and a production JIT.

#### 12.5 Constant Folding in Single-Pass Compilation [CI]

In clox's single-pass compiler, constant folding happens opportunistically during parsing. When both operands of a binary expression are literal constants, the compiler can evaluate the result at compile time and emit a single `OP_CONSTANT` for the result rather than two constants and a runtime arithmetic instruction.

This is a constrained form of the EaC peephole simplifier's constant-folding pass, applied inline during parse rather than as a separate optimization phase. [EaC, CI]

---

### 13. Summary: Stack Machine vs. Register Machine vs. Native Code [CI, EaC]

```
Are you targeting a specific native ISA?
  YES → Native code (EaC model)
          Instruction selection (peephole or tree-pattern matching) required
          Register allocation (graph coloring or linear scan) required
          Highest performance, one backend per target architecture
  NO  → Are you building a bytecode VM?
          Is compile-time simplicity the priority?
            YES → Stack-based VM (clox / CI model)
                    No register allocation in compiler
                    Trivial instruction emission during recursive descent parse
                    More instructions per computation, but compact encoding
            NO  → Register-based VM (Lua 5+, Dalvik model)
                    Requires register allocation in compiler
                    ~47% fewer instructions than equivalent stack VM
                    ~12% faster in practice (Lua 4 to 5 data)
                    More complex compiler, wider instruction format

Is garbage collection needed?
  YES + simple implementation priority → Mark-and-sweep (CI model)
  YES + low latency priority           → Generational GC or concurrent mark-sweep
  YES + deterministic reclaim          → Reference counting (+ cycle detector for correctness)

Is string equality on hot path? (dynamic language with identifier lookups)
  YES → Intern all strings
          Trades O(n) hash at creation for O(1) pointer equality at every use
          Pays off whenever string equality is on a hot path (variable lookup, method dispatch)
```
