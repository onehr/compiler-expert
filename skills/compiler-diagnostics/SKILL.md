---
name: compiler-diagnostics
description: Compiler diagnostics and error reporting expert — covers span tracking, caret-style error display, fix-it hints, error recovery vs reporting separation, and treating diagnostics as a first-class product. Use when designing the error reporting system for a compiler or language tool.
---

# compiler-diagnostics

You are an expert in compiler diagnostics and error reporting. Use this skill when the user is designing or implementing the error reporting system for a compiler, language server, or any language tool. Good diagnostics are a first-class product feature — they are the primary user interface of a compiler.

**Reference material is in [`references/diagnostics-reference.md`](references/diagnostics-reference.md).** That file contains deep, source-verified content from the rustc developer guide and Clang internals manual on all topics below.

---

## Span Design

A span is a half-open byte range `[start, end)` into a source file, plus a file identifier. It is the fundamental currency of all diagnostic information. Key points:

- Store byte offsets (compact, fast); compute `(line, col)` lazily from a line-offset table in a `SourceMap`
- Attach spans to every token, AST node, HIR node, and IR value — carry them through every lowering pass
- Spans in macro-expanded code carry a `SyntaxContext` encoding the expansion chain; use `Span::from_expansion()` to detect macro-generated spans
- Synthetic spans for desugared constructs must point at the original user-written syntax

See reference: **Section 2 — Span Design**

---

## Error Message Quality

- **Caret-style display** (pioneered by Clang): print the source line, then `^` under the primary token, `~` for range extents
- **Primary vs. secondary spans**: one primary (the error site, label should be self-contained), N secondaries with labels for context
- **Severity levels**: `error`, `warning`, `note`, `help` (actionable, may contain suggestions), `suggestion` (structured code replacement)
- **Message style**: lowercase first letter, no trailing period, use backticks for code/identifiers, avoid "illegal" (use "invalid"), be succinct
- **Error codes** (`E0308`, etc.): link to extended `--explain` text; only add when the explanation adds value beyond the inline message
- **Suppress cascading errors** using error-type sentinels (`TyKind::Error` in rustc, `ContainsErrors` bit in Clang)

See reference: **Sections 3, 4, 7 — Caret Display, Diagnostic Structure, Quality Principles**

---

## Fix-it Hints

- A fix-it hint replaces a span with new text; can be applied automatically by `rustfix` / `clang-apply-replacements`
- **Applicability levels** (rustc): `MachineApplicable`, `HasPlaceholders`, `MaybeIncorrect`, `Unspecified` — be conservative
- **Clang rules**: fix-its on errors/warnings must match user intent, Clang must recover as if applied, must not change code semantics; put uncertain fix-its on `note` (not auto-applied)
- Don't phrase suggestions as questions ("did you mean?"); use factual language ("there is a struct with a similar name: X")

See reference: **Section 5 — Fix-it Hints / Suggestions**

---

## Error Recovery vs. Reporting

- **Separate concerns**: recovery is the parser's job (produce an AST); reporting is the diagnostic system's job (format and emit)
- **Error sentinels**: propagate "error type" / `ContainsErrors` through the type system to silence downstream cascade errors
- **Error limit**: stop after N errors (rustc default: 50); make it configurable
- **Buffer early lints**: lints fired before the lint system is ready (e.g., during parsing) must be buffered, not dropped
- **Deferred diagnostics**: errors discovered in later passes must still carry original source spans

See reference: **Section 6 — Error Recovery vs. Error Reporting Separation**

---

## Internationalization

rustc uses **Fluent** for translatable diagnostics. Messages are defined in `.ftl` resource files with typed identifiers; arguments are passed as `DiagArgValue`. The `#[derive(Diagnostic)]` macro validates Fluent messages at compile time.

As of 2024, the rustc translation infrastructure is pending redesign — usage is not mandated. Clang has not yet implemented runtime translation.

See reference: **Section 8 — Internationalization**

---

## Implementation Decision Guide

**Minimal viable system** (5 pieces): span type, caret renderer, severity levels, error-type sentinel, error count limit.

**Full production system** adds: multi-span diagnostics, fix-it hints with applicability, JSON output, struct-based diagnostics (`#[derive(Diagnostic)]`), lint infrastructure, macro span hygiene, i18n.

**Key architecture decisions**: separate builder from emission; make diagnostics strongly typed; decouple rendering from structure; use error types not exceptions; buffer early-phase diagnostics.

See reference: **Section 9 — Implementation Decision Guide**

---

## Reference Sources

- **[`references/diagnostics-reference.md`](references/diagnostics-reference.md)** — comprehensive deep reference with source-verified content from rustc and Clang
- https://rustc-dev-guide.rust-lang.org/diagnostics.html
- https://rustc-dev-guide.rust-lang.org/diagnostics/diagnostic-structs.html
- https://rustc-dev-guide.rust-lang.org/diagnostics/translation.html
- https://clang.llvm.org/docs/InternalsManual.html#the-diagnostic-subsystem

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-expert` — routes here when the question is about error messages or reporting
- Any front-end skill (`compiler-lexer`, `compiler-parser`, `compiler-name-resolution`,
  `compiler-type-system`) — diagnostics is a cross-cutting concern active in all phases

**Trace Down ↓** — this is a leaf skill; no further delegation
- (none — diagnostics infrastructure is consumed by all other phases)

**Related →** — other front-end and cross-cutting skills
- `compiler-lexer` — source spans originate in the lexer
- `compiler-parser` — parse errors need good recovery + messages
- `compiler-name-resolution` — undefined-name errors produced here
- `compiler-type-system` — type errors are the richest diagnostic target
