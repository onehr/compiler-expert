---
name: compiler-mir
description: Mid-level IR (MIR) and borrow checking expert — covers MIR design (simplified CFG), Non-Lexical Lifetimes (NLL), borrow checker implementation, drop elaboration, and encoding ownership semantics in IR. Use when implementing a borrow checker, ownership system, or need a simplified CFG-based IR for analysis.
---

# Compiler MIR (Mid-Level IR) and Borrow Checking

## Overview

MIR (Mid-level Intermediate Representation) is the IR used by `rustc` between HIR (type-checked, desugared source) and LLVM IR (machine-level code). Its design philosophy: **simplify radically** to make analysis and verification tractable. MIR is sometimes described as a "15-statement language" — it has very few node kinds, so the borrow checker and optimizer can be exhaustively complete.

This skill covers:
1. MIR design principles
2. Control flow graph structure
3. Non-Lexical Lifetimes (NLL) and the borrow checker
4. Drop elaboration
5. Encoding ownership semantics in IR

---

## MIR Design Principles

**Why MIR?**

The HIR is too complex for borrow checking. The borrow checker needs to reason about:
- Every possible execution path (control flow)
- Every memory location a value can occupy
- Whether values are initialized, moved, or borrowed at each program point

The MIR makes this tractable by reducing the language to a core of simple operations.

**Core design choices**:

1. **Explicit control flow**: all loops, conditionals, and early returns are represented as explicit CFG edges. No `for`, `while`, `if let` — only `goto`, `switchInt`, and function calls with two successors (return edge and unwind edge).

2. **Explicit moves**: every move is a syntactic move in the MIR. The borrow checker does not need to infer where values move; the MIR makes it explicit.

3. **Explicit drops**: the compiler inserts explicit `drop` terminators at every point where a value is dropped. The borrow checker can reason about drop as a use.

4. **No nesting**: expressions are flattened into a sequence of statements. `a + b * c` becomes `_1 = b * c; _2 = a + _1`.

5. **Explicit temporaries**: all intermediate values are named (as locals `_0`, `_1`, ...). No anonymous temporaries.

6. **Place-based memory model**: a *place* is an expression that describes a memory location: `local`, `local.field`, `*local`, `local[index]`. Assignments are `place = rvalue`.

---

## MIR Structure

```
Function:
  locals: Vec<LocalDecl>        // _0 = return value, _1..=_n = args, rest = temporaries
  basic_blocks: Vec<BasicBlockData>

BasicBlockData:
  statements: Vec<Statement>
  terminator: Terminator

Statement:
  | Assign(Place, Rvalue)       // _1 = some_rvalue
  | StorageLive(Local)          // allocate stack slot
  | StorageDead(Local)          // deallocate stack slot
  | SetDiscriminant(Place, Variant)
  | Nop

Terminator:
  | Goto { target }
  | SwitchInt { discr, targets }
  | Return
  | Call { func, args, destination, target, unwind }
  | Drop { place, target, unwind }
  | Assert { cond, expected, msg, target, unwind }
  | Unreachable

Place:
  | Local(Local)
  | Place.Field(Field)
  | *Place              (deref)
  | Place[index]
  | Place as Variant    (downcast)

Rvalue:
  | Use(Operand)
  | Ref(Region, BorrowKind, Place)
  | AddressOf(Mutability, Place)
  | BinaryOp(BinOp, Operand, Operand)
  | UnaryOp(UnOp, Operand)
  | Aggregate(AggregateKind, Vec<Operand>)
  | Discriminant(Place)
  | Len(Place)

Operand:
  | Copy(Place)
  | Move(Place)
  | Constant(Const)
```

---

## CFG Structure

Each basic block has exactly one terminator. The terminator determines which blocks are successors.

**Entry block**: `START_BLOCK` (index 0)
**Return**: a block whose terminator is `Return`; the return value is in `_0`

**Unwind paths**: every `Call` and `Drop` that could panic has both a normal successor and an unwind successor. The unwind path models stack unwinding (destructors called during a panic). The borrow checker must verify that values are valid along both paths.

---

## Borrow Checking Overview (NLL)

**Non-Lexical Lifetimes (NLL)** — RFC 2094

Before NLL, lifetimes in Rust were tied to lexical scopes. This caused false positives:

```rust
// This should be fine but was rejected before NLL:
let mut x = vec![1, 2, 3];
let last = x.last();    // borrow of x
println!("{:?}", last); // last used here
x.push(4);              // x mutated here — after last is used
```

NLL computes the **liveness** of each borrow and only reports an error if the borrow is still live at the point of conflict.

**Key concepts**:

- **Loan**: a borrow creates a loan `(place, kind, point)` — the place being borrowed, the borrow kind (shared/mutable), and the point where the borrow occurs
- **Region**: each borrow has an associated region (lifetime) that is a set of program points
- **Liveness**: a region variable is live at a program point if the borrow it represents could still be used in the future from that point
- **Constraints**: the borrow checker generates a system of region constraints: `R1 ⊆ R2` means region R1 must include all points of R2. These arise from function signatures (lifetime parameters) and assignments.

**NLL borrow check algorithm (sketch)**:

1. Compute liveness of all locals (which locals are live at each program point)
2. For each borrow, compute the minimal region satisfying all constraints (subset constraints form a lattice; solve by propagation)
3. For each point, check: is there any live loan that conflicts with the operation at this point?
   - A shared borrow of `p` conflicts with a mutation of `p` or a move out of `p`
   - A mutable borrow of `p` conflicts with any other use of `p` (read, write, reborrow)

**Two-phase borrows**: allow a method receiver to be borrowed mutably for the method call even if the same receiver is used in an argument position. The borrow is "reserved" at the first use and only "activated" when needed.

---

## Drop Elaboration

**The problem**: when does a value get dropped? In Rust, values are dropped when they go out of scope — but with borrows, panics, and early returns, the exact drop points are complex.

**Drop elaboration** inserts explicit `Drop` terminators into the MIR at every point where a drop should occur:
- At the end of the block where a local's storage is live
- On the unwind paths through `Call` terminators (stack unwinding)
- On early return paths
- When a value is moved-from (partial moves)

**Drop flags**: for values that might or might not have been initialized at a given point (e.g., after a conditional initialization), the elaborator inserts a boolean drop flag. The `Drop` terminator is guarded by a check of the flag.

```
// Conceptual:
if _drop_flag_x {
    drop(_x);
}
```

**Drop order**: values are dropped in reverse order of initialization (LIFO). The elaborator must respect this order when inserting drops.

---

## Ownership Encoding in IR

**Moves**: in MIR, `Move(place)` in an operand position consumes the value at `place`. After a move, `place` is uninitialized. The borrow checker tracks initialization: it is an error to use an uninitialized place.

**Copies**: `Copy(place)` copies the value. Only types implementing `Copy` may appear as `Copy` operands; the elaborator has already verified this.

**Borrows**: `Ref(region, kind, place)` creates a reference. The region is filled in by the borrow checker to be the minimal region satisfying the constraints.

**Unique ownership invariant**: at any program point, each memory location is either:
- Owned by exactly one local (no borrows active)
- Shared (one or more shared borrows, no mutation)
- Exclusively borrowed (one mutable borrow, no other access)

The borrow checker enforces this invariant.

---

## Polonius

Polonius is the next-generation borrow checker, formulated as a Datalog program. It models the borrow checker as computing relations over facts:

```datalog
// A loan L is live at point P if:
loan_live_at(L, P) :-
    use_of_var_derefs_origin(L, O),
    origin_live_on_entry(O, P).
```

Polonius is strictly more powerful than NLL: it can accept programs that NLL falsely rejects (cases involving conditional borrows and reassignment). It is currently in development for inclusion in `rustc`.

---

## Implementation Checklist

- [ ] Define MIR data types (Place, Rvalue, Statement, Terminator, BasicBlock)
- [ ] Implement HIR → MIR lowering
  - [ ] Desugar all control flow into explicit CFG blocks
  - [ ] Flatten expressions into statement sequences
  - [ ] Name all temporaries
  - [ ] Represent all moves explicitly
- [ ] Implement drop elaboration
  - [ ] Insert `StorageLive`/`StorageDead` for each local
  - [ ] Insert `Drop` terminators on all paths where a drop is needed
  - [ ] Insert drop flags for conditionally-initialized values
- [ ] Implement NLL borrow checker
  - [ ] Compute liveness (backward dataflow)
  - [ ] Generate region constraints from borrows and function signatures
  - [ ] Solve region constraints (propagation to fixed point)
  - [ ] Check for conflicting loans at each point
- [ ] Report borrow check errors with good source spans and explanations
- [ ] Implement two-phase borrow support (if needed)

---

## Planned Sources

- rustc MIR design documentation (rustc-dev-guide: "The MIR" chapter)
- NLL RFC 2094 — Non-Lexical Lifetimes
- Polonius paper: "Polonius: Borrow-checking Rust via a Datalog solver" (Matsakis et al.)
- rustc source: `rustc_mir_build`, `rustc_borrowck`

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-hir` — MIR is the next lowering step after HIR
- `compiler-architect` — routes here when the language needs ownership/borrow checking
- `compiler-expert` — routes here on MIR design / borrow checker / NLL questions

**Trace Down ↓** — next skills after this phase
- `compiler-optimizer` — optimization passes run on MIR (or equivalent control-flow IR)
- `compiler-codegen` — MIR is the direct input to code generation

**Related →** — sibling IR design skills
- `compiler-hir` — the IR layer above; MIR is a simpler, CFG-based view of HIR
- `compiler-elaboration` — for dependent types, elaboration produces the core term that MIR analogs check
