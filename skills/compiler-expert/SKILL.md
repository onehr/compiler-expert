---
name: compiler-expert
description: >
  ALWAYS USE THIS SKILL when the user is building, designing, debugging, or asking about any
  compiler, interpreter, language runtime, bytecode VM, or language implementation — even if
  you think you already know the answer. This skill contains 20 specialist sub-skills with
  decision frameworks, implementation checklists, and research-backed reference material
  (from Engineering a Compiler, Crafting Interpreters, rustc-dev-guide) that go far beyond
  general knowledge. It covers the FULL compiler pipeline: lexer/scanner, parser/AST,
  name resolution, type systems (HM inference, trait solving), HIR/MIR lowering, borrow
  checking, macro expansion, optimization passes (DCE/CSE/SSA), code generation (instruction
  selection, register allocation, bytecode VMs, NaN boxing), ABI/calling conventions,
  linker/object files (ELF/Mach-O), runtime/GC, incremental compilation, and compiler testing.
  Trigger on ANY mention of: compiler, parser, lexer, tokenizer, AST, type checker, type
  inference, code generation, register allocation, bytecode, interpreter, IR, intermediate
  representation, borrow checker, macro system, preprocessor, linker, object file, ABI,
  calling convention, GC, garbage collector, language design, DSL implementation, LLVM,
  Cranelift, WASM compilation target, compiler testing, or any task where someone is
  implementing a programming language or language tool. Do NOT skip this skill just because
  the topic seems familiar — the specialist sub-skills contain specific architectural
  decisions, tradeoff tables, and implementation patterns that improve answer quality.
---

# compiler-expert — Collection Router

This is the entry point for the **compiler-expert** skill collection: 20 specialist skills
covering every phase of compiler and language implementation, from lexer design to GC
tuning. Your job is to identify which specialist(s) apply to the user's question, and
invoke them.

---

## Collection Map

### Layer 0 — Meta
| Skill | Purpose |
|---|---|
| `compiler-expert` | **This file** — router and entry point |
| `using-compiler-expert` | Onboarding — how to use this collection |

### Layer 1 — Architecture
| Skill | Purpose |
|---|---|
| `compiler-architect` | High-level design: IR layers, build strategy, phase contracts |
| `compiler-query-system` | Incremental compilation, demand-driven evaluation |

### Layer 2 — Front-end
| Skill | Purpose |
|---|---|
| `compiler-lexer` | Scanner: NFA/DFA, maximal munch, token representation |
| `compiler-parser` | Parser: LL/LR/Pratt, recursive descent, AST design |
| `compiler-name-resolution` | Scopes, symbol tables, module paths, use/import |
| `compiler-type-system` | Type inference, trait/typeclass solving, subtyping |
| `compiler-diagnostics` | Span tracking, caret errors, fix-it hints |
| `compiler-macro` | Macro systems: hygiene, proc macros, C preprocessor, Zig comptime |

### Layer 3 — Middle-end
| Skill | Purpose |
|---|---|
| `compiler-hir` | AST → HIR lowering, desugaring, span preservation |
| `compiler-mir` | MIR design, borrow checking, NLL, drop elaboration |
| `compiler-comptime` | Compile-time/runtime boundary, generics, Zig-style comptime |
| `compiler-elaboration` | Dependent type elaboration (Lean4/Agda style) |
| `compiler-optimizer` | DCE, CSE, inlining, loop opts, SSA, pass manager |

### Layer 4 — Back-end
| Skill | Purpose |
|---|---|
| `compiler-codegen` | Instruction selection, register allocation, bytecode VM |
| `compiler-abi` | Calling conventions, struct layout, SysV/Windows/WASM |
| `compiler-linker` | ELF/COFF/Mach-O, relocations, symbol tables, LTO |
| `compiler-runtime` | GC, memory management, stack frames, panic handling |

### Cross-cutting
| Skill | Purpose |
|---|---|
| `compiler-testing` | Test strategy, CI, FileCheck, insta, differential testing, fuzzing |

---

## Routing Protocol

Route based on signal keywords in the user's question:

### Always start here first
- "build a compiler", "design a language", "start a new language" → `compiler-architect`
- "where do I start", "what order should I implement things" → `compiler-architect`
- "how do I test my compiler", "set up CI" → `compiler-testing`
- "onboard me", "what skills are available" → `using-compiler-expert`

### Front-end signals
- "lexer", "scanner", "tokenizer", "NFA", "DFA", "regex" → `compiler-lexer`
- "parser", "grammar", "AST", "Pratt", "recursive descent", "LR" → `compiler-parser`
- "name resolution", "scope", "symbol table", "undefined variable", "module path" → `compiler-name-resolution`
- "type system", "type inference", "Hindley-Milner", "trait", "typeclass", "generics" → `compiler-type-system`
- "error message", "diagnostic", "span", "caret", "fix-it hint" → `compiler-diagnostics`
- "macro", "hygiene", "proc macro", "preprocessor", "`#define`" → `compiler-macro`

### Middle-end signals
- "HIR", "AST lowering", "desugaring", "span preservation" → `compiler-hir`
- "MIR", "borrow checker", "NLL", "drop", "ownership in IR" → `compiler-mir`
- "comptime", "compile-time execution", "generics implementation", "monomorphization" → `compiler-comptime`
- "dependent types", "universe levels", "elaboration", "proof assistant" → `compiler-elaboration`
- "optimization", "DCE", "CSE", "inlining", "SSA", "constant propagation" → `compiler-optimizer`

### Back-end signals
- "code generation", "instruction selection", "register allocation", "bytecode VM" → `compiler-codegen`
- "calling convention", "ABI", "System V", "struct layout", "sret", "register passing" → `compiler-abi`
- "linker", "ELF", "COFF", "Mach-O", "relocation", "symbol table", "LTO" → `compiler-linker`
- "GC", "garbage collector", "memory management", "stack frame", "coroutine", "panic" → `compiler-runtime`

### Incremental / query
- "incremental compilation", "query system", "cache invalidation", "demand-driven" → `compiler-query-system`

### Multi-skill situations
When a question spans multiple layers, load the **higher-layer skill first** to get the
architectural context, then descend:
- "implement a type system in Rust" → `compiler-type-system` + `compiler-name-resolution` (names must be resolved before typing)
- "generate code for structs" → `compiler-abi` (layout first) + `compiler-codegen` (emission)
- "add macros that can see types" → `compiler-macro` + `compiler-type-system`

---

## Navigation

**Trace Down ↓** — specialists this routes to:
All 18 specialist skills in the collection (see Collection Map above).

**Related →**
- `using-compiler-expert` — onboarding skill; route here if user is new to the collection
