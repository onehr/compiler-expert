# Macro Systems: Comprehensive Reference

Sources: [chibicc] = rui314/chibicc preprocess.c, [Zig] = ziglang.org/documentation/master, [Rust] = doc.rust-lang.org/reference/macros-by-example.html

---

## 1. Macro System Taxonomy

### Text Substitution — C Preprocessor

The C preprocessor operates on a token stream *before* parsing. It performs pure textual (token-level) substitution with no knowledge of types, scopes, or the AST. Simple to implement as a separate pass; dangerous because it ignores language semantics entirely.

Characteristic problems:
- Macro arguments evaluated multiple times: `#define MAX(a,b) ((a)>(b)?(a):(b))` — calling `MAX(i++, j++)` increments twice.
- No scoping: names introduced by macros pollute the surrounding namespace.
- No type safety: `#define SQ(x) ((x)*(x))` works on any numeric type silently.

### Token-Based Macros — Rust `macro_rules!`

Operates on *token trees* (nested delimiter groups) rather than raw text. Pattern-matching syntax maps token tree patterns to output token trees. Has hygiene for identifiers introduced inside the macro body. Cannot inspect types or perform arbitrary computation.

### Procedural Macros — Rust proc macros

Arbitrary Rust code that receives a `TokenStream` and returns a `TokenStream`. Can do anything a normal Rust program can do at compile time. Three kinds: function-like, derive, and attribute macros. No hygiene by default (must use `proc_macro2::Span::call_site()` / `def_site()` deliberately).

### Comptime — Zig

No macro system at all. The `comptime` keyword marks values and expressions that must be known at compile time. Types are first-class `comptime` values. Generic code is ordinary functions with `comptime` parameters. The same function runs at both compile time and runtime without modification.

### Decision Guide

| Language Goal | Recommended Approach |
|---|---|
| Minimalist C-like, easy to bootstrap | Separate preprocessor pass (text/token substitution) |
| Safe, composable abstractions | Token-tree macros (`macro_rules!` style) |
| Full AST manipulation (derive, DSLs) | Procedural macros |
| Generics, compile-time computation, type-level logic | Comptime (Zig model) |
| Avoid macro system entirely | Invest in comptime + first-class types |

---

## 2. C Preprocessor Implementation (chibicc)

### Architecture

The preprocessor (`preprocess.c`) is a standalone pass. It takes a `Token *` list produced by the lexer and returns a new `Token *` list. The parser never sees preprocessor directives.

```c
// Entry point
Token *preprocess(Token *tok) {
  tok = preprocess2(tok);
  if (cond_incl)
    error_tok(cond_incl->tok, "unterminated conditional directive");
  convert_pp_tokens(tok);
  join_adjacent_string_literals(tok);
  for (Token *t = tok; t; t = t->next)
    t->line_no += t->line_delta;
  return tok;
}
```

### Macro Data Structures

```c
typedef struct Macro Macro;
struct Macro {
  char *name;
  bool is_objlike;        // true = object-like, false = function-like
  MacroParam *params;     // linked list of parameter names
  char *va_args_name;     // "__VA_ARGS__" or named variadic
  Token *body;            // token list of replacement body
  macro_handler_fn *handler; // for built-in dynamic macros (__LINE__ etc.)
};
```

### The Hideset (Anti-Recursion / "Blue Paint")

Each token carries a `Hideset *` — a linked list of macro names. When a macro `M` is expanded, the name `M` is added to the hideset of every token in the expansion result. Before expanding a token as a macro, chibicc checks whether the macro name is in the token's hideset:

```c
static bool expand_macro(Token **rest, Token *tok) {
  if (hideset_contains(tok->hideset, tok->loc, tok->len))
    return false;   // already expanded from this macro — stop
  Macro *m = find_macro(tok);
  if (!m)
    return false;
  ...
}
```

This guarantees termination: `#define T U` / `#define U T` expands `T → U → T` and then stops because `T` is in the hideset of the second `T`. This is the algorithm from Dave Prosser's paper (`cpp.algo.pdf`), which underlies the C standard's wording.

For function-like macros, the hideset used is the *intersection* of the hidesets of the macro name token and the closing parenthesis token, then union with the macro name:

```c
Hideset *hs = hideset_intersection(macro_token->hideset, rparen->hideset);
hs = hideset_union(hs, new_hideset(m->name));
```

### Object-Like Macro Expansion

1. Look up the token in the macro table.
2. Build a new hideset = union(token's existing hideset, {macro_name}).
3. Copy the macro body tokens, stamping each with the new hideset.
4. Splice the body in front of the remaining token stream.

```c
if (m->is_objlike) {
  Hideset *hs = hideset_union(tok->hideset, new_hideset(m->name));
  Token *body = add_hideset(m->body, hs);
  *rest = append(body, tok->next);
  return true;
}
```

### Function-Like Macro Expansion

1. Detect `(` immediately after the macro name with no whitespace (distinguishing it from object-like).
2. Collect arguments by scanning forward, respecting parenthesis nesting depth.
3. Call `subst(m->body, args)` to substitute arguments into the body.
4. Each argument is itself fully macro-expanded before substitution (`preprocess2(arg->tok)`), *unless* it appears after `#` (stringification) or next to `##` (token pasting).

### The `#` (Stringification) Operator

In `subst()`:
```c
if (equal(tok, "#")) {
  MacroArg *arg = find_arg(args, tok->next);
  cur = cur->next = stringize(tok, arg->tok);
  tok = tok->next->next;
  continue;
}
```

`stringize()` calls `join_tokens()` to concatenate the argument's token texts into a string, then creates a new string literal token.

### The `##` (Token Pasting) Operator

```c
static Token *paste(Token *lhs, Token *rhs) {
  char *buf = format("%.*s%.*s", lhs->len, lhs->loc, rhs->len, rhs->loc);
  Token *tok = tokenize(new_file(lhs->file->name, lhs->file->file_no, buf));
  if (tok->next->kind != TK_EOF)
    error_tok(lhs, "pasting forms '%s', an invalid token", buf);
  return tok;
}
```

The concatenated string is re-tokenized. The result must be exactly one valid token or it is an error. `##` cannot appear at the start or end of a macro body.

### `__VA_ARGS__` and Named Variadic Parameters

```c
// In read_macro_params:
if (equal(tok, "...")) {
  *va_args_name = "__VA_ARGS__";
  *rest = skip(tok->next, ")");
  return head.next;
}
// Named variadic: foo...
if (equal(tok->next, "...")) {
  *va_args_name = strndup(tok->loc, tok->len);
  ...
}
```

GNU extension: `, ## __VA_ARGS__` — if `__VA_ARGS__` is empty, the comma is also removed.

`__VA_OPT__(x)` — expands to `x` if variadic args are non-empty, otherwise to nothing.

### `#include` File Search

Two search modes:
- `"foo.h"` (double-quote): first searches the directory of the including file, then falls through to the include path list.
- `<foo.h>` (angle-bracket): only searches the include path list.

```c
if (filename[0] != '/' && is_dquote) {
  char *path = format("%s/%s", dirname(strdup(start->file->name)), filename);
  if (file_exists(path)) {
    tok = include_file(tok, path, start->next->next);
    continue;
  }
}
char *path = search_include_paths(filename);
```

Include guards are detected by pattern-matching the top of each file for `#ifndef FOO_H / #define FOO_H / ... / #endif`. Detected guards are cached; if the guard macro is still defined, the file is skipped entirely without opening it. `#pragma once` is also supported.

### Conditional Compilation

`#if`/`#ifdef`/`#ifndef`/`#elif`/`#else`/`#endif` are handled with a stack of `CondIncl` nodes:

```c
typedef struct CondIncl CondIncl;
struct CondIncl {
  CondIncl *next;
  enum { IN_THEN, IN_ELIF, IN_ELSE } ctx;
  Token *tok;
  bool included;  // whether this branch was taken
};
```

When a branch is not taken, `skip_cond_incl()` fast-forwards the token stream, carefully counting nested `#if`/`#endif` pairs. The `defined(FOO)` operator in `#if` expressions is handled specially:

```c
if (equal(tok, "defined")) {
  Macro *m = find_macro(tok);
  cur = cur->next = new_num_token(m ? 1 : 0, start);
}
```

After all macro expansion of the `#if` expression, any remaining identifiers are replaced with `0` (C standard §6.10.1p4).

### Predefined Macros

Static macros (simple text replacement) are registered via `define_macro()`. Dynamic macros that must compute their value at expansion time use a handler function pointer:

```c
add_builtin("__FILE__", file_macro);
add_builtin("__LINE__", line_macro);
add_builtin("__COUNTER__", counter_macro);
add_builtin("__TIMESTAMP__", timestamp_macro);
```

`file_macro` and `line_macro` chase the `.origin` pointer chain to find the *original* source token (before any macro expansion), so they report the user-visible source location rather than the internal expansion position.

---

## 3. Hygiene

### What the Hygiene Problem Is

When a macro introduces a new identifier binding, it can clash with identifiers in the user's code:

```c
// C — NOT hygienic
#define SWAP(a, b) do { int tmp = a; a = b; b = tmp; } while(0)

int tmp = 42;
int x = 1, y = 2;
SWAP(x, tmp);  // BUG: `tmp` inside macro clashes with user's `tmp`
```

### C Preprocessor: Not Hygienic

The C preprocessor does pure token substitution. There is no concept of scope during expansion. The `do { } while(0)` trick solves the statement-vs-expression problem but does not solve name hygiene. The comma operator trick (`(expr1, expr2)`) is another workaround. All such patterns are fragile.

### Rust `macro_rules!`: Hygienic for Identifiers

[Rust] Local variables and loop labels defined *inside* a `macro_rules!` macro body are placed in the macro's *definition* scope, not the invocation scope. Variables defined *outside* the macro (in user code) and referenced *inside* it are also resolved at definition time for locals.

From the Rust Reference:
> Macros by example have mixed-site hygiene. Loop labels, block labels, and local variables are looked up at the macro definition site while other symbols are looked up at the macro invocation site.

```rust
macro_rules! check {
    () => {
        assert_eq!(x, 1); // uses `x` from definition site
        func();           // uses `func` from invocation site
    };
}
```

Variables introduced by expansion are *not* visible outside the expansion:

```rust
macro_rules! m {
    (define) => { let x = 1; };
    (refer)  => { dbg!(x); };   // compile error — `x` not in scope
}
m!(define);
m!(refer);  // ERROR
```

### How Hygiene Is Implemented

The standard technique (Kohlbecker et al., KFFD) uses *syntax objects* — each identifier carries a *mark* indicating which macro expansion introduced it. Two identifiers with different marks are considered distinct even if they have the same text. Rust's implementation uses *syntax contexts* (`SyntaxContext`) attached to each `Span`. Each macro expansion creates a new context; identifiers with different contexts do not capture each other.

### When Full Hygiene Matters vs. When It Is Overkill

Full hygiene is important when:
- Macros introduce local bindings that users should not see.
- Macros are authored in a library and used in arbitrary user code.
- The language has lexical scoping and closures.

Hygiene is overkill when:
- The macro system is purely for code generation / mechanical repetition.
- All introduced names are prefixed with a sigil or namespace that makes collisions impossible by convention.
- The language is dynamically scoped (rare today).
- Implementing a simple C-like preprocessor where programmer discipline is assumed.

---

## 4. Rust Declarative Macros (`macro_rules!`)

### Token Tree Matching

[Rust] The fundamental unit is a *token tree*: either a single token, or a balanced delimiter pair `()`, `[]`, `{}` containing more token trees. This is the unit that `macro_rules!` patterns match against.

Grammar:
```
MacroRulesDefinition →  macro_rules ! IDENTIFIER MacroRulesDef
MacroRulesDef        →  ( MacroRules ) ;
                      | [ MacroRules ] ;
                      | { MacroRules }
MacroRule            →  MacroMatcher => MacroTranscriber
MacroMatcher         →  ( MacroMatch* )
                      | [ MacroMatch* ]
                      | { MacroMatch* }
MacroMatch           →  Token (except $ and delimiters)
                      | MacroMatcher
                      | $ IDENT : MacroFragSpec
                      | $ ( MacroMatch+ ) MacroRepSep? MacroRepOp
MacroRepOp           →  * | + | ?
```

Rules are tried in order; the first match wins. No backtracking once a rule starts matching (no lookahead beyond one token to resolve ambiguity).

### Fragment Specifiers

| Specifier | Matches |
|---|---|
| `expr` | An expression |
| `stmt` | A statement (without trailing `;` for expr stmts) |
| `ident` | An identifier or keyword |
| `ty` | A type |
| `pat` | A pattern |
| `pat_param` | A pattern without top-level or-patterns |
| `path` | A type-path-style path |
| `block` | A block expression `{ ... }` |
| `item` | An item (fn, struct, impl, ...) |
| `literal` | A literal (including negative numeric literals) |
| `meta` | The contents of an attribute |
| `lifetime` | A lifetime token |
| `tt` | Any single token tree (most permissive) |
| `vis` | A visibility qualifier (possibly empty) |

Fragment specifiers enforce follow-set restrictions to prevent future grammar ambiguities. For example, `expr` may only be followed by `=>`, `,`, or `;`.

### Repetitions

```
$( ... )*    — zero or more
$( ... )+    — one or more
$( ... )?    — zero or one
$( ... sep )* — separated by `sep` token
```

Example:
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

In the transcriber, every metavariable inside a repetition must have been bound in the same repetition depth in the matcher. Multiple metavariables in the same transcriber repetition must have been bound the same number of times.

### Hygiene in Practice

[Rust] Local variables introduced inside a `macro_rules!` body are invisible at the call site:
```rust
macro_rules! foo {
    () => { let x = 1; };
}
foo!();
// `x` is not accessible here — hygiene prevents capture
```

The `$crate` metavariable refers to the crate where the macro is *defined*, allowing macros to reference their own crate's items regardless of where they are invoked:
```rust
#[macro_export]
macro_rules! helped {
    () => { $crate::helper!() }
}
```

### Limitations

- No procedural logic: cannot branch on the *value* of a matched expression, only on its syntactic structure.
- No type information: fragment specifiers match syntactic categories, not semantic types.
- No ability to generate new identifiers (hygiene prevents it; proc macros are needed for this).
- Recursive macros are possible but the recursion must terminate syntactically — each recursive call must consume at least one token.

---

## 5. Zig Comptime

### Core Insight

[Zig] Zig has no macro system. Instead, the `comptime` keyword marks values and code that must be evaluated at compile time. A function marked or called in a `comptime` context is interpreted by the compiler using the same rules as runtime execution. There is no separate "macro language" — it is Zig all the way down.

> "Zig does not special case string formatting in the compiler and instead exposes enough power to accomplish this task in userland. It does so without introducing another language on top of Zig, such as a macro language or a preprocessor language."

### Types as First-Class Comptime Values

Types are values of type `type`. `type` is only usable in comptime expressions:

```zig
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}
```

`T` must be `comptime`-known at the call site. Inside the function, `T` is resolved at compile time, so `if (T == bool)` is a compile-time branch and the compiler generates only the taken branch.

### Comptime Parameters: Generics Without Templates

A `comptime` parameter forces the argument to be compile-time known. This is how Zig implements generics: the same function, instantiated for each distinct set of comptime argument values:

```zig
fn List(comptime T: type) type {
    return struct {
        items: []T,
        len: usize,
    };
}
var list = List(i32){ .items = &buffer, .len = 0 };
```

This is "compile-time duck typing": if an operation on `T` is invalid (e.g., `>` on `bool`), the error is reported when the instantiation is analyzed.

### `@TypeOf`, `@typeInfo` for Reflection

- `@TypeOf(expr)` — returns the type of an expression as a `comptime type` value.
- `@typeInfo(T)` — returns a `std.builtin.Type` tagged union describing the structure of type `T`: `.Int`, `.Float`, `.Struct`, `.Pointer`, `.Fn`, etc.

This enables writing generic dispatch in ordinary Zig:

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

### `inline for` — Loop Unrolling as Comptime

```zig
inline for (format, 0..) |c, i| {
    // c and i are comptime-known; the loop body is specialized per iteration
}
```

`inline for` over a comptime-known range unrolls the loop at compile time. Combined with comptime variables, this allows partial evaluation: the print function in the Zig standard library parses a format string at compile time and generates a specialized print function with no runtime string scanning.

### `comptime` Blocks and Container-Level Expressions

Any expression at container level (outside a function) is implicitly comptime. A programmer can write:

```zig
const first_25_primes = firstNPrimes(25);
```

where `firstNPrimes` is an ordinary Zig function — it runs at compile time and the result is embedded as a constant in the binary. No special annotation on the function is needed.

### @setEvalBranchQuota

Compile-time interpretation has a default budget of 1000 branches to prevent infinite loops. This can be raised:
```zig
@setEvalBranchQuota(10000);
```

### Why This Eliminates Most Need for Macros

- **Generics**: comptime type parameters replace C++ templates and Rust generics syntax.
- **Conditional compilation**: `if (builtin.os.tag == .windows)` at comptime; dead branches are eliminated.
- **Code generation**: functions that return types or generate structs replace `#define`-based tricks.
- **Compile-time assertions**: `comptime { assert(condition); }` replaces `static_assert`.
- **Type-safe printf**: the `print` case study shows format string validation at compile time using ordinary Zig code.

---

## 6. Expansion Order and Phase Separation

### C: Preprocessing Before Parsing

[chibicc] The preprocessor runs as a completely separate pass over the raw token stream. It operates on `Token *` linked lists and knows nothing about C syntax — it cannot inspect whether a token is part of an expression or a declaration. The parser only ever sees the post-preprocessing token stream.

Consequences:
- Macros can produce syntactically invalid fragments as long as they are never used.
- A macro body does not need to be a complete syntactic unit.
- The preprocessor cannot "see" types or scope.
- Source locations in errors refer to the original source, tracked via the `origin` pointer chain in chibicc.

### Rust: Expansion Interleaved with Parsing

[Rust] Rust's macro expansion is interleaved with parsing at the token tree level. The parser can parse up to a macro invocation boundary, hand off to the macro expander, then continue parsing with the result. This means:
- Macros must produce syntactically valid token trees (they are parsed after expansion).
- The expander can distinguish `expr`, `stmt`, `item`, `ty`, etc. syntactic positions.
- Macros cannot span arbitrary token boundaries; the fragment specifier system enforces this.
- Expansion happens after tokenization but before full name resolution and type checking.

Forwarding a matched fragment: when a captured fragment (`$x:expr`) is forwarded to another macro, it arrives as an opaque AST node. The receiving macro cannot match it with literal tokens; it must use a compatible fragment specifier. (`tt` is the escape hatch that accepts any token tree.)

### Zig: No Expansion Phase

[Zig] There is no macro expansion phase. `comptime` code runs *after* type-checking is complete (well, interleaved: the compiler type-checks comptime code using the same type system). Comptime functions see fully typed, resolved values. This is qualitatively more powerful than text-level or token-level macros: a comptime function can branch on the exact type, size, fields, and alignment of its arguments.

The tradeoff is that the "macro" (comptime code) must be valid, well-typed Zig. It cannot produce arbitrary token sequences that would be illegal in isolation.

### Why Phase Order Affects What Macros Can "See"

| System | Phase | Can see types? | Can see scope? | Can produce partial syntax? |
|---|---|---|---|---|
| C preprocessor | Before lexing is complete | No | No | Yes |
| Rust `macro_rules!` | After tokenization, before type-checking | No | Partial (via hygiene) | No — must be valid token trees |
| Rust proc macros | After parsing, before type-checking | No (needs `syn`) | No | No |
| Zig comptime | After type-checking (interleaved) | Yes | Yes | No — must be valid Zig |

---

## 7. Implementation Decision Guide

### For a C-Like Language: Separate Preprocessor Pass

Implement the preprocessor as a separate pass that transforms a raw token stream before the parser sees it. Key decisions:

1. **Token representation**: each token needs a `hideset` (linked list of macro names) to implement the anti-recursion algorithm. Also carry `origin` pointer for error reporting through macro expansions.

2. **Macro table**: a hash map from `char *name` to `Macro *`.

3. **Object-like macros**: detect token in macro table → add name to hideset → splice body.

4. **Function-like macros**: `name` immediately followed by `(` without whitespace → collect arguments (respecting nesting) → substitute via `subst()`.

5. **subst() is the core**: handle `#` (stringize), `##` (paste), argument expansion, and `__VA_ARGS__`. Arguments are fully expanded before substitution unless adjacent to `#` or `##`.

6. **Conditional compilation**: a stack of `CondIncl` structs. When a branch is skipped, use `skip_cond_incl()` to fast-forward past the token stream, counting nested `#if`/`#endif` pairs.

7. **Include guards**: detect the `#ifndef FOO_H / #define FOO_H / ... / #endif` pattern and cache guard names to skip re-reading files.

### For a Rust-Like Language: Token Tree Approach, Integrated with Parser

1. **Token tree representation**: define a `TokenTree` enum: `Token(t)` | `Delimited(delimiter, Vec<TokenTree>)`. The macro system operates on these, not raw tokens.

2. **Fragment specifiers** are essential for preventing ambiguity and for forwarding fragments correctly. Implement follow-set restriction checking.

3. **Hygiene**: attach a `SyntaxContext` to each identifier. When a macro expands, create a fresh context. Use context-aware identifier comparison everywhere (name resolution, binding lookup).

4. **Integration point**: after tokenizing + building token trees, but before name resolution and type checking. The parser calls the macro expander when it encounters a macro invocation.

5. **Recursion limit**: macro expansion can be recursive; implement a depth limit.

### For a Zig-Like Language: No Macro System, Invest in Comptime

1. **First-class types**: `type` is a valid value type. Functions can accept and return `type` values.

2. **Comptime annotation**: add `comptime` as a parameter/variable qualifier. The compiler evaluates comptime expressions using its own interpreter on the typed AST.

3. **Compile-time interpreter**: must support all the language constructs you want available at comptime (loops, function calls, struct creation, etc.). This is the expensive part but eliminates the need for a separate macro language.

4. **`@typeInfo` / `@TypeOf`**: built-in reflection that returns structured type information at comptime. Essential for writing generic library code.

5. **`inline for`**: loops over comptime-known lengths that unroll at compile time. This is the primary mechanism for compile-time code generation.

### Warning Signs Your Macro System Is Getting Too Complex

- **You are parsing inside the macro system**: if your text/token macro system starts needing to understand expressions, types, or blocks to work correctly, you have outgrown it. Move to token trees (Rust) or comptime (Zig).

- **Hygiene is implemented as naming conventions**: `_MACRO_LOCAL_VAR_` prefixes are a sign the language needs actual hygiene support.

- **Macros calling macros deeply**: if macros routinely invoke other macros 5+ levels deep, the system is being used as a programming language. Consider proc macros or comptime.

- **Users must understand expansion order to get correct results**: this indicates phase separation is leaking through the abstraction. The C preprocessor is the canonical example.

- **You have a macro that generates macro definitions**: this is possible in C and Rust but extremely hard to reason about. A sign that the host language's abstraction facilities are insufficient.

- **Debug information is useless**: if error messages point into expanded code rather than the user's source, your `origin`/`Span` tracking is broken. This is a quality-of-life killer.
