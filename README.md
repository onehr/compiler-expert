# compiler-expert

A collection of 20 specialist skills for compiler and language implementation, organized in a layered hierarchy with bidirectional navigation.

Inspired by [actionbook/rust-skills](https://github.com/actionbook/rust-skills) — same format, same philosophy, but for **compiler construction** rather than Rust application development.

---

## Installation

### Claude Code (symlink)

```bash
git clone https://github.com/onehr/compiler-expert.git ~/.claude/skills/compiler-expert-repo
ln -s ~/.claude/skills/compiler-expert-repo/skills/* ~/.claude/skills/
```

### Claude Code (copy)

```bash
git clone https://github.com/onehr/compiler-expert.git /tmp/compiler-expert
cp -r /tmp/compiler-expert/skills/* ~/.claude/skills/
```

### NanoClaw / Container Agent

Copy into `container/skills/`:

```bash
cp -r skills/* /path/to/nanoclaw/container/skills/
```

---

## Collection Map

```
Source Text
     |
     v  Layer 2: Front-end
     compiler-lexer          tokenize source
     compiler-parser         build AST
     compiler-macro          expand macros
     compiler-name-resolution  resolve names/scopes
     compiler-type-system    check/infer types
     compiler-diagnostics    error messages (cross-cutting)
     |
     v  Layer 3: Middle-end
     compiler-hir            lower AST -> typed HIR
     compiler-mir            lower HIR -> control-flow IR + borrow check
     compiler-comptime       compile-time evaluation / generics
     compiler-elaboration    dependent type elaboration (Lean4/Agda style)
     compiler-optimizer      DCE, CSE, inlining, SSA, loop opts
     |
     v  Layer 4: Back-end
     compiler-codegen        instruction selection, register allocation
     compiler-abi            calling conventions, struct layout
     compiler-linker         ELF/COFF/Mach-O, relocations, LTO
     compiler-runtime        GC, memory, stack frames, panics
```

**Layer 1 -- Architecture** (before any phase):
- `compiler-architect` -- IR layer decision, build strategy, phase contracts
- `compiler-query-system` -- add incremental compilation to any architecture

**Layer 0 -- Meta**:
- `compiler-expert` -- router; entry point for all questions
- `using-compiler-expert` -- onboarding guide

**Cross-cutting**:
- `compiler-testing` -- test strategy, CI, FileCheck, insta, differential, fuzzing

---

## Skill Hierarchy

### Three-layer meta-cognition

Each skill follows the **WHY -> WHAT -> HOW** principle:

1. **WHY** -- what problem does this phase solve? What goes wrong if you skip it?
2. **WHAT** -- what are the key design decisions at this phase?
3. **HOW** -- concrete implementation guidance and code patterns

### Bidirectional navigation

Every skill has a `## Navigation` section at the bottom:

```
Trace Up    -- which skill(s) route here (who calls me)
Trace Down  -- which skill(s) I delegate to next (who I call)
Related     -- peer skills at the same layer
```

This makes the skill graph navigable without going through the router.

---

## Skill Index

| Skill | Layer | Description |
|---|---|---|
| `compiler-expert` | 0 | Router/entry point |
| `using-compiler-expert` | 0 | Onboarding |
| `compiler-architect` | 1 | IR design, build strategy, phase contracts |
| `compiler-query-system` | 1 | Incremental/demand-driven compilation |
| `compiler-lexer` | 2 | Scanner: NFA/DFA, token representation |
| `compiler-parser` | 2 | Parser: Pratt/LL/LR, AST design |
| `compiler-name-resolution` | 2 | Scopes, symbol tables, module paths |
| `compiler-type-system` | 2 | HM inference, trait solving, subtyping |
| `compiler-diagnostics` | 2 | Spans, caret errors, fix-it hints |
| `compiler-macro` | 2 | Hygiene, proc macros, preprocessor |
| `compiler-hir` | 3 | AST->HIR lowering, desugaring |
| `compiler-mir` | 3 | CFG IR, borrow checking, NLL |
| `compiler-comptime` | 3 | Compile-time execution, generics |
| `compiler-elaboration` | 3 | Dependent type elaboration |
| `compiler-optimizer` | 3 | DCE, CSE, inlining, SSA, pass manager |
| `compiler-codegen` | 4 | Instruction selection, register allocation |
| `compiler-abi` | 4 | SysV/Windows/WASM calling conventions |
| `compiler-linker` | 4 | ELF/COFF/Mach-O, relocations, LTO |
| `compiler-runtime` | 4 | GC, memory management, stack frames |
| `compiler-testing` | x | Testing strategy, CI, fuzzing |

---

## Usage

**Starting a new compiler project:**
> Invoke `compiler-architect` -- it asks what you're building, makes the IR layer decision, gives you a build plan, and routes to the first specialist.

**Working on a specific phase:**
> Invoke the skill for that phase directly (e.g., `compiler-parser` for parsing questions).

**Unsure which skill applies:**
> Invoke `compiler-expert` -- it maps your question to the right specialist.

**New to the collection:**
> Read `using-compiler-expert` for an overview and quickstart.

---

## Reference Sources

Each skill's `references/` directory contains distilled notes from primary sources:

- **Engineering a Compiler** (Cooper & Torczon) -- lexer, parser, optimizer, codegen chapters
- **Crafting Interpreters** (Nystrom) -- bytecode VM, GC, closures, classes
- **rustc-dev-guide** -- HIR, MIR, query system, diagnostics, type inference
- **Lean 4 / Agda docs** -- elaboration, universe polymorphism, implicit arguments
- **System V AMD64 ABI** -- calling conventions, struct layout, register passing

---

## Demo: crust

[crust](https://github.com/onehr/crust) is a C11 compiler written in Rust, built incrementally using this skill collection. Target: compile [SQLite3](https://sqlite.org/amalgamation.html) (~150K lines of C) correctly.

This serves as the real-world evaluation for the skill collection -- if the skills are good enough, they should guide an AI agent from zero to a working C11 compiler.

---

## Related

- [actionbook/rust-skills](https://github.com/actionbook/rust-skills) -- same format for Rust application development
