---
name: compiler-lexer
description: Lexer/scanner construction expert — covers NFA/DFA, hand-coded vs generator tradeoffs, maximal munch, input buffering, token representation, keyword handling strategies. Use when designing or implementing a lexer, scanner, or tokenizer for any language.
---

# compiler-lexer

You are an expert in lexer and scanner construction. When this skill is active, read `references/lexer-reference.md` for detailed decision frameworks before advising on any lexer design or implementation question. That file contains full coverage of NFA/DFA theory, table-driven vs. hand-coded tradeoffs, maximal munch, input buffering, token representation, keyword handling, and error recovery — distilled from Engineering a Compiler (Cooper & Torczon, Ch. 2) and Crafting Interpreters (Nystrom, Ch. 4 and Ch. 16).

---

## Quick Decision Guide

Use this to orient quickly; consult `references/lexer-reference.md` for full detail.

| Goal | Recommendation |
|---|---|
| Production compiler, complex language | DFA-based, table-driven or direct-coded; consider a generator (flex, logos) |
| Hobby / learning project | Hand-coded recursive scanner — simple, debuggable |
| Maximum performance | Direct-coded DFA with sentinel-based two-buffer input scheme |
| Simple scripting language | Hand-coded with switch/if chains; maximal munch by default |
| Need rich error messages | Hand-coded — generators make error recovery harder to customize |

**Core principle:** The scanner touches every character exactly once. Speed matters more here than in any other compiler pass. Both generated and hand-coded scanners can achieve O(1) per character.

---

## Key Decisions

### 1. NFA vs. DFA

- **NFA** — natural output of Thompson's construction from regular expressions; non-deterministic, used as an intermediate representation
- **DFA** — deterministic; each character drives a single state transition; used at runtime
- **Subset construction** converts NFA → DFA; DFA states are sets of NFA states
- **Key tradeoff:** DFA state explosion is rare in practice for lexer-scale grammars; minimization via Hopcroft's algorithm reduces table size

Read section 2 of `references/lexer-reference.md` for full NFA/DFA detail.

### 2. Table-Driven vs. Direct-Coded vs. Hand-Coded

- **Table-driven:** DFA encoded as a 2D array `next_state[state][char_class]`; easy to generate, cache-unfriendly at scale
- **Direct-coded:** DFA states become code labels / switch cases; faster than table lookup, harder to generate
- **Hand-coded:** Human writes the recognizer directly; most readable, easiest to extend, standard in production compilers (clang, rustc, go)

Read section 3 of `references/lexer-reference.md` for implementation patterns and code examples.

### 3. Maximal Munch

Always prefer the longest matching token. The scanner should not stop at the first valid token boundary when a longer match is possible. This rule resolves most ambiguities (e.g., `>=` vs. `>` followed by `=`).

Read section 4 of `references/lexer-reference.md` for edge cases and exception handling.

### 4. Input Buffering

- **Single buffer:** Simple but requires boundary checks on every character
- **Two-buffer sentinel scheme (EaC):** Eliminates per-character bounds checks; use a sentinel character at the buffer midpoint and end; critical for high-performance scanners
- **String slice / read-all (CI):** Load the entire source into memory; pointer arithmetic into a flat string; practical for most modern compilers where source fits in RAM

Read section 5 of `references/lexer-reference.md` for buffer management details.

### 5. Token Representation

- At minimum: `{type, lexeme_start, lexeme_length, line}`
- Avoid storing the lexeme string in every token when the source is in memory — store a span instead
- Keyword detection: prefer a hash table over encoding keywords directly in the DFA (separates concerns, easier to maintain)

Read section 6 and 7 of `references/lexer-reference.md` for token struct design and keyword handling strategies.

### 6. Error Handling

- On unrecognized character: report the error, emit an error token or skip, and continue scanning — do not abort
- Collect all lexical errors before stopping; users expect all errors at once

Read section 8 of `references/lexer-reference.md` for error recovery patterns.

---

## Common Mistakes

1. **Returning on first match instead of longest match** — violates maximal munch; produces wrong tokens for multi-character operators
2. **Encoding keywords as DFA states** — creates a bloated, unmaintainable DFA; use a symbol table / hash lookup post-scan instead
3. **Storing full lexeme strings in every token** — wastes memory when source is in memory; store (start, length) spans
4. **Aborting on first error** — scanners should recover and report multiple errors in one pass
5. **Conflating scanner and parser responsibilities** — the scanner should not make grammar-level decisions; keep it pure character-to-token
6. **Ignoring whitespace handling** — newlines need explicit handling for languages with significant indentation or automatic semicolon insertion

See section 13 of `references/lexer-reference.md` for the full common mistakes list.

---

## References

- `references/lexer-reference.md` — Full decision frameworks, code examples, tradeoff tables, and implementation patterns
  - Section 1: Core concepts and the three-phase pipeline (RE → NFA → DFA)
  - Section 2: NFA vs. DFA — when each is used
  - Section 3: Table-driven vs. direct-coded vs. hand-coded scanners
  - Section 4: Maximal munch rule and edge cases
  - Section 5: Input buffering strategies
  - Section 6: Token representation choices
  - Section 7: Keyword handling strategies
  - Section 8: Error handling in scanners
  - Section 9: Transition table compression
  - Section 10: Whitespace and newline handling
  - Section 11: Literal handling in hand-coded scanners (CI patterns)
  - Section 12: Practical tradeoffs summary
  - Section 13: Common mistakes and pitfalls
  - Section 14: Quick reference — key algorithms
  - Section 15: CI implementation patterns quick reference

**Primary sources:**
- Engineering a Compiler, Cooper & Torczon (3rd ed., 2023), Chapter 2
- Crafting Interpreters, Robert Nystrom, Chapters 4 and 16

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-architect` — routes here as the first implementation phase
- `compiler-expert` — routes here on lexer/scanner/tokenizer questions

**Trace Down ↓** — next skill after this phase
- `compiler-parser` — consumes the token stream produced by the lexer

**Related →** — peers at the front-end layer
- `compiler-diagnostics` — error reporting uses lexer spans for source locations
- `compiler-macro` — token-level macros (C preprocessor) interact with the lexer
