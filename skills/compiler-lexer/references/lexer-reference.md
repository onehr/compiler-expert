# Lexer / Scanner Reference

**Sources:**
- **[EaC]** Engineering a Compiler (EAC), Chapter 2 — Cooper & Torczon (3rd ed., 2023)
- **[CI]** Crafting Interpreters — Robert Nystrom (craftinginterpreters.com); Chapter 4 (jlox tree-walk scanner in Java) and Chapter 16 (clox bytecode scanner in C)

---

## 1. Core Concepts and Pipeline

### What a Scanner Does

A scanner transforms a character stream into a word stream. For each word it produces a **token**: the pair `⟨lexeme, category⟩`, where:
- **lexeme** — the actual text spelling of the recognized word
- **category** — the syntactic category (terminal symbol) it belongs to

The scanner is the *only* pass that touches every character in the input. Because input size is larger here than in any other pass, speed matters more here than anywhere else. [EaC]

> "Both generated and hand-crafted scanners can be implemented to require just O(1) time per character, so they run in time proportional to the number of characters in the input stream." (p. 28) [EaC]

CI frames the same concept more informally: the scanner "takes in raw source code as a series of characters and groups it into a series of chunks we call tokens. These are the meaningful 'words' and 'punctuation' that make up the language's grammar." [CI]

**Terminology note [CI]:** The terms "scanning" and "lexing" (short for lexical analysis) are used interchangeably in modern practice. Historically, "scanner" referred only to the character-reading phase and "lexer" to the token-recognizing phase, but since reading source into memory is now trivial, the distinction has collapsed.

### Three-Phase Pipeline: RE → NFA → DFA [EaC]

The standard automated approach follows this cycle:

```
Regular Expression
       |
       | Thompson's Construction (§2.4.2)
       ▼
      NFA
       |
       | Subset Construction (§2.4.3)
       ▼
      DFA
       |
       | Hopcroft's Minimization (§2.4.4) [optional but recommended]
       ▼
   Minimal DFA
       |
       | Code generation (§2.5)
       ▼
  Table-driven or Direct-coded Scanner
```

**Decision rule [EaC]:** Always pass through all three phases when using a generator. Skip minimization only if you have a size-insensitive use case — minimization reduces footprint and can improve cache behavior.

**CI's perspective on this pipeline [CI]:** CI explicitly defers to the formal theory here ("other books cover this better than I could") and points to the Dragon Book for RE-to-DFA theory. CI's own scanners are written by hand and bypass the formal pipeline entirely — but CI does connect the dots in Chapter 16, noting that the keyword trie it builds by hand "is exactly a DFA that recognizes Lox keywords," and explaining that tools like Lex/Flex exist precisely to automate this RE-to-DFA transformation.

---

## 2. NFA vs. DFA — When Each Is Used

### NFA (Nondeterministic Finite Automaton) [EaC]

- **When used:** Intermediate representation produced by Thompson's construction. Never executed directly in production scanners.
- **Why intermediate?** NFAs are easy to build compositionally (union, concatenation, closure are mechanical template operations), but expensive to simulate — O(|N|) state tracking per character in the worst case.
- **Key property:** Can have multiple transitions on the same character, and ε-transitions (transitions that consume no input).
- **ε-transitions enable:** Composing sub-NFAs without rewriting structure. E.g., combining FAm and FAn for concatenation mn by adding an ε-transition from FAm's accepting state to FAn's start state.

### DFA (Deterministic Finite Automaton) [EaC, CI]

- **When used:** The execution-time representation. Every scanner runs a DFA (either explicitly via table or implicitly via direct-coded branches). [EaC]
- **Why DFA for execution?** "DFA execution is much easier to simulate than NFA execution." (p. 49) [EaC] The DFA makes exactly one transition per input character, with O(1) cost per transition.
- **Tradeoff vs. NFA:** A DFA can have exponentially more states than the equivalent NFA (up to |2^N| where N is NFA state count). However: "any expansion introduced by the subset construction does not affect the asymptotic running time of the DFA." (p. 51) [EaC] Space is the only concern, not speed.
- **Minimization matters for DFAs** because state count determines cache footprint, not runtime complexity: "A smaller recognizer may fit better into the processor's lowest level of cache memory, producing faster average accesses." (p. 56) [EaC]
- **CI's DFA connection [CI]:** CI explains the DFA concept intuitively by observing that a hand-crafted keyword trie and a number-literal recognizer are both special cases of DFAs: "In a DFA, you have a set of states with transitions between them, forming a graph... each transition is a character that gets matched from the string." CI also explains that Lex converts regex descriptions to DFAs and that most regex engines work the same way.

### Decision: NFA vs. DFA in Your Code

| Situation | Use |
|---|---|
| Specifying/building a recognizer | RE → NFA (Thompson's) [EaC] |
| Execution-time scanning | DFA (always) [EaC] |
| Combining multiple token rules | Build one NFA per rule, merge with ε-transitions, then subset-construct one combined DFA [EaC] |
| Want smallest possible DFA | Apply Hopcroft's minimization after subset construction [EaC] |
| Need simpler implementation, accept slightly larger DFA | Use Brzozowski's algorithm (simpler code, acceptable performance in practice) [EaC] |
| Hand-crafted keyword recognition without a hash table | Encode the keyword set as a trie, implemented as nested switch statements [CI] |

---

## 3. Table-Driven vs. Direct-Coded vs. Hand-Coded Scanners

### 3.1 Table-Driven Scanners [EaC]

**How it works:** A generic skeleton scanner loop drives a language-specific transition table δ. The skeleton code never changes; only the tables change per language.

```
while (state ≠ se) do
    char    ← NextChar()
    col     ← CharClass[char]    // character classification table
    state   ← δ[state, col]      // transition table
```

**Key costs per character:**
- Two memory address calculations
- Two memory loads (CharClass + δ)
- These are O(1) but with non-trivial constants

**When to use:**
- When you want a clean separation of scanner logic and language spec
- When using a generator (lex/flex output is table-driven)
- When code simplicity and maintainability outweigh raw speed
- When the DFA is large and table compression matters more than branch elimination

**Pitfall — quadratic rollback:** The basic table-driven skeleton can exhibit O(n²) rollback behavior on certain inputs. Example: RE `ab | (ab)*c` on input `abababab` forces scanning all characters before determining the longest prefix is `ab`, then re-scanning on the next call. "In the worst case, roll back can create O(n²) behavior." (p. 65) [EaC]

**Fix:** Use the **maximal munch scanner** variant (Fig. 2.13). It maintains a `Failed[state, InputPos]` bit-array to record dead-end transitions so they are not re-explored. The overhead is one bit-test per iteration. Use this when your REs could trigger quadratic rollback. "Most programming languages have simple enough microsyntax that this kind of quadratic roll back cannot occur." (p. 68) [EaC] — but verify this for your language before assuming.

### 3.2 Direct-Coded Scanners [EaC]

**How it works:** No explicit state variable. Each DFA state is a labeled code fragment. Transitions become `goto` statements or branches to state labels. The state machine is encoded in control flow, not data.

```c
s1:
    char   ← NextChar()
    lexeme ← lexeme + char
    push(s1)
    if '0' ≤ char ≤ '9'
        then goto s2
        else goto sout
```

**Why faster:** Eliminates the two-array-lookup overhead per character. Values like `char` and `lexeme` can remain in registers. No 2D address arithmetic.

**Reported speedup:** "They report a factor of five improvement over table-driven implementations." (p. 81, citing Waite and Heuring) [EaC]

**Downside:** The generated code "violates many of the precepts of structured programming." (p. 69) [EaC] — often called "spaghetti code." Humans should not read or debug it; it must be generated.

**When to use:**
- When scanner performance is critical and you are using a generator
- When the scanner is generated (humans do not read it anyway)
- No extra design work for the compiler writer — straightforward for a generator to emit

**When states have too many transitions:** Consider a "computed branch" (jump table into a table of labels) rather than a long if-else chain. [EaC]

### 3.3 Hand-Coded Scanners [EaC, CI]

**Reality check [EaC]:** "In an informal survey of commercial compiler groups, we found that a surprisingly large fraction used hand-coded scanners." (p. 69) GCC 4.0 uses hand-coded scanners in several front ends despite flex existing.

**CI's stance [CI]:** CI builds hand-coded scanners for both of its implementations by choice: "Since our goal is to understand how a scanner does what it does, we won't be delegating that task. We're about handcrafted goods." CI also notes the approach is used by serious implementations: "the same approach is essentially what V8 does, and that's currently one of the world's most sophisticated, fastest language implementations."

**Why hand-coded wins in practice [EaC]:**
1. **Optimization of specific paths** — e.g., if the DFA has only one accepting state, eliminate the stack mechanism for tracking accepting states entirely
2. **Interface optimization** — instead of returning a lexeme string, return the parsed value (e.g., return the integer register number, not the string `"r29"` — avoiding a subsequent conversion)
3. **Custom I/O integration** — fine-grained control over buffering, error reporting, and rollback distance

**Performance ceiling:** A hand-coded scanner can still only achieve O(1) per character. The advantage is in reducing the constant factor and eliminating overhead the generator cannot know to remove. [EaC]

> "We suspect that hand-coded scanners persist for one simple reason: they are small, simple programs that can be fun to write." (p. 69) [EaC]

**When to use:**
- When you have a stable, well-understood language spec
- When scanner performance is on the critical path
- When the scanner-parser interface benefits from custom data types (returning ints, not strings) [EaC]
- When debugging/error messages need tight integration with source position tracking
- When you want to understand the scanner from first principles [CI]

### 3.4 CI's Hand-Coded Scanner Architecture (jlox — Java) [CI]

CI's first implementation (jlox) uses an **eager, all-at-once scanning** approach:

```java
class Scanner {
    private final String source;          // entire source as a string
    private final List<Token> tokens = new ArrayList<>();
    private int start = 0;               // start of current lexeme
    private int current = 0;             // current character position
    private int line = 1;                // current line number

    List<Token> scanTokens() {
        while (!isAtEnd()) {
            start = current;
            scanToken();
        }
        tokens.add(new Token(EOF, "", null, line));
        return tokens;
    }
}
```

**Key design choices:**
- Entire source file is read into a `String` before scanning begins
- `start` and `current` are integer offsets into that string — no pointer arithmetic
- `scanTokens()` returns a complete `List<Token>` before the parser runs (eager/batch model)
- The `EOF` sentinel token is appended explicitly to make the parser cleaner
- Line tracking is done by incrementing `line` whenever `\n` is consumed

**Core primitives used throughout [CI]:**

```java
private char advance() { return source.charAt(current++); }     // consume and return
private char peek()    { return isAtEnd() ? '\0' : source.charAt(current); }    // lookahead 1
private char peekNext() {                                        // lookahead 2
    if (current + 1 >= source.length()) return '\0';
    return source.charAt(current + 1);
}
private boolean match(char expected) {                          // conditional consume
    if (isAtEnd() || source.charAt(current) != expected) return false;
    current++;
    return true;
}
```

CI explicitly limits lookahead to two characters (`peek` and `peekNext`) and makes this visible in the code structure: "I could have made peek() take a parameter for the number of characters ahead to look instead of defining two functions, but that would allow arbitrarily far lookahead. Providing these two functions makes it clearer to a reader of the code that our scanner looks ahead at most two characters."

### 3.5 CI's Hand-Coded Scanner Architecture (clox — C) [CI]

CI's second implementation (clox) uses a fundamentally different **on-demand (lazy) scanning** approach:

```c
typedef struct {
    const char* start;    // pointer to start of current lexeme
    const char* current;  // pointer to current character
    int line;
} Scanner;

Scanner scanner;   // single global instance

Token scanToken();  // called by the compiler one token at a time
```

**Key differences from jlox [CI]:**

1. **On-demand vs. eager:** The compiler calls `scanToken()` each time it needs the next token. There is no pre-built list. "At any point in time, the compiler needs only one or two tokens — remember our grammar requires only a single token of lookahead — so we don't need to keep them all around at the same time."

2. **Pointer-based lexeme representation:** Instead of copying lexeme strings, clox stores `(start, length)` pairs pointing into the original source buffer. "We represent a lexeme by a pointer to its first character and the number of characters it contains. This means we don't need to worry about managing memory for lexemes at all and we can freely copy tokens around."

3. **Tokens passed by value on the C stack:** No heap allocation. `makeToken()` returns a `Token` struct directly.

4. **No literal conversion in the scanner:** jlox converts literal lexemes to runtime values (Java `Double`, `String`) at scan time. clox defers this to the compiler: "we defer converting the literal lexeme to a runtime value until later... right when we are ready to store it in the chunk's constant table."

5. **`TOKEN_ERROR` type for error propagation:** Instead of calling an error reporter directly from the scanner (as jlox does), clox produces a synthetic `TOKEN_ERROR` token carrying the error message as its "lexeme." The compiler detects this type and handles recovery. This decouples the scanner from the error-reporting mechanism.

```c
static Token errorToken(const char* message) {
    Token token;
    token.type = TOKEN_ERROR;
    token.start = message;   // points to a string literal, not source code
    token.length = (int)strlen(message);
    token.line = scanner.line;
    return token;
}
```

6. **`skipWhitespace()` called at the top of each `scanToken()` call:** Rather than handling whitespace inside the main switch, clox separates it into a dedicated function. Comments are also consumed inside `skipWhitespace()`, keeping the main dispatch switch clean.

**Contrast with EaC's table-driven model:** EaC's double-buffer scheme exists because character-by-character I/O was expensive. CI's clox loads the entire source file into a C string before scanning begins, making the buffer management problem moot. The scanner works directly on that string with raw pointer arithmetic. [CI]

### Summary Decision Table

| Criterion | Table-Driven | Direct-Coded | Hand-Coded (EaC style) | Hand-Coded (CI style) |
|---|---|---|---|---|
| Dev effort | Low (generator) [EaC] | Low (generator) [EaC] | High [EaC] | Medium [CI] |
| Runtime speed | Baseline [EaC] | ~5x faster than table [EaC] | Best achievable [EaC] | Excellent in practice [CI] |
| Code readability | High (skeleton) [EaC] | Low (spaghetti) [EaC] | High [EaC] | High [CI] |
| Flexibility to change language | High [EaC] | High [EaC] | Low [EaC] | Low [CI] |
| Custom interface optimization | None | None | Full control [EaC] | Partial [CI] |
| Quadratic rollback risk | Yes (need maximal munch) [EaC] | Yes (need maximal munch) [EaC] | Can be designed away [EaC] | N/A (no rollback) [CI] |
| Memory model | Double-buffer [EaC] | Double-buffer [EaC] | Flexible [EaC] | Full source in memory [CI] |
| Token storage | List/array | List/array | List/array | On-demand (by value) [CI] |

---

## 4. Maximal Munch Rule

### What It Is [EaC, CI]

The scanner always returns the **longest possible prefix** of the remaining input that forms a valid word. This is the default behavior of all DFA-based scanners. [EaC]

**Operational behavior [EaC]:** The scanner runs the DFA until it hits an error state (no valid transition). It then backs up through the state history until it finds an accepting state. The lexeme at that point is the result.

> "Rather than exhausting the input stream, the scanner simulates the DFA until it hits an error — that is, until it is in some state di with input character c such that δ(di, c) = se, the error state." (p. 59) [EaC]

**CI's formulation [CI]:** "When two lexical grammar rules can both match a chunk of code that the scanner is looking at, whichever one matches the most characters wins." CI uses the identifier-vs-keyword problem as its primary example: if we tried to match `or` as a keyword by checking only the first two characters of `orchid`, we would incorrectly emit a keyword token. Maximal munch prevents this — we scan the full identifier `orchid` first, then determine its type.

CI also shows the C example `---a;`: maximal munch forces this to scan as `-- -a;` (decrement followed by negation) rather than `- --a;` (subtraction followed by decrement), even if the latter would parse correctly. The scanner applies the rule mechanically, potentially producing a syntax error that the parser must then handle.

### How the Rollback Works [EaC]

1. The scanner maintains a **stack of states** visited during scanning
2. When the DFA enters error state `se`, it pops states from the stack until it finds an accepting state
3. It truncates the lexeme and calls `RollBack()` to reset the input pointer

**Rollback is necessary because:** The DFA may overshoot the word boundary. E.g., recognizing `>=` when scanning `>>=` — the scanner reads `>`, `>`, `=` before determining the first `>` alone is not valid, then backs up. [EaC]

**CI avoids explicit rollback [CI]:** CI's hand-coded scanners never back up. Instead, they use single-character lookahead (`peek()`) before committing to consume a character. For two-character tokens like `!=`, the scanner sees `!` with `advance()`, then conditionally consumes the `=` only if `match('=')` returns true. If the `=` is not present, `!` alone is returned without any rollback. This approach is only possible because the scanner is hand-coded with knowledge of the language's grammar.

### The Quadratic Rollback Problem [EaC]

**Trigger condition:** An RE where a long match attempt fails, forcing full rollback repeatedly.

**Canonical example from book:** RE `ab | (ab)*c`, input `abababab`
- First call: scans 8 characters, fails, backs up to return `ab` (2 chars)
- Second call: scans 6 characters, backs up to return `ab`
- Third call: scans 4 characters...
- **Total: O(n²) character operations**

**Fix — the Maximal Munch Scanner (Fig. 2.13):** Record `Failed[state, inputPos]` — if a `⟨state, inputPos⟩` pair led to a dead end once, skip it next time. This amortizes rollback cost across the entire input. [EaC]

**When this matters in practice:** "Most programming languages have simple enough microsyntax that this kind of quadratic roll back cannot occur." (p. 68) [EaC] Check your specific REs for this pattern before adding the overhead of the `Failed` table. CI's hand-coded scanners are immune to this by construction.

### Edge Cases and Pitfalls [EaC]

**FORTRAN blanks:** FORTRAN 66 treats blanks as insignificant, so `do10i = 1,100` (a loop) vs. `do10i = 1` (an assignment to variable `do10i`) cannot be distinguished without lookahead past the `=`. The scanner must read the comma to disambiguate. This is a maximal munch failure mode — avoid language designs where whitespace is insignificant.

**Multicharacter delimiters in comments:** The RE for `/* ... */` C-style comments is non-obvious: `/* ( ^* | *+ ^/ )* */`. The complexity arises because `*` can appear inside the comment; the DFA must track whether it has seen a `*` to handle `*/` correctly. "The complexity of the RE and FA for multiline comments arises from the use of multicharacter delimiters." (p. 49) [EaC]

**Identifier vs. keyword ambiguity:** Keywords like `then` also match the identifier RE. Maximal munch will consume the full word either way. The category is resolved by precedence rules or symbol table lookup — not by the scanner stopping early. [EaC, CI]

---

## 5. Input Buffering

### Why Buffering Is Necessary [EaC]

"While character-by-character I/O leads to clean algorithms, the overhead of a function call per character is significant relative to the cost of simulating the DFA." (p. 70) [EaC]

A function call per character is too expensive. The fix: amortize I/O cost with a buffer.

### The Double-Buffer Scheme (§2.5.4) [EaC]

**Structure:** Two adjacent buffers of size n, accessed modulo 2n.

```
NextChar():
    Char  ← Buffer[Input]
    if Char ≠ eof:
        Input ← (Input + 1) mod 2n
        if (Input mod n = 0):
            fill Buffer[Input : Input + n - 1]
            Fence ← (Input + n) mod 2n
    return Char

RollBack():
    if Input = Fence:
        signal rollback error    // can't go back further
    else:
        Input ← (Input - 1) mod 2n
```

**The Fence pointer:** Marks the start of the valid rollback context. Updated each time a new buffer half is filled. Prevents rolling back into data that has been overwritten.

**Practical sizing:** "By choosing a reasonable size for n, such as 2,048, 4,096, or more, the compiler writer can keep the I/O overhead low." (p. 71) [EaC]

**Rollback bound:** "The amount of rollback required by any particular input is bounded by the longest sequence of whitespace-free characters. Few programs contain identifiers with more than 2,048 characters." (p. 72) [EaC] So n=2048 or n=4096 is sufficient for real programs.

### Implementation Notes [EaC]

- `NextChar` and `RollBack` are small enough to **expand inline** (via macro), avoiding function-call overhead
- Use a power-of-2 buffer size so that modulo is a bitwise AND
- `NextChar` amortizes the fill cost: every n characters, one buffer fill; per-character cost approaches zero as n grows

### Interaction with Maximal Munch Scanner [EaC]

The `Failed[state, InputPos]` table in the maximal munch scanner can use `InputPos` modulo buffer size to bound its size: "if the scanner uses a finite input buffer, the number of columns in Failed can be reduced to the size of the input buffer." (p. 67) [EaC]

### CI's Alternative: Full Source in Memory [CI]

CI takes a simpler approach that sidesteps buffering entirely. Both jlox (Java) and clox (C) read the entire source file into memory before scanning begins:

**jlox:**
```java
byte[] bytes = Files.readAllBytes(Paths.get(path));
run(new String(bytes, Charset.defaultCharset()));
```

**clox:**
```c
static char* readFile(const char* path) {
    FILE* file = fopen(path, "rb");
    fseek(file, 0L, SEEK_END);
    size_t fileSize = ftell(file);
    rewind(file);
    char* buffer = (char*)malloc(fileSize + 1);
    size_t bytesRead = fread(buffer, sizeof(char), fileSize, file);
    buffer[bytesRead] = '\0';
    fclose(file);
    return buffer;
}
```

**Tradeoff vs. EaC's double-buffer scheme:**

| Aspect | EaC Double-Buffer | CI Full-Source-in-Memory |
|---|---|---|
| Memory use | O(buffer size) — constant | O(source size) — scales with input |
| Supports rollback? | Yes, up to n characters | clox: no rollback needed; jlox: random access via index |
| Complexity | Higher (Fence pointer, modulo arithmetic) | Very simple |
| Practical for modern hardware? | Yes, but arguably overengineered for most use cases | Yes — source files are small relative to available RAM |
| REPL / interactive use | Works naturally | jlox: works; clox: per-line buffer of 1024 chars |

CI's approach is reasonable for modern systems where source files are small. EaC's double-buffer scheme targets environments where streaming from disk is necessary or source files could be very large. For most language implementation projects today, CI's approach is simpler and sufficient.

---

## 6. Token Representation Choices

### The Token Structure [EaC, CI]

A token is the pair `⟨lexeme, category⟩`. Design choices:

**Category as integer [EaC]:** Most generators assign small integer codes to categories. The DFA final states map to these integers via a `TokenType[state]` table.

**Lexeme as string vs. parsed value [EaC]:** The table-driven and direct-coded scanners typically return the lexeme as a character string. A hand-coded scanner can return a pre-parsed value:

> "If the parser has a syntactic category register name, the scanner might return the actual register number rather than the string that contains it — avoiding that conversion in the parser and eliminating multiple concatenations in the scanner." (p. 70) [EaC]

**Decision [EaC]:** If the same token type always has a deterministic parse (e.g., integer literals, register names), parse it in the scanner. If the parsing is context-dependent, return the string.

### CI's Token Type Design [CI]

CI uses an explicit enum for token types, organized by category:

**jlox (Java) — `TokenType` enum:**
```java
enum TokenType {
    // Single-character tokens
    LEFT_PAREN, RIGHT_PAREN, LEFT_BRACE, RIGHT_BRACE,
    COMMA, DOT, MINUS, PLUS, SEMICOLON, SLASH, STAR,

    // One or two character tokens
    BANG, BANG_EQUAL, EQUAL, EQUAL_EQUAL,
    GREATER, GREATER_EQUAL, LESS, LESS_EQUAL,

    // Literals
    IDENTIFIER, STRING, NUMBER,

    // Keywords
    AND, CLASS, ELSE, FALSE, FUN, FOR, IF, NIL, OR,
    PRINT, RETURN, SUPER, THIS, TRUE, VAR, WHILE,

    EOF
}
```

**clox (C) — `TokenType` enum (identical structure, different naming):**
```c
typedef enum {
    TOKEN_LEFT_PAREN, TOKEN_RIGHT_PAREN, /* ... same categories ... */
    TOKEN_ERROR,   // unique to clox — used for error propagation
    TOKEN_EOF
} TokenType;
```

The `TOKEN_ERROR` type is unique to clox. CI explains it is used to decouple error detection (in the scanner) from error reporting (in the compiler): "the scanner produces a synthetic 'error' token for that error and passes it over to the compiler. This way, the compiler knows an error occurred and can kick off error recovery before reporting it."

### CI's Token Data Structure [CI]

**jlox Token class:**
```java
class Token {
    final TokenType type;
    final String lexeme;   // copy of the matched text
    final Object literal;  // pre-converted runtime value (Double, String, null)
    final int line;
}
```

**clox Token struct:**
```c
typedef struct {
    TokenType type;
    const char* start;  // pointer into source string (no copy)
    int length;         // length of lexeme
    int line;
} Token;
```

**Key differences:**

- **jlox copies the lexeme string; clox uses a pointer + length into the original source.** This makes clox tokens zero-allocation and freely copyable (passing by value on the stack is safe as long as the source outlives all tokens).
- **jlox stores a pre-converted `literal` value; clox stores only the raw lexeme.** CI explains that converting to a runtime value in C would require a union and type tag, adding scanner complexity. Deferring to the compiler keeps the scanner simpler, at the cost of some redundancy.
- **jlox tokens include line number; clox tokens also include line number** — both identify location by line only (not column), which CI acknowledges is minimal but sufficient for the book's purposes.

**Location encoding strategies [CI]:** CI notes two approaches:
1. Store the line number directly in each token (simple, what CI does)
2. Store only the offset from the start of the source; convert to line/column lazily when displaying an error. "An offset can be converted to line and column positions later by looking back at the source file and counting the preceding newlines. That sounds slow, and it is. However, you need to do it only when you need to actually display a line and column to the user. Most tokens never appear in an error message."

### Final-State-to-Category Mapping [EaC]

When building a multi-category DFA:
- Build a **separate NFA** per rule using Thompson's construction
- Merge NFAs with a new start state and ε-transitions to each sub-NFA's start
- Apply subset construction — the result will have distinct final states per category
- Apply Hopcroft's minimization with **initial partition that separates final states by category** (do not group all final states together)

> "If, however, the scanner generator constructs an initial partition that places the final states for each syntactic category in a distinct set in the initial partition, then the rest of the algorithm will maintain that property." (p. 57) [EaC]

**Pitfall — naive minimization destroys category mapping:** "Hopcroft's algorithm immediately combines all of the final states into a single partition, destroying the property that final states map to syntactic categories." (p. 57) [EaC] Always use the modified initial partition.

**Overlapping rules:** When two rules can match the same input (e.g., keyword `then` matches both the keyword rule and the identifier rule), the subset construction merges their final states. Resolution: "scanner generators let the compiler writer specify a precedence among syntactic categories." (p. 59) [EaC] In lex/flex, the first rule listed wins.

---

## 7. Keyword Handling Strategies

The book identifies two canonical approaches (p. 59–60, sidebar "Identifying Keywords"): [EaC]

### Strategy 1: Separate Rule per Keyword [EaC]

Add a distinct RE for each keyword (`if`, `while`, `then`, etc.) to the scanner specification.

**How it works:** The DFA handles keywords naturally — the accepting states for keywords are distinct from the accepting state for identifiers.

**Tradeoffs:**
- Pro: Uniform mechanism — keywords and other categories use the same final-state lookup
- Pro: No separate lookup phase
- Con: More states in the DFA (but still O(1) per character)
- Con: DFA regeneration required when keywords change

### Strategy 2: Identifier + Symbol Table Preloading [EaC]

Recognize all keywords as identifiers first, then check a symbol table.

**How it works:**
1. Scanner recognizes `then` as an identifier
2. Looks up `then` in the symbol table (which was preloaded with all keywords)
3. Returns the keyword category if found

> "If the compiler writer preloads the symbol table with the keywords and their syntactic categories, the scanner will find the keywords as previously seen and categorized identifiers, and will return the appropriate category for each." (p. 60) [EaC]

**Tradeoffs:**
- Pro: DFA unchanged when keywords change (just update the symbol table)
- Pro: Symbol table lookup doubles as the start of the compiler's identifier table
- Pro: Maps identifier names to small integers for efficient later comparison (§4.5)
- Con: Extra lookup per identifier token
- Con: More complex scanner-symbol table interface

### Strategy 3: Identifier + HashMap Lookup [CI — jlox]

CI's jlox uses this approach. A static `HashMap<String, TokenType>` is preloaded with all keywords. After scanning an identifier, the lexeme is looked up in the map:

```java
private static final Map<String, TokenType> keywords;
static {
    keywords = new HashMap<>();
    keywords.put("and", AND);
    keywords.put("class", CLASS);
    keywords.put("else", ELSE);
    // ... all keywords ...
    keywords.put("while", WHILE);
}

private void identifier() {
    while (isAlphaNumeric(peek())) advance();
    String text = source.substring(start, current);
    TokenType type = keywords.get(text);
    if (type == null) type = IDENTIFIER;
    addToken(type);
}
```

This is essentially EaC's Strategy 2 but without a full symbol table — just a keyword lookup table. CI favors it for jlox because it is straightforward in Java where `HashMap` is built-in.

### Strategy 4: Keyword Trie (Hand-Coded DFA) [CI — clox]

CI's clox avoids a hash table entirely (one is not yet built at the scanner stage) and instead implements a **hand-crafted trie** as nested switch statements:

```c
static TokenType identifierType() {
    switch (scanner.start[0]) {
        case 'a': return checkKeyword(1, 2, "nd", TOKEN_AND);
        case 'c': return checkKeyword(1, 4, "lass", TOKEN_CLASS);
        case 'f':
            if (scanner.current - scanner.start > 1) {
                switch (scanner.start[1]) {
                    case 'a': return checkKeyword(2, 3, "lse", TOKEN_FALSE);
                    case 'o': return checkKeyword(2, 1, "r", TOKEN_FOR);
                    case 'u': return checkKeyword(2, 1, "n", TOKEN_FUN);
                }
            }
            break;
        // ...
    }
    return TOKEN_IDENTIFIER;
}

static TokenType checkKeyword(int start, int length,
                               const char* rest, TokenType type) {
    if (scanner.current - scanner.start == start + length &&
        memcmp(scanner.start + start, rest, length) == 0) {
        return type;
    }
    return TOKEN_IDENTIFIER;
}
```

**Why this approach [CI]:** "A hash table would be overkill anyway. To look up a string in a hash table, we need to walk the string to calculate its hash code, find the corresponding bucket in the hash table, and then do a character-by-character equality comparison." The trie bails out immediately on any non-matching character — if the identifier starts with `g`, no keyword is possible without examining any further characters.

CI observes this is essentially a hand-encoded DFA: "Our keyword tree is exactly a DFA that recognizes Lox keywords."

### Which to Choose?

| Situation | Recommended Strategy |
|---|---|
| Language spec is stable, keywords are fixed | Separate rule per keyword (simpler pipeline) [EaC] |
| Language has many keywords (50+) | Symbol table preloading (avoids DFA explosion) [EaC] |
| Building a language workbench / extensible compiler | Symbol table preloading (keywords change at design time) [EaC] |
| Performance critical, symbol table lookup is expensive | Separate rule per keyword [EaC] |
| Hand-coded scanner in a high-level language, hash map available | HashMap lookup (CI Strategy 3) [CI] |
| Hand-coded scanner in C, no hash map, want minimum work | Trie as nested switches (CI Strategy 4) [CI] |

**Note on PL/I as a cautionary tale [EaC]:** PL/I has no reserved keywords — any identifier can be used as a keyword. "The fact that more recent languages abandoned the idea suggests that the complications outweighed any benefits of the extra flexibility." (p. 43) Do not design a language without reserved keywords.

---

## 8. Error Handling in Scanners

### What Constitutes a Lexical Error [EaC, CI]

**EaC's formulation:**
Two cases (p. 33):
1. Some character `xj` takes the FA to the error state `se` — the current prefix is not a valid prefix of any word
2. The FA reaches end-of-input while in a non-accepting state

**Important nuance [EaC]:** If the FA passed through an accepting state on the way to `se`, the input contains a valid word prefix. The scanner uses this to find word boundaries (the maximal munch behavior).

**CI's formulation:** "There are only a couple of errors that get detected during scanning: unterminated strings and unrecognized characters." CI adds a third category implicitly: unexpected end-of-file inside a string literal (the "Unterminated string." error).

### Error Recovery Approaches

**EaC options** (intentionally left vague in EaC: "we will leave this error path deliberately vague at this point," p. 32):

1. **Return an invalid token** — the code in Fig. 2.12/2.13 returns `invalid` when no accepting state is found; the parser then handles it
2. **Panic mode** — skip characters until a recognizable token boundary is found (e.g., whitespace)
3. **Character deletion** — remove the offending character and try again

**CI's jlox approach [CI]:** On encountering an unrecognized character, call the error reporter and then **continue scanning**. Do not stop. The erroneous character is consumed (via the earlier `advance()` call) to prevent an infinite loop, but the scanner keeps going to accumulate further errors:

> "Note also that we keep scanning. There may be other errors later in the program. It gives our users a better experience if we detect as many of those as possible in one go. Otherwise, they see one tiny error and fix it, only to have the next error appear, and so on. Syntax error Whac-A-Mole is no fun."

A `hadError` flag is set, ensuring that even though scanning continues, the resulting tokens will never be executed.

**CI's clox approach [CI]:** Rather than calling an error reporter from the scanner, clox returns a `TOKEN_ERROR` token containing the error message as its lexeme. The scanner always continues. The compiler is responsible for detecting `TOKEN_ERROR` tokens and initiating error recovery. This cleanly separates error detection from error reporting.

> "It's good engineering practice to separate the code that generates the errors from the code that reports them... Various phases of the front end will detect errors, but it's not really their job to know how to present that to a user. In a full-featured language implementation, you will likely have multiple ways errors get displayed: on stderr, in an IDE's error window, logged to a file, etc."

**Coalescing repeated errors [CI]:** CI notes a usability improvement it does not implement: "The code reports each invalid character separately, so this shotguns the user with a blast of errors if they accidentally paste a big blob of weird text. Coalescing a run of invalid characters into a single error would give a nicer user experience."

### The RollBack Error [EaC]

The double-buffer scheme has one explicit error: "if (Input = Fence) then signal roll back error" (Fig. 2.15, p. 71). This fires only if the scanner attempts to roll back farther than the buffer allows — i.e., beyond n characters from the current position. With n=4096, this indicates a pathological input (e.g., a 4096+ character token with no whitespace). [EaC]

### Reporting Position Information [EaC, CI]

The scanner is the only pass that sees every character, making it the natural place to track source position (line number, column number). [EaC]

**EaC:** Does not elaborate on this in Chapter 2, but the `InputPos` counter in §2.5.4 is the structural hook: maintain a parallel line/column counter that advances with `NextChar` and retreats with `RollBack`.

**CI:** Both implementations track only line numbers, incrementing a `line` counter when `\n` is consumed. CI is explicit that this is the minimum: "Even better would be the beginning and end column so they know where in the line. Even better than that is to show the user the offending line." CI acknowledges the practical constraint: "it's a lot of grungy string manipulation code."

CI also describes a lazy approach for column information: store only a byte offset in each token; convert to line/column only when an error actually needs to be displayed. "Most tokens never appear in an error message. For those, the less time you spend calculating position information ahead of time, the better."

---

## 9. Transition Table Compression [EaC]

### Why Compression Matters

An uncompressed DFA table has `|states| × |Σ|` entries. For a real programming language, `|Σ|` might be 128 (ASCII) or 256 (extended ASCII). A scanner with 50 states and ASCII input has 50 × 128 = 6,400 entries.

"When the table size grows larger than the size of the first-level cache, it may cause performance problems." (p. 72) [EaC]

### Column Merging (Character Classification) [EaC]

Observation: many characters cause identical transitions in every state (e.g., digits 1–9 all behave identically in an integer scanner). Identical columns can be merged.

**Implementation:** Add a `CharClass[char]` lookup that maps input characters to column indices. Replace `δ[state, char]` with `δ[state, CharClass[char]]`.

**Cost:** One extra memory load per character (`CharClass`), but `δ` becomes much smaller and more cache-friendly. This trade is almost always worth it.

> "It adds one memory reference from CharClass; in return, the scanner can use a much smaller table representation." (p. 73) [EaC]

**Algorithm** (Fig. 2.17): Compare all pairs of columns in δ. If `δ(k, i) = δ(k, j)` for all rows k, columns i and j are identical and one can be mapped to the other. O(|Σ|²·|states|) worst case, reduceable with population-count signatures. [EaC]

**Note on rows:** "The transition table for a programming language scanner often contains identical columns. (If the DFA is minimal, its rows cannot be identical.)" (p. 72) [EaC] — After minimization, row compression yields nothing. Column compression is where the gains are.

**CI's equivalent [CI]:** CI's hand-coded scanners achieve similar compression naturally. Instead of a `CharClass` table, CI uses helper predicates (`isDigit`, `isAlpha`, `isAlphaNumeric`) that classify characters at scan time:

```java
private boolean isDigit(char c)  { return c >= '0' && c <= '9'; }
private boolean isAlpha(char c)  { return (c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || c == '_'; }
```

CI explicitly avoids `Character.isDigit()` in Java because it accepts Unicode digits beyond ASCII, which is not desired for Lox. This is an important practical note: standard library character classification functions may include characters the language does not intend to allow.

### Row for Error State [EaC]

"The code skeletons in Sections 2.12 and 2.13 create one more opportunity: the row for `se` cannot be referenced and, thus, need not be represented." (p. 73) [EaC] Drop the error-state row to save `|Σ|` entries.

---

## 10. Whitespace and Newline Handling

### Standard Approach [EaC, CI]

**Default best practice [EaC]:** Blanks and tabs are significant word terminators (they do not appear in token REs), and the scanner discards them. "In most languages, whitespace has no intrinsic meaning. Scanners for these languages typically recognize and discard whitespace." (p. 61)

**CI's implementation [CI]:**

jlox handles whitespace in the main switch statement:
```java
case ' ':
case '\r':
case '\t':
    break;     // ignore, no token emitted
case '\n':
    line++;    // track line number, then ignore
    break;
```

clox extracts whitespace skipping into a dedicated `skipWhitespace()` function called at the top of each `scanToken()` invocation. This keeps the main token-dispatch switch uncluttered:
```c
static void skipWhitespace() {
    for (;;) {
        char c = peek();
        switch (c) {
            case ' ': case '\r': case '\t': advance(); break;
            case '\n': scanner.line++; advance(); break;
            case '/':
                if (peekNext() == '/') {
                    while (peek() != '\n' && !isAtEnd()) advance();
                } else { return; }
                break;
            default: return;
        }
    }
}
```

Note that clox includes comment-skipping inside `skipWhitespace()`, treating comments as structurally equivalent to whitespace.

### Extreme Cases [EaC, CI]

Two extremes to avoid:

1. **FORTRAN 66** (blanks insignificant): Requires unbounded lookahead to distinguish `do10i = 1,100` (loop) from `do10i = 1` (assignment). "Few, if any, languages have followed FORTRAN's example." (p. 61) [EaC]

2. **Python** (indentation significant): Requires special handling — add a rule recognizing end-of-line followed by blanks, test lexeme length, emit synthetic block-start/block-end tokens as needed. The scanner must track the previous indentation level. [EaC]

**CI's design note on implicit semicolons [CI]:** CI includes an extended analysis of languages that use newlines as statement terminators. Key observations:
- **Lua** ignores newlines entirely; grammar is designed to not require statement separators in most cases
- **Go** inserts a virtual semicolon after a newline if the preceding token is one of a specific set that can end a statement
- **Python** treats all newlines as significant unless inside brackets or with a trailing backslash
- **JavaScript** uses Automatic Semicolon Insertion (ASI), which CI characterizes as "a mess" — it parses without semicolons and inserts them retroactively when a parse error occurs

CI's recommendation: "If you're designing a new language, you almost surely should avoid an explicit statement terminator... Just make sure you pick a set of rules that make sense for your language's particular grammar and idioms. And don't do what JavaScript did."

---

## 11. Literal Handling in Hand-Coded Scanners [CI]

This section covers patterns CI demonstrates that have no direct EaC equivalent, since EaC focuses on the formal/automated pipeline.

### String Literals [CI]

Both jlox and clox handle string literals with the same basic pattern:
1. Match the opening `"`
2. Consume characters until the closing `"` or end-of-input
3. Track newlines inside the string to maintain accurate line counts
4. Report an error if end-of-input is reached before the closing `"`

jlox strips the surrounding quotes and stores the string value in the token:
```java
String value = source.substring(start + 1, current - 1);
addToken(STRING, value);
```

clox does not strip quotes or convert — it stores only the raw lexeme pointer and lets the compiler do the conversion later.

**Design note [CI]:** "For no particular reason, Lox supports multi-line strings. There are pros and cons to that, but prohibiting them was a little more complex than allowing them, so I left them in."

### Number Literals [CI]

Both implementations use two characters of lookahead to correctly handle decimal points. The `peekNext()` function prevents consuming a `.` that is not followed by a digit (which could be a method call like `123.toString()`):

```java
private void number() {
    while (isDigit(peek())) advance();
    if (peek() == '.' && isDigit(peekNext())) {
        advance();  // consume the '.'
        while (isDigit(peek())) advance();
    }
    addToken(NUMBER, Double.parseDouble(source.substring(start, current)));
}
```

**Design note [CI]:** CI deliberately does not treat `-123` as a number literal. The minus sign is handled as a prefix operator at the parsing stage. CI explains the edge case that motivates this: if `-123` were a literal, then `print -123.abs()` would behave differently from `print -n.abs()` where `n = 123`, because negation has lower precedence than method calls. "No matter what you do, some case ends up weird."

**Leading/trailing decimal points [CI]:** Lox disallows both `.1234` (leading dot) and `1234.` (trailing dot). The trailing dot restriction exists to support future method call syntax on numbers (`123.sqrt()`).

### Handling the `/` Character (Division vs. Comments) [CI]

CI uses the `match()` function to resolve the ambiguity between `/` (division) and `//` (comment start):

```java
case '/':
    if (match('/')) {
        while (peek() != '\n' && !isAtEnd()) advance();  // comment
    } else {
        addToken(SLASH);  // division operator
    }
    break;
```

This is the same pattern as handling `!=` vs `!` — look ahead one character, consume it if it matches, otherwise leave it for the next token. The key insight: the comment content is consumed but no token is emitted; `start` will be reset at the top of the next loop iteration and the comment's text disappears.

---

## 12. Practical Tradeoffs Summary

### The Fundamental Complexity Result [EaC]

**All three scanner implementations (table-driven, direct-coded, hand-coded) achieve O(1) time per character.** This is the critical invariant. The differences are in the constant.

> "The cost of operating an FA is proportional to the length of the input, not to the complexity of the RE or the number of states in the FA. More states need more space, but not more time." (p. 41) [EaC]

This means: invest in RE complexity for correctness/expressiveness without performance penalty. A scanner for 256-register names costs the same per character as one for 32-register names. Only space differs.

### Complexity vs. Specification Clarity Tradeoff [EaC]

The book shows two REs for 32-register names (§2.3.2):
1. Compact RE: `r ( [0...2] ( [0...9] | ε ) | [4...9] | ( 3 ( 0 | 1 | ε )) )` — correct, but "complex and counterintuitive"
2. Enumerated RE: `r0 | r1 | r2 | ... | r31` — "conceptually simpler, but much longer"

Both cost the same at runtime. "The resulting FA still requires one transition per input symbol." (p. 42) [EaC] Choose the version that is clearest to maintain, unless DFA state count becomes a practical problem.

**Third alternative noted by book [EaC]:** "use the RE `r[0...9][0...9]` and to test the register number as an integer." (p. 41) — push the validation into post-scan code. This is often the most maintainable approach when the constraint is numeric.

### When to Use a Scanner Generator vs. Hand-Code

| Factor | Favor Generator | Favor Hand-Code |
|---|---|---|
| Language stability | Unstable (changing) [EaC] | Stable [EaC] |
| Number of token types | Many (>20) [EaC] | Few [EaC] |
| Scanner-parser interface | Generic [EaC] | Custom, type-specific [EaC] |
| Error message quality | Acceptable [EaC] | Needs to be excellent [EaC] |
| Time to first working scanner | Tight deadline [EaC] | Can invest time [EaC] |
| Performance requirement | Moderate [EaC] | Maximum [EaC] |
| Educational / understanding goal | — | Hand-code (CI recommendation) [CI] |

### Simplicity as a Performance Strategy [CI]

CI closes its clox scanner chapter with an observation that complements EaC's formal analysis:

> "We sometimes fall into the trap of thinking that performance comes from complicated data structures, layers of caching, and other fancy optimizations. But, many times, all that's required is to do less work, and I often find that writing the simplest code I can is sufficient to accomplish that."

This applies directly to keyword recognition: the nested-switch trie does less work than a hash table lookup (bails out on first non-matching character), is simpler to write, and requires no additional data structures.

---

## 13. Common Mistakes and Pitfalls

1. **Using naive Hopcroft minimization with a multi-category scanner:** The default initial partition `{DA, D–DA}` merges all accepting states, destroying category information. Always initialize with one partition set per syntactic category. [EaC]

2. **Ignoring quadratic rollback potential:** Before shipping, audit your REs for patterns like `X | X*Y`. The basic skeleton scanner will exhibit O(n²) behavior. Use the maximal munch variant or restructure the RE. [EaC]

3. **Buffering too small for rollback:** If your language allows long tokens (e.g., very long string literals), ensure the double-buffer size n exceeds the maximum token length. With n=4096, tokens over 4096 characters trigger a rollback error. [EaC]

4. **Forgetting the Fence pointer:** Without Fence, `RollBack` can decrement into a buffer half that has been overwritten, returning garbage characters silently. [EaC]

5. **Applying Brzozowski's algorithm to a multi-category scanner without modification:** "Brzozowski's algorithm has no similar fix" (to Hopcroft's category-preserving modification). Use it per-rule, then combine DFAs manually. (p. 82) [EaC]

6. **Building a flat alternation RE for all categories at once:** `(r1 | r2 | ... | rk)` and then minimizing destroys category mappings. Build one NFA per rule, combine at the NFA level, then run the subset construction once. (p. 60) [EaC]

7. **Assuming separate keyword rules are always better:** For languages with 50+ keywords, a single identifier RE + symbol table preloading is simpler to maintain and scales better. [EaC]

8. **Omitting the `se` row from the table but referencing it:** The error-state optimization (omitting the se row) is valid only if the scanning loop exits before taking a transition from se. Verify the loop condition guards this. [EaC]

9. **Confusing "accepts empty string" behavior:** An RE like `[0...9]*` accepts the empty string (zero digits). If used as a token rule this causes the scanner to loop infinitely on non-digit input. Use `[0...9]+` for tokens that must be non-empty. [EaC]

10. **Treating DFA state count as a runtime cost:** More states = more memory, not more time. Do not sacrifice correctness or RE clarity to minimize state count. Minimize the DFA for cache reasons, not algorithmic complexity reasons. [EaC]

11. **Stopping on the first lexical error:** Report the error but continue scanning to accumulate all errors in a single pass. Stopping early forces users to fix errors one at a time. [CI]

12. **Calling error reporting directly from the scanner in a multi-target implementation:** Decouple detection from reporting. Use an `ErrorReporter` interface, a callback, or (in C) an error token type so the scanner is not tightly coupled to a specific output mechanism. [CI]

13. **Using `Character.isDigit()` (Java) or locale-aware classification functions:** These may accept Unicode digits (Devanagari numerals, full-width digits, etc.) that the language's lexical grammar does not intend to accept. Write explicit range comparisons (`c >= '0' && c <= '9'`) for language token rules. [CI]

14. **Scanning keywords character-by-character in a hand-coded scanner without fail-fast logic:** Checking `if (lexeme.equals("false"))` for every identifier scans the entire lexeme every time. A trie or nested-switch approach bails out as soon as a mismatch is found. [CI]

15. **Lifetime errors with pointer-based lexeme storage (C):** If clox-style tokens store `(char*, length)` pointers into the source buffer, the source buffer must outlive all tokens and all uses of their lexemes. Freeing the source string before the compiler finishes will produce silent use-after-free bugs. [CI]

---

## 14. Quick Reference: Key Algorithms

| Algorithm | Input | Output | Complexity | Use When |
|---|---|---|---|---|
| Thompson's Construction | RE | NFA | O(\|RE\|) states | Always, first step in automated pipeline [EaC] |
| Subset Construction | NFA | DFA | O(2^N) states worst case | Convert NFA to executable DFA [EaC] |
| Hopcroft's Minimization | DFA | Minimal DFA | O(\|N\| \|Σ\| log\|N\|) | Reduce DFA size for cache performance [EaC] |
| Brzozowski's Algorithm | NFA or DFA | Minimal DFA | O(2^N) worst case | Simpler implementation, acceptable in practice [EaC] |
| Closure-Free Incremental DFA | Word list | Acyclic DFA | O(total word length) | Keyword tables, hash function replacement [EaC] |
| Keyword Trie (nested switch) | Fixed keyword set | Token type | O(keyword length) worst case | Hand-coded C scanners, no hash table available [CI] |
| HashMap keyword lookup | Fixed keyword set + identifier lexeme | Token type | O(lexeme length) average | Hand-coded scanners in high-level languages [CI] |

---

## 15. CI Implementation Patterns Quick Reference [CI]

### jlox (Java) Scanner State

```java
private final String source;
private final List<Token> tokens = new ArrayList<>();
private int start = 0;    // start of current lexeme
private int current = 0;  // current character
private int line = 1;     // current line number
```

### clox (C) Scanner State

```c
typedef struct {
    const char* start;    // start of current lexeme
    const char* current;  // current character
    int line;
} Scanner;
```

### Fundamental Operations Comparison

| Operation | jlox (Java) | clox (C) |
|---|---|---|
| Advance | `source.charAt(current++)` | `scanner.current++; return scanner.current[-1];` |
| Peek (1 ahead) | `source.charAt(current)` | `*scanner.current` |
| Peek (2 ahead) | `source.charAt(current + 1)` | `scanner.current[1]` |
| Conditional consume | `match(char expected)` | `match(char expected)` (same logic) |
| Emit token | `tokens.add(new Token(...))` | `return makeToken(type);` |
| Emit error | `Lox.error(line, msg)` | `return errorToken(msg);` |
| Lexeme text | `source.substring(start, current)` | `(scanner.start, scanner.current - scanner.start)` |

---

*EaC content extracted from Engineering a Compiler Chapter 2. Page numbers refer to the 2023 (3rd edition) Elsevier printing.*
*CI content extracted from Crafting Interpreters, Chapters 4 and 16 (craftinginterpreters.com).*
