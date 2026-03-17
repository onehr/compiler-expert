# Semantic Analysis & Intermediate Representations
## Decision Frameworks, Tradeoffs, and Practical Guidelines
### Sources: Engineering a Compiler — Cooper & Torczon (3rd ed.) [EaC]; Crafting Interpreters — Robert Nystrom [CI]

---

## PART I: SEMANTIC ANALYSIS (Chapter 4 / Chapter 5 content)

---

### 1. Symbol Table Design — Scoping Strategies and Data Structures

#### The Core Problem
The compiler must map each textual name to a specific runtime entity. The decision is made at multiple levels: how to implement the lookup (mapping), how to store associated metadata (repository), and how to chain tables to reflect scope hierarchies. [EaC]

In a tree-walk interpreter, the symbol table takes the form of runtime environment objects (hashmaps) chained together as a linked list. The same structural principles apply, but the table is live at runtime rather than a purely compile-time construct. [CI]

#### Scoping Strategies

**Lexical Scoping** (dominant choice) [EaC, CI]
- Names are visible in the scope that declares them and all nested scopes
- At any point p, a reference to name n maps to the first declaration found by traversing from the current scope outward to the global scope
- Declarations in inner scopes shadow outer-scope declarations
- The compiler resolves all bindings at compile time; names map to static coordinates `<lexical_level, offset>` [EaC]
- CI's precise rule: "A variable usage refers to the preceding declaration with the same name in the innermost scope that encloses the expression where the variable is used." The word "preceding" means appearing before in program text; "innermost" handles shadowing. [CI]

**Dynamic Scoping** (legacy, mostly deprecated) [EaC]
- A free variable binds to the most recently created instance of that name at runtime
- Easy to implement in an interpreter (runtime name stack); harder to compile efficiently
- Creates bugs that are difficult to detect; avoid unless the source language requires it (e.g., Common Lisp's `special` declarations)

**The Mutable Environment Bug — A Critical Closure Pitfall** [CI]

Even with lexical scoping, a subtle bug arises in tree-walk interpreters that use mutable hashmaps for environments. If a block is represented by a single mutable environment object, a closure captures a *reference* to that mutable map. Any variable declared later in the same block is added to the same map, and the closure sees the new binding — violating the rule that a variable expression always refers to the same declaration.

Example: a function `showA()` closes over its enclosing block environment. If `var a = "block"` is declared in that same block *after* `showA` is defined, a naive interpreter will print "global" on the first call and "block" on the second, even though `a` was never reassigned. This is incorrect.

**Root cause**: the closure should capture a frozen snapshot of the environment at declaration time, not a live reference to a mutable map.

**Two resolution strategies** [CI]:
1. **Persistent environments**: each `var` declaration produces a new immutable environment object that extends the previous one. The closure retains the version that existed when it was created. This is the classic Scheme approach.
2. **Static resolver pass**: resolve all variable references statically before execution. Encode, for each variable use, how many environment hops away the declaration is. At runtime, walk exactly that many links — bypassing any variables declared later in the same logical block.

**Inheritance Hierarchies** (OOL-specific) [EaC]
- A second, orthogonal scope structure layered on top of lexical scoping
- Subclasses conceptually nest inside superclasses for name lookup purposes
- Search order: for an unqualified name in a method, check the lexical hierarchy first; then insert the class hierarchy at the appropriate point
- For qualified names (`obj.member`), resolve the object in the lexical hierarchy, then search the inheritance chain upward

**Closed vs. Open Class Structures** [EaC]
- Closed (C++): All class definitions are present at compile time and cannot change. The compiler fully resolves member names, types, and access methods. Prefer static dispatch; only use dynamic dispatch for explicitly declared `virtual` methods.
- Open (Java, Smalltalk): Class hierarchy can change at runtime (class loader). The compiler must either: rebuild method vectors after each class hierarchy change, defer resolution to a JIT, or rely on runtime name lookup with a method cache.

#### The "Sheaf of Tables" Model [EaC]

Build a distinct hash table for each scope. Link tables together into search paths that reflect the language-specified lookup order. For a method `m` in class `c`:
- The search path combines lexical nesting tables and the inheritance chain from `c` upward
- The link order is language-defined, but the underlying technology (linked tables) stays the same across languages

**Advantage**: Compact, separable, preservable for debuggers and profilers. Can model nested classes in Java, multiple inheritance in C++, and Scheme's `let` scopes all with the same mechanism.

**CI's runtime realization of this model**: In a tree-walk interpreter, each environment object holds a hashmap of `name → value` bindings and a pointer to its enclosing environment. This creates a linked list of hashmaps that is walked at runtime for variable lookup. The structure is identical in concept to the compile-time sheaf of tables, but traversed dynamically. [CI]

#### Data Structure Choices for the Mapping Function [EaC]

| Structure | Lookup Time | Insert | When to Use |
|-----------|-------------|--------|-------------|
| Linear list | O(n) | O(1) | Small scopes (few names), prototypes |
| Balanced BST | O(log n) | O(log n) | Medium scopes where worst-case matters more than constants |
| Unbalanced BST | O(log n) avg, O(n) worst | O(log n) avg | Rarely; risk of adversarial input degeneration |
| Hash map | O(1) expected | O(1) expected | General case; the standard choice |
| Static map (multiset discrimination) | O(1) guaranteed | offline | When you need guaranteed constant time and can afford a pre-scan |

**Hash map** is the practical default for symbol tables because O(1) expected performance handles typical procedure sizes well. The risk of worst-case O(n) behavior from hash collisions is mitigated by using a well-designed hash function. [EaC]

CI uses `HashMap<String, Object>` (Java) for each scope's environment at runtime, confirming hash maps as the natural choice even in a teaching implementation. [CI]

**Multiset discrimination** (Cai and Paige) [EaC]: An offline alternative. Pre-scan the entire token stream, collect all `<name, position>` tuples, sort them lexicographically. This groups all occurrences of each identifier. Assign symbol table indices before parsing begins. Benefits: guaranteed O(1) lookup; table size is known before parsing; no worst-case hash behavior. Cost: an extra pass over the token stream plus a lexicographic sort. Use when worst-case hash behavior is unacceptable.

#### Repository Design Principles [EaC]
- Storage should be contiguous or block-contiguous to improve locality and reduce allocation overhead
- The repository must contain enough information to rebuild the lookup structure (for table expansion and serialization to disk)
- Support changes to the search path as the parser enters/exits scopes
- Keep the index scheme independent of the mapping scheme so each can be expanded independently; ideally the map is sparse and the repository dense

#### IR References: Lexeme vs. Handle [EaC]
- Option A: Store the lexeme (text of the name) in the IR. Simple, but requires a symbol table lookup on every use.
- Option B: Store a handle (pointer or `<table, offset>` pair) to the symbol table entry. Enables direct access and inexpensive equality tests. This is the better choice for IR construction.

**CI's analogous choice**: The resolver stores resolution data (scope depth integer) in a side table keyed by the AST node object's identity. At runtime the interpreter performs a single map lookup on the AST node to retrieve the precomputed depth, then walks that exact number of environment links. This is the tree-walk equivalent of storing a handle instead of a lexeme. [CI]

---

### 2. Attribute Grammars — Synthesized vs. Inherited Attributes [EaC]

#### Synthesized Attributes
- Information flows **bottom-up** in the parse tree: from children to parent
- Natural in LR parsing; the yacc/bison notation models this directly
- Every `$$` value computed from `$1`, `$2`, ... is a synthesized attribute
- Well-suited for: computing expression types, building AST nodes, evaluating constants, emitting three-address code for expressions

**Examples**:
- Type of an expression: `F_+(type(left_operand), type(right_operand))` synthesized up to the `Expr` node
- Value of a positive integer: accumulated left-to-right as the parser reduces `DList → DList digit`

#### Inherited Attributes
- Information flows **top-down** in the parse tree: from parent or siblings to children
- Not directly supported by the bottom-up yacc paradigm
- Work-around: use a global variable (or the symbol table) to pass information "around the tree"

**Example of the work-around**: To propagate the declared type to all names in a declaration list, store the current type in a global variable `CurType` when the `TypeSpec` production reduces. Each name in `NameList` reads `CurType` to obtain its type.

**Alternative**: Restructure the grammar so that the type information is encoded structurally. E.g., replace `Declaration → TypeSpec NameList` with separate `Declaration → int INameList | float FNameList`. This eliminates the global variable but increases grammar size.

#### When to Use Each

- **Synthesized**: prefer whenever possible. They compose naturally with bottom-up parsing, are easier to reason about, and require no extra machinery.
- **Inherited / global communication**: use when the computation genuinely requires information from a surrounding context that cannot be restructured into the grammar. Symbol tables are the canonical solution — they serve as a sanctioned form of nonlocal communication between productions.
- **Attribute grammar systems** (formal): Can handle both synthesized and inherited attributes in a declarative framework. Theoretically cleaner but less practical — not widely adopted due to the lack of well-implemented, easy-to-use tools. yacc/bison won despite being more ad hoc because they were widely distributed and worked.

---

### 3. Type Checking Approaches

#### Static vs. Dynamic Type Checking

**Static (compile-time)** [EaC]:
- The compiler assigns a type to each name and expression from declarations and inference rules
- No runtime tags or case logic needed for type-correct code
- Enables the compiler to select efficient, specialized operations (e.g., integer multiply vs. floating-point multiply)
- Prerequisites: mandatory declarations (or strong inference rules), all called procedures visible with type signatures at compile time

**Dynamic (runtime)** [EaC, CI]:
- Each value carries a runtime tag encoding its type
- Every operation must check tags, select the appropriate variant, perform the operation, and update the tag
- The cost is paid on every execution of the operation, not just once at compile time
- Necessary for dynamically typed languages (Python, Lisp, Smalltalk, APL, Lox) where a variable's type can change between assignments

**The compile-time cost of static checking is paid once; the runtime cost of dynamic checking is paid on every execution.** This is the fundamental asymmetry that motivates static type systems. [EaC]

**CI's practical observation on the static/dynamic tradeoff** [CI]:

The spectrum between fully static and fully dynamic is not binary. Even statically typed languages defer some checks to runtime:
- Java casts: the static type system presumes casts succeed; the JVM inserts a runtime check that throws `ClassCastException` on failure.
- Covariant arrays in Java/C#: storing a `String` into an `Integer[]` variable typed as `Object[]` compiles cleanly but throws `ArrayStoreException` at runtime.

The practical design principle: a language designer can grant users flexibility (such as covariant array subtyping) by deferring specific checks to runtime, without sacrificing the overall soundness of the static type system — at the cost of some runtime overhead and reduced static guarantees.

Conversely, deferring *too many* checks erodes the user's confidence that the static type system catches errors before deployment. The choice of which checks to defer is a deliberate language design decision, not a binary static/dynamic split.

**CI's implementation of dynamic type checking in a tree-walk interpreter** [CI]:

In a dynamically typed tree-walk interpreter, type checking occurs at the point of each operation:
1. Evaluate both operands.
2. Use `instanceof` (or equivalent runtime type test) to determine the actual types.
3. Select the appropriate operation or throw a `RuntimeError` with a source location.

This `instanceof`-based dispatch is exactly the "check tag" operation described in EaC's dynamic type checking model, implemented at the Java level rather than machine level. Example from Lox's `+` operator:
```java
if (left instanceof Double && right instanceof Double)   return (double)left + (double)right;
if (left instanceof String && right instanceof String)   return (String)left + (String)right;
throw new RuntimeError(operator, "Operands must be two numbers or two strings.");
```

Runtime errors must: (a) report the source location (token), (b) unwind the evaluator without crashing the host process, and (c) allow the REPL to continue. Using exceptions for this unwind is idiomatic in tree-walk interpreters.

**Truthiness as implicit boolean coercion** [CI]: Dynamically typed languages must define what counts as "truthy" for non-boolean values. Lox follows Ruby: `false` and `nil` are falsey; everything else is truthy. JavaScript uses a more complex rule (empty string is falsey, `0` is falsey, etc.). This design choice is part of the language specification, not the compiler — but the interpreter must faithfully implement it.

#### Type Inference Strategies [EaC]

**Declarations-first (declare-before-use)**:
- Parse declarations first; populate the symbol table; then process executable statements with full type information
- Enables direct emission of low-level IR in a single pass
- Languages: Pascal, C (mostly), FORTRAN

**Declarations optional, consistent-use inference**:
- Assign a general type to each name; narrow it by examining usage contexts
- Use an iterative fixed-point algorithm over a type lattice if all functions have constant return types
- Requires multiple passes: build abstract IR first, then refine types, then lower to concrete IR
- Languages: Python (to some extent), early FORTRAN (implicit typing by first letter)

**Parametric polymorphism (Hindley-Milner)**:
- Functions like `filter: (α→β) × list(α) → list(β)` have return types that are functions of argument types
- A simple fixed-point over types is insufficient; requires solving equations over the space of types
- Classical approach: unification
- Languages: ML, Haskell, modern Scheme

**Dynamically changing types**:
- Variable's type can change during execution (APL: `a × b` can multiply integers one call and float arrays the next)
- Compiler approach: fall back on interpretation with tagged values; apply check elimination and check motion to remove redundant runtime checks
- JIT compilers can specialize based on observed types at hot call sites

#### Type Equivalence Decisions (Language Designer's Choice, not Compiler Writer's) [EaC]

- **Name equivalence**: two types are equivalent iff the programmer calls them by the same name. `Tree` and `Bush` (structurally identical C structs) are distinct types. Assumes naming is intentional.
- **Structural equivalence**: two types are equivalent iff they have the same structure. `Tree` and `Bush` are equivalent. Assumes structure is what matters.
- The compiler must implement whichever rule the language specifies. Both have been used in production languages.

#### Implicit Conversions (Coercion) [EaC]

Two strategies for handling mixed-type expressions:
1. **Insert explicit conversions early**: apply the language's conversion table during translation; expect optimization to reduce redundant conversions. The IR is cluttered but complete.
2. **Keep conversions implicit until late**: reduce IR clutter; elaborate conversions during a later pass after types are fully resolved.

C/C++ widening rule: convert to the narrowest type that can represent both operands. This is a set of rules the compiler must encode as a lookup table per operator.

---

### 4. Two-Pass vs. Single-Pass Analysis

#### Single-Pass Compilers [EaC]
- Emit assembly or machine code directly during parsing, in one pass over the source
- Historically important when machines were slow and memory was scarce
- Languages designed for single-pass compilation impose constraints: Pascal requires all declarations before any executable statement (so types and storage layout can be resolved before code emission)
- **The "declare before use" rule is essentially a language design constraint to enable single-pass compilation**

**Limitations**:
- Can only use information that has been seen so far in the input
- Cannot optimize across forward references
- Difficult to emit efficient immediate-operand variants when the operand's nature (constant vs. variable) is not yet known at the point of emission
- Floyd (1961): multiple passes produce more efficient code. This observation fundamentally justifies multi-pass compilers.

#### Multi-Pass / Progressive Translation (Modern Practice) [EaC]

**Progressive translation**: the front end builds an initial, partially abstract IR; later passes refine it as more information becomes available.

**When multiple passes are required**:
1. Language requires declarations but allows them in arbitrary order: build an abstract IR with symbolic references; perform type inference and storage layout after parsing; then refine the IR.
2. Language does not require declarations: must build an abstract IR; perform iterative (possibly complex) type inference; then lower to concrete IR.
3. Language requires declarations before use: can perform layout and full type checking inline during parsing; can emit relatively concrete IR in a single pass.

**The choice depends on the source language rules.** If the language rules guarantee that all needed information is available at each point of use during parsing, single-pass is feasible. Otherwise, multiple passes are necessary for correctness or efficiency.

**In almost all modern compilers**, multiple passes are performed regardless, because optimization requires information that only becomes available after the initial IR is complete (e.g., liveness, dataflow).

#### The Separate Resolver Pass — CI's Design Choice [CI]

For a tree-walk interpreter, CI introduces a dedicated **resolver** pass between parsing and execution. This is a single, O(n) walk over the AST that:
- Resolves all variable references to their declaring scope (computing "environment hop distance")
- Detects static errors: use of a variable in its own initializer, duplicate declarations in a local scope, `return` outside a function
- Stores resolution data in a side table (`Map<Expr, Integer> locals`) keyed by AST node identity

**Why a separate pass rather than resolution during parsing or execution?**
- Parsing: feasible (this is what clox does), but complicates the parser
- Execution (dynamic, per-use): correct for pure lexical scoping but slow (re-resolves on each loop iteration) and exposes the mutable-environment closure bug
- Separate pass: cleanest separation of concerns; resolution runs once before execution; natural insertion point for additional static analyses

**Static analysis vs. dynamic execution — key behavioral differences** [CI]:
- No side effects: visiting a print statement does not print anything
- No control flow: loops are visited once; both branches of an `if` are always visited (conservative); logical operators are not short-circuited
- This conservatism is intentional: any branch that *could* execute at runtime must be analyzed statically

**Multiple analyses in one pass vs. separate passes** [CI]: Bundling multiple analyses into a single tree walk is faster (avoids repeated traversal overhead) but increases complexity. The tradeoff mirrors EaC's multi-pass vs. single-pass discussion: favor separate passes for maintainability, single pass for performance.

---

### 5. Name Resolution — Forward References and Mutual Recursion

#### The Forward Reference Problem [EaC]
A forward reference occurs when code uses a name before its declaration. In a single-pass model, this is a problem because the compiler has not yet seen the name's properties.

**Language-level solutions**:
- Pascal: forbid forward references by requiring declarations before use. Procedures can be forward-declared with `forward` to permit mutual recursion.
- C: requires function prototypes (type signatures) for externally defined functions and for forward-called functions. The prototype provides the type signature without the body.
- Fortran: implicit typing allows use before declaration; type is inferred from the first letter of the name.

**Compiler-level solutions**:
- Build an abstract IR with symbolic references; resolve names in a subsequent pass over the complete symbol table.
- For separate compilation: require a type signature (prototype) for each externally defined name. Defer full checking to link time or load time.

#### Mutual Recursion [EaC]
Two or more procedures that call each other (A calls B, B calls A) require that both names be visible in each other's scope at the time of the call.

**Solutions**:
- Two-pass approach: first pass scans all declarations and builds the symbol table; second pass translates procedure bodies with full name visibility.
- Forward declarations: allow the programmer to declare a procedure's signature before its body (Pascal `forward`, C function prototypes). The compiler creates the symbol table entry on the forward declaration; fills in the body later.
- In a multi-pass compiler with an abstract IR, this is handled naturally: all names are resolved in a second pass after the complete IR is built.

**Interprocedural type inference** (for calls): The compiler needs a type signature for every called function. Options:
1. Require all code to be present for compilation (no separate compilation)
2. Require type signatures (function prototypes) for each function
3. Defer type checking to link time or runtime
4. Embed the compiler in a development environment that gathers type information from related compilations

#### Function Name Eagerness vs. Variable Initializer Safety [CI]

In CI's resolver, variable declarations follow a two-step protocol:
1. **Declare**: add the name to the current scope, marked as "not yet initialized" (value `false` in the scope map)
2. **Resolve the initializer** while the variable is declared but not yet defined — so that `var a = a;` can be caught as an error
3. **Define**: mark the name as fully initialized (value `true`)

This catches the use-before-initialization pattern at compile time rather than producing a confusing runtime result.

**Function declarations are handled differently**: the function name is both declared and defined *before* resolving the function body. This intentional asymmetry allows a function to call itself recursively by name inside its own body, which is a well-defined and common pattern. Variables do not get this treatment because self-reference in a variable initializer is almost always a bug.

---

### 6. Variable Resolution — Static Scope Depth Encoding [CI]

This section describes the pattern introduced by CI for efficiently and correctly implementing lexical scoping in a tree-walk interpreter.

#### The Core Technique

At resolve time, for each variable use expression, compute the **scope depth**: the number of environment hops from the current environment to the environment that contains the variable's declaration. Store this as a `(expr_node → depth)` mapping.

At runtime, instead of searching the environment chain by name:
```java
// Instead of dynamic walk (slow, and has the mutable-environment bug):
environment.get(name)  // walks until found

// Use static depth (fast, correct):
environment.getAt(depth, name)  // walks exactly `depth` hops, then looks up
```

The `ancestor(depth)` helper walks exactly `depth` hops up the enclosing chain and returns that environment. Since the resolver already confirmed the variable exists there, no "not found" check is needed at runtime.

#### Global Variables — A Special Case [CI]

Global variables are intentionally excluded from the resolver's scope stack. When the resolver cannot find a variable in any local scope, it leaves the use unresolved. At runtime, unresolved variables are looked up directly and dynamically in the global environment. This is correct because:
- Globals in Lox (and similar languages) are more dynamic: they can be defined at any point in a REPL session
- Applying static resolution to globals would prevent REPL-style incremental definition

This creates a two-tier lookup scheme: static depth-encoded access for locals, dynamic name-based lookup for globals.

#### Storing Resolution Data — Side Table vs. Annotating the AST [CI]

Two options for storing the resolver's output:
1. **Annotate AST nodes directly**: add a `depth` field to each variable expression node. Simple, but requires modifying the AST data structures.
2. **Side table** (`Map<Expr, Integer>`): keyed by AST node object identity (Java reference equality). Does not modify existing classes; easy to discard (clear the map).

CI chooses the side table for its tree-walk interpreter. The same data could also be stored inline in the clox bytecode as an operand to the variable-access instruction, which is more efficient for a bytecode VM.

#### Resolver Error Detection [CI]

The resolver pass is the natural home for static errors that cannot be caught during parsing:
- **Use in own initializer**: if a variable name appears in its own initializer expression, and the variable is currently declared but not yet defined (value = `false`), report a compile error
- **Duplicate declaration in local scope**: if a `var` statement declares a name that already exists in the current (innermost local) scope map, report a compile error (note: duplicate declarations at global scope are allowed for REPL convenience)
- **Return outside function**: track a `currentFunction` context variable through the resolution walk; report an error if a `return` statement is visited while `currentFunction == NONE`

These checks exemplify what CI calls the **semantic analysis** phase: analysis that goes beyond syntactic well-formedness to determine what parts of the program mean and whether those meanings are valid.

---

## PART II: INTERMEDIATE REPRESENTATIONS (Chapter 4 / 5 content)

---

### 1. AST vs. DAG vs. Linear IR — When to Use Each

#### Abstract Syntax Tree (AST) [EaC]

**Structure**: Tree. Each node represents an operator; children represent operands. Interior nodes have implicit names (the subtree they root). Leaves are named values.

**Characteristics**:
- Direct, obvious correspondence to the grammatical structure of the input
- Compact representation of source-level structure
- Interior nodes lack explicit names; values are anonymous
- Naturally represents expressions as trees rather than directed acyclic graphs (each occurrence of the same subexpression creates a separate subtree)

**When to use**:
- Source-to-source translation systems (the AST preserves source structure)
- Early stages of compilation where the goal is to capture source meaning, not expose detail
- Type checking and semantic analysis (the symbol table provides supplementary information about names)
- When the compiler writer wants to defer lowering decisions to later passes

**Limitations**:
- Does not expose sharing of common subexpressions
- Implicit names for intermediate values limit analysis and optimization
- Must be lowered to a linear form for most back-end work

**CI's use of AST as the primary IR** [CI]: In CI's tree-walk interpreter (jlox), the AST is not only the front-end representation — it *is* the executable form. The interpreter implements a Visitor pattern over the AST, evaluating each node recursively in a post-order traversal (children evaluated before parents). This avoids the need for any explicit lowering step and demonstrates that the AST is a fully viable execution IR for interpreted languages, at the cost of performance relative to bytecode.

#### DAG (Directed Acyclic Graph) [EaC]

**Structure**: Like an AST but nodes are shared when they represent the same computation. An expression like `a * b + a * b` has two leaves for `a`, two for `b`, one shared node for `a * b`, and one node for the addition.

**Characteristics**:
- Explicitly captures sharing/common subexpressions
- More compact than the equivalent AST when subexpressions repeat
- Still a graphical, tree-like IR

**When to use**:
- When common subexpression elimination is a primary goal early in compilation
- As part of value numbering within a basic block
- In source-to-source translators that want to compress redundant computations

**Key decision**: converting an expression tree to a DAG requires hashing subexpressions (Ershov's algorithm, dating to the 1950s; popularized in Lisp systems in the early 1970s).

#### Linear IR (Three-Address Code / Quadruples / ILOC) [EaC]

**Structure**: A sequence of instructions, each of the form `op src1, src2 => dst`. Every intermediate value has an explicit name (virtual register). All values are named.

**Characteristics**:
- All names are explicit; no implicit intermediate values
- Close to machine code; easy to emit for most ISAs
- Supports the full range of naming disciplines (from name-reuse to SSA)
- φ-functions for SSA do not fit the fixed-arity three-address scheme; need a side structure for variable-arity operands

**When to use**:
- General-purpose IR for optimization and code generation
- When the compiler needs to analyze value lifetimes and dataflow
- When the target ISA is a register-to-register RISC machine
- As the definitive IR in most production optimizing compilers

#### Stack-Machine Code (Linear, Implicit Names) [EaC]

**Structure**: Operations implicitly consume operands from and push results onto a stack. Names for intermediate values are implicit in the stack position.

**When to use**:
- When the target ISA is a stack machine (Java Virtual Machine bytecode, old Forth systems)
- As a compact encoding; bytecode for JVMs is notably compact
- The Java HotSpot server compiler translates JVM bytecode into a graphical IR for optimization, then generates RISC-like code — illustrating that stack IR is an input format, not an optimization IR

**CI's bytecode VM (clox) also uses a stack-based IR** [CI]: In Part III of CI, the author designs a chunk-based bytecode with a stack-based virtual machine. This is a direct instantiation of EaC's stack-machine model, chosen for its compactness and ease of code generation from expressions.

#### CFG (Control-Flow Graph) — A Meta-IR [EaC]

The CFG is not an alternative to the above but a structural overlay that organizes any linear IR into basic blocks connected by edges. It is discussed separately below (Section 4).

#### Summary Decision Table [EaC]

| IR Form | Naming | Sharing | Proximity to Source | Proximity to Machine | Primary Use |
|---------|--------|---------|--------------------|--------------------|-------------|
| AST | Implicit for intermediates | No | High | Low | Parsing, type checking, source-to-source, tree-walk interpretation [EaC, CI] |
| DAG | Implicit with sharing | Yes | High | Low | Local CSE, value numbering |
| Linear (TAC) | Explicit, all values | No (by default) | Low | High | Optimization, code generation |
| Linear (SSA) | Explicit, unique per def | No (value flow encoded) | Low | Medium | Analysis, optimization |
| Stack code | Implicit (stack position) | No | Medium | Medium (for stack ISAs) | JVM bytecode, compact encoding, bytecode VMs [EaC, CI] |

**Cooper & Torczon note**: "None of the intermediate forms described are definitively the right answer for all compilers or all tasks. Many compilers use multiple IRs — building a second or third one for a particular analysis, then modifying the original." [EaC]

---

### 2. Three-Address Code (TAC) vs. SSA Form [EaC]

#### Three-Address Code

Each operation has the form `op r1, r2 => r3`. A name may be defined multiple times in the procedure. The same virtual register can be the target of several assignments. Example: a loop variable `i` gets a new value on each iteration, but the compiler may use the same name `r5` for each assignment.

**Consequences**:
- Analyzing value flow requires tracking which definition of a name reaches a given use (reaching definitions analysis)
- Optimization algorithms must reason about all possible definitions of a name that could reach each use
- Data structures (like live sets) scale with the number of names; name reuse keeps name spaces small but obscures value flow

#### Static Single-Assignment (SSA) Form

**Definition**: An IR is in SSA form iff:
1. Each definition has a distinct name (subscripted: `x0`, `x1`, `x2`, ...)
2. Each use refers to a single definition

φ-functions are inserted at **join points** (blocks with multiple CFG predecessors) to merge values arriving from different paths. φ-functions select their operand based on which predecessor edge was taken to enter the block. All φ-functions in a block execute in parallel on entry.

**Key properties of SSA**:
- The name of a value encodes where it was defined; a use of `x3` always refers to the unique definition of `x3`
- The compiler can ignore the lifetimes of most values when reasoning about optimizations, because names are not redefined
- The value of a name is available along any path from its definition forward, without further analysis

**When SSA pays off**:
- **Constant propagation**: if `x0 = 5`, then every use of `x0` can be replaced by `5` without tracking where the definition reaches
- **Dead code elimination**: a definition with no uses (in SSA, this is unambiguous) can be removed
- **Copy propagation**: if `y1 = x3`, then uses of `y1` can be replaced by `x3` directly
- **Global value numbering**: because each name has one definition, hash-based value numbering extends naturally from within a block to across blocks
- Any analysis that reasons about the uniqueness of values: in TAC, you must compute reaching definitions first; in SSA, it is built into the IR

**Costs of SSA**:
- φ-functions are a pseudo-operation with unusual parallel semantics; they must be lowered to copies before code generation
- Translating SSA back to executable code (out-of-SSA) is non-trivial; it requires careful handling of the parallel semantics of φ-functions (see lost-copy and swap problems, Section 9.3.5)
- φ-functions have variable numbers of operands; they don't fit the fixed-arity three-address scheme; a side data structure is needed for their operands
- Building SSA has a non-trivial construction cost (dominance frontiers, φ-function placement)

**TAC vs. SSA: the practical decision**:
- Use SSA form if the compiler performs significant global optimization (SSA simplifies almost all optimization algorithms)
- Use TAC (without SSA) for simple compilers that perform only local optimization, or when SSA construction cost is not justified by the optimization benefit
- Most modern production compilers (GCC, LLVM, JVM HotSpot server) build SSA internally for the optimization phase

**Name space size**: the historic FORTRAN experiment in the book illustrates the tension. Completely unique names (one per computation): 985 names for a 210-line SVD routine. Maximum name reuse: ~60 names, but 60% degradation in optimization quality. A disciplined middle ground (unique names per expression, SSA-like rules): ~90 names with full optimization benefit.

---

### 3. High-Level vs. Low-Level IR [EaC]

#### High-Level IR (Near-Source)

**Characteristics**:
- Preserves source-level constructs (array references, structure accesses, procedure calls at source level)
- Uses abstract operations like `load a[i,j]` rather than the address arithmetic that implements it
- Essential information about the program's intent is captured; implementation details are deferred
- The symbol table provides supplementary information (dimensions, lower bounds, element type, etc.)

**When to use**:
- In the early front-end passes where source structure must be preserved for analysis
- For high-level optimizations that depend on source-level semantics (e.g., array dependence analysis, loop transformations, loop-invariant code motion on array references)
- For source-to-source translation tools
- When source-level intent (e.g., "this is an array reference") helps guide optimization

#### Low-Level IR (Near-Machine)

**Characteristics**:
- All address computations are explicit (the address formula for `a[i,j]` is expanded into a sequence of multiplications, additions, and loads)
- Every intermediate value has an explicit name
- Operations map closely to machine instructions
- Information about source-level structure may be lost (the compiler can no longer easily recognize that several operations together constitute an array access)

**When to use**:
- For register allocation, instruction selection, instruction scheduling
- For optimizations that reason about individual memory operations and register values
- When the target ISA details are needed to make correct decisions (e.g., whether an immediate field is large enough to hold an offset)

#### The Progressive Lowering Strategy [EaC]

The practical approach: start with a high-level (near-source) IR for front-end analysis, then progressively lower to machine-level IR as more information is gathered. The compiler can add detail incrementally:
- Initial IR: `load a[i,j]` (abstract, with symbol table reference)
- After storage layout: `load @a + (i-1)*10 + (j-1)` (with concrete dimensions)
- After instruction selection: `loadAI rarp, @offset` (register-based, concrete)

The array reference example from the book makes this concrete: the near-source AST preserves the intent (`a[i,j]`); the low-level tree exposes 7 distinct subexpressions in the address computation. Neither is universally better — it depends on what subsequent phases need.

#### Memory Model Choice Interacts with IR Level [EaC]

Three memory models shape the IR:
1. **Memory-to-memory**: all values have a home in memory; the IR has explicit loads/stores around every operation. Unoptimized code uses few registers. Optimization focuses on promoting values into registers.
2. **Register-to-register**: all unambiguous scalar values are kept in virtual registers; only ambiguous values and structures go to memory. Unoptimized code may use far more virtual registers than physical registers exist. Register allocation is critical and correctness-essential.
3. **Stack model**: values live on an explicit stack in the IR. If the ISA is a stack machine (JVM), this is efficient directly. If the ISA is a RISC/CISC machine, the stack IR must be translated to register-based code before code generation.

**The choice of memory model has a strong influence on the design of the optimizer and back end.** Once chosen, it pervades every IR design decision.

---

### 4. CFG Construction — Basic Blocks and Edge Types [EaC]

#### Basic Blocks

A basic block is a maximal sequence of consecutive instructions with the property that:
- Execution enters at the first instruction only (no jump targets in the middle)
- Execution exits only from the last instruction (no internal branches)

**Finding leaders** (the standard algorithm):
1. The first instruction is a leader
2. Any instruction that is a branch target is a leader
3. Any instruction immediately following a branch is a leader

A basic block consists of a leader and all subsequent instructions up to (but not including) the next leader.

**Why basic blocks matter**: within a basic block, control flows sequentially and predictably. Local optimizations (local value numbering, algebraic simplification, peephole) apply within a block without tracking control flow. Cross-block optimization requires the CFG.

#### CFG Construction from Linear IR

Given basic blocks, build edges:
- If a block ends in a conditional branch `cbr r, L1, L2`: add edges to blocks starting at L1 and L2
- If a block ends in an unconditional branch `br L`: add edge to the block starting at L
- If a block falls through (the last instruction is neither a branch nor a return): add edge to the next block in the linear order

Note: the standard EAC algorithm assumes conditional branches specify **both** the taken and fall-through targets. If only the taken target is specified, `FindLeaders` and `BuildGraph` must be adapted accordingly.

#### Edge Types in the CFG

When the CFG is analyzed with a depth-first spanning tree (DFS), edges fall into four categories:
1. **Tree edges**: edges in the DFS spanning tree; always forward in execution time
2. **Forward edges**: edges from a DFS node to a descendant that is not a direct child
3. **Cross edges**: edges between nodes in different subtrees of the DFS tree
4. **Back edges**: edges from a DFS node to an ancestor in the DFS tree — these identify loops

**Back edges are critical**: a back edge from block `b` to block `h` means that `h` dominates `b` (in reducible CFGs) and that `h` is the header of a loop. Identifying back edges is the first step in loop analysis.

#### Reducibility

Most programs produce **reducible CFGs**: CFGs where all loops can be identified by back edges in any DFS, and back edges go from a node to its dominator. Reducible CFGs support many classical optimizations more efficiently. Irreducible CFGs (which can arise from arbitrary `goto` statements or certain translations) require more complex treatment.

---

### 5. SSA Construction — When to Build SSA and the φ-Function Placement Problem [EaC]

#### When to Build SSA

SSA is built after the initial linear IR and CFG are available. It is:
- Not built during initial translation (too early; the CFG is needed for φ-function placement)
- Built before the bulk of global optimization passes
- Destroyed (out-of-SSA translation) before register allocation, because SSA's parallel φ-function semantics are not compatible with sequential execution

The trigger is a global optimization phase. If the compiler only does local optimization (within basic blocks), SSA is not needed. SSA pays off for constant propagation, dead code elimination, global value numbering, loop-invariant code motion, and most other global optimizations.

#### The φ-Function Placement Problem

**Naive approach**: insert a φ-function for every variable at every block with multiple CFG predecessors. This is correct but inserts many unnecessary φ-functions (ones that merge identical values, or for variables that are dead at that point).

**Better approach**: use the **dominance frontier** (DF) to identify the minimal set of insertion points.

Intuition: if a variable `x` is defined in block `d`, and there is a path from `d` to block `j` through some block `b` where `d` does not dominate `b`... then `j` is in the dominance frontier of `d`, and a φ-function for `x` is needed at `j`.

The dominance frontier of a block `d` is the set of blocks `j` such that `d` dominates a predecessor of `j` but `d` does not strictly dominate `j`. Placing φ-functions at the dominance frontier of every block that defines a variable produces **minimal SSA form**.

**Pruned SSA**: place φ-functions only where a variable is live (using liveness information). Fewer φ-functions, but requires liveness as a prerequisite.

**Semi-pruned SSA**: eliminate φ-functions for names that are never live on entry to any block (names that are local to a single block). A compromise between minimal and pruned.

#### The Construction Algorithm (Sketch)

Phase 1 — Insert φ-functions:
1. For each variable `v` that is defined in block `b`, add `v` to the set of variables defined in `b`
2. Use the DF to compute, for each definition of `v`, all blocks where a φ-function for `v` is needed
3. Insert the φ-function at each such block (a φ-function for a block with `k` predecessors has `k` operands)

Phase 2 — Rename variables:
1. Process blocks in depth-first order
2. For each definition of variable `v`: increment a counter; rename the definition as `v_k`
3. Rewrite all uses of `v` in the current block with the current subscript
4. At the end of a block, update the appropriate φ-function operands in each successor
5. When recursing into a dominator-tree child: push the current renaming; on return, pop it

**The simple approach** (from the book): insert φ-functions at every multi-predecessor block for every virtual register. This inserts too many φ-functions but simplifies the implementation. The full algorithm eliminates extraneous φ-functions during or after construction.

#### φ-Function Semantics — A Critical Detail

All φ-functions at the top of a block execute **in parallel on entry**:
- All φ-functions read their argument values first (in parallel)
- All φ-functions write their result values second (in parallel)

This parallel semantics allows SSA algorithms to ignore the ordering of φ-functions within a block. But it complicates out-of-SSA translation: naively inserting a copy for each φ-function operand on the incoming edge can produce incorrect code when two φ-functions in the same block have each other as operands (the "lost-copy problem" and the "swap problem" in Section 9.3.5).

#### Variable Arity of φ-Functions

A φ-function at a block with `k` predecessors needs `k` operands. This violates the fixed-arity assumption of three-address code. Solutions:
- In an array representation: use a side data structure to hold the φ-function operands (the array stores an index into the side structure)
- In a linked-list or struct representation: use variable-size tuples; the φ-function tuple holds an operand count and a variable-length list of `<value, predecessor_edge>` pairs

---

## Cross-Cutting Tradeoffs

### IR Name Space Design [EaC]

The naming discipline directly affects optimization quality. Key insight from the FORTRAN experiment:

- Maximum unique names (one per computation): large name space, slow compilation, but full optimization visibility
- Maximum name reuse: small name space, fast compilation, but degraded optimization quality
- Disciplined middle ground (one name per distinct expression value): ~same optimization quality as maximum unique names, much smaller name space

Rules for the middle-ground discipline:
1. Each textual expression (e.g., `r17 + r21`) targets the same register wherever it appears (hash-based)
2. In `op ri, rj => rk`, choose `k` such that `i, j < k` (value flow direction encoded in name ordering)
3. Register copies (`ri => rj`) only have `i > j` when `rj` corresponds to a declared variable
4. Each store to memory is followed by a copy of the stored value into the variable's named register

SSA is the modern evolution of this approach: it achieves full optimization visibility with a principled, algorithm-based naming scheme rather than ad-hoc rules.

### Environment / Scope Chain Design Patterns [CI]

For tree-walk interpreters and language runtimes that represent scope chains as linked data structures, CI establishes several design patterns:

**Linked list of hashmaps**: each scope is a hashmap of `name → value`; each environment object holds a pointer to its enclosing environment. Variable lookup walks the chain until the name is found or the chain is exhausted (global not found = runtime error).

**Performance problem with naive lookup**: re-resolving every variable on every access (including in loops) is O(d) per access where d is the nesting depth. For programs with deep nesting and hot inner loops, this is significant.

**Solution**: static scope-depth encoding (see Section 6 above) converts dynamic chain walks into O(1) indexed hops. The resolver precomputes d once per variable-use site; the interpreter executes a fixed-iteration loop at runtime.

**Further optimization** (noted as an exercise in CI): replace the hashmap with an array indexed by a resolver-assigned integer. Variable lookup becomes a two-integer operation (environment depth + slot index) rather than a hash lookup. This is exactly what clox does for local variables, achieving single-instruction variable access.

### Closures and Environment Capture [CI]

Closures require that the environment at the time of function declaration be preserved beyond the lexical lifetime of the declaring scope. Two fundamental design tensions:

1. **Snapshot vs. reference**: a closure should capture the environment *as it was* at declaration time, not a live mutable reference. Mutable reference semantics create the closure bug described in Section 1.
2. **Stack lifetime vs. heap lifetime**: local environments normally live on the call stack and are discarded on return. Closures extend the lifetime of captured environments into the heap. This forces environments that are closed over to be heap-allocated.

**jlox's approach** (CI): all environments are heap-allocated Java objects from the start; the GC handles lifetime. The resolver ensures variables are accessed via fixed depth rather than by scanning a mutable map. This avoids the mutable-environment bug without requiring persistent data structures.

**clox's approach** (CI, Part III): introduces **upvalues** — an explicit representation for closed-over variables. While a function is executing, upvalues point directly into the stack frame where the closed variable lives. When the enclosing function returns, the upvalue is "closed" — the variable's value is copied into a heap-allocated cell that the upvalue then points to. This achieves O(1) access to both open (stack) and closed (heap) upvalues while properly extending the lifetime of captured variables. Upvalues are a well-known technique in Lua's implementation.

**Comparison** [EaC, CI]:
- EaC's static coordinate `<lexical_level, offset>` for lexically scoped languages (Section 1) is the compile-time analog of CI's runtime scope-depth + slot-index scheme
- Both encode the location of a variable as a pair of integers rather than a name string
- EaC's approach works for languages with static type systems and known storage layouts at compile time; CI's approach works for dynamically typed languages where storage layout is determined at runtime

### The Fundamental Compiler Design Tension [EaC]

Every IR design choice involves a tradeoff between:
1. **Compile-time cost**: how expensive is it to build, maintain, and analyze this IR?
2. **Optimization quality**: how much does this IR expose to analysis and transformation?
3. **Proximity to source**: how well does this IR preserve source-level intent?
4. **Proximity to target**: how directly does this IR map to machine instructions?

These four dimensions pull in different directions. A single IR cannot be optimal on all four. The multi-IR approach — using a high-level IR early in the compiler and a low-level IR late — is the standard solution to this tension.

**CI adds a fifth dimension relevant to interpreted languages** [CI]:
5. **Execution model fit**: how well does the IR match the execution strategy (tree-walk, bytecode VM, JIT)? ASTs fit tree-walk interpreters directly; bytecode fits stack-based VMs; neither fits ahead-of-time compiled code well.
