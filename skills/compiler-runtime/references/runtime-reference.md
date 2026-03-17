# Language Runtime and Memory Management — Deep Reference

Sources researched:
- [Go GC: Getting to Go](https://go.dev/blog/ismmkeynote) — Rick Hudson, ISMM 2018 keynote transcript
- [Zig Language Reference: Memory](https://ziglang.org/documentation/master/#Memory)
- [Rust Compiler Dev Guide: Panic Implementation](https://rustc-dev-guide.rust-lang.org/panic-implementation.html)
- [CPython InternalDocs: Garbage Collector Design](https://raw.githubusercontent.com/python/cpython/main/InternalDocs/garbage_collector.md)
- [V8 Blog: Trash Talk — the Orinoco GC](https://v8.dev/blog/trash-talk)

---

## 1. GC Strategy Decision Guide

### When to Use No GC — Manual, Defer, Ownership

**Languages**: C, Zig, Rust (ownership model), assembly

**Zig's model** [Zig]: The language performs *no* memory management on behalf of the programmer. There is no runtime. This is why Zig works in real-time software, OS kernels, embedded devices, and low-latency servers. Every function that allocates takes an explicit `Allocator` parameter — there is no global allocator. Programmers must always be able to answer: *Where are the bytes?*

**Rust's ownership model**: The borrow checker enforces single-owner semantics at compile time. When a value's owner goes out of scope, the compiler inserts a `Drop::drop` call. No GC, no runtime overhead, zero-cost abstractions at the machine level.

**When to choose manual/defer/ownership**:
- Hard real-time constraints (GC pauses unacceptable)
- Embedded / bare-metal targets (no OS, no allocator available)
- OS kernel development
- Libraries that must work in any context (GC would impose a runtime dependency)
- When the programmer can express lifetimes statically

**Trade-offs**: Memory safety requires discipline (C/Zig) or a sophisticated type system (Rust). Cycles are not an issue — without a GC, cycles are just objects that both reference each other; they must be freed manually or broken via weak pointers.

---

### When to Use Reference Counting

**Languages**: Swift (ARC), CPython, Objective-C, Perl, PHP

**Core idea**: Each heap object carries a reference count. Every new reference increments it; every dropped reference decrements it. When the count reaches zero, the object is freed immediately.

**CPython's baseline** [CPython]: Reference counting is CPython's primary GC algorithm. The `sys.getrefcount()` function exposes the current count (always 1 higher than expected since the call itself creates a reference).

**Pros**:
- Deterministic reclamation — freed exactly when the last reference drops
- No stop-the-world pause for normal allocation/deallocation
- Good cache behavior for short-lived objects (freed before memory goes cold)
- Simple to implement

**Cons**:
- **Cycles are never collected by RC alone**: if A → B → A, both reference counts stay above zero even when neither is reachable from any root. CPython had this exact problem with self-referential containers.
- Throughput overhead: every pointer assignment modifies a counter
- Thread-safe RC requires atomic operations — expensive on multicore
- Poor for highly concurrent, allocation-heavy workloads (contention on reference count fields)

**The cycle problem in detail** [CPython]: Consider:
```python
container = []
container.append(container)   # container now holds a ref to itself
del container                  # refcount drops to 1, not 0 — never freed
```
CPython solves this by running a separate cyclic garbage collector (the `gc` module) periodically. The cycle collector uses a modified mark-sweep algorithm over tracked "container" objects.

**Swift ARC** [Zig/Swift analogy]: Swift's ARC is compiler-inserted — the compiler emits `retain`/`release` calls. Three reference types:
- `strong` (default): increments the count
- `weak`: does NOT increment the count; automatically zeroed when the object is deallocated. Used to break reference cycles.
- `unowned`: like `weak` but asserts non-nil on access (crashes if object was deallocated). Used when the programmer guarantees the object outlives the reference.

**When to choose RC**:
- Language semantics require deterministic destruction (file handles, network connections should close promptly)
- Heap objects are predominantly tree-structured (few cycles)
- Latency matters more than throughput
- Integration with foreign code that doesn't tolerate tracing GC's pointer movement

---

### When to Use Mark-and-Sweep

**Languages**: clox, early Go (stop-the-world), Lua, many scripting languages

**Core idea**: Periodically pause the mutator (or run concurrently), walk all reachable objects starting from roots, mark them live, then sweep (free) everything not marked. Reset marks for the next cycle.

**Stop-the-world variant**: Simple. Pause all threads, mark all reachable objects, free the rest, resume. Easy to implement correctly. Unacceptable pause times for large heaps or latency-sensitive applications.

**Incremental mark-and-sweep**: Break the mark phase into small increments interleaved with mutator execution. Requires write barriers to detect when the mutator changes the object graph between increments. Slightly increases total GC time but reduces maximum pause time.

**Concurrent mark-and-sweep** [Go]: Mark phase runs concurrently with the mutator on separate threads. Requires write barriers. The sweep phase can also be mostly concurrent. Go's GC is this model — see Section 4 for details.

**When to choose mark-and-sweep**:
- Pause times can be bounded but need not be zero
- Object graph is complex with many cycles (RC would be expensive)
- Throughput matters and the programmer does not want to manage memory
- A simple implementation is acceptable (stop-the-world for scripting languages)

---

### When to Use Generational GC

**Languages**: JVM (G1, ZGC, Shenandoah), V8/JavaScript [V8], .NET CLR, Ruby (Rincrement)

**The generational hypothesis** [V8]: "Most objects die young." In practice, most allocated objects are short-lived temporaries — they become garbage almost immediately. A generational collector exploits this: collecting only the "young generation" (recently allocated objects) is cheap and collects most garbage without touching long-lived objects.

**V8's heap structure** [V8]:
- **Young generation**: split into "nursery" (newly allocated) and "intermediate" (survived one GC)
- **Old generation**: objects that survived two GCs

Objects move through generations as they survive collections. Collecting only the young generation (Minor GC / Scavenger) is fast because there are few survivors to copy.

**Semi-space design for young generation** [V8]: The young generation is split into two equal halves (from-space and to-space). During a scavenge, live objects are copied from from-space to to-space. When the scavenge completes, from-space and to-space are swapped. The copy step automatically compacts the surviving objects, eliminating fragmentation.

**Write barriers for generational GC** [V8]: When old-generation objects are mutated to point at young-generation objects, these cross-generational pointers must be tracked (the "remembered set"). Otherwise a Minor GC that only scans the young generation could miss roots in the old generation. Write barriers maintain this remembered set.

**Major GC vs Minor GC** [V8]:
- **Minor GC (Scavenger)**: collects young generation only. Parallel in V8 (helper threads). Reduced main-thread time by 20–50%.
- **Major GC (Mark-Compact)**: collects the entire heap. Concurrent marking + parallel compaction. Pause times reduced by up to 50% in heavy WebGL workloads.

**When to choose generational GC**:
- High allocation rates (typical for dynamic languages, functional programming)
- Long-running programs where many objects have divergent lifetimes
- Throughput and low average latency both matter
- Can tolerate occasional major GC pauses (or accept the complexity of concurrent major GC)

---

### Decision Table

| Language | Strategy | Key Reason | Cycle Problem |
|---|---|---|---|
| C | Manual (malloc/free) | No runtime, no overhead | Programmer responsibility |
| Zig | Manual + Allocator + defer | No runtime, configurable | Programmer responsibility |
| Rust | Ownership + Drop | Compile-time safety, zero-cost | Handled by type system (Rc + Weak) |
| Swift | ARC (RC) | Deterministic destruction, ObjC compat | `weak` / `unowned` references |
| CPython | RC + cyclic GC | Legacy, deterministic destruction | Cyclic GC module (mark-sweep variant) |
| clox | Stop-the-world mark-sweep | Simplicity, educational | Handled automatically |
| Go | Concurrent mark-sweep | Low latency, large heaps | Handled automatically |
| V8/JS | Generational (Scavenger + Mark-Compact) | High allocation rate, long-running | Handled automatically |
| JVM | Generational (G1/ZGC) | High throughput + low latency | Handled automatically |

---

## 2. Mark-and-Sweep Implementation

### Tri-Color Invariant

During a mark phase, every object is in exactly one of three states:

- **White**: not yet discovered. After the mark phase completes, white objects are garbage and will be freed.
- **Gray**: discovered (reachable) but not yet fully traced — its references have not yet been scanned. The gray set is the "worklist."
- **Black**: fully traced — all of its references have been followed and their targets made at least gray. Black objects will survive.

**Algorithm**:
1. Mark all root objects gray (stack variables, globals, static data, JIT roots).
2. While the gray set is non-empty:
   - Pop one gray object.
   - For each pointer in that object: if the target is white, color it gray.
   - Color the popped object black.
3. At the end of marking: all white objects are unreachable. Free them.

**The key invariant** (strong tri-color invariant): No black object may point directly to a white object. If this invariant holds throughout marking, then when marking terminates all white objects are truly unreachable.

---

### Stop-the-World vs Incremental vs Concurrent

**Stop-the-world (STW)**:
- The simplest correct implementation.
- All application threads are paused for the entire mark phase.
- Pause time scales with live heap size — unacceptable for heaps > a few hundred MB in latency-sensitive applications.
- No write barriers needed: the mutator cannot change the object graph during GC.

**Incremental** [V8]:
- The mark phase is interleaved with mutator execution in small slices.
- Between increments, the mutator runs and may write new pointers. A **write barrier** must intercept these writes to keep the tri-color invariant valid.
- Does not reduce *total* GC time (may slightly increase it), but caps maximum pause duration.

**Concurrent** [Go, V8]:
- The mark phase runs on background threads while the mutator runs on the main thread(s).
- Write barriers are mandatory and more complex.
- V8: "Concurrent marking happens entirely in the background while JavaScript is executing on the main thread."
- Go: "Concurrent marking tasks are started" as the heap approaches its target limit.
- Most difficult to implement correctly (read/write races on object graph).

---

### Write Barriers: Why Needed for Concurrent GC

A write barrier is code that executes whenever a pointer is written. It is not needed for stop-the-world GC (the mutator is frozen), but is essential for concurrent and incremental GC.

**Problem**: Suppose during concurrent marking, object A (black) gains a reference to object C (white), and simultaneously object B (gray — the only prior path to C) loses its reference to C. Without a write barrier:
- C is now only reachable via A.
- A is black (already scanned), so the marker will not revisit A.
- C remains white and is freed — but it is still live (reachable via A).
- **This is a use-after-free bug.**

**Dijkstra insertion barrier** (forward-looking): When writing `A.field = C`, if A is black and C is white, mark C gray before the write. Ensures newly-added pointers are not missed. Used in V8 concurrent marking.

**Yuasa deletion barrier / SATB** (snapshot-at-the-beginning): When overwriting `A.field` (where A.field currently points to B), mark B gray before the overwrite. This preserves the "snapshot" of the object graph at the start of the GC cycle. Used in Go's GC.

**Go's hybrid write barrier** [Go]: Go 1.17+ uses a hybrid approach. When a pointer is written:
- Shade (gray) the old value (SATB style).
- Shade the new value if it is on the heap and the current goroutine's stack has not been scanned yet.
This avoids rescanning stacks mid-cycle, which was a source of pause-time overhead in earlier Go versions.

**Performance impact of write barriers** [Go]: Write barriers must be compiled into every pointer assignment. Go tried to avoid them in some approaches (ROC — request-oriented collector) but found the write-barrier cost acceptable for the concurrent mark-sweep approach compared to the alternatives.

---

### Heap Sizing Heuristics: When to Trigger a Collection

**clox**: Doubles the GC threshold after each collection. `nextGC = bytesAllocated * GC_HEAP_GROW_FACTOR` (factor 2). Self-tunes: large live sets → less frequent GC. A stress-test mode forces GC on every allocation.

**CPython** [CPython]: `gc.get_threshold()` returns `(threshold0, threshold1, threshold2)` defaults `(700, 10, 10)`. Collection starts when allocations minus deallocations exceeds `threshold0` (700). The fraction of the old generation included in each increment is inversely proportional to `threshold1`.

**Go Pacer** [Go]: Go uses a feedback loop called the "Pacer" (designed by Austin Clements). The pacer:
1. Monitors marking progress and allocation rate.
2. Targets completing the GC cycle just as the heap reaches `GOGC%` more than the live set size (default GOGC=100 means the heap can double before the next cycle).
3. If allocation is outpacing marking, the pacer slows goroutines doing heavy allocation by making them assist with marking (proportional to their allocation).
4. After each cycle, learns from this cycle and previous cycles to predict when to start the next one.

The two GC control knobs in Go [Go]:
- `GOGC` (default 100): ratio of heap growth to trigger next GC. `GOGC=100` means next GC fires when heap size = `liveSet * 2`.
- `GOMEMLIMIT` (formerly `MaxHeap`, not yet released in 2018): hard cap on heap size. When memory pressure is high, the GC can tell the application to shed load rather than OOM.

**V8** [V8]: Major GC starts with concurrent marking "as the heap approaches a dynamically computed limit." The limit is recalculated after each GC based on the live set size.

---

## 3. Reference Counting

### The Basic Algorithm

```
on_assign(object *old_target, object *new_target, field *f):
    if new_target != NULL:
        new_target->refcount++
    *f = new_target
    if old_target != NULL:
        old_target->refcount--
        if old_target->refcount == 0:
            free_object(old_target)

free_object(object *obj):
    for each pointer field f in obj:
        on_assign(f, NULL, &f)   // recursively decrement children
    release_memory(obj)
```

For thread safety, `refcount++` and `refcount--` must be atomic operations (compare-and-swap or explicit memory fences). This is the main throughput bottleneck of RC in multithreaded workloads.

---

### Cycle Detection: CPython's Cyclic GC Module

[CPython] CPython's main GC is RC. The `gc` module is a *supplementary* cycle detector that runs periodically to handle the cycles RC cannot collect.

**Key insight**: CPython only tracks *container* objects (lists, dicts, tuples, class instances, etc.) — objects that can contain references to other objects. Scalars (integers, floats) can never be part of a cycle.

**The algorithm** (simplified from CPython InternalDocs):

Each tracked container has a `PyGC_Head` prepended before the standard `PyObject_HEAD`, providing `_gc_next` and `_gc_prev` fields used to maintain doubly-linked lists (the "generations").

**Cycle detection steps**:
1. Create a copy of the reference count for each candidate object (stored as `gc_ref`).
2. Walk all candidates. For each reference from candidate A to candidate B, decrement `gc_ref` of B.
3. After this pass, objects with `gc_ref > 0` have external references (roots). Objects with `gc_ref == 0` are *tentatively* unreachable.
4. Traverse from rooted objects, marking all reachable objects by restoring their `gc_ref` to 1. Move these back to the reachable set.
5. Whatever remains with `gc_ref == 0` after all transitive reachability has been traced is genuinely unreachable — these form cycles and can be collected.

**Generations** [CPython]: CPython 3.13+ uses two-generation incremental collection:
- **Young generation**: newly allocated objects. Scanned every collection.
- **Old generation**: objects that survived one young-generation collection. Scanned incrementally.
- `gc.get_threshold()` controls collection frequency.
- Objects move from young → old after surviving a young collection.

**Destruction ordering** [CPython]: Destroying unreachable cyclic objects is delicate:
1. Handle weak reference callbacks.
2. Objects with legacy finalizers (`tp_del`) go to `gc.garbage` (not freed).
3. Call `tp_finalize` on finalizable objects.
4. Handle resurrection (finalizers may make unreachable objects reachable again).
5. Clear remaining weak references.
6. Call `tp_clear` on all unreachable objects to break internal links; `refcount` drops to zero, triggering deallocation.

---

### Swift ARC: Strong vs Weak vs Unowned

**Strong** (default): The reference count is incremented. The object lives as long as any strong reference exists.

**Weak** (`weak var` in Swift): Does NOT increment the reference count. Automatically set to `nil` when the object is deallocated. The referencing variable must be declared `Optional` (`T?`). Use to break ownership cycles (e.g., delegate patterns, child → parent back-references).

**Unowned** (`unowned` in Swift): Like weak, does NOT increment the reference count. But asserts the referenced object is non-nil on access — crashes (trap) if the object was already deallocated. Use when the programmer guarantees the object outlives the reference (e.g., closures capturing `self` in a view controller where `self` always outlives the closure).

**Cycle example**:
```swift
class Person {
    var apartment: Apartment?    // strong
}
class Apartment {
    weak var tenant: Person?     // weak — breaks the cycle
}
```
Without `weak`, `Person` and `Apartment` form a retain cycle: neither can be freed even when all external references are dropped.

---

### Performance Characteristics vs Tracing GC

| Property | Reference Counting | Tracing GC (mark-sweep/generational) |
|---|---|---|
| Reclamation latency | Immediate (deterministic) | Deferred (batch) |
| Pause time | None for normal dealloc; possible cascade on large structures | Stop-the-world or bounded concurrent pauses |
| Throughput | Overhead on every pointer write (atomic ops in MT) | Overhead proportional to live set at collection time |
| Space overhead | Per-object counter (typically 8 bytes) | Varies; generational needs write barrier data |
| Cycle handling | Cannot handle natively | Handles trivially |
| Cache behavior | Good for short-lived objects | Can be poor (tracing walks cold objects) |
| Predictability | High (local decision to free) | Lower (GC timing not directly programmer-controlled) |

---

## 4. Go's GC — The "Get GC Out of the Way" Philosophy

[Go] Source: Rick Hudson, ISMM keynote, July 2018.

### Core Design Philosophy

Go's GC goal is simple: **get GC out of the way of the programmer.** The latency SLO target by 2018 was 500 microseconds stop-the-world pause per GC cycle, down from 300–400 *milliseconds* in 2014. This is a 3-orders-of-magnitude improvement over four years.

The constraint that shaped all design decisions: **Go programs have value-oriented memory layouts, interior pointers, no JIT, a fast C FFI, and hundreds of thousands of goroutines.** This rules out moving collectors (interior pointers would break C code; no JIT to patch pointer references). Every GC innovation that works in JVM (card tables for exact interior pointer tracking, JIT support for GC maps) was unavailable or too costly.

---

### Concurrent Mark + Sweep, Tricolor, Write Barrier

Go uses a **non-moving, concurrent, tricolor mark-and-sweep collector**.

**Why non-moving**: Go's value-oriented layout (structs embedded by value, interior pointers, fast C FFI) means moving objects would require updating every interior pointer everywhere they are referenced, including in C stacks. Too complex, too many bugs.

**Why concurrent**: The "tyranny of the 9s" — at scale (e.g., 100 RPC requests per session), a 99th-percentile pause of 10ms means only 37% of users have a consistent sub-10ms experience across a session. To get 99% of users under 10ms, you need 99.99th-percentile pauses under 10ms. Only concurrent GC can achieve this.

**Mark phase**:
- Concurrent marking starts when the heap approaches the pacer's trigger point.
- Background goroutines walk the object graph, graying and then blackening objects.
- A hybrid **write barrier** intercepts all pointer writes on the main goroutine(s) during marking to maintain the tricolor invariant.
- When concurrent marking finishes (or the dynamic allocation limit is reached), the main thread performs a brief "marking finalization" step — the actual stop-the-world pause.

**Sweep phase**:
- Most sweeping is concurrent (background goroutines add freed memory back to size-segregated free lists).
- Memory is not returned to the OS immediately but is held for reuse.

**Key implementation detail** [Go]: Mark bits are stored on the side (not in object headers), encoded as 2-bit fields in a bitmap. This avoids touching object memory during mark (better cache behavior). The bitmap also encodes pointer/scalar information so the GC can stop scanning an object as soon as it reaches the last pointer field.

---

### The Pacer: Controlling GC Pacing

[Go] The Pacer is a feedback controller that determines when to start and how fast to run each GC cycle.

**Goal**: Finish each GC cycle (complete marking) just as the heap reaches the target size (`liveSet * (1 + GOGC/100)`). Starting too late → heap grows beyond the target, wasting memory. Starting too early → wasted CPU on GC.

**Mechanism**:
1. Track allocation rate (bytes/sec) and marking rate (bytes/sec).
2. Estimate: at the current allocation rate, when will the heap reach the target?
3. Ensure marking started early enough that marking completes by that time.
4. If allocation rate spikes (allocation outpacing marking): invoke "GC assists" — goroutines doing heavy allocation are made to assist with marking, proportional to the number of bytes they allocated. This slows the mutator while speeding up the GC.

**GC assist is self-regulating**: A goroutine that allocates a lot must assist a lot, creating a natural feedback loop that prevents the GC from falling behind.

**Result** [Go]: In steady state, GC assists are near zero. The pacer learns from previous cycles and projects start times accurately. The total GC CPU target is configurable (default: 25% of available CPUs during a GC cycle).

---

### Key Design Decisions and Their Rationale

| Decision | Rationale |
|---|---|
| Non-moving collector | Interior pointers + fast C FFI made moving impossible |
| Tricolor concurrent mark | Low latency; avoids stop-the-world for marking |
| Write barriers (hybrid SATB + insertion) | Required for concurrent marking correctness |
| Size-segregated spans | Efficient interior pointer handling (round down to span-size to find object start); low fragmentation |
| Mark bits stored on the side | Avoids touching object memory during mark (cache-friendly) |
| No generational GC (as of 2018) | Write barrier always-on cost outweighed benefits; escape analysis reducing heap allocation; value-orientation reduces pointer chasing |
| Pacer feedback controller | Prevents over/under-triggering GC; adapts to workload |
| GC assists | Self-regulating mechanism to handle allocation rate spikes |

**Why Go chose not to do generational GC** [Go]: Go experimented with a "sticky bits" generational collector and a "request-oriented collector" (ROC). Both were abandoned:
- ROC: required an always-on write barrier; 30–50% slowdown on compiler benchmarks (Go compilers have many pointer writes).
- Generational GC: write barrier overhead (always-on during minor collections) outweighed the savings, especially because escape analysis was getting better at stack-allocating objects that would otherwise be short-lived heap objects.

---

## 5. Manual Memory + Defer (Zig)

[Zig] Source: Zig Language Reference, Memory section.

### The Allocator Interface: Why Every Allocation Takes an Allocator

**Core principle** [Zig]: "The Zig language performs no memory management on behalf of the programmer. This is why Zig has no runtime." Every function that needs to allocate heap memory accepts an `std.mem.Allocator` parameter. There is no `malloc` equivalent called implicitly.

```zig
// A function that allocates takes an Allocator
fn concat(allocator: Allocator, a: []const u8, b: []const u8) ![]u8 {
    const result = try allocator.alloc(u8, a.len + b.len);
    @memcpy(result[0..a.len], a);
    @memcpy(result[a.len..], b);
    return result;
}
```

**Why this design** [Zig]:
- The caller decides the allocation strategy, not the library. A library can be used with any allocator.
- Tests can pass a `std.testing.allocator` that detects leaks.
- Performance-critical code can pass an `ArenaAllocator` that frees everything at once.
- Embedded code can pass a `FixedBufferAllocator` that uses a static buffer with no heap at all.
- There is no implicit global state; subsystems are isolated.

**Allocator interface** (`std.mem.Allocator`): Provides `alloc`, `realloc`, `free`, and `create`/`destroy` (for single objects). The allocator is a fat pointer (pointer to implementation + vtable), enabling runtime polymorphism without overhead in the common case.

---

### `defer` for Cleanup: How It Compiles

```zig
const buf = try allocator.alloc(u8, 1024);
defer allocator.free(buf);
// ... use buf ...
// buf.free() is inserted at every exit path from this scope
```

`defer` statements are executed in LIFO order when the enclosing scope exits — regardless of whether the exit is normal return, early return via error propagation, or `unreachable`. `errdefer` is similar but only triggers on error paths.

**How it compiles**: The compiler transforms `defer` into code that runs at each exit point of the enclosing scope. For a function with multiple return sites, the deferred cleanup is duplicated at each exit. For error-propagating code, `errdefer` is only inserted at error-returning paths. This is a compile-time rewrite, not a runtime stack of closures — zero runtime overhead for tracking deferred operations.

```zig
// errdefer: run cleanup only if this scope exits with an error
const connection = try openConnection();
errdefer connection.close();
const data = try connection.read();  // if this fails, connection.close() runs
```

---

### ArenaAllocator, FixedBufferAllocator, GeneralPurposeAllocator / DebugAllocator

**Choosing an Allocator** [Zig] (from the official decision flowchart):

1. **Writing a library?** → Accept `Allocator` as a parameter. Let the caller decide.
2. **Linking libc?** → `std.heap.c_allocator` wraps `malloc`/`free`.
3. **Maximum bytes bounded at comptime?** → `std.heap.FixedBufferAllocator` — allocates from a stack buffer. Zero heap traffic.
4. **Command-line tool that frees everything at exit?** → `std.heap.ArenaAllocator` wrapping `std.heap.page_allocator`:
   ```zig
   var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
   defer arena.deinit();
   const allocator = arena.allocator();
   // ... allocate as much as needed, never call free individually ...
   // arena.deinit() frees everything at once
   ```
5. **Cyclical pattern (game loop, web request handler)?** → `ArenaAllocator` per cycle — reset the arena at the end of each cycle.
6. **Writing a test?** → `std.testing.allocator` (also a leak detector).
7. **General purpose, debug mode?** → `std.heap.DebugAllocator` — detects leaks, double-frees, use-after-free.
8. **General purpose, release mode?** → `std.heap.smp_allocator` — high-performance multi-threaded allocator.

**ArenaAllocator**: Backs a series of allocations with large pages. `free()` on individual allocations is a no-op. `arena.deinit()` frees everything at once. Ideal for "parse a request, produce a response, throw everything away."

**FixedBufferAllocator**: Takes a static `[]u8` buffer at initialization. All allocations come from this buffer. Fails with `error.OutOfMemory` when the buffer is exhausted. Zero OS interaction.

**DebugAllocator** (formerly `GeneralPurposeAllocator`): In debug builds, tracks all allocations. On `deinit()`, reports any leaked allocations with the allocation stack trace. Detects double-frees and use-after-free by poisoning freed memory.

---

### Testing for Memory Leaks with DebugAllocator / Testing Allocator

[Zig] `std.testing.allocator` is a `DebugAllocator` configured for test builds. The default test runner automatically reports any leaks:

```zig
test "detect leak" {
    const gpa = std.testing.allocator;
    var list: std.ArrayList(u21) = .empty;
    // missing: defer list.deinit(gpa);
    try list.append(gpa, '☔');
    try std.testing.expectEqual(1, list.items.len);
}
// Output: [DebugAllocator] (err): memory address 0x... leaked:
//         (stack trace to the allocation site)
```

The allocator records the stack trace at each allocation. On deallocation mismatch, it prints exactly where the leaked memory was allocated. This gives actionable error messages without requiring a separate tool like Valgrind.

---

## 6. Stack Frames and Calling Conventions

### Frame Layout (x86-64, System V ABI)

```
  High addresses (older frames)
  ┌─────────────────────────────┐
  │  ... caller's frame ...     │
  ├─────────────────────────────┤  ← caller's %rsp before `call` instruction
  │  return address             │  pushed by `call` (8 bytes)
  ├─────────────────────────────┤  ← callee's %rbp (if frame pointer used)
  │  saved %rbp                 │  (8 bytes; optional)
  ├─────────────────────────────┤
  │  callee-saved registers     │  %rbx, %r12–%r15 (if callee uses them)
  ├─────────────────────────────┤
  │  local variables + spills   │  fixed offsets from %rbp (or %rsp)
  ├─────────────────────────────┤
  │  alignment padding          │  %rsp must be 16-byte aligned before `call`
  ├─────────────────────────────┤  ← current %rsp
  │  [ red zone — 128 bytes ]   │  below %rsp, not in the frame
  └─────────────────────────────┘
  Low addresses (newer frames)
```

**Return address**: Pushed onto the stack by the `call` instruction. Points to the instruction immediately after the `call` in the caller. The callee's `ret` instruction pops it and jumps there.

**Saved %rbp**: Optional. When a frame pointer is used (`push %rbp; mov %rsp, %rbp`), unwinding and debugging become simpler — each frame's base address is findable by following the chain of saved %rbp values. Omitting it (`-fomit-frame-pointer`) frees one register and removes prologue/epilogue overhead; DWARF `.eh_frame` unwinding doesn't require it.

**Callee-saved registers** (System V ABI): `%rbx`, `%rbp`, `%r12`–`%r15`. The callee must restore these before returning if it uses them. Caller-saved registers (`%rax`, `%rcx`, `%rdx`, `%rsi`, `%rdi`, `%r8`–`%r11`) can be freely clobbered.

**Slot assignment**: The compiler assigns each local variable (and spilled register) a fixed offset from `%rbp` (or `%rsp` in frameless functions). Stack allocation of a local is free — just adjusting `%rsp` at function entry by the total frame size.

---

### Stack Growth: Fixed-Size vs Segmented vs Stack-Copying

**Fixed-size stacks** (most C programs): A single contiguous block (typically 8 MB on Linux, 1 MB on Windows). Stack overflow → segfault. Simple and fast; every function call is just a `%rsp` decrement.

**Segmented stacks** (early Go): When a goroutine's stack overflows, the runtime allocates a new, larger segment and links it to the old one. A "hot split" performance problem was discovered: code near a segment boundary pays a heavy allocation cost on every call that crosses the boundary. Abandoned in Go 1.4.

**Stack-copying (goroutines, current Go)** [Go]: Goroutines start with a small stack (currently 2 KB). When the stack would overflow, the runtime:
1. Allocates a new, larger stack (2x the current size).
2. Copies the entire old stack to the new stack.
3. Updates all pointers that pointed into the old stack (using GC stack maps that record pointer locations).
4. Resumes on the new stack.

Stack-copying is why Go requires that the GC be able to precisely locate all pointers in a goroutine's stack. The stack maps are generated by the compiler and stored in the binary. "We manage the stacks and their size by copying them and updating pointers in the stack. It's a local operation so it scales fairly well." [Go]

**Goroutine scheduler**: The Go scheduler multiplexes goroutines (M:N threading). Many goroutines → few OS threads. The small initial stack size makes it practical to have hundreds of thousands of goroutines concurrently.

---

### Red Zone: The 128-Byte Scratch Area

The System V AMD64 ABI defines a "red zone" — 128 bytes below the current stack pointer (`%rsp`) that are guaranteed not to be clobbered by signal handlers or interrupt routines on most OS configurations. Leaf functions (functions that do not call other functions) can use this area as scratch space without adjusting `%rsp`, saving the prologue/epilogue overhead.

```asm
; Leaf function using red zone (no sub rsp, N needed)
foo:
    movq %rdi, -8(%rsp)    ; save argument in red zone
    movq %rsi, -16(%rsp)   ; save second argument
    ; ... computation using -8(%rsp) and -16(%rsp) ...
    ret
```

**Caveats**: The red zone is not available in kernel code (interrupts can fire at any time and will clobber memory below `%rsp`) or in code that sets up signal handlers that might be delivered to this stack.

---

## 7. Panic / Exception Handling

[Rust] Source: Rust Compiler Development Guide, Panic Implementation.

### `setjmp` / `longjmp`: Simple, No Destructors

`setjmp(jmp_buf env)` saves the current machine state (register values including `%rsp`, `%rbp`, `%rip`) into `env` and returns 0. `longjmp(jmp_buf env, int val)` restores that state — effectively resuming execution at the `setjmp` site with `setjmp` returning `val`.

```c
jmp_buf env;
if (setjmp(env) == 0) {
    // normal execution
    might_throw(&env);
} else {
    // exception handler
    fprintf(stderr, "caught!\n");
}

void might_throw(jmp_buf *env) {
    longjmp(*env, 1);  // jump back to setjmp site
}
```

**Cost**: Every `try`-equivalent point must save a `jmp_buf` on the stack — non-zero cost on the happy (no-exception) path. Saving all registers on every potential throw site is expensive for code that almost never throws.

**Critical limitation**: Destructors, RAII cleanup, and `defer` do NOT run. `longjmp` unwinds the stack by simply overwriting registers; it does not execute any code in intermediate frames. This makes `setjmp`/`longjmp` inappropriate for C++ exceptions, Rust panics, or any language with RAII semantics.

**Use when**: Simple scripting interpreters, C error handling without RAII, situations where cleanup can be managed explicitly after the `longjmp`.

---

### Table-Based Zero-Cost Unwinding (Itanium ABI, DWARF `.eh_frame`)

The dominant exception model for compiled languages (C++, Rust, Swift on most platforms).

**Key insight**: Make the happy path (no exception) have *zero* overhead. Pay the cost only when an exception actually occurs. This is called "zero-cost exceptions."

**How it works**:
1. The compiler emits **unwind tables** for each function — data (not code) describing:
   - How to restore callee-saved registers for this frame.
   - Where the return address is stored.
   - Which "landing pads" (exception handlers / destructor cleanup code) exist in this function, and for what exception types.
   This data is stored in `.eh_frame` (ELF) or `.pdata`/`.xdata` (PE/COFF) sections.

2. On the **happy path**: no branches, no register saves — the unwind tables are dead data that is never consulted. True zero cost.

3. **When a `throw` / `panic` occurs**:
   - The runtime exception library walks the unwind tables for the current call stack.
   - For each frame, it checks if that frame has a landing pad that handles the current exception.
   - If yes: restore registers for that frame, jump to the landing pad (which calls destructors, then re-throws or handles).
   - If no handler found: call `terminate()`.

4. **DWARF `.eh_frame`**: Encodes frame unwinding instructions in a compact bytecode. The Itanium C++ ABI defines the format; it is used by GCC, Clang, Rust, and Go on Linux.

**Cost**: Landing pads add code to the binary. The unwind tables add read-only data to the binary. Neither adds cost to the instruction stream of the normal execution path.

**Use when**: RAII semantics, C++ destructors, Rust `Drop`, Swift `defer` — any language where cleanup must run during unwinding.

---

### Rust Panic: Unwind or Abort, the `?` Operator

[Rust] In Rust, `panic!` flows through two stages:

**Stage 1 — Invoking the panic macro**:
- `core::panic!` calls `panic_impl` (a `#[lang = "panic_impl"]` lang item with symbol `rust_begin_unwind`).
- `std` provides the implementation: `begin_panic_handler` which is marked `#[panic_handler]` and compiled to the symbol `rust_begin_unwind`.
- At link time, the `core` reference to `rust_begin_unwind` resolves to std's implementation.
- Control passes to `rust_panic_with_hook`, which invokes the global panic hook and checks for double panics.
- Finally calls `__rust_start_panic` — the platform-specific unwinding entry point.

**Stage 2 — The panic runtime**:
Rust provides two panic runtimes, selected at build time:

- **`panic_abort`**: `__rust_start_panic` simply aborts the process. Minimal code size. Used for embedded targets, performance-critical builds.
- **`panic_unwind`**: `__rust_start_panic` unwinds the stack using the platform's unwinding library (libunwind or DWARF `.eh_frame`). Calls all `Drop::drop` implementations ("destructors") for values in frames being unwound. Used for programs that use `std::panic::catch_unwind`.

**The `?` operator as structured error propagation**: Rust strongly discourages using panics for control flow. Instead, functions return `Result<T, E>`. The `?` operator desugars to:
```rust
match expr {
    Ok(val) => val,
    Err(e) => return Err(e.into()),
}
```
This is pure value-passing error propagation with no unwinding, no exception tables, and no landing pads. The compiler can optimize it as aggressively as any other control flow.

**`catch_unwind`**: Wraps a closure and catches any panic that would otherwise unwind past it, converting it into a `Result`. Used in test harnesses and FFI boundaries. In `panic=abort` mode, `catch_unwind` does not stop the abort.

---

### Go `panic` / `recover`: Deferred Functions Run During Unwind

Go's panic model is simpler than Rust's or C++'s but shares the "run cleanup on unwind" property.

**`panic(v)`**: Begins unwinding the current goroutine's stack. As the stack unwinds, deferred functions (registered with `defer`) are executed in LIFO order. This is how deferred cleanup (closing files, unlocking mutexes) runs during a panic.

**`recover()`**: When called inside a deferred function, `recover()` stops the panic and returns the panic value. The goroutine resumes normally (execution continues in the function containing the `recover` call, which is the deferred function's caller after all deferred functions have run). If `recover()` is not called, the panic propagates up the goroutine's stack and eventually terminates the goroutine (and if not caught at the goroutine's root, the entire program).

```go
func safeDiv(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered: %v", r)
        }
    }()
    return a / b, nil  // panics if b == 0
}
```

**Implementation**: Go's `defer` is tracked in the goroutine's G struct at runtime (not as DWARF unwind tables). Each `defer` statement adds an entry to a list; the runtime walks this list during panic unwinding. This is simpler than table-based unwinding but has a small runtime cost for each `defer` statement at its point of registration (though recent Go versions have optimized many common cases to zero cost using static analysis).

---

## 8. Coroutines and Async

### Stackful Coroutines (Goroutines, Green Threads)

**Definition**: A stackful coroutine has its own full call stack. Suspending the coroutine saves the entire stack (or simply leaves it in place on the heap). Resuming restores execution at the exact point where it was suspended.

**Goroutines** [Go]: Each goroutine has its own stack (starts at 2 KB, grows via stack-copying). Goroutines are multiplexed onto OS threads by the Go scheduler. Suspension points are at I/O operations, channel operations, and GC safepoints. Suspension saves the current `%rsp` and a few registers into the goroutine's `G` struct; the OS thread is then free to run another goroutine.

**Advantages**:
- Arbitrary call depth at suspension: a coroutine can suspend inside deeply nested function calls. No restructuring of code required.
- Familiar sequential programming model.
- Easy to integrate with blocking system calls (the scheduler detects when a goroutine blocks on a syscall and moves it to a separate OS thread).

**Disadvantages**:
- Memory: each goroutine needs at least a minimal stack (~2 KB). With 1 million goroutines, that is 2 GB just for stacks.
- Stack growth/copying has overhead when stacks need to grow.
- Pointer validity: when stacks are copied, all pointers into the stack must be updated (requires precise GC stack maps).

**Segmented stacks (historical Go)**: Each stack frame was allocated as a separate heap block. Solved the memory problem but introduced "hot split" performance pathology. Abandoned for stack-copying.

---

### Stackless Coroutines (Rust `async`/`await`, Zig `async`)

**Definition**: A stackless coroutine has *no* dedicated stack. When the coroutine suspends, only the local state that is live across the suspension point is saved, packed into a state machine struct. When the coroutine resumes, it restores that struct and jumps to the appropriate branch of the state machine.

**The state machine transformation** [Rust]:
```rust
async fn fetch_data(url: &str) -> Result<Data, Error> {
    let response = client.get(url).await?;   // suspension point 1
    let text = response.text().await?;       // suspension point 2
    parse(text)
}
```
The compiler transforms this into an enum-based state machine:
```rust
enum FetchDataFuture<'a> {
    Initial { url: &'a str },
    AwaitingGet { /* live vars across .await */ },
    AwaitingText { response: Response },
    Done,
}
impl Future for FetchDataFuture<'_> {
    fn poll(&mut self, cx: &mut Context) -> Poll<Result<Data, Error>> {
        match self {
            FetchDataFuture::Initial { url } => { /* start GET */ }
            FetchDataFuture::AwaitingGet { ... } => { /* check if GET done */ }
            FetchDataFuture::AwaitingText { response } => { /* read text */ }
            FetchDataFuture::Done => panic!("polled after completion"),
        }
    }
}
```
No stack needed — the state struct is allocated once (on the heap or inline in an enclosing future) and reused across all polls.

**Zig async** (note: experimental, see Zig 0.3.0 release notes): Zig had a similar compile-time async transformation. An `async fn` was rewritten into a state machine; the frame struct could be stack-allocated by the caller. This gave zero-heap-allocation async — the caller could allocate the coroutine's frame on its own stack.

**Advantages of stackless**:
- Minimal memory per suspended coroutine: only the live local variables (often tens of bytes).
- No stack growth complications.
- The state machine is a regular struct — can be embedded in other structs, allocated on the stack, stored in arrays.

**Disadvantages**:
- **Color problem**: `async` functions must be called with `await`; they are a different "color" from regular functions. Libraries must be written as async to work in async contexts, or must be duplicated.
- Cannot suspend inside a function call that the coroutine does not own (e.g., cannot `await` inside a closure passed to `sort`). Code must be restructured to use only suspension at top-level `await` expressions.
- More complex error messages and debugging (the state machine is hard to read).

---

### The Suspend Point Problem: What State Must Be Saved

At each suspension point (`await`, channel receive, etc.), the coroutine implementation must save all state that is **live** across the suspension:

- **Local variables**: variables declared before the suspension and used after it. The state machine transformation identifies these via liveness analysis.
- **Arguments**: function arguments that are still needed.
- **Intermediate expressions**: if the programmer writes `let x = (a + b).await?`, the partial computation `a + b` must be saved if `a + b` itself is a future.
- **The program counter**: which suspension point to resume at. In a state machine, this is the enum variant.

**What does NOT need to be saved**:
- Variables that go out of scope before any suspension point.
- Temporary values whose lifetimes do not cross a suspension point.
- The OS stack — the stackless model allocates no OS stack per coroutine.

**Goroutine comparison** [Go]: For goroutines, the "state" is the entire OS-like stack — all local variables, return addresses, and callee-saved registers for all frames on the goroutine's stack. This is much larger per suspended coroutine but requires zero restructuring of code.

---

## Cross-Reference Index

| Topic | Tags | See Also |
|---|---|---|
| Concurrent GC tricolor algorithm | [Go], [V8] | Section 2, Section 4 |
| Write barriers | [Go], [V8] | Section 2.3 |
| GC pacer / pacing | [Go] | Section 4 |
| Reference counting + cycles | [CPython] | Section 3 |
| Cyclic GC algorithm | [CPython] | Section 3.2 |
| Generational hypothesis | [V8], [CPython] | Section 1, Section 3.2, Section 5 |
| Scavenger (minor GC) | [V8] | Section 1 |
| Allocator interface | [Zig] | Section 5 |
| defer / errdefer | [Zig] | Section 5.2 |
| Stack-copying goroutines | [Go] | Section 6.2 |
| Panic implementation | [Rust] | Section 7.3 |
| Zero-cost unwinding | [Rust], general | Section 7.2 |
| Async state machine | [Rust] | Section 8.2 |
| Goroutines vs stackless async | [Go], [Rust] | Section 8 |
