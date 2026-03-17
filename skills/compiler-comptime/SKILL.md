---
name: compiler-comptime
description: Compile-time execution and metaprogramming expert — covers the spectrum from C macros to Zig comptime to dependent types, compile-time/runtime boundary design, generic programming implementation strategies, and constant evaluation. Use when designing metaprogramming facilities, implementing generics, or deciding where to draw the compile-time/runtime boundary.
---

# Compile-Time Execution and Metaprogramming

---

## The Metaprogramming Spectrum

A continuum of power, complexity, and safety:

| Mechanism | Example Languages | How It Works | Safety |
|---|---|---|---|
| Text substitution (macros) | C preprocessor | String replacement before parsing | None — no hygiene, no scope |
| Syntactic macros | Rust `macro_rules!` | Pattern-match token trees, expand to AST | Hygienic by construction |
| Procedural macros | Rust proc-macros, Lisp | Arbitrary code runs on token stream → new tokens | Turing-complete, complex |
| Template metaprogramming | C++ | Type-level computation; Turing-complete by accident | Poor error messages, slow |
| Comptime (value-level) | Zig | Ordinary code marked `comptime`; type is a value | Clean, predictable |
| Staged computation | MetaML, Terra | Explicit quote/splice to control execution stages | Formal, powerful |
| Dependent types | Idris, Lean | Types can depend on values; computation and types unified | Maximum power, high cost |

**Design principle**: choose the weakest mechanism that solves your problem. Each step up the spectrum multiplies expressiveness and implementation complexity.

---

## Compile-Time / Runtime Boundary

### How to Decide What Runs When

Questions to ask about an expression or computation:
1. Are all inputs known at compile time? If yes, it *can* run at compile time.
2. Is the result needed to determine the *type* or *size* of something? If yes, it *must* run at compile time.
3. Is the result needed to generate code (e.g., loop unrolling, specialization)? Compile time.
4. Does it have side effects the user expects to observe at runtime? Runtime.
5. Is it too expensive to compute at compile time (e.g., network I/O, complex search)? Runtime.

### Boundary Design Choices

**Explicit boundary (Zig style)**: programmer marks expressions `comptime`. The compiler evaluates them at compile time; everything else is runtime. Clear, explicit, no surprise phase.

**Implicit constant propagation (C/LLVM style)**: the compiler infers what it can evaluate from constant inputs. No programmer annotation. Simpler to use, harder to reason about what actually runs when.

**Staged programming (MetaML style)**: explicit quotation (`<e>`) and splicing (`~e`) brackets wrap code that runs in a later stage. Formally sound, verbose, rarely used in systems languages.

**Dependent types (Idris/Lean)**: the type system and value system are unified. A type *is* a value. No hard boundary — the elaborator decides what must reduce. Most powerful, most complex.

---

## Zig comptime

### Core Principle

In Zig, `comptime` is not a special metaprogramming sublanguage. It is ordinary Zig code, run at compile time. The same type system, the same semantics, the same syntax.

### Types as Values

In Zig, `type` is a first-class value with a comptime-only type called `type`:
```zig
fn makeSlice(comptime T: type) type {
    return struct {
        ptr: [*]T,
        len: usize,
    };
}
const IntSlice = makeSlice(i32);
```

This replaces C++ templates with ordinary function calls that happen to return types.

### Comptime Parameters

A function parameter declared `comptime` must be known at compile time at every call site. The compiler monomorphizes the function for each distinct comptime argument value:
```zig
fn identity(comptime T: type, value: T) T {
    return value;
}
```

### Inline Control Flow

`comptime` blocks and `inline for` / `inline while` enable compile-time iteration:
```zig
inline for (fields) |field| {
    // runs once per field at compile time; body specialized per field
}
```

### What Cannot Run at Comptime

- Code with runtime-only side effects (I/O, syscalls)
- Pointer arithmetic to addresses that are not comptime-known
- Functions that call into external C libraries (unless marked `comptime`)

---

## Generic Programming: Three Implementation Strategies

### 1. Monomorphization (Zig, Rust, C++ templates)

The compiler generates a separate copy of the generic function/type for each distinct type argument used.

**Pros**: Zero runtime overhead (no boxing, no dispatch); full optimization per instantiation; type errors are precise.

**Cons**: Binary size grows with number of instantiations; compile time grows; not suitable for dynamic dispatch.

### 2. Type Erasure (Java generics, Go pre-1.18 interfaces)

All type parameters are erased to a base type (e.g., `Object`, `interface{}`). A single copy of the code runs for all types, with boxing for value types.

**Pros**: Single binary copy; supports open-world extension; fast compile.

**Cons**: Boxing overhead for value types; no specialization; type safety enforced by casts at runtime boundaries.

### 3. Dictionary Passing (Haskell typeclasses, Rust traits in some codegen)

A dictionary (vtable) of methods for the concrete type is passed as an implicit argument. The generic code dispatches through the dictionary.

**Pros**: Supports dynamic dispatch naturally; no code duplication; supports existential types.

**Cons**: Indirect dispatch overhead; dictionary construction; harder to inline.

### Decision Guide

```
Is this a performance-critical path?
  YES → Monomorphization (Zig/Rust style)
  NO  → Does the set of concrete types need to be open (unknown at compile time)?
          YES → Dictionary passing / type erasure
          NO  → Either works; monomorphization preferred for safety and perf
```

---

## Constant Evaluation

### What Can Be Evaluated at Compile Time

- Integer and float arithmetic on constant operands
- Bitwise operations
- String concatenation (if strings are arrays/slices of constants)
- Control flow over constant conditions
- Function calls where all arguments are constants and the function has no side effects
- Array access with constant indices
- Struct field access
- Type computations (in languages with type-as-value)

### When to Error vs. When to Defer

**Error at compile time** when:
- A value is required at compile time (array length, `comptime` parameter) but is not known
- An operation that is undefined behavior at runtime (integer overflow in Zig comptime) is attempted at compile time — catching it here is better than silently emitting wrong code

**Defer to runtime** when:
- The value *could* be computed at compile time but there is no requirement that it be
- Evaluating it at compile time would significantly increase compile time without benefit

### Constant Folding vs. Constant Propagation

- **Constant folding**: fold a single expression with constant operands: `3 * 4` → `12`
- **Constant propagation (SSCP)**: propagate known-constant values through data flow, enabling further folding downstream

Both are standard optimization passes. In a comptime system, they are applied eagerly and mandatorily (not optionally).

---

## Deep Reference

For full coverage with code examples, syntax cheat sheets, implementation details, and pitfalls, see:

**[references/comptime-reference.md](references/comptime-reference.md)**

Topics covered in depth:
- The full metaprogramming spectrum (C macros → token macros → proc macros → templates → comptime → dependent types) with decision guide
- Zig comptime: `comptime` keyword semantics (parameters, variables, blocks), types as first-class values, `@TypeOf`/`@typeInfo`/`@typeName`, `inline for`/`inline while`, container-level comptime, the `print` case study, what cannot run at comptime, and how the Zig interpreter works
- Rust constant evaluation: `const fn` restrictions, const expressions (exhaustive list from the reference), const contexts, `const` vs `static` items, MIRI as the const evaluator, const generics (`const N: usize`), allowed const parameter types, standalone restriction, ambiguity resolution, exhaustiveness limitation, and `where` clauses
- Compile-time/runtime boundary design: explicit annotation (Zig) vs context-driven (Rust) vs implicit (C), staged programming (MetaML brackets and splices), partial evaluation and Futamura projections, monomorphization vs dictionary passing performance tradeoff
- Generic programming strategies: monomorphization, type erasure/boxing, dictionary passing (static and dynamic), Zig's unified comptime approach, decision table
- Compiler implementation: what an interpreter needs, Zig's full interpreter, Rust's MIR interpreter (MIRI), C++ constexpr evaluators, challenges (I/O, pointer validity, recursion limits, error quality, non-determinism)
- Common pitfalls: comptime explosion, monomorphization bloat, const evaluation error quality, "not comptime-known" vs logic error, const generics trait coherence limitation

Sources researched:
- https://ziglang.org/documentation/master/#comptime
- https://ziglang.org/documentation/master/#Generic-Data-Structures
- https://doc.rust-lang.org/reference/const_eval.html
- https://doc.rust-lang.org/reference/items/constant-items.html
- https://doc.rust-lang.org/reference/items/generics.html#const-generics

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-architect` — comptime boundary design is an architectural decision
- `compiler-expert` — routes here on generics implementation / compile-time execution questions

**Trace Down ↓** — this is a leaf skill; it interacts laterally
- (none — comptime evaluation is layered on top of the existing IR pipeline)

**Related →** — closely related skills
- `compiler-macro` — comptime is the "single-language" alternative to a macro system
- `compiler-type-system` — monomorphization and generic instantiation happen here
- `compiler-optimizer` — compile-time evaluation is a form of partial evaluation / constant folding
