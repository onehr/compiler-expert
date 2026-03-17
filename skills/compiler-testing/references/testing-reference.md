---
name: compiler-testing
description: >
  Compiler testing and CI expert — covers the full testing stack for compiler projects:
  FileCheck-style IR verification, snapshot testing with insta, differential testing against
  reference compilers (gcc/clang), property-based testing with proptest, coverage-guided
  fuzzing with cargo-fuzz, test suite organization (unit/integration/corpus), and CI pipeline
  design for Rust compiler projects. Use when designing a test strategy for a new compiler,
  setting up CI, writing regression tests for a compiler pass, or debugging why a test suite
  is missing a class of bugs.
---

# Compiler Testing and CI — Comprehensive Reference

This document covers the full testing stack for compiler projects written in Rust. It is a
reference for practitioners, not an introduction. All code examples are intended to be
copy-paste usable in a real project.

---

## Part I — Testing Philosophy for Compilers

### The Oracle Problem

Testing a compiler is fundamentally harder than testing most application software because the
"correct" answer is not obvious. For a web server, you know what the response body should be.
For a compiler, you need to know what the correct binary output is for an arbitrary input
program — and that answer is defined by a language specification that can run to hundreds of
pages.

This creates the oracle problem: given a C source file and our compiler's output, how do we
know if the output is right?

There are four common oracle strategies, each with different tradeoffs:

1. **Specification oracle** — you read the spec and manually write the expected output. High
   confidence, high cost, impossible to scale.
2. **Reference compiler oracle** — you compare against gcc or clang. Scalable, but requires
   your compiler to be able to compile the same programs.
3. **Exit-code oracle** — for any program that returns an integer, the exit code is a
   zero-cost oracle. Compile with your compiler AND with gcc; run both; compare exit codes.
4. **Crash oracle** — the compiler must never crash on any input, valid or not. This is the
   weakest oracle but it is free and catches many real bugs.

### The Four Test Goals

A complete test strategy for a compiler must cover all four of these goals:

**Correctness (no miscompilation)** — The compiler produces code that, when run, behaves
according to the language semantics. A miscompilation is the worst failure mode: the code
compiles silently but runs wrong. Differential testing is the primary tool for catching
miscompilations.

**Robustness (no crash on bad input)** — The compiler should never panic, segfault, or produce
an ICE (internal compiler error) on any input, including malformed, malicious, or
syntactically invalid input. Fuzzing is the primary tool for finding robustness failures.

**Performance (no regression)** — Compilation time must not regress. Generated code quality
must not regress. Benchmarks and differential testing of optimization output cover this.

**Diagnostics quality (right errors, right spans)** — When the user writes invalid code, the
compiler should emit the correct error code, point to the correct source location, and produce
a helpful message. Snapshot testing and expected-diagnostic tests cover this.

### The Test Pyramid

Apply the test pyramid rigorously to compiler testing:

```
              /\
             /  \
            / diff\          Differential tests (slow, powerful)
           /  tests \        Run on push to main, not every PR
          /----------\
         /            \
        / integration  \     Integration tests / corpus tests
       /  (corpus ok/)  \    Run on every PR, < 2 minutes
      /------------------\
     /                    \
    /    unit tests        \  #[test] + rstest parameterized
   /  (fast, isolated)      \  Run on every commit, < 30 seconds
  /----------------------------\
```

The key tension in compiler testing is that the most powerful tests (differential, fuzzing)
are the slowest. The solution is to layer them: unit tests run on every commit, differential
tests run on merge to main, fuzzing runs in a dedicated scheduled job.

### The "Exit Code as Oracle" Trick

The cheapest possible correctness check for a C program is its exit code. A C `main` that
returns an integer value makes that value available as the process exit code (modulo 256 on
Unix). This means you can write test programs like:

```c
// tests/differential/cases/add_two_ints.c
int add(int a, int b) { return a + b; }
int main(void) { return add(3, 4) - 7; }
```

This program returns 0 if `add` is correct and non-zero otherwise. You do not need to parse
stdout or compare files. Just check the exit code. This is fast, reliable, and requires no
test infrastructure beyond being able to run a binary.

The constraint is that the oracle value must be in the range 0-127 to be portable. Use
`return (result == expected) ? 0 : 1;` patterns for non-trivial checks.

### Unit Tests vs Differential Tests

There is a critical distinction between what unit tests and differential tests tell you:

- **Unit tests tell you WHERE it broke** — "The type checker rejected a valid pointer cast in
  `sema/cast_check.rs` at line 142."
- **Differential tests tell you THAT it broke** — "The program `array_sum.c` now returns 7
  with our compiler but 0 with gcc."

You need both. A differential test failure starts an investigation; unit tests end it. If you
have only differential tests, debugging is extremely slow. If you have only unit tests, you
will miss whole classes of miscompilations that no single unit tests for.

---

## Part II — Test Suite Structure

### Directory Layout

```
tests/
├── c11-corpus/
│   ├── ok/                     # Programs that must compile and run cleanly
│   │   ├── pointer_deref.c
│   │   ├── struct_return.c
│   │   ├── array_decay.c
│   │   └── ...
│   ├── err/                    # Programs that must produce specific diagnostics
│   │   ├── undeclared_var.c
│   │   ├── undeclared_var.expected_diag
│   │   ├── type_mismatch_assign.c
│   │   ├── type_mismatch_assign.expected_diag
│   │   └── ...
│   └── snapshots/              # insta snapshot files (committed to VCS)
│       ├── snapshot_undefined_name_error.snap
│       ├── snapshot_ast_print_struct.snap
│       └── ...
├── differential/
│   ├── cases/                  # C programs used as differential oracles
│   │   ├── add_two_ints.c
│   │   ├── recursive_fib.c
│   │   ├── struct_field_access.c
│   │   └── ...
│   └── run.py                  # Harness script
├── filecheck/                  # IR-level FileCheck tests
│   ├── codegen_iconst.txt
│   ├── codegen_load_store.txt
│   └── ...
└── fuzz/
    └── fuzz_targets/
        ├── parse.rs
        ├── sema.rs
        └── codegen.rs
```

### Corpus Organization Rules

**`ok/` files** follow strict rules:
- One feature per file. Do not combine features.
- File name describes the feature: `pointer_deref.c`, not `test1.c`.
- Minimal. A corpus file should be 5-30 lines. If it is longer, split it.
- Must produce exit code 0 when compiled and run by the reference compiler.
- No undefined behavior. If the program contains UB, it is not a valid oracle.

**`err/` files** follow strict rules:
- Each file tests exactly one error code.
- Each `.c` file has a `.expected_diag` sidecar in the same directory.
- The `.expected_diag` file contains the expected error code, the line number, and the column
  span. Example format:

```
# undeclared_var.expected_diag
error: E_UNDECLARED_NAME
line: 3
col_start: 16
col_end: 17
```

- The test runner reads the sidecar and verifies the compiler emits exactly that diagnostic.
- Do not put multiple errors in one `err/` file unless you are explicitly testing error
  recovery (in which case, document it clearly).

**Snapshot files** are committed to version control. They are never auto-generated without
review. The workflow is:
1. Write the test.
2. Run `cargo insta test` — this generates the snapshot file in `.snap.new` files.
3. Run `cargo insta review` — this opens a diff for each new snapshot for you to accept or
   reject.
4. Only after review, commit the `.snap` files.
5. In CI, run with `--unreferenced=reject` to fail if any snapshot file is stale.

---

## Part III — Unit Tests with Rust `#[test]` and rstest

### When to Use `#[test]` vs rstest

Use plain `#[test]` for:
- A specific edge case that needs a unique assertion and commentary.
- Tests for internal error paths where the input is constructed programmatically.
- Tests that require significant setup code that would not generalize.

Use `rstest` parameterized tests for:
- Running the same assertion over multiple inputs (values, strings, file paths).
- Corpus-driven tests: "for every file in this directory, this property must hold."
- Table-driven tests where the same test logic applies to many (input, expected_output) pairs.

Add to `Cargo.toml`:
```toml
[dev-dependencies]
rstest = "0.18"
bumpalo = "3"
```

### Pattern: Lexer Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crust_lexer::{lex, TokenKind};

    #[test]
    fn lex_integer_literal() {
        let (tokens, errors) = lex("42");
        assert!(errors.is_empty(), "unexpected lex errors: {errors:?}");
        assert_eq!(tokens.len(), 2); // literal + EOF
        assert!(matches!(tokens[0].kind, TokenKind::IntLiteral(42)));
    }

    #[test]
    fn lex_identifier() {
        let (tokens, errors) = lex("foo_bar");
        assert!(errors.is_empty(), "unexpected lex errors: {errors:?}");
        assert!(matches!(tokens[0].kind, TokenKind::Ident));
        assert_eq!(tokens[0].text, "foo_bar");
    }

    #[test]
    fn lex_unknown_char_produces_error() {
        let (_tokens, errors) = lex("@");
        assert!(!errors.is_empty(), "expected a lex error for '@'");
        assert!(matches!(errors[0], LexError::UnknownChar { ch: '@', .. }));
    }

    // Every error variant must be tested.
    #[test]
    fn lex_unterminated_string_error() {
        let (_tokens, errors) = lex("\"hello");
        assert!(
            errors.iter().any(|e| matches!(e, LexError::UnterminatedString { .. })),
            "expected UnterminatedString error, got: {errors:?}"
        );
    }
}
```

### Pattern: Parser Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn parse(src: &str) -> Result<TranslationUnit, Vec<ParseError>> {
        let (tokens, lex_errs) = crust_lexer::lex(src);
        assert!(lex_errs.is_empty(), "lex errors in parse test input: {lex_errs:?}");
        let arena = bumpalo::Bump::new();
        let mut parser = crust_parser::Parser::new(&tokens, &arena);
        parser.parse_translation_unit()
    }

    #[test]
    fn parse_empty_function() {
        let result = parse("void f(void) {}");
        assert!(result.is_ok(), "expected parse success, got: {result:?}");
    }

    #[test]
    fn parse_missing_closing_brace_error() {
        let result = parse("void f(void) {");
        assert!(result.is_err(), "expected parse error");
        let errors = result.unwrap_err();
        assert!(
            errors.iter().any(|e| matches!(e, ParseError::UnexpectedEof { .. })),
            "expected UnexpectedEof, got: {errors:?}"
        );
    }

    #[test]
    fn parse_pointer_declarator() {
        let result = parse("int *p;");
        assert!(result.is_ok(), "{result:?}");
        let unit = result.unwrap();
        // Verify the AST shape
        assert_eq!(unit.decls.len(), 1);
        assert!(matches!(unit.decls[0].ty, Type::Pointer(_)));
    }
}
```

### Pattern: Semantic Analysis Tests

```rust
#[cfg(test)]
mod sema_tests {
    use crust_sema::{analyze, SemanticError};

    fn sema(src: &str) -> Result<TypedAst, Vec<SemanticError>> {
        let (tokens, lex_errs) = crust_lexer::lex(src);
        assert!(lex_errs.is_empty());
        let arena = bumpalo::Bump::new();
        let mut parser = crust_parser::Parser::new(&tokens, &arena);
        let ast = parser.parse_translation_unit().expect("parse failed in sema test");
        analyze(&ast)
    }

    #[test]
    fn sema_undeclared_variable_error() {
        let result = sema("int f(void) { return x; }");
        assert!(result.is_err(), "expected sema error");
        let errors = result.unwrap_err();
        assert_eq!(errors.len(), 1);
        assert!(
            matches!(
                errors[0],
                SemanticError::UndeclaredName { ref name, .. } if name == "x"
            ),
            "expected UndeclaredName(x), got: {:?}", errors[0]
        );
    }

    #[test]
    fn sema_type_mismatch_in_assignment() {
        let result = sema("void f(void) { int x; x = 1.5; }");
        assert!(result.is_err());
        assert!(
            result.unwrap_err().iter().any(|e| matches!(
                e,
                SemanticError::TypeMismatch { .. }
            )),
            "expected TypeMismatch error"
        );
    }

    #[test]
    fn sema_valid_pointer_cast_succeeds() {
        let result = sema("void f(void) { void *p; int *q = (int *)p; (void)q; }");
        assert!(result.is_ok(), "expected valid cast to succeed: {result:?}");
    }
}
```

### Corpus-Driven Parameterized Tests with rstest

The `#[files(...)]` attribute in rstest generates one test case per matching file. This means
adding a new `.c` file to the corpus automatically creates a new test case without changing
any Rust code.

```rust
use rstest::rstest;

#[rstest]
fn parses_cleanly(#[files("tests/c11-corpus/ok/*.c")] path: std::path::PathBuf) {
    let src = std::fs::read_to_string(&path).unwrap();
    let (tokens, lex_errs) = crust_lexer::lex(&src);
    assert!(
        lex_errs.is_empty(),
        "{path:?}: lex errors: {lex_errs:?}"
    );
    let arena = bumpalo::Bump::new();
    let mut parser = crust_parser::Parser::new(&tokens, &arena);
    assert!(
        parser.parse_translation_unit().is_ok(),
        "{path:?}: parse failed"
    );
}

#[rstest]
fn compiles_cleanly(#[files("tests/c11-corpus/ok/*.c")] path: std::path::PathBuf) {
    let src = std::fs::read_to_string(&path).unwrap();
    let result = crust_driver::compile_to_binary(&src, CompileOptions::default());
    assert!(
        result.is_ok(),
        "{path:?}: compilation failed: {result:?}"
    );
}

#[rstest]
fn emits_expected_diagnostic(
    #[files("tests/c11-corpus/err/*.c")] path: std::path::PathBuf,
) {
    let sidecar = path.with_extension("expected_diag");
    let expected = parse_expected_diag(&sidecar)
        .unwrap_or_else(|e| panic!("{path:?}: failed to read expected_diag: {e}"));

    let src = std::fs::read_to_string(&path).unwrap();
    let result = crust_driver::compile_to_binary(&src, CompileOptions::default());
    assert!(result.is_err(), "{path:?}: expected error but compilation succeeded");

    let errors = result.unwrap_err();
    assert!(
        errors.iter().any(|e| e.code == expected.code
            && e.span.line == expected.line
            && e.span.col_start == expected.col_start),
        "{path:?}: expected diagnostic {expected:?}, got: {errors:?}"
    );
}
```

### Anti-Pattern: Testing Only the Happy Path

Every error enum variant must have at least one test. If `SemanticError` has 12 variants and
your test suite only covers 4 of them, you have no confidence in the other 8.

A practical approach: write a test file named after the error variant. Use a doc-comment on
the variant itself to link to the test:

```rust
pub enum SemanticError {
    /// Test: `tests/c11-corpus/err/undeclared_var.c`
    UndeclaredName { name: String, span: Span },
    /// Test: `tests/c11-corpus/err/type_mismatch_assign.c`
    TypeMismatch { expected: Type, found: Type, span: Span },
    // ...
}
```

---

## Part IV — Snapshot Testing with `insta`

### Setup

```toml
[dev-dependencies]
insta = { version = "1", features = ["yaml"] }
```

### What to Snapshot

**Good snapshot targets:**
- Rendered diagnostic output (the full string the user sees, including caret display)
- AST pretty-prints (when you have a stable pretty-printer)
- HIR or IR dumps (when the IR format is reasonably stable)
- Error message text (to catch unintended wording changes)
- Help text / notes attached to diagnostics

**Bad snapshot targets:**
- Internal struct debug representations (`{:?}`) of data structures that change frequently
  during development — you will spend all your time re-blessing instead of writing features
- Timing data, timestamps, or anything that varies between runs
- File paths from the host machine (they differ between developer machines and CI)
- Memory addresses

### The Three Snapshot Macros

`assert_snapshot!(value)` — value is a `String` or `&str`. Use this for pre-rendered
diagnostic output where you control the exact format.

`assert_debug_snapshot!(value)` — uses `{:?}`. Convenient but fragile; the output changes
whenever you add or rename a field. Use only for stable, simple types.

`assert_yaml_snapshot!(value)` — requires `serde::Serialize` on the type. The best choice
for structured data (AST nodes, diagnostic structs) because the output is readable and stable
across Rust version changes that might affect `{:?}` formatting.

### Workflow

```bash
# Step 1: Run tests. New snapshots appear as .snap.new files; tests fail.
cargo insta test

# Step 2: Review each new/changed snapshot interactively.
cargo insta review
# This shows a diff and prompts: [a]ccept / [r]eject / [s]kip

# Step 3: Commit the .snap files.
git add tests/c11-corpus/snapshots/
git commit -m "test: update snapshots for error rendering"
```

In CI, never run `cargo insta review`. Instead:
```bash
cargo insta test --unreferenced=reject
```

This fails if:
- Any snapshot has changed but not been reviewed (`.snap.new` file exists).
- Any snapshot file exists that is no longer referenced by any test (stale snapshot).

### Naming Conventions

Use descriptive names, not generic numeric names:

```rust
// Bad
insta::assert_snapshot!(rendered);          // generates: snapshot_1.snap

// Good
insta::assert_snapshot!("undefined_name_error", rendered);   // generates: undefined_name_error.snap
insta::assert_snapshot!("ast_print_struct_decl", rendered);
```

### Example: Diagnostic Snapshot Test

```rust
#[test]
fn snapshot_undefined_name_error() {
    let src = "int f(void) { return x; }";
    let ctx = sema(src);
    let rendered = ctx
        .errors
        .iter()
        .map(|e| format!("{e}"))
        .collect::<Vec<_>>()
        .join("\n");
    insta::assert_snapshot!("undefined_name_error", rendered);
}
```

The corresponding snapshot file (`undefined_name_error.snap`) after acceptance:

```
---
source: src/sema/tests.rs
expression: rendered
---
error[E0001]: undeclared name `x`
 --> <stdin>:1:22
  |
1 | int f(void) { return x; }
  |                      ^ not found in this scope
```

This snapshot now serves as a regression test. If you accidentally change the error code,
message text, or caret positioning, the test fails.

### Example: AST Pretty-Print Snapshot

```rust
#[test]
fn snapshot_ast_struct_declaration() {
    let src = "struct Point { int x; int y; };";
    let ast = parse(src).expect("parse failed");
    let pretty = crust_ast::PrettyPrinter::new().print_translation_unit(&ast);
    insta::assert_snapshot!("ast_struct_declaration", pretty);
}
```

---

## Part V — FileCheck-Style IR Verification

### The Problem

LLVM's `FileCheck` tool is the standard way to verify IR output in LLVM-based compilers: you
embed `; CHECK: pattern` comments in the test file and run FileCheck to verify they match the
IR. If your compiler is not LLVM-based, or you want IR tests that run as part of `cargo test`
without external tool dependencies, you need a pure-Rust substitute.

### Pure-Rust FileCheck Substitute

```rust
/// Minimal FileCheck-style IR verifier.
///
/// `patterns` is a slice of `(pattern, must_match)` pairs.
/// - `must_match = true` corresponds to `; CHECK:` — the pattern must appear in order.
/// - `must_match = false` corresponds to `; CHECK-NOT:` — the pattern must NOT appear
///   anywhere after the current position.
///
/// Ordering is enforced: each CHECK pattern must appear AFTER the previous one.
fn check_ir(ir: &str, patterns: &[(&str, bool)]) {
    let mut last_pos = 0;
    for (pat, must_match) in patterns {
        if *must_match {
            let search_slice = &ir[last_pos..];
            let found = search_slice.find(pat);
            assert!(
                found.is_some(),
                "CHECK: {pat:?} not found in IR after position {last_pos}.\nFull IR:\n{ir}"
            );
            last_pos += found.unwrap() + pat.len();
        } else {
            let search_slice = &ir[last_pos..];
            assert!(
                !search_slice.contains(pat),
                "CHECK-NOT: {pat:?} unexpectedly found in IR after position {last_pos}.\nFull IR:\n{ir}"
            );
        }
    }
}

#[test]
fn codegen_return_constant_uses_iconst() {
    let ir = compile_to_ir("int f(void) { return 42; }");
    check_ir(&ir, &[
        ("iconst",  true),   // CHECK: iconst
        ("call",    false),  // CHECK-NOT: call (no function call overhead)
    ]);
}

#[test]
fn codegen_struct_field_load_uses_offset() {
    let ir = compile_to_ir(
        "struct P { int x; int y; }; int get_y(struct P p) { return p.y; }"
    );
    check_ir(&ir, &[
        ("load",    true),   // CHECK: load (at least one load)
        ("offset",  true),   // CHECK: offset (field access via offset calculation)
        ("malloc",  false),  // CHECK-NOT: malloc (no heap allocation for value types)
    ]);
}

#[test]
fn codegen_short_circuit_and_skips_rhs() {
    // In `a && b`, if `a` is false, `b` must not be evaluated.
    // Verify this in the IR by checking for a conditional branch.
    let ir = compile_to_ir("int f(int a, int b) { return a && b; }");
    check_ir(&ir, &[
        ("branch",  true),   // CHECK: branch (short-circuit requires a branch)
    ]);
}
```

### LLVM FileCheck Directive Reference

If you are using LLVM FileCheck directly (via the `llvm-filecheck` crate or the external
binary), the following directives are available:

| Directive         | Meaning                                                              |
|-------------------|----------------------------------------------------------------------|
| `CHECK: pattern`  | Pattern must appear after the previous CHECK match.                  |
| `CHECK-NEXT: p`   | Pattern must appear on the very next non-empty line after previous.  |
| `CHECK-NOT: p`    | Pattern must NOT appear between this and the next positive CHECK.    |
| `CHECK-DAG: p`    | Pattern must appear somewhere in the block (order-independent).      |
| `CHECK-LABEL: p`  | Anchors a new section; resets ordering context.                      |
| `CHECK-SAME: p`   | Pattern must appear on the same line as the previous CHECK match.    |
| `{{regex}}`       | Regex match within a pattern. E.g., `iconst {{[0-9]+}}`.            |
| `[[name:regex]]`  | Capture group. E.g., `[[REG:%[a-z]+]]` then use `[[REG]]` later.   |

Example FileCheck test file (`tests/filecheck/codegen_iconst.txt`):

```
; RUN: crust-compile %s | filecheck %s
; CHECK-LABEL: define i32 @f
; CHECK:       iconst {{[0-9]+}}
; CHECK-NOT:   call
```

### Using `llvm-filecheck` Crate in Tests

```toml
[dev-dependencies]
llvm-filecheck = "0.9"
```

```rust
#[test]
fn filecheck_return_constant() {
    let ir = compile_to_ir("int f(void) { return 42; }");
    let checks = r#"
CHECK-LABEL: define i32 @f
CHECK: iconst 42
CHECK-NOT: call
"#;
    llvm_filecheck::check(checks, &ir)
        .expect("FileCheck assertions failed");
}
```

---

## Part VI — Differential Testing

Differential testing compares your compiler's output against a reference implementation
(typically gcc or clang). For C compilers, the exit code oracle is the most practical
approach: compile the same C program with both compilers, run both binaries, and compare exit
codes.

### When to Use Differential Testing

- Once your compiler can compile approximately 30% or more of valid C programs. Below this
  threshold, most test cases will fail for trivial reasons (missing features), making the
  signal-to-noise ratio too low.
- After any optimizer change. Even a small optimization pass can introduce miscompilations.
- After any ABI or calling-convention change.
- For every new language construct before closing the feature. The differential test should
  be written before the feature is considered done.

### The Differential Harness

```python
#!/usr/bin/env python3
"""
Differential test harness: compare our compiler's output against gcc.

Usage:
    python3 tests/differential/run.py
    python3 tests/differential/run.py tests/differential/cases/add_two_ints.c

Each test case in tests/differential/cases/ is compiled with:
  - Our compiler (crust-compile)
  - gcc -O0

Both binaries are executed and their exit codes are compared. A mismatch is a test failure.
"""

import subprocess
import sys
import os
import tempfile
from pathlib import Path
from dataclasses import dataclass
from typing import Optional


CRUST_COMPILE = os.environ.get("CRUST_COMPILE", "target/debug/crust-compile")
GCC = os.environ.get("GCC", "gcc")
CASES_DIR = Path("tests/differential/cases")
TIMEOUT_SECS = 5


@dataclass
class TestResult:
    path: Path
    passed: bool
    failure_reason: Optional[str] = None


def compile_with_crust(src: Path, out: Path) -> subprocess.CompletedProcess:
    return subprocess.run(
        [CRUST_COMPILE, str(src), "-o", str(out)],
        capture_output=True,
        text=True,
        timeout=TIMEOUT_SECS,
    )


def compile_with_gcc(src: Path, out: Path) -> subprocess.CompletedProcess:
    return subprocess.run(
        [GCC, "-O0", str(src), "-o", str(out)],
        capture_output=True,
        text=True,
        timeout=TIMEOUT_SECS,
    )


def run_binary(binary: Path) -> Optional[int]:
    try:
        result = subprocess.run(
            [str(binary)],
            capture_output=True,
            timeout=TIMEOUT_SECS,
        )
        return result.returncode
    except subprocess.TimeoutExpired:
        return None


def run_case(src: Path) -> TestResult:
    with tempfile.TemporaryDirectory() as tmpdir:
        tmp = Path(tmpdir)
        crust_bin = tmp / "crust_out"
        gcc_bin = tmp / "gcc_out"

        # Compile with our compiler
        crust_result = compile_with_crust(src, crust_bin)
        if crust_result.returncode != 0:
            return TestResult(
                path=src,
                passed=False,
                failure_reason=(
                    f"crust-compile failed to compile:\n{crust_result.stderr}"
                ),
            )

        # Compile with gcc
        gcc_result = compile_with_gcc(src, gcc_bin)
        if gcc_result.returncode != 0:
            return TestResult(
                path=src,
                passed=False,
                failure_reason=(
                    f"gcc failed to compile (test case may be invalid):\n{gcc_result.stderr}"
                ),
            )

        # Run both
        crust_exit = run_binary(crust_bin)
        gcc_exit = run_binary(gcc_bin)

        if crust_exit is None:
            return TestResult(path=src, passed=False, failure_reason="crust binary timed out")
        if gcc_exit is None:
            return TestResult(path=src, passed=False, failure_reason="gcc binary timed out")

        if crust_exit != gcc_exit:
            return TestResult(
                path=src,
                passed=False,
                failure_reason=(
                    f"exit code mismatch: crust={crust_exit}, gcc={gcc_exit}"
                ),
            )

        return TestResult(path=src, passed=True)


def main(argv: list[str]) -> int:
    if len(argv) > 1:
        cases = [Path(p) for p in argv[1:]]
    else:
        cases = sorted(CASES_DIR.glob("*.c"))

    if not cases:
        print("No test cases found.", file=sys.stderr)
        return 1

    results = []
    for case in cases:
        result = run_case(case)
        results.append(result)
        status = "PASS" if result.passed else "FAIL"
        print(f"  [{status}] {case.name}")
        if not result.passed:
            print(f"         {result.failure_reason}")

    passed = sum(1 for r in results if r.passed)
    total = len(results)
    print(f"\n{passed}/{total} passed")

    return 0 if passed == total else 1


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

### Writing Good Differential Test Cases

**Must be defined-behavior C.** If the program contains undefined behavior, different
compilers are allowed to produce different results — both are technically correct. UB in a
differential test case produces false positives.

```c
// WRONG: integer overflow is UB in C
int main(void) { return 2147483647 + 1; }

// RIGHT: use unsigned arithmetic for overflow
int main(void) {
    unsigned int x = 2147483647u;
    return (int)((x + 1u) == 2147483648u ? 0 : 1);
}
```

**Return value is the oracle.** The return value of `main` (0-127) is the test result. Do not
use printf and compare stdout unless you need to test string output specifically.

```c
// Good: minimal, uses return as oracle
int square(int x) { return x * x; }
int main(void) { return square(7) - 49; }  // returns 0 if correct

// Acceptable: uses a boolean check for clarity
int main(void) {
    int result = square(7);
    return (result == 49) ? 0 : 1;
}
```

**One feature per file.** If a test fails, you want to know which feature caused it.

**Minimal.** 10-20 lines maximum. If the test is longer, it likely tests multiple features.

### The Optimization Differential Trick

Compile the same program four ways:

1. Your compiler at `-O0`
2. gcc at `-O0`
3. gcc at `-O1`
4. gcc at `-O2` / `-O3`

Your `-O0` output should produce the same exit code as gcc's `-O0` output. If gcc's `-O1`
and `-O2` outputs disagree with gcc's `-O0`, the program likely contains UB (since optimizer
transforms are valid). Use this to filter out UB-containing test cases automatically.

```python
def is_ub_program(src: Path) -> bool:
    """Returns True if gcc disagrees with itself at different opt levels (likely UB)."""
    with tempfile.TemporaryDirectory() as tmpdir:
        tmp = Path(tmpdir)
        exits = {}
        for opt in ["-O0", "-O1", "-O2"]:
            out = tmp / f"gcc_{opt[1:]}"
            r = subprocess.run([GCC, opt, str(src), "-o", str(out)], capture_output=True)
            if r.returncode != 0:
                return False  # compile error, not a UB issue
            exits[opt] = run_binary(out)
        return len(set(exits.values())) > 1
```

---

## Part VII — Property-Based Testing with proptest

Property-based testing generates random inputs and verifies that a specified property holds.
For compilers, the most valuable properties are structural invariants that should hold for all
inputs, not just hand-written test cases.

### Setup

```toml
[dev-dependencies]
proptest = "1"
```

### The Three Key Properties

**Property 1: Parse-print roundtrip**

If your compiler has a pretty-printer, then `parse(print(ast))` should produce an AST
equivalent to the original. This catches both printer bugs and parser bugs.

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn parse_print_roundtrip(src in arb_valid_c_expr()) {
        let ast1 = parse(&src).expect("first parse failed");
        let printed = PrettyPrinter::new().print_expr(&ast1.body[0]);
        let ast2 = parse(&printed).expect("second parse failed");
        prop_assert_eq!(
            normalize_ast(&ast1),
            normalize_ast(&ast2),
            "roundtrip failed for src={src:?}\nprinted={printed:?}"
        );
    }
}
```

**Property 2: Lex-relex consistency**

Lexing the same input twice should produce the same tokens. This catches stateful bugs in
the lexer.

```rust
proptest! {
    #[test]
    fn lex_is_deterministic(src in "[ -~]{0,200}") {
        let (tokens1, errors1) = crust_lexer::lex(&src);
        let (tokens2, errors2) = crust_lexer::lex(&src);
        prop_assert_eq!(tokens1, tokens2, "lex produced different tokens on second run");
        prop_assert_eq!(errors1, errors2, "lex produced different errors on second run");
    }
}
```

**Property 3: No panic on arbitrary input**

The compiler must never panic on any syntactically valid UTF-8 input. This is the most
important property for robustness.

```rust
proptest! {
    #[test]
    fn no_panic_on_arbitrary_input(src in ".*") {
        // Must not panic. The result can be any error.
        let (tokens, _) = crust_lexer::lex(&src);
        let arena = bumpalo::Bump::new();
        let mut parser = crust_parser::Parser::new(&tokens, &arena);
        let _ = parser.parse_translation_unit();
    }
}
```

### Custom Arbitrary Implementation for AST Nodes

For more targeted property tests, generate structured AST nodes rather than random strings.

```rust
use proptest::prelude::*;
use proptest::strategy::BoxedStrategy;

fn arb_int_literal() -> impl Strategy<Value = Expr> {
    any::<i32>().prop_map(|n| Expr::IntLiteral(n))
}

fn arb_binop() -> impl Strategy<Value = BinOp> {
    prop_oneof![
        Just(BinOp::Add),
        Just(BinOp::Sub),
        Just(BinOp::Mul),
    ]
}

fn arb_expr() -> BoxedStrategy<Expr> {
    let leaf = prop_oneof![
        arb_int_literal(),
        Just(Expr::Var("x".to_string())),
        Just(Expr::Var("y".to_string())),
    ];
    leaf.prop_recursive(
        4,    // max depth
        64,   // max nodes
        8,    // items per collection
        |inner| {
            (arb_binop(), inner.clone(), inner.clone()).prop_map(|(op, lhs, rhs)| {
                Expr::BinOp {
                    op,
                    lhs: Box::new(lhs),
                    rhs: Box::new(rhs),
                }
            })
        },
    ).boxed()
}

proptest! {
    #[test]
    fn codegen_expr_does_not_panic(expr in arb_expr()) {
        // Filter out expressions that would cause division by zero
        prop_assume!(!contains_div_by_zero(&expr));
        let _ = crust_codegen::emit_expr(&expr);
    }
}
```

### Using `prop_assume!` for Invalid Inputs

`prop_assume!(condition)` causes proptest to discard the current case and generate a new one
if the condition is false. Use this to filter out obviously invalid programs rather than
writing a full validator in the strategy.

```rust
proptest! {
    #[test]
    fn type_checked_programs_compile(expr in arb_expr()) {
        // Only test programs that our simple type-checker says are well-typed.
        // This is not a full type-checker; just a quick sanity filter.
        prop_assume!(quick_type_check(&expr).is_ok());
        let result = crust_codegen::emit_expr(&expr);
        prop_assert!(
            result.is_ok(),
            "well-typed expr failed to compile: {expr:?}: {result:?}"
        );
    }
}
```

Do not use `prop_assume!` excessively. If more than 90% of generated inputs are being
discarded, the strategy is too broad. Narrow it instead.

---

## Part VIII — Fuzzing with cargo-fuzz

Fuzzing generates semi-random inputs to find crashes and panics. It is the most effective way
to find robustness bugs (unexpected panics, index-out-of-bounds, stack overflows) that no
human would think to test.

### Setup

```bash
cargo install cargo-fuzz
# In your compiler crate root:
cargo fuzz init
cargo fuzz add parse
cargo fuzz add sema
cargo fuzz add codegen
```

This creates `fuzz/Cargo.toml` and `fuzz/fuzz_targets/` in your crate.

```toml
# fuzz/Cargo.toml — add your compiler crates as dependencies
[dependencies]
crust_lexer = { path = ".." }
crust_parser = { path = "../crust_parser" }
crust_sema = { path = "../crust_sema" }
```

### The Three Essential Fuzz Targets

**Target 1: Parser fuzzer**

The parser must never panic on arbitrary byte input. This is the weakest but most important
invariant.

```rust
// fuzz/fuzz_targets/parse.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if let Ok(src) = std::str::from_utf8(data) {
        let (tokens, _lex_errors) = crust_lexer::lex(src);
        let arena = bumpalo::Bump::new();
        let mut parser = crust_parser::Parser::new(&tokens, &arena);
        // Must not panic. Errors are acceptable, panics are not.
        let _ = parser.parse_translation_unit();
    }
});
```

**Target 2: Sema fuzzer**

Run the full front-end pipeline. Requires the parser to already be panic-free (fix all
parser crashes before adding this target, or they will dominate the fuzz results).

```rust
// fuzz/fuzz_targets/sema.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if let Ok(src) = std::str::from_utf8(data) {
        let (tokens, _) = crust_lexer::lex(src);
        let arena = bumpalo::Bump::new();
        let mut parser = crust_parser::Parser::new(&tokens, &arena);
        if let Ok(ast) = parser.parse_translation_unit() {
            // Sema must not panic even on syntactically valid but semantically invalid ASTs.
            let _ = crust_sema::analyze(&ast);
        }
    }
});
```

**Target 3: Codegen fuzzer**

For the codegen fuzzer, it is more effective to generate structured AST inputs (using
`arbitrary`) rather than random bytes, because most random byte inputs will fail to parse
and the codegen will never be reached.

```rust
// fuzz/fuzz_targets/codegen.rs
#![no_main]
use arbitrary::Arbitrary;
use libfuzzer_sys::fuzz_target;

// Derive Arbitrary on your AST types for structured fuzzing.
// See: https://docs.rs/arbitrary

fuzz_target!(|ast: crust_ast::TranslationUnit| {
    // Only fuzz codegen with ASTs that pass type-checking,
    // to avoid false positives from obviously invalid programs.
    if crust_sema::analyze(&ast).is_ok() {
        let _ = crust_codegen::emit(&ast);
    }
});
```

To use `arbitrary::Arbitrary`, add to your AST crate:
```toml
[dependencies]
arbitrary = { version = "1", features = ["derive"], optional = true }

[features]
fuzzing = ["arbitrary"]
```

And derive `Arbitrary` on all AST node types:
```rust
#[cfg_attr(feature = "fuzzing", derive(arbitrary::Arbitrary))]
pub struct TranslationUnit {
    pub decls: Vec<Decl>,
}
```

### Running the Fuzzer

**In CI (30-second smoke test):**
```yaml
- name: Fuzz (30s smoke)
  run: |
    cargo fuzz run parse -- -max_total_time=30
    cargo fuzz run sema  -- -max_total_time=30
```

**Locally until a crash is found:**
```bash
# 4 parallel fuzzing processes, inputs up to 1024 bytes
cargo fuzz run parse -- -max_len=1024 -jobs=4

# Or run indefinitely (stop manually):
cargo fuzz run parse
```

**Reproducing a crash artifact:**
```bash
# Artifacts are saved in fuzz/artifacts/<target>/
cargo fuzz run parse fuzz/artifacts/parse/crash-abc1234567890abcdef

# Or run the minimized version (often easier to debug):
cargo fuzz tmin parse fuzz/artifacts/parse/crash-abc1234567890abcdef
```

**Viewing the corpus:**
```bash
ls fuzz/corpus/parse/   # Files that increased coverage
```

### Managing Fuzz Corpus in CI

Fuzz corpus files should be committed to the repository so that every CI run starts from a
meaningful baseline rather than from zero:

```bash
# After a fuzzing session, merge new corpus into the committed corpus
cargo fuzz cmin parse   # minimize the corpus
cp fuzz/corpus/parse/* tests/fuzz/corpus/parse/
git add tests/fuzz/corpus/parse/
git commit -m "test: update fuzz corpus for parse target"
```

---

## Part IX — CI Pipeline Design

### Full GitHub Actions Workflow

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

env:
  CARGO_TERM_COLOR: always
  RUST_BACKTRACE: 1

jobs:
  fmt:
    name: Format
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt
      - run: cargo fmt --all --check

  clippy:
    name: Clippy
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy
      - uses: Swatinem/rust-cache@v2
      - run: |
          cargo clippy --workspace --all-targets -- \
            -D warnings \
            -D clippy::unwrap_used \
            -D clippy::expect_used \
            -D clippy::panic

  build-and-test:
    name: Test (${{ matrix.os }} / ${{ matrix.toolchain }})
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest]
        toolchain: [stable, beta]
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: ${{ matrix.toolchain }}
      - uses: Swatinem/rust-cache@v2
      - name: Build
        run: cargo build --workspace
      - name: Unit + integration tests
        run: cargo test --workspace
      - name: Snapshot check
        run: cargo insta test --unreferenced=reject
        env:
          INSTA_UPDATE: no

  fuzz-smoke:
    name: Fuzz smoke (30s)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@nightly
      - uses: Swatinem/rust-cache@v2
      - run: cargo install cargo-fuzz
      - run: cargo fuzz run parse -- -max_total_time=30
      - run: cargo fuzz run sema  -- -max_total_time=30

  differential:
    name: Differential tests
    # Only run differential tests on push to main, not on every PR.
    # They are slow and require a working compiler; PRs often break this.
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - name: Install gcc
        run: sudo apt-get install -y gcc
      - name: Build compiler
        run: cargo build --release
      - name: Run differential tests
        run: |
          CRUST_COMPILE=target/release/crust-compile \
          python3 tests/differential/run.py
        timeout-minutes: 10
```

### OS Matrix Notes

- **ubuntu-latest** — primary CI target; most CI work happens here
- **macos-latest** — catches macOS-specific behavior (different libc, Apple clang)
- **windows-latest** — optional; add only if you need Windows support, as C toolchain setup
  on Windows in CI is complex (MSVC vs MinGW vs WSL)

Use `fail-fast: false` so that all matrix combinations run even if one fails. This gives you
a complete picture of failures rather than stopping at the first one.

### Rust Toolchain Matrix Notes

- **stable** — must always pass
- **beta** — catches regressions before they reach stable; warnings on beta often become
  errors on the next stable
- **nightly** — only include if you use nightly features; otherwise nightly failures are noise

### Test Timing Budget

Enforce these budgets to keep the feedback loop fast:

| Test category           | Budget   | Runs on             |
|-------------------------|----------|---------------------|
| Unit tests              | < 30s    | Every commit        |
| Integration / corpus    | < 2 min  | Every PR            |
| Snapshot check          | < 1 min  | Every PR            |
| Fuzz smoke (30s)        | < 5 min  | Every PR            |
| Differential suite      | < 10 min | Push to main only   |
| Full fuzz session       | Unlimited| Scheduled / manual  |

If unit tests start taking longer than 30 seconds, investigate. Common causes:
- A test is doing I/O (reading the corpus) when it should be using an in-memory fixture.
- A test is compiling a large file when it should be testing a small, isolated unit.
- Tests are running serially due to a mutex on shared state.

### Gate Rules (What Must Pass Before Merge)

These checks must all pass before a PR can merge:

```bash
# 1. Format
cargo fmt --all --check

# 2. Lint (zero warnings, no unwrap_used)
cargo clippy --workspace --all-targets -- -D warnings -D clippy::unwrap_used

# 3. All tests pass
cargo test --workspace

# 4. No stale or unreviewed snapshots
INSTA_UPDATE=no cargo insta test --unreferenced=reject

# 5. Fuzz smoke (catches obvious panics)
cargo fuzz run parse -- -max_total_time=30
```

The `clippy::unwrap_used` lint is particularly valuable for compiler code. A `.unwrap()` in
a compiler's parsing or analysis path is a potential crash on user input. Use `expect("...")
` with a message explaining why the invariant holds, or return a `Result`.

---

## Part X — Test Quality Metrics and Anti-Patterns

### Metrics Worth Tracking

**Line coverage** is a useful baseline metric but not a goal in itself. 80% line coverage
with bad tests is worse than 60% line coverage with good tests. Track it to find
uncovered code, not to hit a target.

Generate coverage with:
```bash
cargo install cargo-tarpaulin
cargo tarpaulin --workspace --out Html
```

Or with LLVM coverage:
```bash
RUSTFLAGS="-C instrument-coverage" cargo test
grcov . -s . --binary-path ./target/debug/ -t html --branch --ignore-not-existing -o ./coverage/
```

**Number of fuzzer-found crashes** should be zero after the initial hardening phase. Track
this in a log file and review any new crash before closing a release.

**Differential test pass rate** against gcc -O0 is the most meaningful correctness metric.
Track it over time. A regression here means a miscompilation was introduced.

**Error coverage** — what percentage of `SemanticError` variants have a test in `err/`?
This is easy to measure with a script:

```bash
# Count variants defined
grep -c "^    [A-Z][A-Za-z]* {" src/sema/error.rs

# Count variants with a test sidecar
ls tests/c11-corpus/err/*.expected_diag | wc -l
```

### Anti-Patterns

**1. Snapshot everything blindly**

Snapshots of wrong output become regression tests for wrong behavior after a `--bless`. Before
accepting a snapshot, read it and verify it is correct. The review step in `cargo insta review`
is not bureaucracy — it is the actual test.

**2. Only testing happy paths**

A test suite with 200 tests for valid programs and 3 tests for error cases will miss most
diagnostic bugs. Every `SemanticError` variant, every `ParseError` variant, and every
`LexError` variant must have at least one test in `err/`.

**3. Integration-only tests**

If all your tests go through the full pipeline (lex → parse → sema → codegen → link → run),
debugging is extremely slow. When a test fails, you have no idea which phase introduced the
bug. Every module should have unit tests that test it in isolation.

**4. Testing internal state instead of observable behavior**

Do not write tests like:
```rust
// BAD: testing internal field values
assert_eq!(parser.current_token_index, 5);
assert_eq!(sema_context.symbol_table.len(), 3);
```

Test observable behavior instead: return values, diagnostic output, generated IR, exit codes.
Internal state tests break constantly during refactoring and provide little confidence that
the compiler is correct.

**5. Uninformative assertion messages**

```rust
// BAD: failure message is useless
assert!(result.is_ok());

// GOOD: failure message includes the actual error
assert!(result.is_ok(), "expected success, got: {result:?}");

// BAD: no context on which corpus file failed
assert!(parser.parse_translation_unit().is_ok());

// GOOD: includes the file path in the failure
assert!(
    parser.parse_translation_unit().is_ok(),
    "{path:?}: parse failed"
);
```

**6. Mutable global state in tests**

`#[test]` functions run in parallel by default. Avoid writing output to a fixed path like
`/tmp/test_output.o` — two tests will collide. Use `tempfile::tempdir()` for all output files:

```rust
use tempfile::tempdir;

#[test]
fn compile_and_run() {
    let dir = tempdir().expect("failed to create tempdir");
    let out = dir.path().join("output");
    let result = compile_to_binary("int main(void) { return 0; }", &out);
    assert!(result.is_ok(), "{result:?}");
    // dir is automatically cleaned up when dropped
}
```

**7. Flaky differential tests from UB**

A differential test that sometimes passes and sometimes fails is almost always caused by
undefined behavior in the test case. When a test starts flaking, first check if the program
contains UB by comparing gcc -O0, -O1, -O2. If the results differ, the program is invalid
and must be removed or rewritten.

---

## Part XI — Quick-Reference Checklist

Use this checklist when adding a new compiler feature. Do not close the feature until every
item is checked.

```
New Feature Checklist
=====================

Corpus
  □ At least one ok/ corpus file exercising the feature
      - File name describes the feature: <feature_name>.c
      - Minimal (< 30 lines)
      - No undefined behavior
      - Returns 0 when compiled correctly

  □ At least one err/ corpus file for each new diagnostic code
      - One error per file
      - Has a .expected_diag sidecar
      - Tests the correct line and column span

Snapshot Tests
  □ Snapshot test for each new diagnostic message
      - Uses descriptive name, not snapshot_1
      - Snapshot has been reviewed (cargo insta review), not auto-blessed
      - .snap file committed to VCS

Unit Tests
  □ Unit test for each new public function in each affected crate
  □ Unit test for each new SemanticError variant
  □ Unit test for each new ParseError variant

Differential Tests
  □ Differential test case for any new code generation
      - Minimal (< 20 lines)
      - Defined behavior only
      - Uses return value as oracle

CI Checks (all must pass)
  □ cargo fmt --all --check
  □ cargo clippy --workspace -- -D warnings -D clippy::unwrap_used
  □ cargo test --workspace
  □ INSTA_UPDATE=no cargo insta test --unreferenced=reject
  □ cargo fuzz run parse -- -max_total_time=30 (no new crashes)
```

---

## Appendix A — Recommended Crate Versions (2025)

```toml
[dev-dependencies]
rstest        = "0.18"
insta         = { version = "1", features = ["yaml"] }
proptest      = "1"
tempfile      = "3"
bumpalo       = "3"

# Optional: for structured fuzzing
arbitrary     = { version = "1", features = ["derive"] }

# Optional: for LLVM FileCheck integration
llvm-filecheck = "0.9"
```

## Appendix B — Common Failure Modes and Diagnosis

| Symptom | Likely cause | How to diagnose |
|---|---|---|
| Differential test fails on one case | Miscompilation | Add IR dump; run with -v; add unit test for the pass |
| Fuzz target finds crash immediately | Panic in common code path | Read the crash artifact; `cargo fuzz tmin` it first |
| Snapshot test fails in CI but passes locally | Snapshot not committed | Run `git status`; look for `.snap.new` files |
| Corpus test passes but program is wrong | Exit code oracle not used | Verify test actually runs the binary |
| Clippy `unwrap_used` fires in new code | `.unwrap()` on fallible path | Replace with `?`, `expect("invariant: ...")`, or explicit error handling |
| Test suite takes > 5 minutes | I/O in unit tests | Find tests reading large files; use in-memory fixtures |

## Appendix C — Testing Checklist for Compiler Passes

When adding a new compiler pass (e.g., a constant folding optimization, a new lowering pass,
or a new analysis), use this more specific checklist:

```
Compiler Pass Checklist
=======================

  □ Unit test: pass is a no-op on the empty program
  □ Unit test: pass produces correct output for the minimal triggering case
  □ Unit test: pass is idempotent (running it twice gives the same result)
  □ Unit test: pass does not change semantics (add differential test)
  □ FileCheck test: IR output contains expected patterns
  □ FileCheck test: IR output does NOT contain forbidden patterns (CHECK-NOT)
  □ Fuzz: existing fuzz targets still pass after pass is added to the pipeline
  □ Snapshot: if pass produces user-visible output (warnings, notes), snapshot it
```

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-architect` — routes here after the first end-to-end milestone compiles and runs
- `compiler-expert` — routes here on any question about testing, CI, or test coverage

**Trace Down ↓** — this is a leaf skill (cross-cutting)
- (none — testing strategy applies to all other skills' outputs)

**Related →** — testing applies to every layer
- `compiler-lexer` — unit test token types and spans
- `compiler-parser` — snapshot-test AST shapes
- `compiler-codegen` — differential-test output vs. gcc/clang
- `compiler-optimizer` — FileCheck-test IR transformations
- `compiler-abi` — test struct layout and calling convention correctness
