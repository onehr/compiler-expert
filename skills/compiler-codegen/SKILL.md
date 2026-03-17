---
name: compiler-codegen
description: Code generation expert — covers instruction selection (BURS, tree-pattern matching), register allocation (graph coloring, linear scan), bytecode VM design (stack vs register), NaN boxing, value representation, and native code emission. Use when implementing a code generator, bytecode VM, or register allocator.
---

# Compiler Code Generation

Full reference material is in `references/codegen-reference.md` (sourced from Engineering a Compiler [EaC] Ch. 11 & 13, and Crafting Interpreters [CI]).

---

## Key Decision Points

### Instruction Selection Strategy

**Choose peephole selection** when:
- IR is linear (not tree-shaped)
- You need a late-stage cleanup pass over inefficiencies from prior phases
- Toolchain already provides RTL infrastructure (e.g., GCC-family)

**Choose tree-pattern matching (BURS)** when:
- IR is tree-shaped (AST or close to it)
- You need explicit cost-driven selection
- ISA has complex address modes requiring multi-op IR pattern recognition
- You want local optimality guarantees per subtree

**Avoid ad-hoc matching** for anything beyond prototypes: no cost model, no retargetability.

See: `references/codegen-reference.md` Part I, Sections 2–4

---

### Register Allocation Strategy

```
Tight compile-time budget? (JIT)
  YES → Linear scan (O(n log n), good for hot-path JIT)
  NO  → SSA already available + code quality critical?
          YES → SSA-based allocation (chordal graph, optimal coloring)
          NO  → Global graph-coloring (Chaitin-Briggs)

Single basic block scope?
  YES → Local allocator (Best's algorithm — optimal for uniform spill costs)
```

**Coalescing strategy**:
- Conservative (Briggs): safe default, cannot worsen colorability
- Aggressive (Chaitin): more copies eliminated, occasional extra spills
- Biased coloring: lowest risk, good default when static analysis is hard
- Iterated: best quality, higher compile-time cost

**Spill priority** (never spill infinite-cost LRs; always spill negative-cost LRs):
- Prefer rematerializable > clean > dirty when next-use distances are similar
- Metric: `cost / degree` (standard) or `cost / degree²` (high-pressure graphs)

See: `references/codegen-reference.md` Part II

---

### Bytecode VM: Stack vs. Register

| Criterion | Stack VM | Register VM |
|---|---|---|
| Compiler complexity | Low — no register allocation needed | Higher — need register allocation |
| Instruction count | More (explicit push/pop) | ~47% fewer (Lua 4→5 data) |
| Performance gain | Baseline | ~12% faster in practice |
| Encoding width | Narrow (no register operands) | Wider (3 register operands typical) |
| First implementation | Strongly recommended | After gaining experience |

**Stack VM** (clox model): postorder AST traversal drives emission trivially. No register allocation problem exists — stack slot is determined by expression evaluation order.

**Register VM** (Lua 5+, Dalvik): requires a register allocator in the compiler front-end. Justified when the ~12% performance gain matters.

See: `references/codegen-reference.md` Part III, Sections 1 & 4

---

### Value Representation: Tagged Union vs. NaN Boxing

| Representation | Size | Portability | Number access |
|---|---|---|---|
| Tagged union | 16 bytes (64-bit) | Universal | Struct field read |
| NaN boxing | 8 bytes | 64-bit IEEE only | memcpy type pun (optimized away) |

**NaN boxing insight**: IEEE 754 quiet NaN has 51 spare mantissa bits. On 64-bit platforms, pointers use only 48 bits. The remaining bits carry type tags. Result: halved Value size, better cache utilization.

**Use NaN boxing when**: targeting 64-bit IEEE platforms and memory bandwidth is a concern (large arrays, closures with many upvalues).

See: `references/codegen-reference.md` Part III, Section 6

---

### Native Code Emission Pipeline

```
IR (SSA or linear)
  ↓ Instruction Selection (peephole or tree-pattern)
    ↓ Register Allocation (graph coloring or linear scan)
      ↓ Spill code insertion (store after def, load before use)
        ↓ Re-allocation (1-3 iterations typical)
          ↓ Native code emission
```

**Selection-allocation interaction**: Keep them separate when possible. Merge only when ISA forces pre-assignment (e.g., IA-32 multiply requires AX/DX).

---

## Critical Implementation Details

- **BURS tile algorithm**: single bottom-up postorder pass; tracks minimum-cost rule per (node, class) pair; local optimality, not global
- **Graph coloring convergence**: build interference graph → coalesce copies → estimate spill costs → Simplify/Select → insert spills → repeat (1-3 iterations)
- **Linear scan limitation**: intervals overestimate true live ranges (ignores control flow); may cause unnecessary spills; compensated by fast O(n log n) build
- **SSA-based allocation**: chordal interference graph enables polynomial-time optimal coloring; requires out-of-SSA translation afterward (inserts copies)
- **String interning invariant**: after interning, pointer equality implies value equality — O(1) string comparison on the hot path

---

## Sources

Full annotated notes in `references/codegen-reference.md`:
- Part I: Instruction Selection — EaC Chapter 11
- Part II: Register Allocation — EaC Chapter 13
- Part III: Bytecode VM Design — CI (Crafting Interpreters, clox chapters 14–30)

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-optimizer` — optimized IR is the input to code generation
- `compiler-mir` — for compilers without an optimizer, MIR feeds directly into codegen
- `compiler-architect` — routes here as the eighth (final) implementation phase
- `compiler-expert` — routes here on code generation / register allocation / VM design questions

**Trace Down ↓** — next skills during/after code generation
- `compiler-abi` — calling conventions and struct layout must be applied during code emission
- `compiler-linker` — generated object files are consumed by the linker

**Related →** — peers at the back-end layer
- `compiler-abi` — tightly coupled: instruction emission depends on ABI decisions
- `compiler-runtime` — generated code must interoperate with the runtime (GC barriers, stack introspection)
