# Compile-Time Execution: Deep Reference

Sources researched:
- https://ziglang.org/documentation/master/#comptime (Zig language reference, comptime section)
- https://ziglang.org/documentation/master/#Generic-Data-Structures (Zig generic data structures)
- https://doc.rust-lang.org/reference/const_eval.html (Rust reference: constant evaluation)
- https://doc.rust-lang.org/reference/items/constant-items.html (Rust reference: constant items)
- https://doc.rust-lang.org/reference/items/generics.html#const-generics (Rust reference: const generics)

---

## 1. The Metaprogramming Spectrum

Metaprogramming means writing code that operates on or generates other code. The mechanisms span a wide spectrum of power and safety.

### 1.1 Text Substitution — C Preprocessor Macros

The C preprocessor runs before parsing. It sees the source as a stream of tokens and performs textual replacement:

```c
#define MAX(a, b) ((a) > (b) ? (a) : (b))
```

**Characteristics**:
- No type information whatsoever — the preprocessor does not parse types
- No scope hygiene — identifiers capture from the surrounding context
- No recursion limit enforced by the language spec (implementation-defined)
- Order of evaluation problems: `MAX(x++, y++)` evaluates `x++` twice
- Results in completely different code for each instantiation (textual substitution)

**When to use**: Almost never for new code. Legacy interop, platform-specific header guards, compile-time string concatenation when nothing else is available.

### 1.2 Token Macros — Rust `macro_rules!`

Rust `macro_rules!` pattern-match on token trees (not on the full AST) and expand to new token trees. They are hygienic: identifiers introduced inside a macro do not accidentally capture identifiers from the call site.

```rust
macro_rules! vec_literal {
    ( $( $x:expr ),* ) => {
        {
            let mut v = Vec::new();
            $( v.push($x); )*
            v
        }
    };
}
```

**Characteristics**:
- Hygienic — macro-introduced bindings are scoped to the macro expansion
- Pattern matching is on token trees, not semantic AST nodes
- Pre-semantic: the macro sees tokens, not types
- Cannot express arbitrary type-directed behavior (for that, use proc macros or traits)
- Expansion happens before type inference

**When to use**: Repetitive syntactic boilerplate that follows a pattern. Implementing DSLs that map closely to Rust syntax. Replacing C-style variadic function-like macros.

### 1.3 Procedural Macros — Rust Proc Macros

Rust procedural macros are Rust functions compiled into a special crate that receives a `TokenStream` and returns a `TokenStream`. They run at compile time in the compiler host process.

Three forms:
- `#[derive(MyTrait)]`: auto-implements a trait for a struct/enum
- `#[my_attribute]`: transforms an item
- `my_macro!(...)`: function-like, full control

**Characteristics**:
- Arbitrary computation at compile time (the proc macro is a full Rust program)
- Can inspect the AST via the `syn` crate (parses token stream into semantic tree)
- Operates before type checking — cannot query types directly
- Errors are reported as token-level spans
- The proc macro crate must compile for the host, not the target

**When to use**: Deriving complex trait implementations. Generating code from external schemas (SQL, protobuf). Any case where `macro_rules!` is insufficient because you need to inspect struct field names or types.

### 1.4 Template Metaprogramming — C++ Templates

C++ templates were designed for code parameterization but turned out to be Turing-complete at the type level (Veldhuizen 2003). The compilation model: when a template is instantiated with a concrete type, the compiler generates a new copy of the function/class for that type.

```cpp
template<typename T>
T max(T a, T b) { return a > b ? a : b; }
```

Turing-completeness emerged from:
- Template specialization (pattern matching on types)
- Recursive template instantiation (loops)
- Integer template parameters as values
- `static_assert` and `std::enable_if` for conditions

**Characteristics**:
- Error messages are famously incomprehensible (the instantiation stack can be hundreds of lines)
- Slow compilation: each instantiation recompiles the full template body
- No explicit separation between "type-level" and "value-level" computation
- `constexpr` (C++11/14/17/20) added explicit compile-time value computation

**When to use**: C++ interop or existing C++ codebase. Otherwise, modern alternatives (Rust const generics, Zig comptime) are cleaner.

### 1.5 Comptime — Zig

Zig's `comptime` is the central mechanism for both metaprogramming and generics. It is not a separate sublanguage: the same Zig code that runs at runtime can run at compile time, with the `comptime` keyword marking what must be evaluated compile-time.

See Section 2 for full details. [Zig]

### 1.6 Dependent Types — Lean 4 / Agda / Idris

In a dependently typed language, types can depend on values. The type of a vector might be `Vec T n` where `n : Nat` is the length — a runtime value. Propositions are types; proofs are programs (Curry-Howard correspondence).

```lean4
-- n is a natural number VALUE in the type
def replicate {α : Type} (n : Nat) (a : α) : Vector α n := ...
```

**Characteristics**:
- The compile-time / runtime distinction is replaced by the elaborator deciding what must reduce for type checking
- Can express invariants that are impossible in simpler systems (e.g., sorted lists, size-indexed arrays)
- High cognitive overhead; complex type errors
- Primarily used in proof assistants and research languages; production use is rare but growing (Lean 4, Idris 2)

**When to use**: Formal verification, proof-carrying code, research. Not for typical application or systems programming.

### 1.7 Decision Guide: Which Mechanism to Use

```
Need: generate code for multiple types
  → Is the set of types known at compile time?
      YES, finite: Zig comptime or Rust const generics
      YES, open: Rust trait generics (monomorphization) or trait objects (dynamic dispatch)
      NO (dynamic plugin system): trait objects + vtable

Need: syntactic transformation (new syntax, DSL)
  → Is the transformation purely syntactic?
      YES: macro_rules! (Rust) or simple text macros
      NO (need to inspect types/names): Rust proc macros or Zig comptime

Need: invariants enforced by the type system
  → Are they expressible with traits/interfaces?
      YES: trait bounds / interface constraints
      NO: dependent types (Lean/Idris) or encode via newtype/phantom type tricks

Need: constant folding / partial evaluation
  → Zig: comptime expressions
  → Rust: const fn + const items
  → C++: constexpr
  → C: the preprocessor (weak) or compiler optimizer (implicit)
```

---

## 2. Zig Comptime in Depth

[Zig]

### 2.1 The Core Concept

From the official documentation:

> "Zig places importance on the concept of whether an expression is known at compile-time. There are a few different places this concept is used, and these building blocks are used to keep the language small, readable, and powerful."

Zig has a single mechanism — `comptime` — that replaces:
- Generics / templates
- Macros
- Metaprogramming sublanguages
- Conditional compilation (`#ifdef`)
- Compile-time constants

### 2.2 `comptime` Keyword Semantics

`comptime` can appear in three positions:

**1. `comptime` parameter**: the argument at the call site must be compile-time known.

```zig
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}

// Call site:
const result = max(f32, 1.0, 2.0);  // T = f32 is comptime-known
```

**2. `comptime var`**: a mutable variable that must be comptime-known at all uses. Useful inside `inline for`/`while` loops.

```zig
comptime var i = 0;
inline while (i < cmd_fns.len) : (i += 1) {
    if (cmd_fns[i].name[0] == prefix_char) {
        result = cmd_fns[i].func(result);
    }
}
```

**3. `comptime` block/expression**: force evaluation at compile time. All variables inside are comptime; all control flow is evaluated; any attempt to do something runtime-only causes a compile error.

```zig
const primes = comptime firstNPrimes(25);  // computed at compile time
```

### 2.3 Types as First-Class `comptime` Values

The key insight in Zig: `type` is a first-class value. Its type is `type`. A variable of type `type` holds a type (like `i32`, `bool`, or a struct definition). Types can only exist in comptime context.

```zig
// `type` is the type of types
const T: type = i32;
const U: type = []const u8;

// Functions can take and return types
fn makeOptional(comptime T: type) type {
    return ?T;  // returns the optional type wrapping T
}
```

This is how Zig achieves generics without a separate generics system.

### 2.4 Generic Functions via `comptime T: type`

```zig
fn identity(comptime T: type, value: T) T {
    return value;
}

// Each call with a distinct T generates a separate monomorphized version
const x = identity(i32, 42);
const y = identity([]const u8, "hello");
```

**Compile-time duck typing**: the type parameter `T` is checked by actually type-checking the function body with the concrete `T` substituted. There is no explicit constraint syntax (no trait bounds). If `T` does not support the operations used, the error appears when the function is instantiated — like C++ templates but with better error messages because Zig has a simpler type system.

```zig
// This works for any T that supports >
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}

// Calling with bool causes a compile error:
// "operator > not allowed for type 'bool'"
// _ = max(bool, true, false);
```

The programmer can add explicit branching on type information to handle more cases:

```zig
fn max(comptime T: type, a: T, b: T) T {
    if (T == bool) {
        return a or b;
    } else if (a > b) {
        return a;
    } else {
        return b;
    }
}
```

Because `T` is comptime-known, the `if (T == bool)` branch is evaluated at compile time, and the dead branch is eliminated entirely. The generated code for `max(bool, ...)` contains only the `a or b` path.

### 2.5 Generic Data Structures: `fn List(comptime T: type) type { ... }`

Generic data structures are functions that take a comptime type and return a struct type:

```zig
fn List(comptime T: type) type {
    return struct {
        items: []T,
        len: usize,
    };
}

// Instantiation:
var buffer: [10]i32 = undefined;
var list = List(i32){
    .items = &buffer,
    .len = 0,
};
```

From the official documentation:

> "That's it. It's a function that returns an anonymous struct. For the purposes of error messages and debugging, Zig infers the name 'List(i32)' from the function name and parameters invoked when creating the anonymous struct."

A more complex generic with methods:

```zig
fn Stack(comptime T: type) type {
    return struct {
        const Self = @This();  // refers to this anonymous struct

        data: []T,
        top: usize,

        pub fn push(self: *Self, val: T) void {
            self.data[self.top] = val;
            self.top += 1;
        }

        pub fn pop(self: *Self) T {
            self.top -= 1;
            return self.data[self.top];
        }
    };
}
```

### 2.6 Reflection: `@TypeOf`, `@typeInfo`, `@typeName`

Zig provides builtin functions for type reflection, all comptime-only:

**`@TypeOf(expr)`**: returns the type of an expression. Works on any expression.

```zig
const x: i32 = 42;
const T = @TypeOf(x);  // T == i32
```

**`@typeInfo(T)`**: returns a tagged union `std.builtin.Type` describing the type's structure. This enables exhaustive introspection.

```zig
const info = @typeInfo(T);
switch (info) {
    .int => |int_info| {
        // int_info.bits, int_info.signedness
    },
    .pointer => |ptr_info| {
        // ptr_info.child, ptr_info.size, ptr_info.is_const
    },
    .@"struct" => |struct_info| {
        // struct_info.fields: []const std.builtin.Type.StructField
        for (struct_info.fields) |field| {
            _ = field.name;
            _ = field.type;
        }
    },
    .array => |arr| { _ = arr.child; _ = arr.len; },
    .float, .bool, .void, .noreturn => ...,
    // ... many more variants
}
```

**`@typeName(T)`**: returns the name of a type as a `[]const u8` compile-time string.

```zig
const name = @typeName(i32);  // "i32"
const name2 = @typeName([]const u8);  // "[]const u8"
```

The `printValue` function in Zig's standard library print implementation:

```zig
pub fn printValue(self: *Writer, value: anytype) !void {
    switch (@typeInfo(@TypeOf(value))) {
        .int => return self.writeInt(value),
        .float => return self.writeFloat(value),
        .pointer => return self.write(value),
        else => @compileError("Unable to print type '" ++ @typeName(@TypeOf(value)) ++ "'"),
    }
}
```

### 2.7 `inline for` and `inline while`

`inline for` and `inline while` iterate at compile time: the loop body is unrolled, with each iteration specialized to the comptime-known loop variable.

```zig
// inline for over an array — each iteration is compiled separately
inline for (fields, 0..) |field, i| {
    // field is comptime-known here; can use it in types
    const FieldType = @TypeOf(@field(value, field.name));
    _ = FieldType;
}
```

From the official documentation on `performFn`:

```zig
fn performFn(comptime prefix_char: u8, start_value: i32) i32 {
    var result: i32 = start_value;
    comptime var i = 0;
    inline while (i < cmd_fns.len) : (i += 1) {
        if (cmd_fns[i].name[0] == prefix_char) {
            result = cmd_fns[i].func(result);
        }
    }
    return result;
}
```

The compiler generates a separate `performFn` for each distinct `prefix_char` value, with the `inline while` fully unrolled and dead-code-eliminated for each.

### 2.8 Container-Level Expressions Are Implicitly Comptime

Any expression at the top level of a container (struct, union, enum, file) is implicitly evaluated at compile time. This enables complex compile-time initialization:

```zig
const first_25_primes = firstNPrimes(25);   // computed at compile time
const sum_of_first_25_primes = sum(&first_25_primes);  // also comptime

fn firstNPrimes(comptime n: usize) [n]i32 {
    // This function runs at compile time when called at container level
    var prime_list: [n]i32 = undefined;
    // ... sieve logic
    return prime_list;
}
```

The LLVM IR generated for the above contains only the precomputed constants — no runtime computation.

### 2.9 Case Study: `print` in Zig

The Zig standard library `print` is implemented in userland using comptime. From the documentation:

```zig
pub fn print(self: *Writer, comptime format: []const u8, args: anytype) anyerror!void {
    comptime var start_index: usize = 0;
    comptime var state = State.start;
    comptime var next_arg: usize = 0;

    inline for (format, 0..) |c, i| {
        switch (state) {
            State.start => switch (c) {
                '{' => { ... state = State.open_brace; },
                '}' => { ... state = State.close_brace; },
                else => {},
            },
            State.open_brace => switch (c) {
                '{' => { state = State.start; start_index = i; },
                '}' => { try self.printValue(args[next_arg]); next_arg += 1; ... },
                else => @compileError("Unknown format character: " ++ [1]u8{c}),
            },
            // ...
        }
    }
    comptime {
        if (args.len != next_arg) @compileError("Unused arguments");
        if (state != State.start) @compileError("Incomplete format string: " ++ format);
    }
}
```

The format string is parsed at compile time. The `inline for` unrolls across each character. The resulting generated function is specialized to the specific format string and argument types — no runtime format parsing, no dynamic dispatch on argument types. The format string validation (wrong number of arguments, unknown format specifier) becomes a compile error.

From the documentation:

> "Zig does not special case string formatting in the compiler and instead exposes enough power to accomplish this task in userland. It does so without introducing another language on top of Zig, such as a macro language or a preprocessor language. It's Zig all the way down."

### 2.10 What Cannot Be Done at Comptime

From the official docs:

```zig
extern fn exit() noreturn;

test "foo" {
    comptime {
        exit();  // ERROR: comptime call of extern function
    }
}
```

The following are not allowed in comptime contexts:
- Calling `extern` functions (they have undefined semantics at compile time)
- I/O (reading files, printing at runtime, syscalls)
- Pointer arithmetic to addresses that are not comptime-known (pointers to heap memory, stack memory allocated at runtime)
- Spawning threads
- Any operation with global runtime side effects

Within a `comptime` expression, the documentation specifies:
- All variables are comptime variables
- All `if`, `while`, `for`, and `switch` are evaluated at compile time (or a compile error if impossible)
- All `return` and `try` are invalid unless the function itself is called at compile time
- All function calls cause the compiler to interpret the function at compile time

### 2.11 How Zig Compiles Comptime: Full Interpreter in the Compiler

Zig embeds a complete interpreter for Zig code in the compiler. When an expression is marked `comptime`, the compiler switches from code generation mode to interpretation mode and runs the Zig interpreter.

The interpreter handles:
- Zig value semantics (integers, floats, arrays, structs, unions, pointers to comptime-known data)
- Function calls (including recursive)
- Builtin functions (`@typeInfo`, `@TypeOf`, etc.)
- Control flow (`if`, `while`, `for`, `switch`)
- Type operations (creating struct types, function types, array types)

This is a substantial engineering investment. The interpreter must faithfully implement Zig semantics, including overflow checking, sentinel-terminated arrays, etc. The benefit is that the same semantics apply at both compile time and runtime — there is no discrepancy between what the programmer writes and what the compiler evaluates.

**Branch quota**: to prevent infinite loops in comptime, Zig imposes a default limit of 1000 branches. This can be raised with `@setEvalBranchQuota(n)`:

```zig
comptime {
    @setEvalBranchQuota(10000);
    const result = computeSomethingExpensive();
    _ = result;
}
```

---

## 3. Rust Constant Evaluation

[Rust]

### 3.1 Const Functions: `const fn`

A `const fn` is a function that can be called from a const context. It is defined with the `const` qualifier:

```rust
const fn square(x: i32) -> i32 { x * x }

const VALUE: i32 = square(12);  // evaluated at compile time
```

From the reference:

> "When called from a const context, a const function is interpreted by the compiler at compile time. The interpretation happens in the environment of the compilation target and not the host. So `usize` is 32 bits if you are compiling against a 32-bit system, irrelevant of whether you are building on a 64-bit or a 32-bit system."

> "When a const function is called from outside a const context, it behaves the same as if it did not have the `const` qualifier."

**Restrictions on `const fn` bodies**:
- The body may only use constant expressions
- Async is not allowed
- Parameter and return types are restricted to those compatible with a const context

The restrictions have relaxed significantly across Rust versions (const fn stabilization tracks). As of stable Rust (2024), `const fn` supports: arithmetic, logical operations, conditionals, loops, let bindings, pattern matching, and many standard library operations.

### 3.2 Constant Expressions

From the Rust reference: "Certain forms of expressions, called constant expressions, can be evaluated at compile time."

The exhaustive list includes:
- Literals
- Const parameters
- Paths to functions and constants (no recursive constant definitions)
- Paths to statics with restrictions (no writes, no reads from extern statics, no reads from mutable statics except in static initializers)
- Tuple, array, struct expressions
- Block expressions (including `unsafe` and `const` blocks)
- `let` statements and irrefutable patterns (including mutable bindings, assignment, compound assignment)
- Field and index expressions
- Range expressions
- Closure expressions that do not capture variables from the environment
- Arithmetic, logical, comparison operators on primitive types
- Borrows (with complex restrictions around interior mutability and temporary lifetime extension — see the reference for full details)
- Dereference expressions
- Cast expressions (except pointer-to-address and function-pointer-to-address casts)
- Calls to `const fn` and `const` methods
- `loop` and `while` expressions
- `if` and `match` expressions

**Key constraint from the reference**: "Behaviors such as out of bounds array indexing or overflow are compiler errors if the value must be evaluated at compile time (i.e. in const contexts). Otherwise, these behaviors are warnings, but will likely panic at run-time."

### 3.3 Const Contexts

A const context is a location where the Rust compiler requires a constant expression:

1. Array type length expressions: `[T; N]` — `N` must be const
2. Array repeat length expressions: `[0u8; N]` — `N` must be const
3. Initializers of `const` items
4. Initializers of `static` items
5. Initializers of `enum` discriminants
6. Const generic arguments
7. Const blocks: `const { ... }`

**Restriction on const contexts in types**: "Const contexts that are used as parts of types (array type and repeat length expressions as well as const generic arguments) can only make restricted use of surrounding generic parameters: such an expression must either be a single bare const generic parameter, or an arbitrary expression not making use of any generics."

### 3.4 `const` Items vs `static` Items

**`const` items**:
- Inlined wherever used — no stable memory address
- Essentially a named constant expression
- Can have destructors (destructor runs when the value goes out of scope at the use site)
- Always evaluated at compile time: "Free constants are always evaluated at compile-time to surface panics. This happens even within an unused function."
- Type must have `'static` lifetime

```rust
const BIT1: u32 = 1 << 0;
const STRING: &'static str = "bitstring";

// Compile-time panic — even if unused:
const PANIC: () = std::unimplemented!();
```

**`static` items**:
- Have a stable memory address for the lifetime of the program
- All static items use the same memory location across the entire crate
- Require `unsafe` to mutate (`static mut`)
- Read from `extern` statics is not allowed in const evaluation

**Key difference**: `const` values are conceptually inlined at each use site (no single address), while `static` values have exactly one address in the final binary.

### 3.5 MIRI as the Const Evaluator

Rust uses MIRI (Mid-level IR Interpreter) to evaluate constants. MIRI interprets Rust's MIR (Mid-level Intermediate Representation), a simplified, typed, control-flow-graph form of Rust code produced after type checking and borrow checking.

MIRI's role:
- Evaluates `const` items and `const fn` calls at compile time
- Detects undefined behavior in const evaluation (dangling pointers, out-of-bounds, etc.)
- Also used as a standalone tool for detecting UB at runtime in test code

MIRI operates after borrow checking, so it can assume memory safety has been verified. This simplifies the interpreter's job compared to Zig's comptime interpreter, which must enforce all of Zig's safety rules itself.

### 3.6 Const Generics: `fn foo<const N: usize>()`

[Rust]

Const generic parameters allow items to be generic over constant values, not just types:

```rust
fn zeros<const N: usize>() -> [u8; N] {
    [0; N]
}

struct Array<T, const N: usize>([T; N]);

impl<T, const N: usize> Array<T, N> {
    const SIZE: usize = N;
}
```

From the reference:

> "Const generic parameters allow items to be generic over constant values. The const identifier introduces a name in the value namespace for the constant parameter, and all instances of the item must be instantiated with a value of the given type."

**Allowed types for const parameters** (as of stable Rust, 2024): `u8`, `u16`, `u32`, `u64`, `u128`, `usize`, `i8`, `i16`, `i32`, `i64`, `i128`, `isize`, `char`, `bool`.

**Where const parameters can appear**:
- As an applied const in signatures: `fn foo<const N: usize>(arr: [i32; N])`
- In the body of functions: `let x: [i32; N]; println!("{}", N * 2);`
- As fields: `struct Foo<const N: usize>([i32; N]);`
- As associated constants: `const CONST: usize = N * 4;`
- As associated types: `type Output = [i32; N];`

**Where they cannot appear**: inside item definitions within a function body (no `static BAD: [usize; N]`, no `type BadAlias = [usize; N]` inside a function body).

**Standalone restriction**: in type positions and array repeat expressions, const parameters must appear as a "standalone" argument — a single path segment or block expression, not combined in an arithmetic expression:

```rust
// NOT allowed: arithmetic in type position
fn bad<const N: usize>() -> [u8; {N + 1}] { ... }

// Allowed: N alone, or in a block if needed elsewhere
fn good<const N: usize>() -> [u8; N] { ... }
```

**Ambiguity resolution**: when a generic argument could be resolved as either a type or a const, it is always resolved as a type. Use braces to force const interpretation:

```rust
type N = u32;
struct Foo<const N: usize>;
// ERROR: N is interpreted as the type alias N
fn foo<const N: usize>() -> Foo<N> { todo!() }
// OK: braces force const interpretation
fn bar<const N: usize>() -> Foo<{ N }> { todo!() }
```

**Exhaustiveness limitation**: the compiler does not consider whether all const values of a type are covered when checking trait bounds:

```rust
struct Foo<const B: bool>;
trait Bar {}
impl Bar for Foo<true> {}
impl Bar for Foo<false> {}

fn needs_bar(_: impl Bar) {}
fn generic<const B: bool>() {
    let v = Foo::<B>;
    needs_bar(v);  // ERROR: trait bound `Foo<B>: Bar` is not satisfied
    // Even though all concrete values (true, false) implement Bar,
    // the compiler cannot verify this for abstract B.
}
```

### 3.7 `where` Clauses on Const Generics

Where clauses can constrain const generic parameters. From the reference:

```rust
struct A<T>
where
    T: Iterator,
    T::Item: Copy,
    String: PartialEq<T>,
{
    f: T,
}
```

For const generics, there is no direct `where` clause syntax for constraining the value of a const parameter (no `where N > 0`). Workarounds:
- Use `static_assert` or trait bounds with helper traits
- Use the `generic_const_exprs` unstable feature for expressions in const contexts

---

## 4. Compile-Time / Runtime Boundary Design

### 4.1 The Core Question

For any computation, the designer must decide: does this run at compile time, at runtime, or can it run at either?

**Forces pushing toward compile time**:
- The result is needed to determine the type or layout of a value (array length, struct field type)
- The result is a constant that would otherwise require a runtime load
- The computation is expensive but the inputs are known at compile time
- You want to catch errors earlier (format string validation, dimension checking)

**Forces pushing toward runtime**:
- The inputs are not known until runtime (user input, file contents, network data)
- The computation has observable side effects (I/O, state mutation)
- Compile-time evaluation would unacceptably increase build times
- The system needs to be dynamically configurable without recompilation

### 4.2 Explicit Annotation vs. Inference

**Zig: explicit annotation**. The programmer writes `comptime` to declare what must run at compile time. The compiler enforces the annotation — if an expression depends on a runtime value, marking it `comptime` is an error.

Benefits:
- The programmer's intent is always visible in the source
- No "accidental" compile-time evaluation
- Errors are precise: "unable to resolve comptime value" at the specific site

**Rust: context-driven with inference**. A `const fn` can run at either compile time or runtime depending on where it is called. In a const context, the call is interpreted at compile time; outside, it runs at runtime. The programmer does not explicitly mark call sites.

Benefits:
- Less annotation overhead
- A function written as `const fn` works at both call sites automatically

Drawbacks:
- It is less visible at the call site whether computation is happening at compile time
- You cannot force compile-time evaluation without putting the call in a const context

**C: implicit via optimizer**. The compiler infers what it can fold, based on constant inputs. The programmer has no control beyond `#define` and `constexpr` (C++). No guarantee that a computation actually runs at compile time unless you check the generated assembly.

### 4.3 Staged Programming: MetaML / MetaOCaml

Staged programming formalizes the distinction between code generation stages using explicit quotation brackets:
- `<e>`: quote — turns expression `e` into a code value (deferred to next stage)
- `~e`: splice — executes a code value in the current context

This gives type-safe, compositional multi-stage programs where the code generator and generated code are in the same language but explicitly separated.

MetaOCaml uses this approach. The type `'a code` represents a closed expression of type `'a` to be evaluated in the next stage.

In practice, staged programming is primarily a research tool. Zig's `comptime` achieves most of the practical goals without formal brackets, at the cost of some theoretical composability.

### 4.4 Partial Evaluation: Futamura Projections

Partial evaluation is a program transformation: given a program `P(s, d)` with static inputs `s` (known at specialization time) and dynamic inputs `d`, produce a residual program `P_s(d)` that is equivalent to `P(s, d)` for all `d`.

The three Futamura projections:
1. Specializing an interpreter with a specific program: `pe(interpreter, program)` → compiled program
2. Specializing the partial evaluator with the interpreter: `pe(pe, interpreter)` → a compiler
3. Specializing the partial evaluator with itself: `pe(pe, pe)` → a compiler generator

Comptime evaluation is a form of offline partial evaluation: the programmer (or type system) marks which values are static. The compiler then partially evaluates the program with respect to those static values.

### 4.5 The Performance Argument: Monomorphization vs. Dictionary Passing

**Monomorphization** (the comptime/template approach): for each distinct set of type/const arguments, the compiler generates a separate copy of the function, optimized for those specific types.

```
foo<i32>(x) → foo_i32(x)     // specialized, inlineable
foo<f64>(x) → foo_f64(x)     // specialized, no boxing
```

**Dictionary passing** (Haskell typeclasses, Rust trait objects): a single copy of the function is compiled, with an extra argument carrying a vtable (dictionary) of method implementations.

```
foo(dict, x) → indirect call through dict  // one copy, indirect dispatch
```

**Performance comparison**:
- Monomorphization: zero runtime overhead from dispatch; excellent optimization (inlining, specialization); but binary size grows linearly with number of instantiations
- Dictionary passing: constant binary size; indirect call overhead (typically 1-3 cycles + branch predictor miss); prevents inlining through the vtable

**Compile time**:
- Monomorphization: each instantiation requires re-analyzing the function body; O(instantiations × function complexity) compile time
- Dictionary passing: function analyzed once; lower compile time

---

## 5. Generic Programming Strategies

### 5.1 Monomorphization (C++, Rust, Zig)

The compiler generates a separate instantiation of the generic code for each distinct type argument used in the program.

```rust
// Rust — each use with a distinct T generates a copy
fn max<T: PartialOrd>(a: T, b: T) -> T {
    if a > b { a } else { b }
}

let x = max(1i32, 2i32);    // generates max_i32
let y = max(1.0f64, 2.0);   // generates max_f64
```

```zig
// Zig — identical mechanism via comptime
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}
```

**Pros**: no runtime overhead; full optimization per type; no boxing; type errors are precise
**Cons**: binary size grows; compile time grows; not suitable for open-world extensibility (all types must be known at link time in the simplest case)

### 5.2 Type Erasure / Boxing (Java, early Go)

All type parameters are erased to a common representation. In Java, generics are erased to `Object` at compile time; casts are inserted at use sites; value types are boxed.

```java
// Source
List<Integer> list = new ArrayList<>();
// After erasure
List list = new ArrayList();
Integer x = (Integer) list.get(0);  // cast inserted
```

**Pros**: single copy of code; works with open-world types (plugins, serialization); fast build
**Cons**: boxing overhead for value types (heap allocation + indirection); no specialization; runtime type checks at boundaries

### 5.3 Dictionary Passing (Haskell Typeclasses, Rust Trait Objects)

A dictionary of method implementations is passed (implicitly or explicitly) to the generic function. Haskell typeclasses are compiled this way; Rust's `dyn Trait` uses fat pointers (data pointer + vtable pointer).

```rust
// Rust trait object (dictionary passing)
fn print_debug(x: &dyn Debug) {
    // indirect call through x's vtable
    println!("{:?}", x);
}
```

**Pros**: one copy of the code; dynamic dispatch supports runtime polymorphism; works with trait objects across crates
**Cons**: indirect dispatch overhead; no specialization; fat pointers (2 words instead of 1); no inlining through the vtable without LTO

### 5.4 Zig's Approach: Monomorphization via Comptime, No Separate Generics

Zig has no generics system separate from `comptime`. The mechanism for generics is: write a function that takes comptime type parameters, and return different concrete types or produce different code based on those parameters. The compiler monomorphizes each distinct instantiation.

This is architecturally simpler than systems with separate "generics" syntax:
- One mechanism (comptime) handles both metaprogramming and generics
- The compiler's type checker handles generics by re-checking with concrete types, not by operating on abstract type variables
- Error messages point directly to the instantiation site

### 5.5 Decision Table

| Strategy | Code size | Runtime overhead | Compile time | Open-world | Use when |
|---|---|---|---|---|---|
| Monomorphization | O(instantiations) | Zero | O(instantiations) | No | Performance-critical, known type set |
| Type erasure | O(1) | Boxing + cast | O(1) | Yes | JVM/GC languages, legacy Java |
| Dictionary passing (static) | O(1) | Indirect call | O(1) | Partial | Haskell, Rust generic functions with trait bounds |
| Dictionary passing (dynamic) | O(1) | Indirect call | O(1) | Yes | Rust `dyn Trait`, plugin systems, heterogeneous collections |

For Rust specifically:
- `impl Trait` / `T: Trait` in function signatures → static dispatch (monomorphization by default)
- `dyn Trait` → dynamic dispatch (dictionary passing at runtime)
- `const N: usize` → monomorphization over constant values

---

## 6. Implementation of Comptime: What the Compiler Needs

### 6.1 The Core Requirement: An Interpreter

To evaluate expressions at compile time, the compiler must contain an interpreter for a subset of the source language. The interpreter must:
- Represent values (integers, floats, booleans, aggregates, references to comptime-known data)
- Execute operations (arithmetic, memory reads/writes on comptime data, control flow)
- Handle function calls (including recursive, up to some limit)
- Interface with the type system (for type-as-value operations)
- Report errors with source locations from the original code, not the interpreter's stack

### 6.2 Zig's Approach: Full Interpreter in the Compiler

Zig embeds a complete interpreter for Zig in the `zig` compiler binary. The interpreter:
- Handles all of Zig semantics (including comptime-known pointer arithmetic, comptime-known slice operations, etc.)
- Is invoked when the compiler encounters a `comptime` block, a `comptime` parameter with a known value, or a container-level expression
- Produces Zig values that the compiler then uses in later compilation stages (e.g., a returned type becomes a type node in the AST/IR)
- Enforces Zig's safety checks at compile time (integer overflow in comptime is a hard error, not UB)

The interpreter is tightly integrated with the type checker: the result of comptime evaluation is fed back into type checking, which may trigger more comptime evaluation (e.g., if a struct type is generated, its fields must be type-checked).

### 6.3 Rust's Approach: MIR Interpreter (MIRI) for Const Evaluation

Rust compiles source code to MIR (Mid-level Intermediate Representation) — a simplified, typed CFG form — before lowering to LLVM IR. The const evaluator operates on MIR.

MIRI:
- Interprets MIR instructions
- Models memory as a map from `AllocId` to allocation contents
- Handles borrows, raw pointers, and unsafe code
- Tracks provenance (which allocation a pointer came from) to detect use-after-free and pointer arithmetic errors

The advantage of operating on MIR (post-type-checking) is that the interpreter can assume type safety. The disadvantage is that MIR is further from the source, making error messages harder to map back to user code.

MIRI also serves as a research tool for testing Rust's memory model (Stacked Borrows, Tree Borrows) and can be run on test code to detect UB that would otherwise be silent.

### 6.4 C++: `constexpr` Evaluated During Template Instantiation

C++ `constexpr` functions are evaluated by a compiler-internal interpreter during compilation, typically during or after template instantiation. Each compiler (GCC, Clang, MSVC) has its own `constexpr` evaluator, with slightly different limitations.

C++ `constexpr` has significantly expanded across standards:
- C++11: simple, single-return-statement functions
- C++14: allowed local variables, loops, conditionals
- C++17: `if constexpr` for compile-time branching
- C++20: `consteval` (must evaluate at compile time, like `comptime` blocks), `constinit` (must be constant-initialized), `constexpr` virtual functions, `constexpr` new/delete (allocation in constexpr)

### 6.5 Challenges

**I/O restrictions**: it is unclear what filesystem state or system state means at compile time. I/O is generally forbidden. Exceptions: reading embedded files (`@embedFile` in Zig), reading compile-time variables.

**Pointer validity**: pointers computed at comptime refer to comptime-only allocations. These cannot be used at runtime (the comptime allocation does not persist to the final binary in general). Both Zig and Rust explicitly track this.

**Recursion limits**: to prevent infinite loops from hanging the compiler:
- Zig: default 1000-branch quota, configurable with `@setEvalBranchQuota`
- Rust: hardcoded recursion limit (configurable with `#![recursion_limit = "..."]`)
- C++: implementation-defined (GCC defaults to 512 for template instantiation depth)

**Error quality**: errors that originate in compile-time evaluation need to be reported with the original source location, not the interpreter's call stack. This is an ongoing engineering challenge in all languages. Zig's interpreter produces error messages with source locations and notes; Rust's MIRI errors in const context are generally good.

**Non-determinism**: const evaluation must be deterministic and target-independent (with the exception of `usize`/`isize` sizes). Rust's const eval is forbidden from reading mutable statics or extern statics to preserve this property.

---

## 7. Common Pitfalls

### 7.1 Comptime Explosion: Too Much Work at Compile Time

**Problem**: putting expensive computations in comptime/const contexts makes incremental compilation slower. Every change to a file that triggers recompilation also re-evaluates all comptime in that file.

**Manifestation**: a codebase where changing a configuration constant forces recompilation of thousands of generic instantiations; build times measured in minutes.

**Mitigation**:
- Keep comptime computations simple and bounded
- Use `@setEvalBranchQuota` judiciously in Zig rather than raising it globally
- In Rust, prefer `const fn` over complex `const` expressions for better incremental compilation
- Profile build times with `-ftime-report` (GCC/Clang) or Zig's `--verbose-compile`

### 7.2 Monomorphization Bloat: Binary Size

**Problem**: each unique combination of type parameters generates a new copy of the function body. A function `fn foo<T, U, V>()` used with 10 types for T, 10 for U, and 10 for V generates up to 1000 copies.

**Manifestation**: Rust programs using complex generics can have binaries measured in tens or hundreds of megabytes. The Rust standard library uses internal `Box<dyn Error>` in several places specifically to reduce monomorphization bloat.

**Mitigation**:
- Move non-type-dependent code out of generic functions
- Use dynamic dispatch (`dyn Trait`) for code paths where performance is not critical
- In Rust: `#[inline(never)]` to prevent inlining of large generic functions
- In Zig: factor out common logic into non-generic helper functions called from within the generic

### 7.3 Const Evaluation Errors: Hard to Debug

**Problem**: errors that occur during compile-time evaluation often have poor error messages, especially when they arise deep in a recursion or generic instantiation.

**Rust example**:
```rust
const PANIC: () = std::unimplemented!();
// error[E0080]: evaluation of constant value failed
//   at /path/to/libstd: called `Option::unwrap()` on a `None` value
```

The error points into the standard library, not to the user's code.

**Zig example**:
```
test_fibonacci_comptime_overflow.zig:5:28: error: overflow of integer type 'u32' with value '-1'
    return fibonacci(index - 1) + fibonacci(index - 2);
                    ~~~~~~^~~
note: called at comptime here (7 times)
```

Zig gives call-stack context ("called at comptime here (7 times)").

**Mitigation**:
- Test comptime logic separately with small inputs
- Use `@compileLog` in Zig to print intermediate comptime values
- In Rust, use `const _: () = assert!(...)` to add diagnostic constants
- Keep comptime functions simple enough that their behavior is obvious

### 7.4 Distinguishing "Not Known at Comptime" from "Error"

**Problem**: in Zig, a common confusion is whether something that fails in a comptime context is a logic error in the programmer's code, or whether the programmer simply needs to mark something as runtime.

For example:
```zig
fn processInput(input: []const u8) void {
    comptime var idx = 0;  // ERROR if input is runtime
    // ...
}
```

The error "unable to resolve comptime value" for `input` means `input` is a runtime value and cannot be used to drive a `comptime var`. The fix might be to remove `comptime` from `idx` (make it runtime) or rethink the design.

**Rust equivalent**: the error "attempt to use a non-constant value in a constant" when trying to use a runtime variable in a const context.

**Mitigation**:
- Understand the compile-time / runtime distinction clearly before using `comptime`
- When the error says "not comptime-known," ask: "Is this input truly known at compile time? Should it be?"
- Use `@inComptime()` in Zig to introspect whether code is executing in a comptime context (useful for writing functions that behave differently at comptime vs. runtime)

```zig
fn maybeComptime(value: i32) i32 {
    if (@inComptime()) {
        return value * 2;  // comptime path
    } else {
        return value * 2;  // runtime path (same here, but could differ)
    }
}
```

### 7.5 Monomorphization and Trait/Interface Coherence

**Rust-specific**: const generics have exhaustiveness limitations. Implementing a trait for all `Foo<true>` and all `Foo<false>` does not satisfy a bound `Foo<B>: Bar` for abstract `B`:

```rust
// Even with these two impls:
impl Bar for Foo<true> {}
impl Bar for Foo<false> {}

// This is still an error:
fn generic<const B: bool>() {
    needs_bar(Foo::<B>);  // ERROR: trait bound not satisfied
}
```

This is a fundamental limitation of the current const generics implementation — the trait solver does not reason about exhaustiveness over const values.

---

## Quick Reference: Syntax Cheat Sheet

### Zig

```zig
// Comptime parameter
fn f(comptime T: type) T { ... }

// Comptime variable
comptime var x: usize = 0;

// Comptime block
const c = comptime blk: { ... break :blk value; };

// Comptime expression
const c = comptime someFunction();

// Type reflection
@TypeOf(expr)                     // type of expression
@typeInfo(T)                      // std.builtin.Type tagged union
@typeName(T)                      // []const u8 name
@sizeOf(T)                        // byte size
@bitSizeOf(T)                     // bit size
@alignOf(T)                       // alignment
@hasDecl(T, "name")               // true if T has a declaration named "name"
@hasField(T, "name")              // true if T has a field named "name"

// Inline loops
inline for (slice, 0..) |item, i| { ... }
inline while (condition) { ... }

// Compile-time errors and logging
@compileError("message")
@compileLog(value)

// Generic data structure
fn List(comptime T: type) type {
    return struct { items: []T, len: usize };
}

// Check if in comptime context
if (@inComptime()) { ... }

// Set evaluation branch quota
@setEvalBranchQuota(10000);
```

### Rust

```rust
// Const function
const fn square(x: i32) -> i32 { x * x }

// Const item
const N: usize = 42;

// Static item
static DATA: [u8; 3] = [1, 2, 3];

// Const generic parameter
fn zeros<const N: usize>() -> [u8; N] { [0; N] }

// Const generic struct
struct Array<T, const N: usize>([T; N]);

// Const block
let x = const { 1 + 2 };

// Compile-time assertion
const _: () = assert!(usize::BITS == 64);

// if constexpr equivalent (Rust)
// Use trait-based specialization or const fn with if

// Type information (runtime)
use std::any::TypeId;
TypeId::of::<T>()    // type identity (not full reflection)
std::any::type_name::<T>()  // human-readable name
```
