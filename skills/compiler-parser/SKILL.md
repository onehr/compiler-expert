---
name: compiler-parser
description: Parser construction expert — covers LL/LR/Pratt parsing, recursive descent implementation, operator precedence, AST design, error recovery, and the Visitor pattern. Use when designing or implementing a parser, grammar, or AST for any language.
---

# compiler-parser

You are an expert in parser construction. When this skill is active, read `references/parser-reference.md` for detailed decision frameworks before advising on any parser design or implementation question. That file contains full coverage of LL vs. LR, recursive descent, operator precedence, Pratt parsing, AST node design, error recovery strategies, and the Visitor pattern — distilled from Engineering a Compiler (Cooper & Torczon, Ch. 3) and Crafting Interpreters (Nystrom, Ch. 5, 6, and 8).

---

## Quick Decision Guide

Use this to orient quickly; consult `references/parser-reference.md` for full detail.

| Goal | Recommendation |
|---|---|
| Production compiler, readable code | Hand-coded recursive descent (LL) — used by clang, rustc, go |
| Generator-based, any grammar | LALR(1) generator (yacc/bison, ANTLR) — handles more grammars than LL(1) |
| Expression-heavy language, elegant precedence | Pratt parser (top-down operator precedence) |
| Learning / small language | Hand-coded recursive descent; straightforward to implement and debug |
| Maximum generality (ambiguous grammars OK) | Earley parser or GLR — O(n^3) worst case, rarely needed |

**Core principle (EaC, p. 135):** "Almost all programming-language constructs can be written in LR(1) or LL(1) form." Starting with a parser generator is always better than writing one from scratch.

---

## Key Decisions

### 1. Grammar Class Selection

The grammar hierarchy from most to least general:

```
Arbitrary CFGs  (O(n^3), Earley)
  └─ LR(1)      (bottom-up, linear, 1 lookahead)
       └─ LL(1)  (top-down, linear, 1 lookahead)
            └─ Regular grammars
```

Most languages are LR(1) or LL(1). Pick top-down (LL) for hand-coding; bottom-up (LR) when using a generator.

Read section 1 and 2 of `references/parser-reference.md` for the full grammar hierarchy and comparison.

### 2. Top-Down (LL) vs. Bottom-Up (LR)

- **LL (recursive descent):** One function per non-terminal; reads left-to-right and builds the parse tree top-down; natural for hand-coding; struggles with left recursion (must eliminate it)
- **LR (shift-reduce):** Reads left-to-right, reduces rightmost derivation; handles left recursion naturally; table-driven; harder to write by hand but powerful with a generator
- **Practical rule:** Hand-code recursive descent. Use a generator (bison, ANTLR) when you need LR power or have a complex grammar.

Read section 2 and 3 of `references/parser-reference.md` for full comparison and LL(1) implementation guidance.

### 3. LL(1) Conflicts and Grammar Transformation

- **Left recursion** must be eliminated for LL parsers — use the standard left-recursion removal transformation
- **Common prefix / left factoring** — factor out shared prefixes to produce deterministic choice
- **FIRST and FOLLOW sets** — used to build the predictive parsing table; a conflict (two productions in the same cell) means the grammar is not LL(1)

Read sections 4 and 5 of `references/parser-reference.md` for FIRST/FOLLOW computation and conflict resolution.

### 4. Pratt Parsing for Expressions

Pratt parsing (top-down operator precedence) handles expression grammars with arbitrary precedence levels and associativity without left-recursion elimination or complex grammar rewrites. Each token type carries a `left_binding_power` (infix) and a `null_denotation` function (prefix). Highly recommended for expression-heavy languages.

Read section 12 of `references/parser-reference.md` for a full Pratt parser walkthrough.

### 5. AST Node Design

Two main approaches:
- **Tagged union / sum type:** One `NodeKind` enum + a union of payload structs; cache-friendly; common in C implementations
- **Class hierarchy with virtual dispatch:** One class per node kind; natural in OOP languages; enables the Visitor pattern cleanly

**Visitor pattern:** Separates operations (type checking, code generation, pretty printing) from node structure. Define a `Visitor` interface with one `visit_*` method per node type; each node implements `accept(visitor)`. Avoids adding methods to every node class for every new operation.

Read section 10 of `references/parser-reference.md` for AST design patterns and Visitor implementation.

### 6. Error Recovery

Parsers must recover from syntax errors and continue parsing to report multiple errors in one pass. Main strategies:
- **Panic mode:** Discard tokens until a synchronization token (`;`, `}`, keyword) is found; simple and effective
- **Error productions:** Add explicit error productions to the grammar; catches common mistakes with better messages
- **Phrase-level recovery:** Insert or delete tokens locally to repair the parse; fragile, hard to maintain

Recursive descent enables clean panic-mode recovery with explicit `synchronize()` methods.

Read section 9 of `references/parser-reference.md` for full error recovery strategies.

---

## Common Mistakes

1. **Not eliminating left recursion before writing a recursive descent parser** — causes infinite recursion; transform the grammar first
2. **Building a concrete syntax tree instead of an AST** — include only semantically meaningful nodes; omit parentheses, punctuation, and other structural tokens
3. **Mixing parsing and semantic analysis** — keep the parser focused on syntax; defer type checking, name resolution, and constant folding to separate passes
4. **Aborting on the first parse error** — implement synchronization so the parser recovers and reports multiple errors
5. **Encoding operator precedence with ambiguous grammar rules** — use explicit precedence declarations or Pratt binding powers instead of relying on the parser to resolve ambiguity at runtime
6. **Choosing a generator before understanding the grammar class** — verify the grammar is LL(1) or LALR(1) before selecting a tool; conflicts are painful to debug after the fact

See section 13 of `references/parser-reference.md` for the full common mistakes and warnings list.

---

## References

- `references/parser-reference.md` — Full decision frameworks, code examples, tradeoff tables, and implementation patterns
  - Section 1: Grammar hierarchy and parser selection overview
  - Section 2: Top-down (LL) vs. bottom-up (LR) parsers
  - Section 3: Recursive descent vs. table-driven LL(1)
  - Section 4: FIRST and FOLLOW sets — computation and use
  - Section 5: LL(1) grammar conflicts and resolution
  - Section 6: LR(1) vs. LALR(1) vs. SLR(1)
  - Section 7: Operator precedence and associativity in grammars
  - Section 8: Grammar ambiguity — detection and resolution
  - Section 9: Error recovery strategies
  - Section 10: AST node design patterns
  - Section 11: Implementation tradeoffs — table-driven vs. direct-coded vs. hand-coded
  - Section 12: Pratt parsing (top-down operator precedence)
  - Section 13: Common mistakes and warnings
  - Section 14: Grammar transformation checklist
  - Section 15: LL(1) parsing condition — formal statement
  - Section 16: LR(1) parsing — key concepts summary

**Primary sources:**
- Engineering a Compiler, Cooper & Torczon (3rd ed., 2023), Chapter 3
- Crafting Interpreters, Robert Nystrom, Chapters 5, 6, and 8

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-architect` — routes here as the second implementation phase
- `compiler-expert` — routes here on parser/grammar/AST questions

**Trace Down ↓** — next skills after this phase
- `compiler-name-resolution` — name resolution walks the AST produced by the parser
- `compiler-hir` — if using a HIR layer, the AST feeds directly into HIR lowering
- `compiler-macro` — syntactic macros (macro_rules!, proc macros) expand during or after parsing

**Related →** — peers at the front-end layer
- `compiler-lexer` — provides the token stream this parser consumes
- `compiler-diagnostics` — error recovery during parsing needs good span+message support
