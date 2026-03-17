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

# compiler-testing

You are an expert in compiler testing strategy and CI design. This skill covers the full testing stack for compiler projects written in Rust.

Full code examples, CI YAML, Python harnesses, and proptest strategies are in `references/testing-reference.md`. This file contains decision guides, key patterns, and the quick-reference checklist.

---

## The Oracle Problem — Decision Guide

Testing a compiler requires an oracle: how do you know if the output is right? Four strategies, ranked by cost:

| Oracle | Description | When to use |
|---|---|---|
| **Exit code** | Both compilers compile the same `.c`; compare exit codes (0-127) | Best default; zero infrastructure |
| **Reference compiler** | Compare output against gcc/clang | After ~30% of valid C programs compile |
| **Specification** | Manually write expected output from the spec | High-confidence edge cases only; expensive to scale |
| **Crash oracle** | Must never panic on any input, valid or invalid | Fuzzing; weakest but free |

**The exit-code trick** is the most practical: `int main(void) { return (result == expected) ? 0 : 1; }`. The process exit code is a zero-cost oracle for any C program.

---

## The Four Test Goals

Every complete compiler test strategy must address all four:

1. **Correctness** — no miscompilation. Code compiles silently but runs wrong is the worst failure. → *Differential testing*
2. **Robustness** — no crash on any input (malformed, malicious, invalid). → *Fuzzing*
3. **Performance** — no compilation speed or code quality regression. → *Benchmarks + differential optimization tests*
4. **Diagnostic quality** — right errors, right spans, right messages. → *Snapshot testing + `err/` corpus*

---

## The Test Pyramid

```
          /\
         /  \
        / diff\        Differential (slow, powerful) — push to main only
       /  tests \
      /----------\
     / integration\    Corpus tests (ok/ + err/) — every PR, < 2 min
    /--------------\
   /  unit tests    \  #[test] + rstest — every commit, < 30s
  /------------------\
```

Unit tests tell you **WHERE** it broke. Differential tests tell you **THAT** it broke. You need both.

---

## Test Suite Directory Layout

```
tests/
├── c11-corpus/
│   ├── ok/                     # Programs that must compile and run
│   ├── err/                    # Programs that must produce specific diagnostics
│   │   ├── undeclared_var.c
│   │   └── undeclared_var.expected_diag  # sidecar: error code + line + col
│   └── snapshots/              # insta .snap files (committed to VCS)
├── differential/
│   ├── cases/                  # C programs for gcc comparison
│   └── run.py                  # Harness script
├── filecheck/                  # IR-level CHECK patterns
└── fuzz/
    └── fuzz_targets/
        ├── parse.rs
        ├── sema.rs
        └── codegen.rs
```

### Corpus File Rules

**`ok/` files:**
- One feature per file; name describes the feature (`pointer_deref.c` not `test1.c`)
- 5–30 lines; minimal; no undefined behavior
- Returns 0 when compiled correctly (exit code is the oracle)

**`err/` files:**
- One error variant per file; paired `.expected_diag` sidecar
- Format: `error: E_CODE`, `line: N`, `col_start: N`, `col_end: N`
- Every `SemanticError`/`ParseError`/`LexError` variant must have at least one file here

---

## Unit Tests — Key Patterns

Use plain `#[test]` for specific edge cases; use `rstest` for parameterized / corpus-driven tests.

**Corpus-driven with rstest** (auto-discovers new `.c` files):
```rust
#[rstest]
fn parses_cleanly(#[files("tests/c11-corpus/ok/*.c")] path: std::path::PathBuf) {
    let src = std::fs::read_to_string(&path).unwrap();
    let (tokens, lex_errs) = crust_lexer::lex(&src);
    assert!(lex_errs.is_empty(), "{path:?}: lex errors: {lex_errs:?}");
    let arena = bumpalo::Bump::new();
    let mut parser = crust_parser::Parser::new(&tokens, &arena);
    assert!(parser.parse_translation_unit().is_ok(), "{path:?}: parse failed");
}
```

**Anti-pattern:** Testing only happy paths. Every error enum variant must have at least one test. Link the test in a doc-comment on the variant:
```rust
pub enum SemanticError {
    /// Test: `tests/c11-corpus/err/undeclared_var.c`
    UndeclaredName { name: String, span: Span },
}
```

See `references/testing-reference.md §Part III` for full lexer, parser, and sema test patterns.

---

## Snapshot Testing with `insta`

**Good snapshot targets:** rendered diagnostic output, AST pretty-prints, IR dumps, error message text.
**Bad snapshot targets:** `{:?}` of frequently-changing structs, timestamps, file paths, memory addresses.

Workflow:
```bash
cargo insta test      # generates .snap.new files, tests fail
cargo insta review    # diff each new snapshot: [a]ccept / [r]eject / [s]kip
git add tests/.../*.snap && git commit
```

In CI: `INSTA_UPDATE=no cargo insta test --unreferenced=reject`
— fails on unreviewed snapshots AND on stale snapshots (no longer referenced by any test).

**Key rule:** The review step is the actual test. Never auto-bless without reading the output.

See `references/testing-reference.md §Part IV` for full examples including diagnostic and AST snapshots.

---

## FileCheck-Style IR Verification

For Cranelift/custom IR, embed assertions in tests using a minimal pure-Rust checker or the `llvm-filecheck` crate:

```rust
fn check_ir(ir: &str, patterns: &[(&str, bool)]) { /* CHECK / CHECK-NOT in order */ }

#[test]
fn codegen_return_constant_uses_iconst() {
    let ir = compile_to_ir("int f(void) { return 42; }");
    check_ir(&ir, &[
        ("iconst", true),   // CHECK: iconst
        ("call",   false),  // CHECK-NOT: call
    ]);
}
```

Use CHECK-NOT to assert that no heap allocation, no extra calls, no pessimized code was emitted.

See `references/testing-reference.md §Part V` for the full `check_ir` implementation and LLVM FileCheck directive reference.

---

## Differential Testing

Compare your compiler vs gcc by exit code. Only feasible once ~30% of valid C programs compile.

**When to run:**
- After any optimizer change
- After any ABI or calling-convention change
- For every new language construct before closing the feature

**UB filter trick:** If `gcc -O0`, `gcc -O1`, and `gcc -O2` disagree with each other, the program contains UB and is not a valid oracle. Filter automatically:
```python
def is_ub_program(src: Path) -> bool:
    exits = {opt: run_gcc(src, opt) for opt in ["-O0", "-O1", "-O2"]}
    return len(set(exits.values())) > 1
```

**Test case rules:** Must be defined-behavior C; return value is the oracle (0-127); one feature per file; ≤20 lines.

See `references/testing-reference.md §Part VI` for the full Python harness.

---

## Property-Based Testing with `proptest`

Three properties that should hold for all inputs:

1. **Parse-print roundtrip** — `parse(print(ast))` ≡ original ast (catches printer + parser bugs)
2. **Lex determinism** — lexing the same input twice gives identical tokens
3. **No panic on arbitrary input** — the most important robustness property:

```rust
proptest! {
    #[test]
    fn no_panic_on_arbitrary_input(src in ".*") {
        let (tokens, _) = crust_lexer::lex(&src);
        let arena = bumpalo::Bump::new();
        let mut parser = crust_parser::Parser::new(&tokens, &arena);
        let _ = parser.parse_translation_unit();
    }
}
```

Use `prop_assume!` to discard invalid inputs (e.g., UB programs). If >90% of inputs are discarded, narrow the strategy instead.

See `references/testing-reference.md §Part VII` for custom `arb_expr()` strategies and structured generation.

---

## Fuzzing with `cargo-fuzz`

Three essential targets in order of priority:

1. **Parser fuzzer** — must never panic on arbitrary bytes; fix all parser crashes before adding more targets
2. **Sema fuzzer** — runs full front-end; requires parser to already be panic-free
3. **Codegen fuzzer** — use `arbitrary::Arbitrary` on AST types for structured fuzzing (random bytes rarely reach codegen)

```bash
cargo fuzz run parse -- -max_total_time=30   # CI smoke test
cargo fuzz run parse -- -max_len=1024 -jobs=4  # Local session
cargo fuzz tmin parse fuzz/artifacts/parse/crash-XXXX  # Minimize crash
```

Commit corpus files to the repo (`fuzz/corpus/`) so CI starts from a meaningful baseline, not zero.

See `references/testing-reference.md §Part VIII` for full fuzz target code and `arbitrary::Arbitrary` setup.

---

## CI Pipeline Design

### Test Timing Budget

| Category | Budget | Runs on |
|---|---|---|
| Unit tests | < 30s | Every commit |
| Integration / corpus | < 2 min | Every PR |
| Snapshot check | < 1 min | Every PR |
| Fuzz smoke (30s) | < 5 min | Every PR |
| Differential suite | < 10 min | Push to main only |
| Full fuzz session | Unlimited | Scheduled / manual |

### Gate Rules (must pass before merge)

```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings -D clippy::unwrap_used -D clippy::panic
cargo test --workspace
INSTA_UPDATE=no cargo insta test --unreferenced=reject
cargo fuzz run parse -- -max_total_time=30
```

`clippy::unwrap_used` in compiler code is critical: a `.unwrap()` in a parsing or analysis path is a potential crash on user input.

See `references/testing-reference.md §Part IX` for the full GitHub Actions YAML (ubuntu + macOS, stable + beta, fuzz smoke job, differential job on main only).

---

## Anti-Patterns

1. **Snapshot blindly**: blessing wrong output locks in wrong behavior. The review step IS the test.
2. **Happy-path only**: every error variant must be tested.
3. **Integration-only**: if all tests run the full pipeline, you can't debug which phase broke.
4. **Testing internal state**: test observable behavior (return values, IR, exit codes), not `parser.current_token_index`.
5. **Uninformative assertions**: always include `{result:?}` and `{path:?}` in failure messages.
6. **Global mutable state**: use `tempfile::tempdir()` for output; `#[test]` runs in parallel.
7. **Flaky differential tests**: UB in test cases causes non-determinism. Filter with the optimization disagreement trick.

---

## Quick-Reference Checklist (New Feature)

```
Corpus
  □ ok/ file: feature_name.c, < 30 lines, no UB, returns 0
  □ err/ file per new diagnostic code + .expected_diag sidecar

Snapshot Tests
  □ Snapshot per new diagnostic message (descriptive name, reviewed)
  □ .snap file committed to VCS

Unit Tests
  □ Test per new public function
  □ Test per new SemanticError / ParseError variant

Differential Tests
  □ Minimal case (< 20 lines), defined behavior, return-value oracle

CI Checks
  □ cargo fmt --all --check
  □ cargo clippy -- -D warnings -D clippy::unwrap_used
  □ cargo test --workspace
  □ INSTA_UPDATE=no cargo insta test --unreferenced=reject
  □ cargo fuzz run parse -- -max_total_time=30
```

---

## Common Failure Modes

| Symptom | Likely cause |
|---|---|
| Differential test fails on one case | Miscompilation — add IR dump, write unit test for the pass |
| Fuzz finds crash immediately | Panic in common code path — `cargo fuzz tmin` the artifact first |
| Snapshot fails in CI but passes locally | `.snap.new` file not committed |
| Corpus test passes but program is wrong | Exit code oracle not actually used |
| Test suite > 5 minutes | I/O in unit tests — use in-memory fixtures |

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-architect` — routes here after first end-to-end milestone
- `compiler-expert` — routes here on testing, CI, or coverage questions

**Trace Down ↓** — leaf skill (cross-cutting)
- (none — testing strategy applies to all other skills' outputs)

**Related →**
- `compiler-lexer` — unit test token types and spans
- `compiler-parser` — snapshot-test AST shapes
- `compiler-codegen` — differential-test output vs gcc/clang
- `compiler-optimizer` — FileCheck-test IR transformations
