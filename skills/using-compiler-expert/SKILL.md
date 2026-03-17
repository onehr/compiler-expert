---
name: using-compiler-expert
description: >
  Onboarding guide for the compiler-expert skill collection. Use when the user asks "what
  compiler skills do you have?", "how do I use the compiler expert?", "what's in this
  collection?", or wants an overview of available compiler implementation expertise.
  Also use at the start of a new compiler project to orient the user toward the right
  skills for their phase.
---

# Using compiler-expert

Welcome to the **compiler-expert** collection — 20 specialist skills covering the full
stack of compiler and language implementation, from source text to native binary.

---

## What's in this collection

The collection is organized in four implementation layers plus cross-cutting concerns:

```
Source Text
    │
    ▼ [Layer 2: Front-end]
    compiler-lexer        ← tokenize source
    compiler-parser       ← build AST
    compiler-macro        ← expand macros
    compiler-name-resolution ← resolve names
    compiler-type-system  ← check types
    compiler-diagnostics  ← emit errors (cross-cutting)
    │
    ▼ [Layer 3: Middle-end]
    compiler-hir          ← lower AST → typed HIR
    compiler-mir          ← lower HIR → control-flow IR
    compiler-comptime     ← evaluate at compile time
    compiler-elaboration  ← for dependently-typed languages
    compiler-optimizer    ← optimize IR
    │
    ▼ [Layer 4: Back-end]
    compiler-codegen      ← select instructions, allocate registers
    compiler-abi          ← apply calling conventions
    compiler-linker       ← produce binary
    compiler-runtime      ← GC, memory, panics
```

Overarching:
- **`compiler-architect`** (Layer 1) — make the big design decisions before diving in
- **`compiler-query-system`** (Layer 1) — add incremental compilation
- **`compiler-testing`** (cross-cutting) — test strategy and CI for any layer

---

## How to get started

**Starting a new compiler project?**
→ Begin with `compiler-architect`. It will ask what you're building, make the IR layer
decision, and give you an ordered build plan. It will tell you which skill to invoke next.

**Continuing an existing project?**
→ Jump directly to the skill for your current phase. If unsure which phase you're in,
ask `compiler-architect` for a diagnosis.

**Just want to learn?**
→ Read the skills in layer order: architect → lexer → parser → name-resolution →
type-system → hir → codegen → abi → linker. Each skill explains its phase, the key
decisions, and how it connects to the next.

---

## Skill invocation pattern

Each skill in this collection follows a consistent structure:

1. **Decision points first** — what choices you need to make at this phase
2. **Implementation guidance** — how to do it correctly
3. **Navigation section** — which skills sit above/below/beside this one

When a skill says "Trace Down ↓ → compiler-foo", it means: once you've finished
this phase, the next specialist to invoke is `compiler-foo`.

---

## Navigation

**Trace Down ↓**
- `compiler-expert` — the router; invoke for any compiler/language question
- `compiler-architect` — start here for new projects

**Related →**
- `compiler-expert` — routes any question to the right specialist
