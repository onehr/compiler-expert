# Dependent Type Elaboration: Comprehensive Reference

*Sources: Lean 4 Metaprogramming Book (leanprover-community.github.io), Agda docs (agda.readthedocs.io),
Bidirectional Typing survey (arXiv:1908.05839), Lean 4 Language Reference.*

---

## 1. What Elaboration Is

### The Gap: Surface Language vs Core Calculus

A dependently typed language has two distinct syntactic layers:

**Surface term** (what the programmer writes):
- Omitted (implicit) arguments: `id zero` instead of `@id Nat zero`
- Underscores / holes: `f _ x` where `_` should be inferred
- Syntactic sugar: `do` notation, `match`, `where` clauses, `∀ x y, P`
- Overloaded notation, dot-accessor syntax (`.add`), anonymous constructor syntax
- Type ascriptions that need checking: `(e : T)`
- Universe levels omitted: `List α` instead of `@List.{u} α`

**Core term** (fully explicit, fully annotated):
- Every argument is present — implicit, instance, and universe arguments all written out
- Every binder carries its type annotation
- No sugar: all pattern matches are `casesOn`, all `do` is desugared to `bind`
- Every constant is applied to its universe arguments: `@List.{0}`
- The type checker for the core is a simple recursive descent algorithm — no heuristics

**The elaborator** bridges the gap. It is a *type-directed translation* from surface to core.

### Elaboration = Type Checking + Implicit Filling + Universe Inference

Elaboration encompasses:
1. **Type checking / inference** — verifying or synthesizing types of terms
2. **Implicit argument insertion** — inserting metavariables where arguments are omitted
3. **Universe inference** — assigning universe levels to `Sort ?u` holes
4. **Instance synthesis** — finding typeclass instances
5. **Constraint solving** — running unification to solve all accumulated metavariables
6. **Desugaring** — translating syntactic sugar to core forms

The *output* of elaboration is a fully-annotated core term where every type is explicit. You can give this term to a simple, decidable kernel type checker that needs no heuristics.

### Why a Separate Elaborator Phase

A monolithic type checker that handles both the surface and core simultaneously would be:
- Difficult to specify (mixing heuristics with logical rules)
- Hard to trust (the trusted kernel grows)
- Hard to extend (adding a new sugar requires touching the kernel)

The standard architecture instead has:
- **Trusted kernel**: small, simple, decidable checker for the core calculus. This is what you stake correctness on.
- **Untrusted elaborator**: the large, complex, heuristic-laden pass. Mistakes here produce a term that the kernel will reject — safety is maintained.

[Lean4]: The elaborator lives in `Lean.Elab.*`. The kernel checker is in `Lean.Kernel`. Only the kernel needs to be trusted.
[Agda]: Same architecture: `Agda.TypeChecking.*` for elaboration, separate kernel for decidable checking.

---

## 2. Bidirectional Type Checking

Bidirectional type checking (Pierce & Turner 1998, Dunfield & Krishnaswami 2019) structures the type checker around two *modes* or *judgments*:

### The Two Modes

```
Γ ⊢ e ⇒ A      -- SYNTHESIS (inference) mode:
                --   given context Γ and term e,
                --   produce (synthesize) a type A

Γ ⊢ e ⇐ A      -- CHECKING mode:
                --   given context Γ, term e, and expected type A,
                --   verify that e has type A
```

Information flows differently in each mode:
- **Synthesis** flows *outward* (bottom-up / synthesized attribute): the term determines its type
- **Checking** flows *inward* (top-down / inherited attribute): the expected type flows into the term

### When to Use Each Mode

**Introduction forms CHECK** — because introductions produce values of a known type. The expected type tells us what we're constructing:
- Lambda `λ x. e` checks against `(x : A) → B` — we know `x : A` from the expected type
- Pair `(a, b)` checks against `A × B`
- Constructor `inl e` checks against `A + B`

**Elimination forms SYNTHESIZE** — because eliminations consume a term and the output type depends on what's being consumed:
- Variable `x` synthesizes its declared type from context
- Application `f a` synthesizes: first synthesize type of `f`, then check `a` against the domain
- Projection `p.1` synthesizes: synthesize type of `p`, then return the first component type

**Subsumption / mode switch**: when you have a synthesized type but need checking mode (or vice versa):
```
Γ ⊢ e ⇒ A    A ≡ B          -- if synthesis gives A and expected is B,
──────────────────────           check that A ≡ B (definitional equality)
Γ ⊢ e ⇐ B
```

**Type annotation** switches from checking to synthesis:
```
Γ ⊢ A : Type    Γ ⊢ e ⇐ A
──────────────────────────────    -- (e : A) synthesizes A
Γ ⊢ (e : A) ⇒ A
```

### Algorithm Sketch with Rules

```
-- Variable (synthesize)
x : A ∈ Γ
──────────────
Γ ⊢ x ⇒ A

-- Lambda (check)
Γ, x : A ⊢ e ⇐ B
────────────────────────────
Γ ⊢ (λ x. e) ⇐ (x : A) → B

-- Lambda (synthesize — requires annotation on x)
Γ, x : A ⊢ e ⇒ B
────────────────────────────────
Γ ⊢ (λ (x : A). e) ⇒ (x : A) → B

-- Application (synthesize)
Γ ⊢ f ⇒ (x : A) → B    Γ ⊢ a ⇐ A
────────────────────────────────────
Γ ⊢ f a ⇒ B[x := a]

-- Annotation (synthesize from check)
Γ ⊢ A : Type    Γ ⊢ e ⇐ A
──────────────────────────────
Γ ⊢ (e : A) ⇒ A

-- Subsumption (check from synthesize)
Γ ⊢ e ⇒ A    Γ ⊢ A ≡ B
────────────────────────    -- ≡ is definitional equality
Γ ⊢ e ⇐ B

-- Pi type (synthesize)
Γ ⊢ A ⇒ Type i    Γ, x : A ⊢ B ⇒ Type j
──────────────────────────────────────────
Γ ⊢ (x : A) → B ⇒ Type (max i j)
```

### Why Bidirectional Wins Over Full Inference

- Hindley-Milner handles `let`-polymorphism but fails on dependent types (type checking is undecidable with full HM + dependent types)
- Bidirectional type checking is *predictable*: programmers know when they need annotations
- It propagates type information top-down, which helps resolve ambiguities (e.g., `[]` can check against `List Nat` without an annotation)
- It integrates cleanly with metavariables: checking mode lets you assert the expected type into the metavariable

---

## 3. Implicit Arguments

### What They Are

Implicit arguments are function parameters the user omits at call sites. The elaborator fills them in by creating *metavariables* (unification variables) and solving them through type unification.

```
-- Surface
id zero         -- user writes this

-- After elaboration
@id Nat zero    -- elaborator fills in the Nat
```

### Syntax

[Lean4]:
- `{α : Type u}` — implicit argument, filled by unification
- `⦃α : Type u⦄` or `{{α : Type u}}` — strict implicit, only inserted when an explicit argument follows
- `[inst : Add α]` — instance argument, filled by typeclass synthesis
- `(x : α := default)` — optional argument with default

[Agda]:
- `{A : Set}` — implicit argument
- `{{inst : Monad M}}` or `⦃inst : Monad M⦄` — instance argument
- Named access: `{A = myType}` at call site
- Tactic arguments: `{@(tactic t) n : Nat}` — solved by running tactic `t`

### Metavariables: Placeholders for Unknown Terms

When the elaborator encounters an implicit argument it cannot immediately determine, it creates a **metavariable** `?M` — a typed hole in the term:

```
?M : Nat          -- a metavariable of type Nat
?M : Type u       -- a metavariable for a type at universe u
```

Each metavariable has:
- A **unique identifier** (MVarId in Lean 4)
- A **local context** (Γ): the free variables in scope when the hole was created
- A **type**: what type the eventual solution must have
- An optional **assignment**: the term that fills it (once solved)

[Lean4]: `mkFreshExprMVar` creates a new metavariable. `instantiateMVars` replaces all assigned metavariables in a term with their solutions.

### Unification: How Metavariables Get Solved

The elaborator accumulates **unification constraints** of the form `?M ≡ t` (metavariable equals term) or `A ≡ B` (two types must be definitionally equal). Solving these constraints assigns values to metavariables.

[Lean4]: `isDefEq ctx A B` checks definitional equality and may assign metavariables as a side effect — it is a *unifying* definitional equality check, not a pure predicate.

### Implicit Argument Insertion Algorithm

At each application site `f a₁ ... aₙ`:

1. Synthesize the type of `f`: get `(x₁ : A₁) → ... → (xₙ : Aₙ) → B`
2. For each parameter, look at its binder info:
   - **Explicit**: the next provided argument fills it
   - **Implicit `{}`**: insert a fresh metavariable `?Mᵢ` and try to solve it via unification
   - **Instance `[]`**: launch instance search for the required type
3. After processing all explicit arguments, insert trailing implicits if the result type doesn't match the expected type

**Eager vs lazy insertion**: most elaborators insert implicits eagerly (as many as needed to make types match). Lean 4 inserts strict implicits `⦃⦄` only when the next token is an explicit argument.

### Named Implicit Arguments

[Agda]: Call sites can supply implicit args by name:
```agda
subst C {y = myExpr} eq cx   -- supply the 'y' implicit by name, skip others
```

[Lean4]:
```lean
def f {n : Nat} (x : Fin n) := ...
f (n := 5) ⟨3, by omega⟩    -- explicitly name the implicit
```

---

## 4. Universe Levels

### The Hierarchy

To avoid Russell's paradox (`Type : Type` is inconsistent — Girard's paradox), dependent type theories use a **universe hierarchy**:

```
Prop  :  Type 0  :  Type 1  :  Type 2  :  ...

Prop = Sort 0
Type 0 = Sort 1
Type 1 = Sort 2
Type u = Sort (u+1)
```

[Lean4]:
- `Prop` = `Sort 0`: impredicative universe of propositions. `(∀ x : α, P x) : Prop` whenever `P x : Prop`, regardless of the universe of `α`.
- `Type 0` = `Sort 1`: the universe of ordinary small types. `Nat : Type 0`, `Bool : Type 0`.
- `Type u` = `Sort (u+1)`: the universe at level `u`.
- `Sort u`: uniform notation; `Prop = Sort 0`, `Type u = Sort (u+1)`.

[Agda]:
- `Set` = `Set₀`: ordinary types
- `Set₁`, `Set₂`, ...: the hierarchy
- No `Prop` by default (Agda is proof-irrelevant via `--prop` flag or `Prop` universe)
- `Setω`: the type of `(n : Level) → Set n`, above the whole hierarchy

### Why Universes Are Needed: Russell's Paradox

If `Type : Type` held, we could construct Girard's paradox (the type-theoretic analogue of Russell's paradox) and derive `False`. The universe hierarchy prevents this: `Type u : Type (u+1)` but not `Type u : Type u`.

### Universe Polymorphism

Rather than writing separate definitions at each universe level, functions can be *polymorphic* over universe levels.

[Lean4]:
```lean
-- Automatically universe-polymorphic (Lean infers the level variable u)
def id {α : Type u} (x : α) : α := x
-- Lean desugars this to:
-- def id.{u} {α : Type u} (x : α) : α := x

-- Universe-polymorphic identity for Sort (works for both Prop and Type)
def id' {α : Sort u} (x : α) : α := x
```

[Agda]:
```agda
-- Explicit level parameter
id : {n : Level} {A : Set n} → A → A
id x = x

-- Universe-polymorphic product
data _×_ {n m : Level} (A : Set n) (B : Set m) : Set (n ⊔ m) where
  _,_ : A → B → A × B

-- Universe-polymorphic list (level preserved)
data List {n : Level} (A : Set n) : Set n where
  []   : List A
  _::_ : A → List A → List A
```

### Universe Level Arithmetic

[Agda]: Universe levels are values of type `Level`. Operations:
- `lzero : Level` — the base level
- `lsuc : Level → Level` — successor; `Set (lsuc n) : Set (lsuc (lsuc n))`
- `_⊔_ : Level → Level → Level` — least upper bound (max)

Properties the compiler simplifies automatically:
- Commutativity: `a ⊔ b = b ⊔ a`
- Idempotence: `a ⊔ a = a`
- Neutrality of `lzero`: `a ⊔ lzero = a`
- Subsumption: `a ⊔ lsuc a = lsuc a`
- Distributivity: `lsuc (a ⊔ b) = lsuc a ⊔ lsuc b`

[Lean4]: Level expressions (`Lean.Level`) include:
- `Level.zero`, `Level.succ l`, `Level.max l₁ l₂`, `Level.imax l₁ l₂`
- `imax` is special: `imax u 0 = 0`, `imax u v = max u v` otherwise (handles `Prop`'s impredicativity)
- Universe parameters in definitions: `def foo.{u, v} ...`

### Universe Inference vs Explicit Annotation

[Lean4]: Lean automatically infers universe level variables for definitions. You almost never need to write `.{u}` explicitly. The elaborator collects universe level constraints and solves them:
- `Π (x : A) B : Sort (imax u v)` where `A : Sort u` and `B : Sort v`
- `A × B : Sort (max u v)` where `A : Sort u` and `B : Sort v`

When Lean cannot determine a universe level, it reports an error like "failed to synthesize universe level".

### The Setω / Typeω Boundary

[Agda]: `(n : Level) → Set n` does not live in any `Set ℓ` — it lives in `Setω`. Agda introduces `Setω` as the type of such expressions. Note: `Setω : Setω` is allowed via `--omega-in-omega`, but makes the system logically inconsistent.

[Lean4]: `Type*` notation refers to `Type u` for a fresh universe variable `u`. There is no `Typeω`; instead, Lean uses universe polymorphism to handle the same cases.

### Typical Universe Typing Rules

```
A : Sort u    B[x : A] : Sort v
────────────────────────────────────
(x : A) → B : Sort (imax u v)

-- imax u 0 = 0  (Pi into Prop is Prop, regardless of domain level)
-- imax u v = max u v  (otherwise)

A : Sort u    B[x : A] : Sort v
─────────────────────────────────────
(x : A) × B : Sort (max u v)          -- Sigma type

A : Sort u
─────────────────
List A : Sort u   -- Type constructors preserve universe level
```

---

## 5. Metavariables and Unification

### Creating Metavariables

Metavariables are created when the elaborator encounters an unknown:
1. An implicit argument `{α : Type}` in a function application
2. A hole `_` written by the user
3. A fresh goal in tactic mode
4. A universe level that must be inferred

[Lean4]: `MetaM.mkFreshExprMVar (type : Option Expr) : MetaM Expr`

Each metavariable `?M` has:
- A *type* (e.g., `?M : Nat`)
- A *local context* Γ (the free variables in scope at the point `?M` was created)
- A solution is a closed term of type `?M.type` using only variables in Γ

### Higher-Order Unification

In dependent type theory, unification is *higher-order* (terms can be functions applied to terms). **Higher-order unification is undecidable in general** (Huet 1975). In practice, elaborators implement:

1. **First-order unification** — most cases: compare term heads, decompose recursively
2. **Pattern unification** — a decidable fragment (Miller 1991)
3. **Heuristics** — for common cases that fall outside the decidable fragment
4. **Postponement** — when a problem is not yet solvable

### Pattern Unification (Miller's Algorithm)

A unification problem is in **pattern form** when it looks like:

```
?M x₁ x₂ ... xₙ ≡ t
```

where `x₁ ... xₙ` are **distinct bound variables** (locally bound, not metavariables).

If the free variables of `t` are all in `{x₁, ..., xₙ}`, there is a **unique solution**:

```
?M := λ x₁ x₂ ... xₙ. t
```

This is decidable and sound. Pattern unification covers the vast majority of practical elaboration cases.

**Example** (Lean/Agda style):
```
-- Synthesizing: map f [] ≡ []
-- Constraint: ?ResultType ≡ List ?ElemType
-- Pattern: ?ResultType is applied to no variables → solution: List ?ElemType
```

### Flex-Flex Problems

When both sides of a unification equation have metavariable heads, the problem is **flex-flex**. There may be infinitely many solutions. Elaborators typically:
- Defer flex-flex problems to the constraint queue
- Apply a "most general unifier" heuristic (e.g., `?A ≡ ?B` → assign `?A := ?B`)
- Fail if after all other constraints are solved, a flex-flex remains unsolved

### Postponed Constraints

When a unification problem cannot be solved immediately (the head is an unsolved metavariable, or we are waiting for more type information), it is **postponed** — added to a constraint queue to be retried later.

[Lean4]: The elaborator queues constraints via *synthetic metavariables* with associated solving strategies:
- `typeClass`: to be solved by instance synthesis
- `coe`: to be solved by coercion insertion
- `tactic`: to be solved by running a tactic
- `postponed`: to be retried when more information is available

Elaborators throw `Except.postpone` (or equivalent) to signal "try me later".

### Metavariable Dependency Tracking

A metavariable `?M` created in context `Γ = [x₁ : A₁, ..., xₙ : Aₙ]` may only be solved by a term that uses variables from Γ. Solutions that would reference free variables *not* in Γ are scope errors.

[Lean4]: `withContext mvarId computation` runs `computation` in the local context of metavariable `mvarId`. This ensures that expressions constructed for `?M` are valid in its scope.

### Definitional Equality and Reduction

Unification in dependent type theory is modulo *definitional equality* (not just syntactic equality). Two terms are definitionally equal if they reduce to the same normal form under:

- **β-reduction**: `(λ x. t) a` → `t[x := a]`
- **η-reduction**: `λ x. f x` → `f` (when `x` not free in `f`)
- **δ-unfolding**: unfold defined constants to their definitions
- **ι-reduction**: reduce recursors/eliminators applied to constructors

[Lean4]: `MetaM.isDefEq : Expr → Expr → MetaM Bool` — checks definitional equality and assigns metavariables. Transparency mode controls δ-unfolding:
- `reducible`: only `@[reducible]` defs
- `instances`: reducible + `@[instance]`
- `default`: all non-`@[irreducible]` defs
- `all`: everything

[Lean4]: `MetaM.whnf : Expr → MetaM Expr` — weak head normal form: reduce until the head is a constructor, lambda, pi, or stuck (metavariable, free variable).

---

## 6. Instance Search (Typeclass Resolution)

### What Instances Are

A **typeclass** is a record (or predicate) parameterized by a type (or types). An **instance** is a proof/value that fills the typeclass for a specific type. Instance arguments are filled automatically by searching a global database.

[Lean4]:
```lean
class Add (α : Type u) where
  add : α → α → α

instance : Add Nat where
  add := Nat.add

instance [Add α] : Add (List α) where   -- conditional instance
  add xs ys := List.zipWith Add.add xs ys
```

[Agda]:
```agda
record Monoid {a} (A : Set a) : Set a where
  field
    mempty : A
    _<>_   : A → A → A

instance
  ListMonoid : ∀ {a} {A : Set a} → Monoid (List A)
  ListMonoid = record { mempty = []; _<>_ = _++_ }
```

### The Search Algorithm: Backward Chaining

Instance resolution is **backward chaining** (like Prolog resolution):

1. **Goal**: find an instance of type `T`
2. **Candidates**: collect all instances whose head type unifies with `T`
   - Top-level `instance` declarations
   - Local `instance` hypotheses in context
   - `let`-bound instance values
3. **Select**: apply the candidate, generating subgoals for its premises
4. **Recurse**: solve subgoals recursively
5. **Depth limit**: abort after a maximum search depth to prevent infinite loops

[Lean4] resolution also looks at `@[instance]`-tagged definitions. Resolution can be debugged with `set_option trace.Meta.synthInstance true`.

[Agda] resolution steps:
1. Verify the goal shape is `{Γ} → C vs` (named type applied to args)
2. Collect candidates (let-bound, top-level, local instance args)
3. Unify candidate types with target
4. Remove overlapped candidates (using `OVERLAPPABLE`/`OVERLAPPING` pragmas)
5. If exactly one candidate remains, commit; else error

### Coherence: At Most One Instance

**Coherence**: for a given type `T`, there should be at most one instance of typeclass `C T`. Multiple instances create **incoherence** — different parts of a program may use different instances, leading to inconsistency.

[Lean4]: Uses the diamond problem heuristic — if two instance paths produce definitionally equal results, Lean accepts them. The `[instance]` attribute controls which definitions participate in synthesis.

[Agda]: Explicit overlap pragmas (`OVERLAPPABLE`, `OVERLAPPING`, `OVERLAPS`, `INCOHERENT`) control the resolution of multiple candidates. `--backtracking-instance-search` enables recursive resolution but may slow search significantly.

### Debugging Instance Search

[Lean4]:
```lean
#check (inferInstance : Add Nat)   -- shows the found instance
#synth Add Nat                      -- runs synthesis, shows result
set_option trace.Meta.synthInstance true in
#check (0 : Nat) + 1               -- trace the search
```

[Agda]:
```agda
-- Explicitly provide instance to bypass resolution:
_==_ {A} {{eqA}} a b

-- Use -v 20 flag or interactive mode to trace instance resolution
```

---

## 7. The Elaboration Algorithm (Sketch)

### Phase 0: Pre-Elaboration

- **Macro expansion**: expand user-defined macros and notation to core syntax
- **Name resolution**: resolve unqualified names to fully-qualified names using the current namespace and open directives
- **Operator fixity**: apply fixity/precedence/associativity rules to desugar infix notation into function applications
- **Syntax desugaring**: `do` notation → `bind`/`pure`; `match` → `casesOn` eliminators; `where` clauses → let bindings; structure field access `.field` → projection

### Phase 1: Core Elaboration Loop (Bidirectional)

The main elaboration function has signature:

```
-- Lean 4 style (TermElabM monad)
elab : Syntax → Option Expr → TermElabM Expr
-- arg 1: surface syntax
-- arg 2: expected type (None if unknown)
-- result: core Expr (fully annotated)
```

Key dispatch cases:

```
elab (Var x) expectedTy =
  ty ← lookupContext x         -- synthesize from context
  unifyWithExpected ty expectedTy
  return (Expr.fvar x)

elab (App f arg) expectedTy =
  fExpr ← elab f None          -- synthesize type of f
  fTy ← whnf (typeOf fExpr)
  match fTy with
  | Expr.forallE x A B binderInfo →
      match binderInfo with
      | implicit →
          meta ← mkFreshExprMVar (some A)    -- insert implicit metavar
          result ← elab (App (App f meta) arg) expectedTy
          return result
      | explicit →
          argExpr ← elab arg (some A)        -- check arg against domain
          return Expr.app fExpr argExpr
      | instImplicit →
          inst ← synthInstance A             -- typeclass search
          return Expr.app fExpr inst
  | _ → throwError "not a function"

elab (Lam x body) (some (Expr.forallE _ A B _)) =
  bodyExpr ← withLocalDecl x A (elab body (some B))   -- check body
  return Expr.lam x A bodyExpr BinderInfo.default

elab (Lam (x : A') body) None =
  A ← elab A' None                          -- must have annotated binder
  bodyExpr ← withLocalDecl x A (elab body None)
  B ← inferType bodyExpr
  return Expr.lam x A bodyExpr BinderInfo.default

elab (Ascription e ty) _ =
  tyExpr ← elab ty None                     -- elaborate the type annotation
  eExpr ← elab e (some tyExpr)              -- check e against it
  return eExpr

elab (Hole _) expectedTy =
  ty ← expectedTy.getOrElse (mkFreshTypeMVar ())
  mkFreshExprMVar (some ty)                 -- return an unsolved metavar
```

### Phase 2: Constraint Queue Processing

After the main elaboration pass, the elaborator processes its queue of postponed constraints:

1. **Universe constraints**: solve universe level inequalities. A simple topological sort / constraint propagation over `Level` expressions.
2. **Unification constraints**: retry any postponed unification problems now that more metavariables may be assigned.
3. **Instance synthesis**: run backward chaining for unresolved `[instance]` metavariables.
4. **Default instances**: apply `@[defaultInstance]` if no other instance was found.

### Phase 3: Post-Elaboration Checks

- **Unsolved metavariables**: if any `?M` remains unassigned, report "unknown `?M`" error
- **Universe consistency**: verify all universe constraints are satisfiable (no cycle `u < u`)
- **Type correctness check**: optionally re-run the kernel type checker on the produced core term to catch elaborator bugs
- **Definitional equality**: verify that any user-provided expected type matches the elaborated type

---

## 8. Lean 4 Elaboration Architecture

[Lean4]

The Lean 4 elaborator is structured as a stack of monads (each extends the previous):

```
CoreM          -- environment, options, logging, name generation
  └── MetaM    -- metavariable state, local context, definitional equality
        └── TermElabM  -- term elaboration: expected type, macro expansion
              └── TacticM    -- tactic proof state: goals (list of mvar)
                    └── CommandElabM  -- top-level command elaboration
```

Key types:
- `Lean.Expr`: the core AST. Constructors: `bvar`, `fvar`, `mvar`, `sort`, `const`, `app`, `lam`, `forallE`, `letE`, `lit`, `mdata`, `proj`
- `Lean.Level`: universe level expressions: `zero`, `succ`, `max`, `imax`, `param`, `mvar`
- `Lean.MVarId`: unique identifier for a metavariable
- `Lean.LocalContext`: ordered map of `FVarId → LocalDecl` (hypotheses)
- `Lean.MetavarContext`: map of `MVarId → MetavarDecl` (metavariable state)

Key elaboration functions:
```lean
-- Elaborate a syntax node with an optional expected type
TermElabM.elabTerm : Syntax → Option Expr → TermElabM Expr

-- Check a term has a specific type (checking mode)
TermElabM.elabTermEnsuringType : Syntax → Expr → TermElabM Expr

-- Create a fresh metavariable
MetaM.mkFreshExprMVar : Option Expr → MetaM Expr

-- Unifying definitional equality (assigns metavariables)
MetaM.isDefEq : Expr → Expr → MetaM Bool

-- Weak head normal form
MetaM.whnf : Expr → MetaM Expr

-- Typeclass synthesis
Meta.synthInstance : Expr → MetaM (Option Expr)

-- Instantiate all assigned metavariables in an expression
MetaM.instantiateMVars : Expr → MetaM Expr
```

---

## 9. Agda Elaboration Notes

[Agda]

### Implicit Argument Features

```agda
-- Basic implicit
id : {A : Set} → A → A
id x = x

-- Usage: supply all, some, or named
id zero            -- infer A = Nat
id {Nat} zero      -- explicit A = Nat
id {A = Nat} zero  -- named

-- Unnamed implicit (solved by η-expansion / proof search)
data SList (bound : Nat) : Set where
  scons : (head : Nat) → {head ≤ bound} → SList head → SList bound
-- The {head ≤ bound} argument has no name; resolved by instance search or unification

-- Lambda with implicit
id' = λ {A} → id {A}    -- explicit pattern on implicit arg
z1 = λ {A} x → x        -- same as id
z2 = λ x → x            -- omit implicit binder, Agda infers it

-- Tactic arguments
the-best-number : {@(tactic clever-search) n : Nat} → Nat
```

### Instance Arguments

```agda
-- Instance argument: {{...}} or ⦃...⦄
_==_ : {A : Set} {{eqA : Eq A}} → A → A → Bool

-- Instance declaration
instance
  NatEq : Eq Nat
  NatEq = record { _==_ = Nat._==_ }

-- Overlap control pragmas
{-# OVERLAPPABLE ListEq #-}    -- can be superseded by more specific instance
{-# OVERLAPPING NatEq #-}      -- supersedes less specific instances
```

### Dot Patterns (Forced Patterns)

In dependent pattern matching, some arguments are *forced* by the types of others:

```agda
-- The n in the result is forced by the length type
head : {A : Set} {n : Nat} → Vec A (suc n) → A
head (x :: _) = x
-- Agda writes the forced pattern as: head {n = .n} (x :: _) = x
-- The dot means: this value is determined by unification, not by pattern matching
```

### With Abstraction

`with` clauses let you pattern-match on a subterm by abstracting it from the goal:

```agda
filter : {A : Set} → (A → Bool) → List A → List A
filter p [] = []
filter p (x :: xs) with p x
... | true  = x :: filter p xs
... | false = filter p xs
```

The `with` keyword replaces all occurrences of `p x` in the goal with a fresh variable, then lets you pattern-match on it.

---

## 10. Implementation Decision Guide

### Do You Need Full Dependent Types?

Before implementing full dependent type elaboration, consider the spectrum:

| Feature | Complexity | Languages Using It |
|---------|-----------|-------------------|
| Monomorphic type checking | Very low | Early ML, Fortran |
| Hindley-Milner inference | Medium | Haskell, OCaml, Rust |
| System F / rank-N polymorphism | High | GHC with extensions |
| Bidirectional + implicits (no dependent types) | High | Many modern languages |
| Dependent types with full elaboration | Very high | Lean 4, Agda, Coq, Idris |

**Minimal useful setup** (covers 90% of needs without full dependent types):
- Bidirectional type checking
- Implicit arguments filled by first-order unification
- Basic typeclass/trait resolution
- No universe polymorphism needed

### HM vs Bidirectional vs Full Elaboration

**Hindley-Milner**:
- Pro: fully automatic (no annotations), familiar
- Con: no dependent types, limited polymorphism, principal type property but hard to extend
- Use when: functional language without dependent types

**Bidirectional type checking**:
- Pro: predictable annotation requirement, extends to dependent types, good error messages
- Con: programmers need to annotate some terms
- Use when: any modern language; baseline for dependently-typed systems

**Full dependent type elaboration**:
- Pro: Curry-Howard, proofs as programs, extremely expressive
- Con: months of engineering work, undecidable unification, complex error messages
- Use when: building a proof assistant or a language where correctness is the primary goal

### The Cost of Full Elaboration

Implementing a production-quality elaborator is a multi-year project:
- **Lean 4** took ~5 years from scratch
- **Agda** has been under development for 20+ years
- **Coq/Rocq**: 30+ years

Rough work estimates for a research-quality (not production) elaborator:
- Core infrastructure (metavariables, whnf, isDefEq): 2–3 months
- Pattern unification + constraint queue: 1–2 months
- Bidirectional elaboration of Pi/Sigma/Let: 2–4 months
- Universe level inference: 1–2 months
- Instance synthesis: 2–4 months
- Error messages: 2–4 months (ongoing)
- Syntax sugar / macros: 2–6 months

### Minimal Elaboration: Implicit Arguments + Basic Unification

A practical subset to implement first:

```
1. Core calculus: just Pi (no Sigma, no inductives yet)
2. Metavariables: create, assign, instantiate
3. WHNF: beta reduction only
4. Definitional equality: syntactic + beta
5. Pattern unification: the Miller fragment
6. Implicit argument insertion: create metavar, unify types
7. Bidirectional checking/synthesis
```

This gives you implicit argument inference similar to what ML has with `'a` variables, plus basic dependent types. Add universe levels, instance search, and inductives as separate passes later.

### Error Message Design for Elaboration

The most user-visible part of an elaborator is its error messages. Key principles:
- Always show **what was inferred** vs **what was expected**
- Show the **context**: which implicit was being solved, what constraint failed
- Show the **hole context**: if `?M` is unsolved, show what variables are in scope
- Use **error ranges** pointing to the exact surface term that caused the failure
- For universe errors: show the full constraint chain that led to `u < u`

---

## 11. Key References

| Resource | What It Covers |
|----------|---------------|
| Dunfield & Krishnaswami (2019), arXiv:1908.05839 | Comprehensive survey of bidirectional type checking |
| Ulf Norell's PhD thesis (2007) | Original design of Agda's elaboration algorithm |
| Lean 4 Metaprogramming Book (leanprover-community.github.io) | MetaM, TermElabM, metavariables in Lean 4 |
| Agda docs: implicit-arguments, universe-levels, instance-arguments | Definitive Agda reference |
| Andreas Abel — NbE for definitional equality | Normalization-by-evaluation for isDefEq |
| Conor McBride — "Outrageous but Meaningful Coincidences" | Universe-polymorphic elaboration |
| de Bruijn (1970) | Original AUTOMATH — first elaboration system |
| Miller (1991) — "A Logic Programming Language with Lambda-Abstraction..." | Pattern unification algorithm |

---

## Appendix A: Cheat Sheet — Lean 4 Binder Syntax

| Syntax | Kind | Filled By |
|--------|------|-----------|
| `(x : T)` | explicit | user must provide |
| `{x : T}` | implicit | unification |
| `⦃x : T⦄` | strict implicit | unification, only inserted before explicit arg |
| `[inst : T]` | instance | typeclass synthesis |
| `(x : T := default)` | optional | default value if not provided |
| `(x : T := by tactic)` | auto-param | tactic proof |

## Appendix B: Cheat Sheet — Agda Binder Syntax

| Syntax | Kind | Filled By |
|--------|------|-----------|
| `(x : T)` | explicit | user must provide |
| `{x : T}` | implicit | unification |
| `{{inst : T}}` or `⦃inst : T⦄` | instance | instance resolution algorithm |
| `{@(tactic t) x : T}` | tactic | running tactic `t` |
| `{x : T}` unnamed | implicit | η-expansion or instance search |

## Appendix C: Universe Level Comparison

| Concept | Lean 4 | Agda |
|---------|--------|------|
| Base level | `0` | `lzero` |
| Successor | `Level.succ u` | `lsuc n` |
| Max | `Level.max u v` | `n ⊔ m` |
| Impredicative max | `Level.imax u v` | — (no Prop by default) |
| Sort universe | `Sort u` | `Set n` |
| Propositions | `Prop = Sort 0` | `Set lzero` (or `--prop`) |
| Above hierarchy | — | `Setω` |
| Auto-inference | yes (most cases) | explicit Level params needed |
