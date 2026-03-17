---
name: compiler-macro
description: Macro system and metaprogramming expert — covers declarative macros (macro_rules!), procedural macros, hygienic macro expansion, C preprocessor design, and Zig-style comptime. Use when designing a macro system, preprocessor, or compile-time metaprogramming facility for a language.
---

# compiler-macro

You are an expert in macro systems and compile-time metaprogramming. Use this skill when the user is designing or implementing a macro system, preprocessor, or any compile-time code generation facility for a programming language.

Deep reference material is in `references/macro-reference.md` — link there for code details; use this file for design decisions and key patterns.

---

## Macro System Taxonomy — Decision Guide

| Language Goal | Recommended Approach |
|---|---|
| Minimalist C-like, easy to bootstrap | Separate preprocessor pass (text/token substitution) |
| Safe, composable abstractions | Token-tree macros (`macro_rules!` style) |
| Full AST manipulation (derive, DSLs) | Procedural macros |
| Generics, compile-time computation, type-level logic | Comptime (Zig model) |
| Avoid macro system entirely | Invest in comptime + first-class types |

**Key question: do you need hygiene?** Textual macros are easy to implement but cause name capture bugs. Hygienic systems are harder to build but safer for users. See `references/macro-reference.md §3` for the hygiene problem and Kohlbecker's algorithm.

---

## C Preprocessor Implementation

The preprocessor runs as a standalone pass on the raw token stream — the parser never sees directives. [Source: chibicc `preprocess.c`]

### Core Data Structures

Each token carries a **hideset** (linked list of macro names) — this is the anti-recursion mechanism:

```c
typedef struct Macro Macro;
struct Macro {
  char *name;
  bool is_objlike;        // object-like vs function-like
  MacroParam *params;
  char *va_args_name;     // "__VA_ARGS__" or named variadic
  Token *body;
  macro_handler_fn *handler; // for __LINE__, __FILE__, etc.
};
```

### The Hideset (Anti-Recursion / "Blue Paint")

Before expanding a macro, check whether its name is already in the token's hideset. If so, skip — this guarantees termination for mutual recursion like `#define T U` / `#define U T`.

```c
static bool expand_macro(Token **rest, Token *tok) {
    if (hideset_contains(tok->hideset, tok->loc, tok->len))
        return false;   // already expanded — stop
    Macro *m = find_macro(tok);
    ...
}
```

After expansion, stamp the hideset of each result token with the macro name. For function-like macros, use the **intersection** of the macro name token's hideset and the closing `)` hideset, then union with the macro name. This is the Dave Prosser algorithm underlying the C standard.

### Object-Like Expansion

```c
if (m->is_objlike) {
    Hideset *hs = hideset_union(tok->hideset, new_hideset(m->name));
    Token *body = add_hideset(m->body, hs);
    *rest = append(body, tok->next);
    return true;
}
```

### Function-Like Expansion via `subst()`

`subst(body, args)` is the core of function-like macro expansion. It handles:

- **`#` (stringification)**: join the argument's token texts into a string literal.
- **`##` (token pasting)**: concatenate two token texts, then re-tokenize — result must be exactly one valid token.
- **Argument pre-expansion**: arguments are fully macro-expanded *before* substitution, **except** when adjacent to `#` or `##`.
- **`__VA_ARGS__`** and GNU `, ## __VA_ARGS__` (remove preceding comma when variadic args are empty).

### `#include` and Include Guards

- `"foo.h"`: search directory of including file first, then include path list.
- `<foo.h>`: only search include path list.

Include guard optimization: detect `#ifndef FOO_H / #define FOO_H / ... / #endif` at the top of a file. Cache the guard name; skip re-opening the file if the guard macro is still defined. `#pragma once` also supported.

### Conditional Compilation

Stack of `CondIncl` nodes (`IN_THEN / IN_ELIF / IN_ELSE`, plus `included` flag). When a branch is not taken, `skip_cond_incl()` fast-forwards, counting nested `#if`/`#endif`. After all macro expansion of `#if` expressions, remaining identifiers are replaced with `0` (C standard §6.10.1p4).

### Predefined Dynamic Macros

```c
add_builtin("__FILE__", file_macro);
add_builtin("__LINE__", line_macro);
add_builtin("__COUNTER__", counter_macro);
```

These use a handler function pointer. `file_macro` and `line_macro` chase the `.origin` pointer chain to the original source token for user-visible locations.

> See `references/macro-reference.md §2` for full chibicc code listings.

---

## Rust `macro_rules!` — Token-Tree Macros

### Token Trees

The fundamental unit: either a single token, or a balanced delimiter pair `()`, `[]`, `{}` containing more token trees. Macros receive and produce `TokenTree` sequences — not raw text.

Rules are tried in order; first match wins. No backtracking once a rule starts matching.

### Fragment Specifiers

| Specifier | Matches |
|---|---|
| `expr` | An expression |
| `stmt` | A statement |
| `ident` | An identifier or keyword |
| `ty` | A type |
| `pat` | A pattern |
| `path` | A type-path-style path |
| `block` | A block `{ ... }` |
| `item` | A top-level item (fn, struct, impl…) |
| `literal` | A literal value |
| `meta` | Attribute contents |
| `lifetime` | A lifetime |
| `tt` | Any single token tree (escape hatch) |
| `vis` | A visibility qualifier |

Fragment specifiers enforce **follow-set restrictions** (e.g., `expr` may only be followed by `=>`, `,`, `;`) to prevent future grammar ambiguities.

### Repetitions

```
$( ... )*       — zero or more
$( ... )+       — one or more
$( ... )?       — zero or one
$( ... sep )*   — separated by `sep` token
```

Every metavariable in a transcriber repetition must have been bound at the same repetition depth in the matcher.

```rust
macro_rules! vec_literal {
    ( $( $x:expr ),* ) => {{
        let mut v = Vec::new();
        $( v.push($x); )*
        v
    }};
}
```

### Hygiene

Local variables introduced inside a `macro_rules!` body are **invisible at the call site** — they live in the macro's definition scope. `$crate` refers to the crate where the macro is *defined*, enabling cross-crate macros:

```rust
#[macro_export]
macro_rules! helped {
    () => { $crate::helper!() }
}
```

Implementation uses `SyntaxContext` (a mark on each span). Two identifiers with different contexts are considered distinct even if same text. See `references/macro-reference.md §3–4` for Kohlbecker's algorithm and Rust's mixed-site hygiene rules.

### Limitations

- No procedural logic — can only branch on syntactic structure, not values.
- No type information — fragment specifiers match syntax categories only.
- Cannot generate new identifiers (hygiene prevents it) — use proc macros for that.
- Recursive macros must consume at least one token per recursion to terminate.

---

## Zig Comptime — No Macro System Needed

### Core Insight

Zig has no macro system. `comptime` marks values and expressions that must be evaluated at compile time. Types are first-class values of type `type`. Generic code is ordinary functions with `comptime` parameters. One language, zero phase distinction for the programmer.

### Generics via Comptime Parameters

```zig
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}
```

```zig
fn List(comptime T: type) type {
    return struct { items: []T, len: usize };
}
```

### Compile-Time Reflection

- `@TypeOf(expr)` — returns the type as a comptime `type` value.
- `@typeInfo(T)` — returns `std.builtin.Type` tagged union (`.Int`, `.Float`, `.Struct`, `.Pointer`, `.Fn`, etc.).

```zig
pub fn printValue(self: *Writer, value: anytype) !void {
    switch (@typeInfo(@TypeOf(value))) {
        .int    => return self.writeInt(value),
        .float  => return self.writeFloat(value),
        .pointer => return self.write(value),
        else    => @compileError("Unable to print type '" ++
                       @typeName(@TypeOf(value)) ++ "'"),
    }
}
```

### `inline for` — Compile-Time Loop Unrolling

```zig
inline for (format, 0..) |c, i| {
    // c and i are comptime-known; loop body specialized per iteration
}
```

Combined with comptime, the Zig stdlib's `print` parses a format string at compile time and generates a specialized function — no runtime string scanning.

### Why This Eliminates Most Macros

- **Generics**: comptime type parameters replace C++ templates.
- **Conditional compilation**: `if (builtin.os.tag == .windows)` at comptime; dead branches eliminated.
- **Code generation**: functions returning types replace `#define` tricks.
- **Compile-time assertions**: `comptime { assert(condition); }` replaces `static_assert`.
- **Type-safe printf**: format string validated at compile time by ordinary Zig code.

Branch budget: 1000 branches by default; raise with `@setEvalBranchQuota(N)`.

> See `references/macro-reference.md §5–6` for full Zig listings.

---

## Expansion Order and Phase Separation

| System | Phase | Sees types? | Sees scope? | Partial syntax OK? |
|---|---|---|---|---|
| C preprocessor | Before parsing | No | No | Yes |
| Rust `macro_rules!` | After tokenization, before type-check | No | Partial (hygiene) | No |
| Rust proc macros | After parsing, before type-check | No | No | No |
| Zig comptime | Interleaved with type-checking | Yes | Yes | No |

C preprocessor consequence: a macro body need not be a complete syntactic unit — it can produce anything as long as the call site is never reached. Rust consequence: macros must produce valid token trees; the expander distinguishes `expr`/`stmt`/`item` positions.

---

## Warning Signs Your Macro System Is Getting Too Complex

- **Parsing inside the macro system**: if your text/token macro needs to understand expressions or types, you've outgrown it. Move to token trees or comptime.
- **Hygiene as naming conventions**: `_MACRO_LOCAL_VAR_` prefixes signal that the language needs actual hygiene.
- **Macros calling macros 5+ levels deep**: the system is being used as a programming language. Consider proc macros or comptime.
- **Users must understand expansion order**: phase separation is leaking. C preprocessor is the canonical example.
- **A macro generates macro definitions**: possible in C and Rust, nearly impossible to reason about.
- **Debug info points into expanded code**: `origin`/`Span` tracking is broken — quality-of-life killer.

---

## Implementation Checklist

**Separate preprocessor (C-like):**
- [ ] Token `hideset` field (linked list of macro names)
- [ ] `origin` pointer chain for error location tracking
- [ ] Macro table (`HashMap<name, Macro>`)
- [ ] Object-like expansion with hideset stamping
- [ ] Function-like expansion: arg collection, `subst()`, hideset intersection
- [ ] `subst()`: handle `#`, `##`, pre-expand args, `__VA_ARGS__`
- [ ] Conditional compilation stack (`CondIncl`)
- [ ] Include path search + include guard optimization
- [ ] Dynamic built-ins (`__FILE__`, `__LINE__`, `__COUNTER__`)

**Token-tree macros (Rust-like):**
- [ ] `TokenTree` enum: `Token(t)` | `Delimited(delimiter, Vec<TokenTree>)`
- [ ] Fragment specifiers + follow-set restriction checking
- [ ] Pattern matcher: rules tried in order, first match wins
- [ ] `SyntaxContext` on identifiers for hygiene
- [ ] Expansion interleaved with parser, at macro invocation boundary
- [ ] Recursion depth limit

**Comptime (Zig-like):**
- [ ] `type` is a valid value type; `comptime` parameter qualifier
- [ ] Compile-time interpreter over typed AST
- [ ] `@typeInfo` / `@TypeOf` built-ins returning structured type info
- [ ] `inline for` loop unrolling
- [ ] Branch budget (`@setEvalBranchQuota`)

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-architect` — macro system design is part of the front-end architecture
- `compiler-expert` — routes here on macro/hygiene/preprocessor questions

**Trace Down ↓** — next skills after this phase
- `compiler-parser` — macro expansion produces token trees or AST nodes that re-enter parsing
- `compiler-name-resolution` — hygienic macros must interact with the name resolver

**Related →** — peers and related skills
- `compiler-comptime` — comptime is the "no-macro-language" alternative to procedural macros
- `compiler-type-system` — attribute macros and derive macros need type information
