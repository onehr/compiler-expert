# Compiler Diagnostics: Deep Reference

Sources researched:
- https://rustc-dev-guide.rust-lang.org/diagnostics.html [rustc-dev-guide]
- https://rustc-dev-guide.rust-lang.org/diagnostics/diagnostic-structs.html [rustc-dev-guide]
- https://rustc-dev-guide.rust-lang.org/diagnostics/translation.html [rustc-dev-guide]
- https://clang.llvm.org/docs/InternalsManual.html#the-diagnostic-subsystem [Clang]

---

## 1. Why Diagnostics Are a First-Class Product

[rustc-dev-guide] The rustc documentation opens bluntly: "A lot of effort has been put into making rustc have great error messages." This framing — effort as a deliberate investment — signals the Rust team's philosophy: error messages are not an afterthought but a core deliverable.

The compiler UX argument is straightforward. A compiler's primary user interface is its terminal output. For most users, especially beginners, the error message is the only explanation they receive of why their code does not work. An opaque or misleading error message forces the user to:
- Search externally for the meaning of an error code
- Form incorrect mental models of what went wrong
- Abandon the language as hostile

Rust explicitly acknowledges this in its style guide: "Keep in mind that Rust's learning curve is rather steep, and that the compiler messages are an important learning tool." This is an admission that the compiler must compensate for the language's complexity through superior diagnostics.

[Clang] Clang's design philosophy is similar. Its internal documentation states: "The Clang Diagnostics subsystem is an important part of how the compiler communicates with the human." The emphasis on communication as the primary goal — not merely error detection — shapes every design decision downstream.

The practical consequence is that diagnostic quality is a release-blocking concern. rustc maintainers routinely file issues for regressions in error message quality, and improvements to error messages are a recognized class of contribution.

---

## 2. Span Design

### What a Span Is

[rustc-dev-guide] In rustc, a `Span` is the primary data structure for representing a location in the code being compiled. Conceptually, a span is a half-open byte range `[lo, hi)` within a source file, combined with a reference to which file it belongs to (via the `SourceMap`). Spans are attached to most constructs in HIR and MIR.

A `Span` can be looked up in a `SourceMap` to obtain a "snippet" — the actual source text of the range — using methods like `span_to_snippet`. The `SourceMap` also computes `(line, column)` information lazily from a line-offset table; the span itself stores only byte offsets, which are compact and fast to manipulate.

[Clang] Clang uses `SourceLocation`, a 32-bit opaque value encoding a file/offset pair, and `SourceRange` (a pair of `SourceLocation`). The `SourceManager` class performs all lookups: file identification, `(line, col)` computation, and macro expansion context. The separation between the lightweight `SourceLocation` value and the heavier `SourceManager` lookup is the same compact/lazy split used by rustc.

Clang also distinguishes `CharSourceRange` from `SourceRange`: a `CharSourceRange` is token-granularity (ending after the last character of the last token), while a `SourceRange` is token-start-granularity. This matters when a fix-it must replace exactly the characters the user wrote.

### Multi-Span Diagnostics

[rustc-dev-guide] A rustc diagnostic has one primary span and zero or more secondary spans. The canonical rendered form is:

```
error[E0000]: main error message
  --> file.rs:LL:CC
   |
LL | <code>
   | -^^^^- secondary label
   |  |
   |  primary label
   |
   = note: note without a Span, created with `.note`
note: sub-diagnostic message for `.span_note`
  --> file.rs:LL:CC
   |
LL | more code
   |      ^^^^
```

The primary span is the site of the error; its label should contain enough text to explain the problem even when displayed in isolation (as in an IDE tooltip). Secondary spans add context: "this variable was defined here", "conflicting type bound comes from here", "previous borrow occurs here."

When multiple span labels would overlap or produce cluttered output, the implementation is expected to adapt — for example, the if/else type-mismatch error uses different span layouts depending on whether the arms are on the same line.

### Synthetic Spans for Desugared Code

[rustc-dev-guide] The compiler desugars many constructs (for loops to iterators, async/await to state machines, etc.) before or during HIR lowering. Desugared code must still carry accurate spans pointing to the original user-written syntax. If the desugaring of `for x in iter { ... }` produces a type error on the internal `IntoIterator::into_iter` call, the span must point to the `iter` expression the user wrote, not to the synthetic AST node inside the desugaring.

This is achieved by assigning the original source span to every synthetic node produced by lowering, rather than using dummy spans.

### Span Hygiene for Macros

[rustc-dev-guide] When an error originates inside a macro expansion, the span situation is more complex. rustc uses a `SyntaxContext` (hygiene context) attached to each span to encode the expansion chain. This allows the compiler to render both the call site of the macro and the definition site, depending on which is more useful.

The field `Span::from_expansion()` returns `true` if the span comes from a macro expansion. Code that generates diagnostics often checks this to avoid producing confusing messages about macro-generated syntax that the user never wrote. For example, the `while_true` lint explicitly skips when `!lit.span.from_expansion()` to avoid firing on `while true { }` patterns generated by macros.

---

## 3. Caret Display

[Clang] The caret-style display was established by Clang and has become the universal standard. The core idea: print the source line, then on the next line print a caret (`^`) under the exact column of the error token, with tildes (`~`) extending left and right to cover the full span of a source range. Example from Clang's documentation:

```
t.c:38:15: error: invalid operands to binary expression ('int *' and '_Complex float')
P = (P-42) + Gamma*4;
    ~~~~~~ ^ ~~~~~~~
```

The column calculation works as follows:
1. Given the byte offset of the primary span, look up the source line by scanning for newlines up to that offset.
2. The column is the number of characters (or bytes, depending on how the compiler handles Unicode) from the start of the line to the span start.
3. Print the source line verbatim.
4. Print a row of spaces equal in width to the column offset, then `^` for the primary token, then `~` for secondary ranges.

[rustc-dev-guide] rustc's terminal renderer adds color and labels on top of the same base layout. The `^` underline is rendered in red for errors and yellow for warnings. Secondary spans use `-` or `~` in their respective colors. Labels appear inline after the underlines, or on subsequent lines if they would overlap.

The location header `  --> file.rs:LL:CC` gives the file path, line number (1-based), and column number (1-based) of the primary span's start position.

---

## 4. Diagnostic Structure

### Severity Levels

[rustc-dev-guide] rustc recognizes the following diagnostic levels:

- **error**: The program is invalid; compilation cannot succeed. Emitted when the compiler detects a problem that makes it unable to compile the program.
- **warning**: Something odd or potentially problematic, but compilation continues. Warnings have a high bar to avoid "warning fatigue" and false positives.
- **note**: Additional context or information explaining why the error or warning occurred. Notes should not contain actionable suggestions.
- **help**: Actionable information telling the user how to fix the problem. Often includes a suggestion string and an `Applicability` level. The error/warning should not itself suggest fixes; that belongs in `help`.
- **suggestion**: A structured code change, which is a special subtype of help. Suggestions can be machine-applicable.

[rustc-dev-guide] Lint levels (separate from diagnostic levels) are: `forbid`, `deny` (equivalent to error), `warn`, `allow`. `forbid` should never be the default for a lint. Lints that default to `allow` include those with high false positive rates, overly opinionated lints, and experimental lints.

[Clang] Clang defines its severity levels in `Diagnostic*Kinds.td` files as: `NOTE`, `REMARK`, `WARNING`, `EXTENSION`, `EXTWARN`, `ERROR`. These are mapped to the output enum `Diagnostic::Level`: `{Ignored, Note, Remark, Warning, Error, Fatal}`. `EXTENSION` is ignored by default; `EXTWARN` warns by default. `Fatal` should only be used for errors so severe that error recovery cannot sensibly continue (e.g., failure to `#include` a file).

The mapping between severity declaration and output level is configurable: `-pedantic` maps `EXTENSION` to `Warning`; `-pedantic-errors` maps it to `Error`. Individual diagnostics can be remapped via `-W` flags. The only non-remappable diagnostics are `NOTE` (always follows the severity of the preceding diagnostic) and errors (can only be mapped to `Fatal`, never downgraded to warnings).

### Primary Message, Labels, Notes, and Suggestions

[rustc-dev-guide] The anatomy of a rustc diagnostic:

1. **Level + Code + Message**: e.g., `error[E0308]: mismatched types`. The message should be general enough to make sense in isolation (as a standalone string in an IDE).
2. **File/line/column header**: `  --> src/main.rs:5:13`
3. **Source window with span annotations**: the source line(s), underlines, and inline labels.
4. **Sub-diagnostics**: zero or more notes, helps, and suggestions. Notes provide context; helps provide actionable steps; suggestions provide specific code.

The text style guide specifies:
- Start with a lowercase letter; do not end with punctuation.
- Do not use the word "illegal" (use "invalid").
- Use backticks for code and identifiers in messages: `` the identifier `foo.bar` is invalid ``.
- Be succinct; users see these messages repeatedly.
- Do not refer to the compiler as "Rust" or "rustc"; say "the compiler."
- Use the Oxford comma in lists.

### Rustc's `Diagnostic` Struct Fields

[rustc-dev-guide] The `Diag` builder (formerly `DiagnosticBuilder`) is obtained from `DiagCtxt` methods such as `struct_span_err`. It accumulates:
- Primary message (a `DiagMessage`)
- Primary span (a `Span`, set via `set_span` or `#[primary_span]`)
- Span labels (via `span_label`)
- Notes, helps, warnings (via `.note()`, `.help()`, `.warn()`)
- Suggestions (via `span_suggestion`, with `Applicability`)
- Error code (optional, used for `--explain`)

The `Diag` is emitted by calling `.emit()`. Failing to emit or cancel a `Diag` produces an Internal Compiler Error (ICE) — a design that prevents diagnostics from being silently dropped.

[rustc-dev-guide] For the derive-based API, a diagnostic is a struct annotated with `#[derive(Diagnostic)]`:

```rust
#[derive(Diagnostic)]
#[diag("field `{$field_name}` is already declared", code = E0124)]
pub struct FieldAlreadyDeclared {
    pub field_name: Ident,
    #[primary_span]
    #[label("field already declared")]
    pub span: Span,
    #[label("`{$field_name}` first declared here")]
    pub prev_span: Span,
}
```

Fields without annotations are passed as arguments to the Fluent message template. The derive generates an implementation of `Diagnostic::into_diag` that builds and returns the `Diag` object.

### Clang's `DiagnosticsEngine` Approach

[Clang] In Clang, diagnostics are declared in `Diagnostic*Kinds.td` TableGen files. Each entry specifies: a unique enum name (e.g., `err_typecheck_invalid_operands`), a severity, and an English format string with `%0`..`%9` argument slots.

The `Diag()` helper method creates and dispatches a diagnostic:

```cpp
if (various things that are bad)
  Diag(Loc, diag::err_typecheck_invalid_operands)
    << lex->getType() << rex->getType()
    << lex->getSourceRange() << rex->getSourceRange();
```

Arguments are appended with `<<`. `SourceRange` arguments are also passed via `<<` and handled separately from typed arguments. The `DiagnosticsEngine` maps the diagnostic to an output level, then invokes the `DiagnosticConsumer` interface.

The `DiagnosticConsumer` (e.g., `TextDiagnosticPrinter`) is responsible for rendering. This separation allows the same diagnostic data to be rendered as terminal text, JSON, HTML, or any other format without changing the emission code.

---

## 5. Fix-it Hints and Suggestions

### Machine-Applicable vs. Human-Readable Suggestions

[rustc-dev-guide] Suggestions have four applicability levels:

- **`MachineApplicable`**: Can be applied mechanically and safely. The replacement is syntactically and semantically correct. Tools like `rustfix` apply these automatically when `--fix` is passed.
- **`HasPlaceholders`**: Cannot be applied mechanically because the suggestion contains placeholder text the user must fill in. Example: `try adding a type: let x: <type>`.
- **`MaybeIncorrect`**: The suggestion may or may not be correct depending on the user's intent. Requires human review.
- **`Unspecified`**: We do not know which of the above applies.

Be conservative: use `MachineApplicable` only when you are confident the replacement is correct in all cases where the error fires.

[Clang] Clang's fix-it hints must obey three rules:
1. They should only be used when there is high confidence they match the user's intent (since `-Xclang -fixit` applies them automatically).
2. Clang must recover from the error as if the fix-it had been applied.
3. Fix-its on warnings must not change the meaning of the code (they may clarify intent, e.g., adding parentheses when precedence is unclear).

If a fix-it cannot obey these rules, it must be placed on a `note`, not on the warning or error. Fix-its on notes are not applied automatically.

### How `rustc --fix` Uses Machine-Applicable Suggestions

[rustc-dev-guide] When `rustfix` processes a crate, it runs the compiler with `--error-format json` to obtain structured diagnostic output. The JSON representation of each suggestion includes:
- The span to replace (`byte_start`, `byte_end`, `file_name`)
- The replacement text (`suggested_replacement`)
- The applicability (`suggestion_applicability`)

`rustfix` collects all `MachineApplicable` suggestions and applies them to the source file as text replacements. The resulting file is then re-compiled to verify the fix did not introduce new errors.

The JSON diagnostic format also includes a `rendered` field containing the human-readable text representation, so tools can display the original diagnostic alongside the suggestion without recompiling.

### When to Offer a Suggestion vs. Just Explain the Error

[rustc-dev-guide] Style guidelines:
- Suggestions should not be phrased as questions. Avoid "did you mean X?" — prefer "there is a struct with a similar name: X" or "use X instead."
- The suggestion message should not say "the following" or "as shown" — the span conveys location.
- The message may contain function, variable, or type names, but should avoid whole expressions.
- Do not suggest a fix in the error or warning message itself; suggestions belong in `help` sub-diagnostics.

Offer a suggestion when:
- The fix is unambiguous and high-confidence (e.g., a missing semicolon, a typo in an identifier name, a straightforward type annotation).
- The fix is mechanical and the compiler has enough information to generate correct replacement text.

Do not offer a suggestion (use a `note` or `help` text instead) when:
- Multiple valid fixes exist that depend on user intent.
- The fix would require understanding of the program's semantics that the compiler does not have.
- The suggestion would require the user to fill in non-trivial content.

---

## 6. Error Recovery vs. Error Reporting Separation

### The Parser Continues After Errors

[rustc-dev-guide] Error recovery and error reporting are distinct responsibilities. The parser's job is to continue producing a usable AST even in the presence of syntax errors; the diagnostic system's job is to communicate those errors to the user.

rustc implements several recovery strategies: inserting missing tokens (e.g., inferring a missing `;`), skipping to a synchronization point (e.g., the next statement boundary), and producing "error nodes" in the AST that propagate a "this subtree contains an error" flag.

[Clang] Clang's AST has a `ContainsErrors` bit on nodes. When an expression or declaration contains errors, the `ContainsErrors` bit is set. Subsequent analysis passes check this bit to suppress downstream diagnostics that would be caused by the malformed node rather than by a genuine secondary error.

Clang also has a `RecoveryExpr` node type: when a parse or semantic error occurs in an expression context, Clang can synthesize a `RecoveryExpr` with the correct type (often `int` or an error type) so that the rest of the expression context can still be type-checked and further errors reported.

### Delayed vs. Immediate Errors

[rustc-dev-guide] Some errors must be delayed until more information is available. For example, the borrow checker may discover type errors during monomorphization that could not be reported at the point of the definition. rustc's query system allows diagnostics to be buffered and re-associated with the original source spans when they are finally emitted.

The `buffer_lint` mechanism on `Session` and `ParseSess` is used for lints that fire before the linting system is initialized (e.g., during parsing or macro expansion). These buffered lints are held until the linting system is ready.

In the parser, `rustc_ast` cannot depend on `rustc_lint` (to avoid circular dependencies), so it defines its own buffered lint type. After macro expansion, these lints are moved into `Session::buffered_lints`.

The flag `-Z treat-err-as-bug=N` causes the Nth error to be treated as an ICE, printing a stack trace. This is used by compiler developers to locate where a diagnostic is emitted when grepping for the message text is insufficient.

### Error Count Limits and Cascading Error Suppression

[rustc-dev-guide] rustc stops compilation after a configurable number of errors (default: 50) to avoid overwhelming the user with error floods caused by a single root error. This limit is configurable via `-Z error-limit`.

Cascading errors — secondary errors caused by the same root problem — are suppressed using "poisoned" types. When the type checker cannot determine the type of an expression due to an earlier error, it assigns the expression a special "error type" (`TyKind::Error`). Subsequent type-checking operations on the error type are silently ignored, preventing the single root error from generating dozens of confusing downstream messages.

[Clang] Clang has a similar error limit. The `DiagnosticsEngine` can be configured with a maximum error count; once reached, further errors are suppressed (though notes on existing errors are still attached).

---

## 7. Diagnostic Quality Principles

[rustc-dev-guide] The rustc style guide encodes the following quality principles:

**Point to the cause, not the symptom.** Span the smallest amount of code that still signifies the issue. If the problem is a missing `mut` qualifier, span the variable binding, not the entire function body.

**Explain what the user probably meant.** The compiler has context the user does not: the full type of an expression, the expected type from the call site, the set of trait bounds. Use that context in the message. The `#[rustc_on_unimplemented]` attribute allows library authors to annotate traits with custom diagnostic messages that fire when a trait bound is not satisfied — the message can reference the concrete type and the missing implementation.

**Show the fix, not just the problem.** Where possible, include a `help` sub-diagnostic that shows the corrected code. The correction must be accurate enough that the user can apply it directly.

**The "expected X, found Y" pattern.** The most common pattern in rustc diagnostics is `error[E0308]: mismatched types` followed by a note like `expected type X, found type Y`. The context provided by both types allows the user to understand not just that there is a mismatch but what the mismatch is and where it comes from.

[Clang] Clang's wording guidelines:
- Diagnostics do not start with a capital letter and do not end with punctuation (except a trailing `?` is allowed, e.g., "unknown identifier %0; did you mean %1?").
- Wording should be actionable. Prefer "missing semicolon" over "syntax error" (not actionable) or "expected unqualified-id" (uses standards terminology).
- Wording should explain what is wrong rather than restate what the code does. Prefer "type %0 requires a value in the range %1 to %2" over "%0 is invalid."
- Wording should have enough context to help the user in a complex expression: "both sides of the %0 binary operator are identical" rather than "identical operands to binary operator."
- Do not use random English strings as arguments to diagnostics. This prevents localization. Use `%select{option0|option1|option2}N` for localized alternatives.

[rustc-dev-guide] rustc style also specifies: try not to emit multiple error messages for the same error; when the compiler has too little information, consult the team about adding attributes (like `#[rustc_on_unimplemented]`) to library code.

---

## 8. Internationalization: rustc's Translation Infrastructure

[rustc-dev-guide] rustc's diagnostic infrastructure supports translatable diagnostics using **Fluent**, a localization system built around "asymmetric localization." The key insight of Fluent is that translations should be able to express things differently from the source language without requiring changes to the compiler source. Before Fluent, rustc used string interpolation (`format!`), which is difficult to translate because:
- Some languages require different word order.
- Grammatical agreement (number, gender, case) may require different interpolation points.
- A natural translation may require more or fewer substituted values than the English string.

**Fluent resource files** define messages in the form:
```
typeck_address_of_temporary_taken = cannot take address of a temporary
    .label = temporary value
```

The identifier `typeck_address_of_temporary_taken` names the message. Sub-messages (for labels, notes, etc.) are "attributes" using the `.attribute-name` syntax. Arguments are referenced as `{$arg_name}`:
```
typeck_struct_expr_non_exhaustive =
    cannot create non-exhaustive {$what} using struct expression
```

**Guidelines for message naming:** Use `_` as the word separator (not `-`, despite Fluent's convention), to match Rust identifier conventions. The only exception is leading `-` in names like `-passes_see_issue`.

**Compile-time validation:** rustc's `#[derive(Diagnostic)]` macro validates Fluent messages at compile time. Parse errors in Fluent resources cause build failures. Duplicate message identifiers also produce compile errors.

**Typed identifiers:** The `DiagMessage` type can hold either a legacy `String` (non-translatable) or a typed Fluent message identifier. All existing diagnostic APIs accept anything convertible to `DiagMessage`, so legacy non-translatable diagnostics continue to work unchanged. New diagnostics should use typed identifiers.

**Arguments to Fluent messages** are provided via `set_arg(name, value)`. Values are `DiagArgValue` (string or number). Compiler types like `Ty<'tcx>` implement `IntoDiagArg` with conversions to string or number. The derive macro inserts `set_arg` calls automatically for struct fields.

**Current status (as of 2024):** The translation infrastructure is pending a redesign. The current system works but causes contributor friction. Usage is not mandated — contributors may use the existing infrastructure or bypass it when they need more flexibility. See the tracking issue https://github.com/rust-lang/rust/issues/132181.

[Clang] Clang's current status: "Not possible yet! Diagnostic strings should be written in UTF-8, the client can translate to the relevant code page if needed. Each translation completely replaces the format string for the diagnostic." Clang recognizes the need but has not implemented it; the `%select`, `%plural`, `%ordinal`, and `%s` format specifiers were designed with localization in mind (avoiding raw English strings as arguments), but actual runtime translation infrastructure does not exist.

---

## 9. Implementation Decision Guide: Minimal Viable vs. Full System

### Minimal Viable Diagnostics

For a new compiler or language tool, a minimal viable diagnostic system needs:

1. **A span type**: Store `(file_id, byte_start, byte_end)`. Use a `SourceMap` to compute `(line, col)` lazily. File IDs can be indices into a file table.

2. **Caret rendering**: Given a span, print the source line and a `^` underline. Two lines of code, but deeply useful.

3. **Severity levels**: At minimum, `Error` and `Warning`. Add `Note` and `Help` when you need to attach context.

4. **Error poisoning**: Define an "error type" sentinel that silences downstream diagnostics. This single decision prevents error floods.

5. **Error count limit**: Hard-stop after 20–50 errors. Make it configurable.

### Full Production System

A full system adds:

6. **Multi-span diagnostics**: Primary span + labeled secondary spans. The Clang/rustc approach: one primary, N secondaries with independent labels.

7. **Fix-it hints with applicability levels**: Four-tier system (MachineApplicable, HasPlaceholders, MaybeIncorrect, Unspecified). Required for `--fix`-style tooling.

8. **JSON output format**: Parallel to text output; consumed by IDEs and tools. The `rendered` field (human text embedded in JSON) is a quality-of-life feature for tools.

9. **Structured diagnostic types** (`#[derive(Diagnostic)]`): Separates diagnostic definition from emission. Enforces that every diagnostic has a span; prevents ICEs from unfinalized builders.

10. **Lint infrastructure**: Separate from hard errors. Requires lint levels (`allow/warn/deny/forbid`), `#[allow(...)]` attribute handling, buffering for early-phase lints.

11. **Macro span hygiene**: Expansion chains in span data. Required to produce useful errors on macro-expanded code.

12. **Translation/i18n**: Fluent or equivalent. Defer this; it is a significant infrastructure investment and rustc itself has not completed it.

### Key Architecture Decisions

**Separate the diagnostic builder from emission.** (rustc's `Diag` pattern, Clang's `<<` chaining.) This allows diagnostics to be constructed incrementally as more information becomes available, and ensures the diagnostic is complete before it is shown to the user.

**Make diagnostics strongly typed.** (rustc's `#[derive(Diagnostic)]`.) Struct-based diagnostics catch missing spans at compile time and make it impossible to forget to emit a diagnostic.

**Decouple rendering from structure.** (Clang's `DiagnosticConsumer`.) The same diagnostic data should be renderable as terminal text, JSON, or IDE annotations. Hardcoding the rendering into the emission site is a common mistake that makes tooling impossible.

**Use error types, not exceptions.** (Clang's `ContainsErrors` bit, rustc's `TyKind::Error`.) Propagating error sentinels through the type system is simpler and more predictable than using exception-based control flow for error recovery.

**Buffer early-phase diagnostics.** (rustc's `buffer_lint`, `ParseSess::buffered_lints`.) Diagnostics emitted before the diagnostic system is initialized must be buffered, not dropped. Design for this from the start.

---

## Code Reference: Key Types and APIs

### rustc

| Type/Function | Location | Purpose |
|---|---|---|
| `Span` | `rustc_span` | Byte range + file reference |
| `SourceMap` | `rustc_span` | Maps spans to file/line/col |
| `DiagCtxt` | `rustc_errors` | Creates and emits diagnostics |
| `Diag<'a, G>` | `rustc_errors` | Builder for a single diagnostic |
| `DiagMessage` | `rustc_error_messages` | Translatable or legacy message |
| `Applicability` | `rustc_errors` | Suggestion confidence level |
| `#[derive(Diagnostic)]` | `rustc_macros` | Struct-based diagnostic definition |
| `#[derive(Subdiagnostic)]` | `rustc_macros` | Struct-based sub-diagnostic |
| `buffer_lint` | `Session`, `ParseSess` | Buffer lint for deferred emission |

### Clang

| Type/Function | Location | Purpose |
|---|---|---|
| `SourceLocation` | `clang/Basic/SourceLocation.h` | 32-bit opaque file+offset value |
| `SourceRange` | `clang/Basic/SourceLocation.h` | Token-start-granularity range |
| `CharSourceRange` | `clang/Basic/SourceLocation.h` | Character-granularity range |
| `SourceManager` | `clang/Basic/SourceManager.h` | All location lookups |
| `DiagnosticsEngine` | `clang/Basic/Diagnostic.h` | Diagnostic dispatch and mapping |
| `DiagnosticConsumer` | `clang/Basic/Diagnostic.h` | Rendering interface |
| `FixItHint` | `clang/Basic/Diagnostic.h` | Fix-it hint data |
| `Diag(Loc, diag::X)` | Sema, Parser, etc. | Emit a diagnostic inline |
| `Diagnostic*Kinds.td` | `clang/Basic/` | TableGen diagnostic declarations |
