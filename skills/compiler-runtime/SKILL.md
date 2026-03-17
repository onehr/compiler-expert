---
name: compiler-runtime
description: Language runtime and memory management expert — covers GC strategies (mark-and-sweep, reference counting, generational), manual memory with defer semantics, stack frame layout, exception/panic handling, and coroutine/async implementation. Use when designing a language runtime, GC, or memory management strategy.
---

# Language Runtime and Memory Management

---

## GC Strategy Decision Guide

```
Does correctness require automatic memory management?
  NO  → Manual memory + defer (Zig/C style) — see below
  YES → Is low latency (< 1ms pauses) a hard requirement?
          YES → Concurrent / incremental GC (Go, Go 1.14+, Azul C4)
                  or reference counting + cycle collector (Swift, CPython)
          NO  → Is throughput the priority?
                  YES → Generational GC (JVM G1/ZGC, CPython 3.13+)
                  NO  → Simple mark-and-sweep (clox, scripting languages)

Does the language allow cycles in the object graph?
  YES + RC chosen → Must add a cycle detector (CPython gc module, Swift weak refs)
  NO  (DAG only)  → RC alone is sufficient (Swift with discipline, Rust Rc<T>)
```

---

## Mark-and-Sweep Implementation

Reference material for clox's implementation is in `references/codegen-reference.md` (see Part III, Section 11). Key design decisions summarized here:

### Tri-Color Marking

Three object states during collection:
- **White**: not yet visited — may be collected if still white after mark phase
- **Gray**: discovered (reachable) but children not yet traced — on worklist
- **Black**: fully traced — will survive

**Algorithm**:
1. Mark roots gray (stack values, globals, open upvalues, compiler roots)
2. While gray worklist non-empty: pop one object, mark it black, mark all its referenced objects gray
3. Sweep: walk all heap objects; free white objects, reset black objects to white for next cycle

### Write Barriers (for incremental/concurrent GC)

When GC runs concurrently with the mutator, the mutator can turn a black object gray again by writing a white pointer into it (Dijkstra tricolor invariant violation). A **write barrier** intercepts all pointer writes to maintain the invariant:

- **Insertion barrier**: when writing pointer `B → C`, mark `C` gray if `B` is black
- **Deletion barrier** (SATB — snapshot-at-the-beginning): when overwriting pointer `B → C`, mark `C` gray before the overwrite (preserves the snapshot)

clox's stop-the-world GC does not need write barriers — the mutator is paused for the entire mark phase.

### Heap Growth Heuristic

clox doubles the GC threshold after each collection:
```
nextGC = bytesAllocated * GC_HEAP_GROW_FACTOR   // factor of 2
```
This self-tunes: programs with large live sets trigger GC less often. Programs with small live sets trigger it aggressively. A stress-test mode (`DEBUG_STRESS_GC`) forces GC on every allocation — invaluable for finding GC bugs.

See full annotated notes: `references/codegen-reference.md` Part III §11

---

## Reference Counting

### How It Works

Each heap object carries a count of references to it. On assignment, increment the new target's count and decrement the old target's count. When a count reaches zero, free the object immediately.

### Strengths

- Deterministic reclamation: objects freed the instant their last reference drops
- No stop-the-world pause: reclamation is interleaved with the mutator
- Cache-friendly for short-lived objects (freed before going cold)

### Weaknesses

- **Cycles are never collected**: A → B → A keeps both alive forever
- **Throughput overhead**: every pointer assignment modifies counters
- **Atomic operations**: thread-safe RC requires atomic increment/decrement (expensive on multicore)

### Swift / CPython Approach

**Swift**: ARC (Automatic Reference Counting). Compiler inserts `retain`/`release` calls. Weak references (`weak var`) break cycles — they do not increment the count and are zeroed when the object is freed. Unowned references (`unowned`) are like weak but assert non-nil.

**CPython**: RC for everything, with a cycle detector (`gc` module) that runs periodically. The cycle detector uses a mark-sweep variant that only walks objects in "generation" containers known to possibly form cycles.

---

## Manual Memory + Defer (Zig / Go Approach)

### Core Idea

The programmer allocates and frees memory explicitly but uses `defer` to guarantee cleanup at scope exit regardless of how the scope is exited (normal return, early return, panic/error propagation).

```zig
const buf = try allocator.alloc(u8, 1024);
defer allocator.free(buf);
// ... use buf ...
// buf is freed when this scope exits, even if an error occurs above
```

### Zig Allocator Design

Every function that allocates takes an `Allocator` argument — there is no global heap. This makes allocation strategy configurable and testable:
- `GeneralPurposeAllocator`: default; leak detection in debug builds
- `ArenaAllocator`: bulk-free all at once when the arena is destroyed
- `FixedBufferAllocator`: allocates from a stack buffer, zero heap traffic
- `c_allocator`: wraps `malloc`/`free` for C interop

Benefits: tests can use a deterministic allocator; subsystems can use arenas for bulk allocation without individual frees; no implicit global state.

### Compile-Time Resource Tracking

Zig's `defer` is a compile-time construct. In a future Zig direction, comptime analysis can detect when a resource is acquired without a corresponding `defer` — a form of static resource leak detection without a GC.

---

## Stack Frame Layout

A call frame on x86-64 (System V ABI) looks like:

```
  High addresses
  ┌─────────────────────┐
  │ caller's frame ...  │
  ├─────────────────────┤  ← caller's rsp before call
  │ return address      │  pushed by `call` instruction
  ├─────────────────────┤  ← rbp (callee's frame pointer, if used)
  │ saved rbp           │
  ├─────────────────────┤
  │ callee-saved regs   │  (rbx, r12–r15 if callee uses them)
  ├─────────────────────┤
  │ local variables     │  compiler-assigned offsets from rbp
  ├─────────────────────┤
  │ [alignment padding] │  ensure rsp is 16-byte aligned before next call
  ├─────────────────────┤  ← rsp (top of stack)
  │ red zone (128B)     │  not in frame, but usable by leaf functions
  └─────────────────────┘
  Low addresses
```

**Frame pointer (rbp)**: optional. Omitting it (`-fomit-frame-pointer`) frees one register and reduces prologue/epilogue cost. Required for some stack unwinding implementations; DWARF-based unwinding does not need it.

**Slot assignment**: compiler assigns a fixed offset from rbp (or rsp in frameless functions) to each local variable. Stack allocation of a local is free — just adjusting rsp at function entry.

---

## Exception / Panic Handling

Three implementation strategies with very different cost profiles:

### 1. setjmp / longjmp (C-style)

`setjmp` saves the current register state (including `rsp`, `rbp`, `rip`) into a `jmp_buf`. `longjmp` restores that state from any point in the call stack, effectively "returning" to the setjmp site.

**Cost**: every call site that might throw must save `jmp_buf` on the stack — non-zero cost on the happy path. Destructors/defer not run automatically (no stack unwinding).

**Use when**: no destructors, no RAII, simple scripting languages, C code.

### 2. Stack Unwinding (C++ exceptions, Rust `?` with Drop)

The compiler emits unwind tables (`.eh_frame` on ELF, `.pdata` on PE) describing how to unwind each stack frame: which callee-saved registers to restore, where the previous frame's return address is, and which destructors to call.

**Cost on happy path**: zero for the code itself; the unwind tables are read-only data that is never accessed unless an exception occurs. This is the "zero-cost exception" model.

**Cost on exception path**: expensive — must walk the unwind tables, execute landing pads, and call destructors in order. Acceptable because exceptions are rare.

**Use when**: RAII semantics, destructors, Rust-style Drop, zero-cost on happy path required.

### 3. Explicit Error Values (Go, Zig)

Errors are values returned from functions. No unwinding, no tables. `defer` handles cleanup. The compiler can optimize error paths normally.

**Cost**: return value overhead on every call, error check on every call. Predictable, branch-predictable in common cases.

**Use when**: errors are expected and frequent, predictable performance more important than implicit propagation, no runtime cost for cleanup setup.

---

## Deep Reference Material

### Primary Reference: `references/runtime-reference.md`

Comprehensive deep-dive covering all major runtime topics, researched from primary sources:

- **Go GC** — Rick Hudson's ISMM 2018 keynote ([go.dev/blog/ismmkeynote](https://go.dev/blog/ismmkeynote)): tri-color concurrent mark-sweep, the Pacer feedback controller, why Go rejected generational GC, write barrier evolution, the "tyranny of the 9s" latency problem, 3 orders-of-magnitude pause time improvement 2014–2018.
- **Zig memory model** — Official Zig Language Reference ([ziglang.org/documentation/master/#Memory](https://ziglang.org/documentation/master/#Memory)): allocator interface design rationale, `defer`/`errdefer` compilation, all standard allocators (Arena, FixedBuffer, DebugAllocator, smp_allocator), the allocator decision flowchart, leak detection via testing allocator.
- **Rust panic implementation** — Rust Compiler Dev Guide ([rustc-dev-guide.rust-lang.org/panic-implementation.html](https://rustc-dev-guide.rust-lang.org/panic-implementation.html)): two-stage dispatch (core lang item → std handler → panic runtime), `panic_abort` vs `panic_unwind`, how `__rust_start_panic` crosses the FFI boundary, `catch_unwind`, the `?` operator as structured propagation.
- **CPython cyclic GC** — CPython InternalDocs ([github.com/python/cpython/blob/main/InternalDocs/garbage_collector.md](https://github.com/python/cpython/blob/main/InternalDocs/garbage_collector.md)): `PyGC_Head` doubly-linked list layout, the `gc_ref` copy algorithm for cycle detection, two-generation incremental collection, destruction ordering (weak refs → finalizers → resurrection handling → `tp_clear`), free-threaded build GC differences.
- **V8 Orinoco GC** — V8 Blog ([v8.dev/blog/trash-talk](https://v8.dev/blog/trash-talk)): generational heap layout (nursery/intermediate/old), parallel Scavenger, concurrent major GC (Mark-Compact), parallel/incremental/concurrent distinction, idle-time GC, measured results (20–50% main-thread scavenge reduction, 50% pause reduction in heavy WebGL).

**Topics covered in the reference**:
1. GC Strategy Decision Guide with decision table (8 languages)
2. Mark-and-sweep: tri-color invariant, STW vs incremental vs concurrent, Dijkstra vs Yuasa write barriers, heap sizing heuristics
3. Reference counting: algorithm, CPython cycle detection, Swift ARC (strong/weak/unowned), RC vs tracing performance comparison
4. Go's GC: design philosophy, concurrent mark+sweep, Pacer, key decisions table
5. Zig manual memory: allocator interface, defer compilation, all standard allocators, leak testing
6. Stack frames: x86-64 frame layout diagram, fixed/segmented/copying stacks, red zone
7. Panic/exception handling: setjmp/longjmp, zero-cost DWARF unwinding, Rust panic two-stage dispatch, Go panic/recover
8. Coroutines/async: stackful goroutines vs stackless state machines, suspend point analysis, the color problem

### Other References

- Crafting Interpreters (Nystrom) — clox GC chapter: annotated notes in `references/codegen-reference.md` Part III §11
- "A Unified Theory of Garbage Collection" (Bacon, Cheng, Rajan) — RC and tracing as duals

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-architect` — runtime design (GC strategy, memory model) is an architectural decision
- `compiler-expert` — routes here on GC / memory management / stack frame / coroutine questions

**Trace Down ↓** — runtime design influences multiple downstream skills
- (none — runtime is designed in parallel with codegen, not strictly downstream)

**Related →** — closely related skills
- `compiler-codegen` — generated code must emit GC barriers and use the runtime ABI
- `compiler-abi` — stack layout and frame format are part of both the runtime and ABI contract
- `compiler-mir` — ownership/borrow checking is the compile-time analog of runtime memory safety
