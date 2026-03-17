# Parser Reference: Decision Frameworks & Tradeoffs
## Sources: Engineering a Compiler, Cooper & Torczon (Ch. 3) [EaC]; Crafting Interpreters, Robert Nystrom (Ch. 5, 6, 8) [CI]

---

## 1. Overview: Grammar Hierarchy and Parser Selection

Context-free grammars (CFGs) nest in a hierarchy of increasing restrictiveness and parsability: [EaC]

```
Arbitrary CFGs (O(n^3) e.g. Earley's algorithm)
  └─ LR(1) grammars (bottom-up, linear time, 1 lookahead)
       └─ LL(1) grammars (top-down, linear time, 1 lookahead)
            └─ Regular grammars (same power as regular expressions)
```

**Key insight (p. 135):** "Almost all programming-language constructs can be written in LR(1) or LL(1) form. Thus, most compilers use a parser based on one of these two classes of CFGs. Some constructs fall in the gap between LR(1) and LL(1); they do not appear to be particularly useful." [EaC]

**Practical rule (p. 154):** "Starting with an available parser generator is always better than implementing a parser generator from scratch." [EaC]

**CI perspective:** CFGs are what allow grammars to express arbitrary nesting (unlike regular grammars). A key intuition: a formal grammar plays a "game" in two directions — forward (derivation/generation) and backward (parsing). A recursive nonterminal in the grammar is what separates context-free from regular. [CI]

**Terminology alignment (CI, Ch. 5):** [CI]

| Terminology          | Lexical grammar | Syntactic grammar |
|----------------------|-----------------|-------------------|
| The "alphabet" is... | Characters      | Tokens            |
| A "string" is...     | Lexeme or token | Expression        |
| Implemented by...    | Scanner         | Parser            |

---

## 2. Top-Down (LL) vs. Bottom-Up (LR) Parsers

### Core Conceptual Difference

- **Top-down parsers** begin with the grammar's start symbol (root) and expand toward leaves. They build a leftmost derivation. At each step, the parser predicts which production to apply using the next lookahead symbol. [EaC]
- **Bottom-up parsers** begin with the input words (leaves) and reduce toward the start symbol (root). They build a rightmost derivation in reverse. At each step, the parser finds a "handle" — the right-hand side of a production to reduce. [EaC]

CI frames this as direction of tree traversal: recursive descent is a "top-down parser" because it starts from the outermost grammar rule (`expression`) and works its way into nested subexpressions before reaching leaves. LR parsers start with primary expressions and compose them upward. Note that "high" and "low" precedence use the opposite metaphor — in a top-down parser you reach lowest-precedence expressions first, because they may contain higher-precedence subexpressions. [CI]

### When to Use LL(1) / Top-Down

Use LL(1) or recursive descent when: [EaC, CI]
- You are **hand-coding** the parser (recursive descent is the natural choice)
- The grammar is **small and amenable** to manual inspection
- You need **high-quality, customized error messages** — recursive-descent parsers can pinpoint errors precisely because the parser knows exactly what it expected
- The source language grammar is already LL(1) or can easily be made LL(1) after left recursion elimination and left factoring
- You want **simplicity and maintainability** over handling the broadest grammar class

CI adds: GCC, V8 (Chrome's JavaScript engine), and Roslyn (C# compiler) all use hand-written recursive descent. "Recursive descent parsers are fast, robust, and can support sophisticated error handling." Don't underestimate it due to its simplicity. [CI]

**Limitation:** LL(1) cannot handle left recursion directly; it requires grammar transformation. Also, some languages have no LL(1) equivalent, requiring LR(1). [EaC]

### When to Use LR(1) / Bottom-Up

Use LR(1) when: [EaC]
- You are using a **parser generator** (LR(1) generators are widely available)
- The grammar **cannot be expressed** as LL(1) (falls in the LL(1)/LR(1) gap)
- You want the **largest class of unambiguous CFGs** accepted automatically
- You need **strong error localization** — LR(1) parsers detect errors at the earliest possible point, before reading any symbol beyond those needed to prove the input erroneous (p. 127)

**Limitation:** LR(1) table construction requires scrupulous bookkeeping; it is a task that "should be automated and relegated to a computer" (p. 141). Hand-building LR(1) tables is "tedious and error-prone." [EaC]

### Summary Comparison Table

| Property                     | LL(1) / Recursive Descent        | LR(1) / Table-Driven Bottom-Up   |
|------------------------------|----------------------------------|----------------------------------|
| Grammar class                | Smaller (strict subset of LR(1)) | Larger                           |
| Derivation built             | Leftmost                         | Rightmost (in reverse)           |
| Parsing direction            | Root → leaves                    | Leaves → root                    |
| Handles left recursion       | No (must eliminate)              | Yes                              |
| Hand-coding ease             | Easy (recursive descent)         | Very hard                        |
| Tool generation              | Straightforward                  | Standard (yacc, bison, ANTLR)    |
| Error messages               | Excellent (natural in RD)        | Good (detects errors early)      |
| Table size                   | Small                            | Larger                           |
| Speed                        | Fast (RD can beat table-driven)  | Fast; direct encoding can win    |
| Ambiguity handling           | Manual grammar rewrite           | Conflicts reported by generator  |

**Quote (p. 4154):** "A compiler writer who wants to construct a hand-coded parser, for whatever reason, is well advised to use the top-down, recursive-descent method." [EaC]

---

## 3. Recursive Descent vs. Table-Driven LL(1)

Both are top-down, predictive parsers, but implemented differently.

### Recursive Descent

- Structured as a set of **mutually recursive procedures**, one per nonterminal. [EaC]
- The **grammar guides the code structure** directly. [EaC]
- No parse table — logic is embedded in control flow and conditionals. [EaC]
- Advantages (p. 1664–1667): [EaC]
  - "Can produce accurate, informative error messages" — the natural location for error generation is when the parser fails to find an expected terminal
  - Compact, efficient, readable
  - A "well-constructed top-down, recursive-descent parser can be faster than a table-driven LR(1) parser" (p. 4149)
  - Easier to **finesse context-sensitive ambiguities** (e.g., distinguishing array subscripts from function calls by checking a symbol table)
- Disadvantage: Must be hand-coded from grammar, tedious for large grammars. [EaC]

CI provides the direct translation mapping from grammar notation to code: [CI]

| Grammar notation | Code representation               |
|------------------|-----------------------------------|
| Terminal         | Code to match and consume a token |
| Nonterminal      | Call to that rule's function      |
| `\|`             | `if` or `switch` statement        |
| `*` or `+`       | `while` or `for` loop             |
| `?`              | `if` statement                    |

"The descent is described as 'recursive' because when a grammar rule refers to itself — directly or indirectly — that translates to a recursive function call." [CI]

### CI's Concrete Recursive Descent Implementation (jlox, Java)

The core parser class structure: [CI]

```java
class Parser {
    private final List<Token> tokens;
    private int current = 0;

    Parser(List<Token> tokens) {
        this.tokens = tokens;
    }
}
```

Primitive token operations that all parsing methods rely on: [CI]

```java
private boolean match(TokenType... types) {
    for (TokenType type : types) {
        if (check(type)) { advance(); return true; }
    }
    return false;
}

private boolean check(TokenType type) {
    if (isAtEnd()) return false;
    return peek().type == type;
}

private Token advance() {
    if (!isAtEnd()) current++;
    return previous();
}

private boolean isAtEnd()  { return peek().type == EOF; }
private Token peek()       { return tokens.get(current); }
private Token previous()   { return tokens.get(current - 1); }
```

The key pattern for parsing a left-associative binary operator (e.g., equality): [CI]

```java
private Expr equality() {
    Expr expr = comparison();                   // parse left operand

    while (match(BANG_EQUAL, EQUAL_EQUAL)) {   // loop: grammar's ( ... )*
        Token operator = previous();
        Expr right = comparison();             // parse right operand
        expr = new Expr.Binary(expr, operator, right); // build left-assoc tree
    }

    return expr;
}
```

"As we zip through a sequence of equality expressions, [storing back into `expr`] creates a left-associative nested tree of binary operator nodes." If no equality operator is found, the method simply delegates upward to the next higher precedence level — this is how the stratified grammar encodes precedence. [CI]

Mandatory token consumption (used to require closing delimiters): [CI]

```java
private Token consume(TokenType type, String message) {
    if (check(type)) return advance();
    throw error(peek(), message);
}
```

### Table-Driven LL(1)

- Uses a **parse table** (nonterminal × terminal → production number) and a generic skeleton parser loop. [EaC]
- Generated automatically from a grammar by a parser generator. [EaC]
- Advantages: [EaC]
  - Can be generated automatically from START sets
  - Separates grammar specification from parsing engine
  - A parser generator can emit either a table-driven parser or a recursive-descent parser from the same grammar
- Disadvantage: Table lookup adds overhead; less intuitive to debug; harder to add custom error messages. [EaC]

**Key insight (p. 1748):** "Such a system [LL(1) generator emitting recursive-descent code] could combine the speed and locality of a recursive-descent parser with the convenience of a grammar-based generator." [EaC]

**Decision rule:** For hand-written parsers, use recursive descent. For generated parsers, use table-driven LL(1) or LR(1) depending on grammar class needed. [EaC]

---

## 4. FIRST and FOLLOW Sets: How to Compute, When They Matter

### FIRST Sets

**Definition (p. 1162):** For a grammar symbol α, `FIRST(α)` is the set of terminals that can appear at the left end of a sentential form derived from α. [EaC]

- For terminal t: `FIRST(t) = {t}`
- For nonterminal A: computed by iterating over productions

**Algorithm (Fig. 3.7):** [EaC]
1. Initialize `FIRST` for all terminals, ε, and eof to themselves.
2. For each production `A → β1 β2 ... βk`:
   - Start with `FIRST(β1) - {ε}`
   - If ε ∈ FIRST(βi), include `FIRST(βi+1) - {ε}`
   - Continue until hitting a symbol whose FIRST doesn't include ε
   - If all symbols can derive ε, add ε to `FIRST(A)`
3. Repeat until no changes (fixed-point computation)

**Warning (p. 2503):** "In our experience, this use of FIRST(δa) [in the LR(1) closure computation] is the point in the LR(1) table construction where a human is most likely to make a mistake." [EaC]

### FOLLOW Sets

**Definition (p. 1289):** For a nonterminal A, `FOLLOW(A)` contains all words that can occur immediately after A in a sentence. Domain: NT. Range: T ∪ {eof}. [EaC]

**Algorithm (Fig. 3.8):** [EaC]
1. Initialize all FOLLOW sets to ∅
2. Set `FOLLOW(Start) = {eof}`
3. For each production `A → β1 β2 ... βk`, iterate right-to-left:
   - Maintain a TRAILER variable initialized to `FOLLOW(A)`
   - For each nonterminal βi: `FOLLOW(βi) = FOLLOW(βi) ∪ TRAILER`
   - Update TRAILER: if ε ∈ FIRST(βi), `TRAILER = TRAILER ∪ (FIRST(βi) - {ε})`; else `TRAILER = FIRST(βi)`
4. Repeat until fixed point.

### START Sets: Combining FIRST and FOLLOW

**Definition (p. 1387):** For production `A → β`: [EaC]
```
START(A → β) = FIRST(β)                          if ε ∉ FIRST(β)
             = (FIRST(β) - {ε}) ∪ FOLLOW(A)      if ε ∈ FIRST(β)
```

**When they matter:** [EaC]
- **FIRST sets** determine which production to expand in LL(1) when lookahead hits a non-empty right-hand side.
- **FOLLOW sets** are needed specifically for ε-productions — they tell the parser when it is correct to derive the empty string (lookahead ∈ FOLLOW(A) → use ε production; otherwise, syntax error).
- **START sets** are what the LL(1) parser actually uses: `START(A → βi) ∩ START(A → βj) = ∅` for all i ≠ j is the LL(1) condition.

---

## 5. LL(1) Grammar Conflicts and How to Resolve Them

A grammar fails the LL(1) condition when two or more productions for the same nonterminal have overlapping START sets. Two main causes:

### Problem 1: Left Recursion

**Definition:** A grammar is left recursive if `A →+ Aβ` for some A and β.

**Why it breaks top-down parsing (p. 855):** The parser applies `A → Aβ` repeatedly, never generating a leading terminal, looping infinitely. [EaC]

CI frames the same issue in terms of implementation: a left-recursive rule causes a function to immediately call itself, which calls itself again, until a stack overflow kills the process. "This is why left recursion is problematic for recursive descent." [CI]

**Fix: Eliminate left recursion** [EaC]

Direct left recursion transformation:
```
Before:             After:
Fee → Fee α         Fee  → β Fee'
    | β             Fee' → α Fee'
                        | ε
```

For the classic expression grammar:
```
Before (left-recursive):    After (right-recursive):
Expr → Expr + Term          Expr  → Term Expr'
     | Expr - Term          Expr' → + Term Expr'
     | Term                       | - Term Expr'
                                  | ε
```

**CI's alternative (for implementation):** Rather than the two-nonterminal transformation, CI uses an iterative `while`-loop pattern that avoids left recursion while still producing a left-associative tree: [CI]

```
factor → unary ( ( "/" | "*" ) unary )* ;
```

This flat EBNF rule `X → Y (op Y)*` directly translates to the while-loop pattern shown in Section 3 above. It "matches the same syntax as the previous [left-recursive] rule, but better mirrors the code we'll write to parse." Both approaches produce a left-associative AST.

**Important caveat (p. 932):** "Note that we did not simply rewrite `Expr → Expr + Term` as `Expr → Term + Expr`. That change would effectively change the associativity of addition." The right-recursive transformation preserves left-associativity in the resulting parse tree. [EaC]

**Indirect left recursion:** Use the systematic algorithm (Fig. 3.6): [EaC]
1. Order nonterminals A0, A1, ..., An
2. For each Ai, substitute away any Aj (j < i) from the RHS using forward substitution
3. Eliminate any resulting direct left recursion
4. New nonterminals added are always right-recursive; ignore them in the outer loop.

### Problem 2: Common Prefixes (Solved by Left Factoring)

When multiple productions for A begin with the same terminal(s), their START sets overlap.

**Example:** [EaC]
```
Factor → name           # all three start with "name" → overlapping START sets
       | name [ ArgList ]
       | name ( ArgList )
```

**Fix: Left Factoring (p. 1457)** [EaC]
Extract the common prefix into a new nonterminal:
```
Factor    → name Arguments
Arguments → [ ArgList ]
          | ( ArgList )
          | ε
```

**Transformation pattern:** [EaC]
```
A → αβ1 | αβ2 | ... | αβn | γ1 | γ2 | ... | γj
becomes:
A → αB | γ1 | γ2 | ... | γj
B → β1 | β2 | ... | βn
```

**Warning (p. 1484):** "Left-factoring can often eliminate the need to backtrack. However, some context-free languages have no backtrack-free grammar. They must be parsed with a more general technique, such as the LR(1) parsers." [EaC]

**Important limit (p. 1491):** "In general, however, it is undecidable whether or not a backtrack-free grammar exists for an arbitrary context-free language." When in doubt, try the transformation; if the grammar still doesn't satisfy LL(1), use LR(1). [EaC]

---

## 6. LR(1) vs. LALR(1) vs. SLR(1)

### Three LR Construction Algorithms (p. 4101–4131) [EaC]

All three produce shift-reduce parsers using Action/Goto tables. They differ in how they compute lookahead information and thus in table size and grammar class accepted.

| Property               | SLR(1)               | LALR(1)              | LR(1) (Canonical)    |
|------------------------|----------------------|----------------------|----------------------|
| Grammar class          | Smallest             | Middle               | Largest              |
| Table size             | Smallest             | Same as SLR(1)       | Largest              |
| Lookahead source       | FOLLOW sets          | Merged item sets     | Full LR(1) items     |
| Practical use          | Simple, limited      | Most common (yacc)   | Full generality      |

**SLR(1):** Uses global FOLLOW sets instead of per-item lookahead symbols. This eliminates the need for lookahead in items, producing fewer states. Accepts many grammars of practical interest but rejects some valid LR(1) grammars. [EaC]

**LALR(1) (p. 4116):** Relies on the observation that "core items in the set representing a state are critical and that the remaining items can be added to a set by computing its closure." Merges sets that share the same core items and computes closure afterward. Produces table sizes similar to SLR(1). This is what yacc/bison generate. [EaC]

**Canonical LR(1):** Most general. Tracks distinct lookahead symbols per item, distinguishing states that LALR(1) would merge. Produces the largest tables but accepts the widest class of grammars. [EaC]

**Critical insight (p. 4129–4132):** "Any language that has an LR(1) grammar also has both an SLR(1) grammar and an LALR(1) grammar. The grammars for these more restrictive construction algorithms must be shaped in ways that let the algorithms differentiate between shift actions and reduce actions." [EaC]

**When you need canonical LR(1) over LALR(1):** [EaC]
- When LALR(1) produces conflicts (shift-reduce or reduce-reduce) that the canonical construction would resolve — this indicates the grammar has states that LALR merges incorrectly
- In practice, most real programming languages fit in LALR(1). If the parser generator reports conflicts, rewrite the grammar rather than upgrading to canonical LR(1).

**Practical recommendation (p. 4162):** "In choosing between LR(1) and LL(1) grammars, the choice often depends on the availability of tools. In practice, few, if any, programming-language constructs fall in the gap between LR(1) grammars and LL(1) grammars." [EaC]

---

## 7. Operator Precedence and Associativity in Grammars

### Encoding Precedence via Grammar Levels (p. 556–563) [EaC]

The key insight: encode precedence levels as distinct nonterminals in a hierarchy. Each level handles one tier of operators, and higher-precedence operators are deeper in the grammar (closer to the leaves).

**Classic expression grammar (Fig. 3.2):** [EaC]
```
Expr   → Expr + Term | Expr - Term | Term      (lowest: +, -)
Term   → Term × Factor | Term ÷ Factor | Factor (medium: ×, ÷)
Factor → ( Expr ) | num | name                 (highest: parens)
```

A postorder tree-walk of this grammar produces the correct evaluation order (×/÷ before +/-).

**Rule:** To add more precedence levels, add more grammar levels (nonterminals). Higher-precedence operators live deeper in the grammar. [EaC]

**CI's concrete Lox expression grammar** uses the same stratification principle with EBNF notation: [CI]

```
expression → equality ;
equality   → comparison ( ( "!=" | "==" ) comparison )* ;
comparison → term ( ( ">" | ">=" | "<" | "<=" ) term )* ;
term       → factor ( ( "-" | "+" ) factor )* ;
factor     → unary ( ( "/" | "*" ) unary )* ;
unary      → ( "!" | "-" ) unary | primary ;
primary    → NUMBER | STRING | "true" | "false" | "nil"
           | "(" expression ")" ;
```

CI's Lox precedence table (low to high), matching C's precedence rules: [CI]

| Name       | Operators        | Associativity |
|------------|------------------|---------------|
| Equality   | `== !=`          | Left          |
| Comparison | `> >= < <=`      | Left          |
| Term       | `- +`            | Left          |
| Factor     | `/ *`            | Left          |
| Unary      | `! -`            | Right         |

**Examples of typical precedence order (low to high) in C-family languages:** [EaC]
- Assignment (←) — lowest
- Addition/subtraction (+, -)
- Multiplication/division (×, ÷)
- Unary operators (-, !, type casts)
- Array subscripts, function calls
- Parentheses — highest

**CI design note — Logic versus History:** When designing a new language's precedence table, there is a tension between internal logical consistency (e.g., placing bitwise operators above equality since `flags & MASK == VALUE` almost always means `(flags & MASK) == VALUE`) and familiarity from existing languages (C got this wrong; users are used to the mistake). No perfect answer exists — use language context to choose. [CI]

### Handling Associativity

**Left-associativity (p. 890):** Achieved by left recursion in the grammar: [EaC]
```
Expr → Expr + Term    (left-recursive → left-associative)
```
After transforming to right recursion for LL(1), the associativity is preserved by the specific form of the transformation:
```
Expr  → Term Expr'
Expr' → + Term Expr' | ε
```
Do NOT naively rewrite `Expr → Expr + Term` as `Expr → Term + Expr` — this changes addition to right-associative. [EaC]

CI notes a practical caveat on floating-point: "In principle, it doesn't matter whether you treat multiplication as left- or right-associative — you get the same result either way. Alas, in the real world with limited precision, roundoff and overflow mean that associativity can affect the result." [CI]

**Right-associativity:** Use right recursion directly (e.g., exponentiation, assignment operators). [EaC]

CI assignment grammar (right-associative, achieved by recursive call rather than loop): [CI]
```
expression → assignment ;
assignment → IDENTIFIER "=" assignment | equality ;
```
And the corresponding parser code:
```java
private Expr assignment() {
    Expr expr = equality();         // parse left side as if expression
    if (match(EQUAL)) {
        Token equals = previous();
        Expr value = assignment();  // recurse for right-associativity
        if (expr instanceof Expr.Variable) {
            Token name = ((Expr.Variable)expr).name;
            return new Expr.Assign(name, value);
        }
        error(equals, "Invalid assignment target.");
    }
    return expr;
}
```
The "trick" here: parse the left-hand side as a normal expression, then if `=` follows, reinterpret the already-parsed expression as an l-value. "This means we can parse the left-hand side as if it were an expression and then after the fact produce a syntax tree that turns it into an assignment target." [CI]

### Handling Unary Operators (Section 3.5.2) [EaC]

Insert a new grammar level (nonterminal) between the appropriate binary operator levels:
```
Term  → Term × Value | Term ÷ Value | Value
Value → |Factor| | Factor             (absolute value, higher than × ÷)
Factor → ( Expr ) | num | name
```

For right-binding unary operators that can repeat (e.g., C's `**p`):
```
Value → * Value    (right-recursive → right-associative, allows repetition)
```

For unary minus where repetition should be prevented, use a non-recursive rule:
```
Value → - Factor | Factor
```

CI handles this with a right-recursive rule that bottoms out at `primary`: [CI]
```
unary → ( "!" | "-" ) unary | primary ;
```
```java
private Expr unary() {
    if (match(BANG, MINUS)) {
        Token operator = previous();
        Expr right = unary();
        return new Expr.Unary(operator, right);
    }
    return primary();
}
```

---

## 8. Grammar Ambiguity: Detection and Resolution

### What Ambiguity Means

A grammar G is ambiguous if some sentence in L(G) has **more than one rightmost (or leftmost) derivation**, equivalently, more than one parse tree. Ambiguous grammars are problematic because the compiler cannot know which meaning to assign to a program (p. 418). [EaC]

CI demonstrates the problem concretely: the simple flat grammar `binary → expression operator expression` allows `6 / 3 - 1` to be parsed as either `(6 / 3) - 1` or `6 / (3 - 1)`, yielding different numeric results. "The way mathematicians have addressed this ambiguity since blackboards were first invented is by defining rules for precedence and associativity." The fix is stratifying the grammar into precedence levels (see Section 7). [CI]

### Detection

LR(1) table construction **automatically detects** ambiguity as conflicts: [EaC]

1. **Shift-reduce conflict (p. 3194):** Occurs when the table-filling algorithm tries to define one Action[i, a] entry as both a shift and a reduce. "Arises because rule 2 in the grammar is a prefix of rule 3." Manifests the classic if-then-else ambiguity.

2. **Reduce-reduce conflict (p. 3214):** Occurs when two different productions `A → γδ` and `B → γδ` have the same right-hand side, and the same state CCi contains both `[A → γδ •, a]` and `[B → γδ •, a]`. This generates two conflicting reduce actions for Action[i, a].

**Important:** "Manually determining that a grammar has the LR(1) property is tedious and error-prone... the method of choice for determining if a given grammar has the LR(1) property is to invoke an LR(1) parser generator on it. If the process succeeds, the grammar has the LR(1) property" (p. 3225). [EaC]

### Resolution Strategies

**Strategy 1: Rewrite the grammar (the clean solution)** [EaC]

For the classic if-then-else ambiguity (p. 452–475):
```
# Ambiguous:
Stmt → if Expr then Stmt
     | if Expr then Stmt else Stmt
     | Other

# Unambiguous (binds else to innermost if):
Stmt     → if Expr then Stmt
         | if Expr then WithElse else Stmt
         | Other
WithElse → if Expr then WithElse else WithElse
         | Other
```
The rewrite restricts what can appear in the `then` clause of an if-then-else.

**Strategy 2: Use shift over reduce (pragmatic for if-then-else)** [EaC]

"As an alternative, an LR(1) parser generator can simply resolve shift-reduce conflicts in favor of the shift action. This choice forces the parser to recognize the longer production. In the case of the if-then-else grammar, that decision binds each else to the innermost unmatched ifthen—precisely the rule that the disambiguated grammar from Section 3.2.3 enforces" (p. 3209).

**Strategy 3: Disambiguate with semantic information (context-sensitive ambiguity)** [EaC]

For the FORTRAN/PL/I/ADA problem where `fee(i,j)` could be an array reference or function call (p. 3403–3478):

- Option A: **Combine both into one production**, defer resolution to semantic analysis where type information is available.
- Option B: **Scanner-parser cooperation** — have the scanner classify identifiers using the symbol table (requires define-before-use):
  ```
  Subscript → variable-name ( ArgList )
  Call      → function-name ( ArgList )
  ```
  "Rewritten in this way, the grammar is unambiguous. The scanner returns a distinct syntactic category in each case."

**Prevention:** "Of course, the language designer can avoid this kind of ambiguity. Languages such as C and BCPL use different syntax for function calls and array element references, which eliminates the ambiguity" (p. 3485). [EaC]

---

## 9. Error Recovery Strategies

**Goal:** A parser should find as many syntax errors as possible in each compilation, not halt at the first one. [EaC, CI]

CI articulates the requirements more fully. A parser must: [CI]
1. **Detect and report the error** — a malformed syntax tree passed to the interpreter can cause arbitrary bad behavior downstream.
2. **Avoid crashing or hanging** — users actively use the parser to discover what syntax is valid; it must be robust to invalid input.

And ideally should: [CI]
- **Be fast** — modern IDEs reparse after every keystroke; millisecond latency is expected.
- **Report as many distinct errors as possible** — stopping at the first is annoying; users want to see all errors at once.
- **Minimize cascaded errors** — after the first real error, the parser may generate phantom errors that disappear when the original is fixed. The tension: report more errors vs. avoid ghost errors.

### Strategy 1: Panic Mode (Synchronization)

Most common approach. When an error is found: [EaC]
1. Discard input symbols until a **synchronizing token** is found (e.g., semicolons in Algol-like languages)
2. Reset parser state to one consistent with having recognized a complete syntactic unit

**In recursive-descent parsers (p. 3290):** Simply discard words in a loop until the synchronizing token appears, then return control to the statement-parsing routine as if it succeeded. May use `setjmp/longjmp` or exception handling. [EaC]

**In LR(1) parsers (p. 3294):** [EaC]
1. Discard input until the synchronizing token (e.g., semicolon)
2. Scan backward down the parse stack to find state s where `Goto[s, Stmt]` is a valid (non-error) entry
3. Discard stack entries down to state s
4. Push `Goto[s, Stmt]` onto the stack and resume parsing

**CI's concrete implementation of panic mode in jlox (Java):** [CI]

CI uses Java exceptions to implement the call-stack unwinding needed in recursive descent. This elegantly leverages the language runtime to unwind all the in-progress parse frames back to the synchronization point — no explicit stack tracking needed.

Step 1: A sentinel exception type:
```java
private static class ParseError extends RuntimeException {}
```

Step 2: On encountering a token that cannot start a valid construct, report and throw:
```java
private Token consume(TokenType type, String message) {
    if (check(type)) return advance();
    throw error(peek(), message);
}

private ParseError error(Token token, String message) {
    Lox.error(token, message);   // display to user with line info
    return new ParseError();     // return rather than throw, so
}                                // caller can decide whether to unwind
```

The `error()` method returns the exception rather than throwing it, giving each call site the choice: some errors (e.g., too many arguments) don't warrant panic mode and the parser simply reports and continues. Others (e.g., missing closing paren) do, and the caller throws.

Step 3: The `synchronize()` method discards tokens until a statement boundary: [CI]
```java
private void synchronize() {
    advance();
    while (!isAtEnd()) {
        if (previous().type == SEMICOLON) return;  // just ended a statement
        switch (peek().type) {
            case CLASS: case FUN: case VAR: case FOR:
            case IF: case WHILE: case PRINT: case RETURN:
                return;  // about to start a new statement
        }
        advance();
    }
}
```

Step 4: The declaration loop at the top of the parse catches `ParseError` and calls `synchronize()`: [CI]
```java
private Stmt declaration() {
    try {
        if (match(VAR)) return varDeclaration();
        return statement();
    } catch (ParseError error) {
        synchronize();
        return null;
    }
}
```

"The traditional place in the grammar to synchronize is between statements." Synchronization on statement boundaries is reliable because semicolons and statement-starting keywords (`if`, `for`, `return`, etc.) are strong boundaries. CI notes the approximation: a semicolon inside a `for` loop's clauses could confuse the heuristic, "but that's OK. We've already reported the first error precisely, so everything after that is kind of 'best effort'." [CI]

CI also distinguishes when NOT to enter panic mode: for localized violations like "too many function arguments," the parser reports the error and continues without throwing, because it isn't in a confused state. [CI]

### Strategy 2: Error Productions

In table-driven parsers, the compiler writer can add **error productions** — productions with a reserved `error` keyword and synchronizing tokens — to tell the parser generator where to synchronize. This is the yacc/bison approach. [EaC]

CI describes the same technique: augment the grammar with a rule that successfully matches erroneous syntax, parse it safely, and report it as an error. "Error productions work well because you, the parser author, know how the code is wrong and what the user was likely trying to do." Example: extending `unary` to accept `+` even though Lox has no unary `+`, allowing the parser to give the message "Unary '+' expressions are not supported" rather than going into panic mode. "Mature parsers tend to accumulate error productions like barnacles since they help users fix common mistakes." [CI]

### Strategy 3: Parse-and-Continue

After error recovery, the parser must decide whether to generate code. "The compiler should not try to generate and optimize code for a syntactically invalid program. This requires simple handshaking between the error-recovery apparatus and the driver that invokes the compiler's various passes in order to halt after an unsuccessful parse" (p. 3307). [EaC]

CI's implementation: the `parse()` entry point catches the top-level `ParseError` and returns `null` instead of a tree. The driver checks `hadError` before invoking subsequent phases: [CI]
```java
Expr parse() {
    try {
        return expression();
    } catch (ParseError error) {
        return null;
    }
}
// In driver:
if (hadError) return;
interpreter.interpret(expression);
```

### Error Localization Quality

**LR(1) parsers have excellent error localization (p. 2289):** "The LR(1) parser detects syntax errors with a simple mechanism: it finds an invalid table entry. It detects the error as soon as possible, before it reads a word beyond those needed to prove the input erroneous. This property localizes the error to a specific point in the input." [EaC]

**LL(1) / recursive-descent parsers** also localize errors well because they know, at each point, exactly what set of words can appear next (the START sets). [EaC]

**Backtracking parsers** are worse: "Backtracking increases the asymptotic cost of parsing; in practice, it is an expensive way to find syntax errors" (p. 732). [EaC]

**CI on error reporting quality:** CI reports errors with token-level location information (line number and the token's lexeme), making the message "show the token's location and the token itself." This pattern — tracking the offending token rather than just a line number — is useful throughout the interpreter because "we use tokens throughout the interpreter to track locations in code." [CI]

---

## 10. AST Node Design Patterns

### Parse Trees vs. Abstract Syntax Trees [CI]

In a **parse tree**, every single grammar production becomes a node. An **abstract syntax tree (AST)** elides productions not needed by later phases — intermediate chain-through rules (like `expression → equality → comparison → ...` when no actual operators exist) are collapsed, keeping only the meaningful nodes.

### Typed Node Hierarchy (CI, jlox) [CI]

Rather than a single `Expression` class with an arbitrary list of children, CI uses a separate subclass per expression kind, exploiting the type system for correctness:

```java
abstract class Expr {
    static class Binary extends Expr {
        final Expr left;
        final Token operator;
        final Expr right;
        Binary(Expr left, Token operator, Expr right) { ... }
    }
    static class Unary extends Expr {
        final Token operator;
        final Expr right;
        Unary(Token operator, Expr right) { ... }
    }
    static class Literal extends Expr {
        final Object value;
        Literal(Object value) { ... }
    }
    static class Grouping extends Expr {
        final Expr expression;
        Grouping(Expr expression) { ... }
    }
}
```

This ensures that, e.g., accessing a second operand on a unary expression is a compile error, not a runtime null pointer. [CI]

**Key design choice:** These tree classes are "dumb structures" — bags of typed data with no behavior. They exist to enable the parser and interpreter to communicate. "The problem is that these tree classes aren't owned by any single domain... Trees span the border between those territories, which means they are really owned by neither." [CI]

**Separate `Stmt` hierarchy:** Expressions and statements are disjoint syntactically, so CI uses two separate class hierarchies (`Expr` and `Stmt`). This lets the Java compiler catch errors like passing a statement where an expression is expected. [CI]

### The Expression Problem [CI]

The AST design reflects a fundamental tension known as the **expression problem**: you have a set of types and a set of operations, and you need an implementation for each (type × operation) pair.

- **Object-oriented style** (rows = types): Easy to add new types (subclasses), hard to add new operations (must modify all classes).
- **Functional style** (columns = operations): Easy to add new operations (new functions with pattern matching), hard to add new types (must update all functions).

For AST nodes, operations (interpret, type-check, pretty-print) are added more often than node types, which makes the OO default (methods on node classes) a poor fit. [CI]

### The Visitor Pattern (CI, jlox) [CI]

CI's solution: the Visitor pattern approximates the functional style within Java — "it lets us add new columns to that table easily. We can define all of the behavior for a new operation on a set of types in one place, without having to touch the types themselves."

The core mechanism:
1. Define a `Visitor<R>` interface with one method per node type
2. Add a single abstract `accept(Visitor<R> v)` method to the base class
3. Each subclass implements `accept()` by calling the appropriate `visit*` method

```java
abstract class Expr {
    interface Visitor<R> {
        R visitBinaryExpr(Binary expr);
        R visitUnaryExpr(Unary expr);
        R visitLiteralExpr(Literal expr);
        R visitGroupingExpr(Grouping expr);
    }

    abstract <R> R accept(Visitor<R> visitor);

    static class Binary extends Expr {
        // ... fields ...
        @Override
        <R> R accept(Visitor<R> visitor) {
            return visitor.visitBinaryExpr(this);
        }
    }
}
```

Each operation (interpreter, pretty-printer, type-checker) becomes a class implementing `Visitor<R>`:

```java
class AstPrinter implements Expr.Visitor<String> {
    String print(Expr expr) { return expr.accept(this); }

    @Override
    public String visitBinaryExpr(Expr.Binary expr) {
        return parenthesize(expr.operator.lexeme, expr.left, expr.right);
    }
    // ...
    private String parenthesize(String name, Expr... exprs) {
        StringBuilder b = new StringBuilder();
        b.append("(").append(name);
        for (Expr expr : exprs) { b.append(" "); b.append(expr.accept(this)); }
        b.append(")");
        return b.toString();
    }
}
```

**Use of generics:** CI refines the pattern to use `Visitor<R>` (generic return type) rather than `void` visitors, so operations can produce values. The return type is fixed per operation class but varies between operations. [CI]

**Important clarification:** The Visitor pattern is not fundamentally about trees — it works equally well on a flat set of types. The recursion in `AstPrinter` comes from the tree structure of the data, not from the Visitor pattern itself. [CI]

### Metaprogramming AST Classes [CI]

With 21+ node types each requiring fields, constructors, and `accept()` overrides, CI uses a small code-generation script (`GenerateAst.java`) to emit the Java boilerplate from a compact description:

```java
defineAst(outputDir, "Expr", Arrays.asList(
    "Binary   : Expr left, Token operator, Expr right",
    "Grouping : Expr expression",
    "Literal  : Object value",
    "Unary    : Token operator, Expr right"
));
```

This approach — treating the AST definition as a domain-specific language with a dedicated generator — keeps the canonical definition compact and ensures consistency across all node types. [CI]

---

## 11. Implementation Tradeoffs: Table-Driven vs. Direct-Coded vs. Hand-Coded

### Table-Driven Parsers [EaC]

- Use generic skeleton parser + table lookup (Action/Goto)
- Easy to regenerate from grammar changes
- Table size can be large; techniques to shrink:
  1. **Shrink the grammar** — combine terminal symbols that behave identically (e.g., `num` and `name` both into `val`; `+` and `-` into `addsub`) — "produced a 48 percent reduction in table size" (p. 3699)
  2. **Combine identical rows/columns** — states with identical Action rows can share one row; "further reduce the Action table from 132 entries to 102 entries, an additional 23 percent reduction" (p. 4057). Together: 65% reduction.
  3. **Use sparse matrix representation** — tradeoff: saves memory but adds access overhead

### Direct-Coded Parsers (p. 4079) [EaC]

Each state becomes a small case statement or if-then-else that tests the next symbol. Action/Goto tables replaced with code. "While its structure makes it almost unreadable by humans, it should execute more quickly than the corresponding table-driven parser." Achieves strong cache locality if related states are co-located in memory.

### Grammar Optimization for Speed (Section 3.6.1) [EaC]

Eliminate "useless productions" — productions with a single-symbol RHS that just chain through without adding value:
```
Factor → ( Expr ) | num | name
```
Fold Factor's alternatives directly into Term's productions. This removes one derivation level, reducing reductions (LR(1)) or procedure calls (recursive descent) at the cost of more productions and potentially larger tables.

**Tradeoff warning (p. 3655):** "Folding away useless productions has its costs. In an LR(1) parser, it can make the tables larger... The resulting parser performs fewer reductions (and runs faster), but has larger tables." [EaC]

---

## 12. Pratt Parsing (Top-Down Operator Precedence)

Pratt parsing, also called top-down operator precedence parsing, is an alternative to nonterminal-stratified recursive descent for handling operator precedence. Where stratified recursive descent encodes precedence in the grammar structure (one nonterminal per level), Pratt parsing encodes it in numeric precedence values associated with tokens.

CI uses Pratt parsing in clox (the C implementation), contrasting with jlox's stratified recursive descent.

**Core idea:** Each token type has two associated parsing functions:
- A **prefix parse function** — called when the token appears at the beginning of an expression (handles literals, unary operators, grouping)
- An **infix parse function** — called when the token appears after an already-parsed left-hand operand (handles binary operators)

Each infix operator also carries a **precedence level** (a numeric value). The central `parsePrecedence(minPrec)` function drives parsing: it calls the current token's prefix function to get a left-hand side, then repeatedly consumes infix operators whose precedence is at least `minPrec`, calling their infix functions.

**The parse table in clox:**
```c
typedef enum {
    PREC_NONE,
    PREC_ASSIGNMENT,  // =
    PREC_OR,          // or
    PREC_AND,         // and
    PREC_EQUALITY,    // == !=
    PREC_COMPARISON,  // < > <= >=
    PREC_TERM,        // + -
    PREC_FACTOR,      // * /
    PREC_UNARY,       // ! -
    PREC_CALL,        // . ()
    PREC_PRIMARY
} Precedence;

typedef void (*ParseFn)();

typedef struct {
    ParseFn prefix;
    ParseFn infix;
    Precedence precedence;
} ParseRule;
```

Each `TokenType` maps to a `ParseRule`. Tokens that never begin an expression have `NULL` as their prefix function; tokens that never appear in infix position have `NULL` as their infix function.

**The core function:**
```c
static void parsePrecedence(Precedence precedence) {
    advance();
    ParseFn prefixRule = getRule(parser.previous.type)->prefix;
    if (prefixRule == NULL) {
        error("Expect expression.");
        return;
    }
    prefixRule();                           // parse prefix (e.g., literal, unary)

    while (precedence <= getRule(parser.current.type)->precedence) {
        advance();
        ParseFn infixRule = getRule(parser.previous.type)->infix;
        infixRule();                        // parse infix (e.g., binary op)
    }
}
```

**Comparison with stratified recursive descent:**

| Property                  | Stratified Recursive Descent [CI/jlox] | Pratt Parsing [CI/clox]             |
|---------------------------|----------------------------------------|--------------------------------------|
| Precedence representation | One function per grammar level         | Numeric value per token              |
| Adding new operators      | Add/modify grammar functions           | Add entry to parse table             |
| Left recursion            | Eliminated via EBNF loops              | Never arises (data-driven loop)      |
| Readability               | Grammar structure visible in code      | More compact but less obvious        |
| Flexibility               | Fixed at grammar design time           | Parse functions composable at runtime|
| Used in                   | jlox (Java tree-walk interpreter)      | clox (C bytecode compiler)           |

**Advantages of Pratt parsing:** compact implementation; adding operators requires only a table entry; handles mixed-precedence operator sequences without deep call stacks; integrates neatly with single-pass bytecode generation because the "current expression being parsed" maps directly to the code generation context.

**When to use Pratt parsing vs. stratified recursive descent:** Both correctly handle precedence and associativity. Stratified RD is more readable and maps directly to the grammar; Pratt is more data-driven and scales better when the operator table is large or dynamic. For a language with a fixed, modest set of operators, either works. Pratt parsing is commonly used in production single-pass compilers and interpreters (e.g., Pratt's original paper described it for Lisp, and it's used in various JavaScript engines).

---

## 13. Common Mistakes and Warnings

### Grammar Design Mistakes

1. **Using left recursion in a grammar intended for top-down parsing** — causes infinite loops (stack overflow in recursive descent). Always eliminate before building an LL(1) or recursive-descent parser. [EaC, CI]

2. **Simply reversing a left-recursive rule** — e.g., rewriting `Expr → Expr + Term` as `Expr → Term + Expr` — this **changes associativity** from left to right. Use the proper two-nonterminal transformation (or CI's EBNF loop pattern). [EaC]

3. **Not left-factoring when common prefixes exist** — if multiple productions start with the same token(s), the LL(1) parser cannot choose among them deterministically. Always left-factor. [EaC]

4. **Assuming left-factoring always works** — "some context-free languages have no backtrack-free grammar" (p. 1485). If left-factoring doesn't resolve conflicts, use LR(1). [EaC]

5. **Ignoring operator precedence and associativity in grammar design** — the simple ambiguous grammar `Expr → Expr Op Expr | name` ignores both. Always encode levels via nonterminal hierarchy (or a Pratt parse table). [EaC, CI]

6. **Identical RHS for different nonterminals** — e.g., `Subscript → name(ArgList)` and `Call → name(ArgList)` causes a reduce-reduce conflict. Resolve by combining into one production and using semantic analysis, or by enriching the scanner's classification. [EaC]

7. **Stopping at the first error** — the default behavior of simple parsers. Real compilers must implement error recovery to report multiple errors per compilation. Use panic mode + synchronize. [EaC, CI]

8. **Generating code after parse errors** — the compiler must not attempt optimization passes on syntactically invalid programs. Always handshake between error recovery and subsequent passes. [EaC, CI]

9. **Allowing cascaded errors to flood the user** — after the first real error, the parser's confusion can create phantom errors. Minimize cascaded errors by synchronizing promptly and not reporting errors from within the recovery zone. [CI]

10. **Mistaking l-values and r-values in assignment parsing** — `a = b` and `a + b = c` look similar at the start. The standard technique (CI): parse the left-hand side as an expression, then check whether it is a valid assignment target (`Expr.Variable`, `Expr.Get`) before constructing the assignment node. Report an error without panicking if it isn't. [CI]

### LR(1) Table Construction Mistakes

11. **Computing FIRST(δa) incorrectly in closure** — specifically in the LR(1) closure function where δ is the suffix after C in an item `[A → β•Cδ, a]`. The book explicitly calls this "the point where a human is most likely to make a mistake" (p. 2503). [EaC]

12. **Attempting to hand-build LR(1) tables** — "The table-construction algorithm requires scrupulous bookkeeping; it is a prime example of the kind of task that should be automated and relegated to a computer" (p. 2304). Use a parser generator. [EaC]

13. **Ignoring shift-reduce conflicts** — many tools will silently resolve them by favoring shift. This can accidentally implement the correct behavior (as with if-then-else), but should be explicitly understood and documented, not accidentally relied upon. [EaC]

14. **Combining rows/columns that only differ in error entries** — "Combining such columns produces the same behavior on correct inputs. It does change the parser's behavior on erroneous inputs and may impede the parser's ability to provide accurate and helpful error messages" (p. 4066). [EaC]

---

## 14. Quick Reference: Grammar Transformation Checklist

When converting a grammar for use in a top-down (LL(1)) parser: [EaC]

- [ ] Check for and eliminate all **direct left recursion** (`A → Aα`) using the two-nonterminal transformation
- [ ] Check for and eliminate all **indirect left recursion** using the systematic algorithm (Fig. 3.6)
- [ ] Compute **FIRST sets** for all grammar symbols
- [ ] Apply **left factoring** to any nonterminal where two productions have common prefixes (overlapping FIRST sets on their RHS)
- [ ] Compute **FOLLOW sets** for all nonterminals
- [ ] Compute **START sets** for all productions
- [ ] Verify the **LL(1) condition**: for any nonterminal A, all pairs of alternative RHS must have disjoint START sets
- [ ] If LL(1) condition fails after all transformations, switch to LR(1)

When using an LR(1) parser generator: [EaC]

- [ ] Submit the grammar to the generator
- [ ] If **shift-reduce conflict**: grammar likely has a prefix issue (like if-then-else); resolve by rewriting or explicitly choose shift
- [ ] If **reduce-reduce conflict**: grammar has two productions with identical RHS; resolve by combining into one or enriching scanner classification
- [ ] Consider LALR(1) first (yacc/bison default); only fall back to canonical LR(1) if LALR reports conflicts that are not genuine ambiguities

When implementing a hand-written recursive descent parser (CI checklist): [CI]

- [ ] Stratify the grammar into one function per precedence level (or build a Pratt parse table for each token)
- [ ] Use the EBNF loop pattern `X → Y (op Y)*` for left-associative binary operators (translates to a `while` loop)
- [ ] Use right-recursive call for right-associative operators (assignment)
- [ ] Implement `consume()` to require and advance past mandatory tokens, throwing `ParseError` on mismatch
- [ ] Implement `synchronize()` to discard tokens to the next statement boundary
- [ ] Wrap the declaration-level parse loop in a try/catch that calls `synchronize()` on `ParseError`
- [ ] After parse, gate subsequent compiler phases on `hadError`
- [ ] Use a separate `Stmt` class hierarchy from `Expr` so the type system enforces the syntactic distinction

---

## 15. The LL(1) Parsing Condition: Formal Statement [EaC]

A grammar is LL(1) if and only if, for any nonterminal A with multiple right-hand sides `A → β1 | β2 | ... | βn`:

```
START(A → βi) ∩ START(A → βj) = ∅   for all 1 ≤ i, j ≤ n, i ≠ j
```

Where:
```
START(A → β) = FIRST(β)                       if ε ∉ FIRST(β)
             = (FIRST(β) - {ε}) ∪ FOLLOW(A)   if ε ∈ FIRST(β)
```

If this condition holds for every nonterminal, the grammar can be parsed left-to-right with one symbol of lookahead, without backtracking.

The LL(1) table-construction algorithm then fills `Table[A, w] = p` for each production p of the form `A → β` and each `w ∈ START(A → β)`. If any table entry is multiply defined, the grammar is not LL(1).

---

## 16. LR(1) Parsing: Key Concepts Summary [EaC]

**LR(1) item:** `[A → β • γ, a]` — production + position of stacktop + lookahead terminal.

Three types of items:
- **Possibility** `[A → • βγ, a]`: A valid completion path; parser hasn't started on A yet
- **Partially complete** `[A → β • γ, a]`: Parser has recognized β; needs to find γ
- **Complete** `[A → βγ •, a]`: Handle found; reduce βγ to A if lookahead is a

**Canonical collection CC:** Set of all parser states, where each state CCi is a set of LR(1) items. Built by fixed-point computation using `closure` and `goto` operations.

**Action table:** For state i and terminal a:
- `shift j`: shift a onto stack, move to state j
- `reduce A → β`: pop |β| items, push A, use Goto table
- `accept`: input successfully parsed
- `error`: syntax error

**Goto table:** For state i and nonterminal A, gives next state after reducing to A.

**Time complexity:** O(input size + derivation length) — one shift per word, one reduce per derivation step.

**Error detection strength:** LR(1) parsers detect errors at the earliest possible point — before reading any input beyond what is needed to prove the input invalid (p. 2294). [EaC]
