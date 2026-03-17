# Type System Implementation — Deep Reference
## Decision Frameworks, Algorithms, and Tradeoffs
### Sources: okmij.org (Rémy/OCaml), rustc-dev-guide, Swift Book, Pierce TAPL

---

## PART I: HINDLEY-MILNER TYPE INFERENCE — FULL ALGORITHM W

[HM-algo]

### 1.1 Type Representation

Before writing Algorithm W, define the type language:

```
Type τ ::=
  | α                    -- type variable (unification variable)
  | T                    -- base type (Int, Bool, String, …)
  | τ₁ → τ₂             -- function type
  | C τ₁ … τₙ           -- type constructor applied to args
                         --   (List α, Option α, Pair α β, …)

TypeScheme σ ::=
  | τ                    -- monotype (no quantification)
  | ∀α. σ               -- polymorphic type (type schema)
```

In an implementation, type variables are mutable reference cells:

```
type tv = Unbound of string * level   (* free variable, with level *)
        | Link of typ                  (* bound, points to another type *)

type typ =
  | TVar of tv ref
  | TBase of string
  | TArrow of typ * typ
  | TApp of string * typ list          (* e.g. TApp("List", [α]) *)

type scheme =
  | Mono of typ
  | Poly of string list * typ          (* ∀α₁…αₙ. τ *)
```

Key invariant: follow Link chains eagerly. Write a `repr` function that chases links:

```ocaml
let rec repr = function
  | TVar {contents = Link t} -> repr t
  | t -> t
```

---

### 1.2 Fresh Type Variable Generation

Every unknown expression gets a fresh type variable. Use a global counter:

```ocaml
let next_id = ref 0
let gensym () =
  let id = !next_id in
  incr next_id;
  "α" ^ string_of_int id

let newvar () = TVar (ref (Unbound (gensym (), !current_level)))
```

The `current_level` tracks let-nesting depth for Rémy-style generalization (see §1.5).

---

### 1.3 Constraint Generation Rules (Typing Rules for Algorithm W)

Algorithm W generates constraints by recursively typing expressions. The rules:

**Var rule** — look up the type scheme in the environment, instantiate it:
```
Γ(x) = ∀α₁…αₙ. τ
---------------------
Γ ⊢ x : [β₁/α₁,…,βₙ/αₙ]τ   where β₁…βₙ are fresh
```
Implementation: `inst` replaces each `QVar` (quantified variable) with a fresh `TVar`.

**Lam rule** — introduce a fresh variable for the parameter:
```
β fresh    Γ, x:β ⊢ e : τ
--------------------------
Γ ⊢ (fun x -> e) : β → τ
```

**App rule** — create a fresh result variable, unify function type:
```
Γ ⊢ e₁ : τ₁    Γ ⊢ e₂ : τ₂    β fresh    unify(τ₁, τ₂→β)
-------------------------------------------------------------
Γ ⊢ e₁ e₂ : β
```

**Let rule** — generalize the bound expression's type:
```
Γ ⊢ e₁ : τ₁    Γ, x:GEN(Γ,τ₁) ⊢ e₂ : τ₂
-------------------------------------------
Γ ⊢ let x = e₁ in e₂ : τ₂
```
The critical point: `GEN(Γ, τ)` generalizes only type variables NOT free in `Γ`. This is what gives let-polymorphism.

**Algorithm W in pseudocode:**

```
function W(Γ, e):
  match e with
  | Var x ->
      σ = Γ.lookup(x)          -- look up type scheme
      return inst(σ)            -- instantiate quantified vars

  | Lam(x, body) ->
      β = newvar()              -- fresh type variable for x
      τ = W(Γ ∪ {x: β}, body)  -- type body with x:β in env
      return TArrow(β, τ)

  | App(e1, e2) ->
      τ₁ = W(Γ, e1)
      τ₂ = W(Γ, e2)
      β = newvar()              -- fresh result type
      unify(τ₁, TArrow(τ₂, β))
      return β

  | Let(x, e1, e2) ->
      enter_level()             -- for Rémy generalization
      τ₁ = W(Γ, e1)
      leave_level()
      σ = generalize(Γ, τ₁)    -- quantify free vars not in Γ
      return W(Γ ∪ {x: σ}, e2)
```

---

### 1.4 Robinson Unification Algorithm with Occurs Check

[HM-algo]

Unification finds the most general substitution `σ` such that `σ(τ₁) = σ(τ₂)`. It modifies type variables in place (destructive unification using reference cells).

**Full pseudocode with occurs check:**

```
function unify(t1, t2):
  t1 = repr(t1)    -- chase links
  t2 = repr(t2)    -- chase links

  if t1 == t2:     -- physical equality: same pointer, done
      return

  match (t1, t2) with

  | (TVar tv, t) | (t, TVar tv) ->
      -- tv must be an Unbound variable at this point
      occurs_check(tv, t)        -- prevent infinite types
      update_levels(tv, t)       -- Rémy level adjustment
      tv.contents := Link t      -- BIND: overwrite with link

  | (TBase a, TBase b) ->
      if a ≠ b: raise TypeError("cannot unify " ^ a ^ " with " ^ b)

  | (TArrow(t1a, t1b), TArrow(t2a, t2b)) ->
      unify(t1a, t2a)
      unify(t1b, t2b)

  | (TApp(c1, args1), TApp(c2, args2)) ->
      if c1 ≠ c2: raise TypeError(...)
      List.iter2 unify args1 args2

  | _ ->
      raise TypeError("cannot unify " ^ show(t1) ^ " with " ^ show(t2))
```

**Occurs check:**

```
function occurs_check(tv, t):
  t = repr(t)
  match t with
  | TVar tv2 ->
      if tv == tv2:       -- same variable: infinite type!
          raise TypeError("occurs check failed: cyclic type")
      -- else: different unbound variable, ok
  | TBase _       -> ()   -- no variables inside
  | TArrow(a, b)  -> occurs_check(tv, a); occurs_check(tv, b)
  | TApp(_, args) -> List.iter (occurs_check tv) args
```

**Why occurs check matters:** Without it, unifying `α` with `α → α` succeeds by setting `α := α → α`. But `α` appears inside `α → α`, so the resulting "type" is the infinite tree `(... → ...) → (... → ...)`. Following links during any subsequent traversal will loop forever. Occurs check prevents this by failing early. Some languages (Prolog, some ML dialects with `-rectypes`) intentionally omit it to allow recursive types, but this requires special cycle-aware traversal everywhere.

---

### 1.5 Let-Polymorphism and Generalization

**Naive generalization:**

```
function gen_naive(Γ, τ):
  -- quantify all free type variables in τ not in Γ
  env_vars = free_vars_in_env(Γ)
  τ_vars   = free_vars(τ)
  to_quantify = τ_vars - env_vars
  return Poly(to_quantify, τ)
```

**The problem with naive generalization:** Computing `free_vars_in_env(Γ)` requires scanning every type in every binding. In a program with N let-bindings, this is O(N) per let, O(N²) total. The original Caml compiler actually did this, and bootstrapping took 20 minutes. [okmij.org]

---

### 1.6 Didier Rémy's Efficient Generalization (Levels/Ranks)

[HM-algo, okmij.org]

**The key insight:** Instead of scanning the environment at generalization time, track *where* each type variable was created using a numeric *level* (De Bruijn level = let-nesting depth). A type variable can only be generalized if its level is greater than the current let-level — meaning it was created inside the current let-binding and thus not accessible from the outer environment.

**How levels work:**

```
current_level = 1     -- global, starts at 1

-- Entering a let-binding increments the level:
function enter_level(): current_level += 1
function leave_level(): current_level -= 1

-- Fresh variables receive the current level:
function newvar():
    return TVar(ref (Unbound(gensym(), current_level)))
```

**The critical invariant:** A free type variable with level `l` was allocated in a let at depth `l`. If `l > current_level`, the let that created it has already exited — the variable is "orphaned" and safe to quantify. If `l <= current_level`, the variable is still owned by a live let-binding and must NOT be quantified.

**Level-aware generalization:**

```
function gen(τ):
  τ = repr(τ)
  match τ with
  | TVar {Unbound(name, l)} when l > current_level ->
      -- level is above current: owned by a finished let, safe to quantify
      QVar name                   -- replace with quantified variable
  | TVar {Unbound(_, _)} ->
      τ                           -- level is current or below: don't quantify
  | TVar {Link t} ->
      gen(t)
  | TArrow(a, b) ->
      TArrow(gen(a), gen(b))
  | TApp(c, args) ->
      TApp(c, List.map gen args)
  | _ -> τ
```

**Level adjustment during unification:** When unifying a free variable `tv` at level `l` with a type `t`, any free variables in `t` that have a level *higher* than `l` must have their level lowered to `l`. This is because `tv` (at level `l`) may be used in a context that survives beyond the let that created those higher-level variables. The `occurs_check` function in §1.4 doubles as the level-update pass:

```
function occurs_check_and_update(tv_level, t):
  t = repr(t)
  match t with
  | TVar {Unbound(name, l)} ->
      -- prevent occurs check failure first
      if tv == t: raise TypeError("cyclic type")
      -- adjust level: min of current var's level and the binding var's level
      let min_l = min l tv_level in
      t.contents := Unbound(name, min_l)
  | TArrow(a, b) ->
      occurs_check_and_update(tv_level, a)
      occurs_check_and_update(tv_level, b)
  | ...
```

**Why this is efficient:** The occurs-check traversal, which we must do anyway to prevent cycles, simultaneously propagates levels. Generalization itself is just a single traversal of the type being generalized. No environment scanning is required — O(size of type), not O(size of program).

**Analogy to garbage collection:** Level tracking is analogous to generational GC. Type variables are allocated in the "innermost region" (highest level). During generalization (minor collection), variables that belong only to the current region are quantified (collected). Variables referenced from outer regions (lower levels) are promoted (level lowered) rather than collected.

**The `sound_lazy` optimization** (full Rémy algorithm): Composite types `TArrow`, `TApp`, etc. also carry two level fields: `level_old` (current upper bound) and `level_new` (pending update target). This lets unification be amortized — level propagation is deferred until gen/inst time, reducing repeated traversal of the same subtree. OCaml uses a variant of this. Key detail: `generic_level` is a sentinel value (e.g., `max_int`) marking a type as fully generalized; `marked_level` = -1 marks nodes currently being traversed to detect cycles cheaply. [okmij.org]

---

### 1.7 The Value Restriction

[HM-algo]

**The problem:** Naively, `let r = ref []` should get type `∀α. α list ref`. But then:
```
let r = ref []
let () = r := [1]    -- instantiate α = int
let () = print (List.hd (!r))  (* instantiate α = string — unsound! *)
```

**The value restriction (Tofte 1990, Wright 1995):** Generalization at a `let`-binding is only allowed if the right-hand side is *syntactically a value* (a lambda, literal, constructor, or variable — not an application or any expression that could have side effects). The formal test is called "nonexpansive":

```
nonexpansive(e) iff e is:
  | Var _
  | Literal _
  | Lam _
  | Constructor(_, args) where all args are nonexpansive
  | not (App _ | Let _ | Seq _ | ...)
```

If the RHS is expansive, its type variables are NOT generalized — they become *weak type variables* (written `'_a` in OCaml, sometimes shown as `'_weak1`).

**Why this is sound:** An application may allocate a mutable cell. If we generalize the type of that cell, we allow it to be used at multiple incompatible types, breaking type safety. By restricting generalization to syntactic values (which cannot have observable side effects by definition), we ensure soundness.

**OCaml relaxation:** OCaml uses a relaxed value restriction: functions whose body is a value are also considered nonexpansive, enabling more polymorphism in practice without sacrificing soundness.

---

## PART II: TRAIT / TYPECLASS CONSTRAINT SOLVING

[rustc-dev-guide]

### 2.1 What Constraints Are

A trait bound like `fn sort<T: Ord>(v: &mut Vec<T>)` introduces a *predicate*: `T: Ord`. During type checking, every usage that requires `Ord` operations generates an *obligation* — a predicate that must be proven. The constraint solver's job is to prove all obligations by finding appropriate *impls* or *where-clause* assumptions.

**Obligation**: a trait reference needing resolution. Example: `isize: Clone`.

**Obligaton sources:**
1. Trait bounds on generic parameters (`T: Clone`)
2. Supertraits (if `T: Iterator`, then also `T: ?Sized`)
3. Trait method calls that require a bound
4. Coercions and auto-derefs
5. Operator desugaring (`a + b` requires `T: Add`)

---

### 2.2 Three-Phase Resolution (rustc old solver)

[rustc-dev-guide]

Trait resolution in rustc consists of three interacting components:

**1. Selection** — Given a single obligation `T: Trait`, decide:
- `Ok(Some(vtable))` — solvable; `vtable` describes which impl to use and may create nested obligations
- `Ok(None)` — ambiguous (inference variables still unresolved)
- `Err(_)` — definitely fails

**2. Fulfillment** — A worklist that accumulates obligations and drives selection to completion. Each successfully selected obligation may emit nested obligations (from the impl's where-clauses) back into the worklist.

**3. Evaluation** — Checks whether an obligation *could* hold without committing any inference variables. Used during winnowing (see below) and cycle detection.

---

### 2.3 Candidate Assembly and Winnowing

[rustc-dev-guide]

**Candidate assembly:** For an obligation `T: Trait`, collect all potentially applicable candidates:
- **impl candidates**: scan all impls of `Trait` in scope, attempt to unify the `Self` type with `T`
- **where-clause candidates**: look in the current parameter environment for `T: Trait` assumptions
- **built-in candidates**: auto-traits (`Send`, `Sync`, `Sized`), closures, function pointers
- **object candidates**: `dyn Trait` implements `Trait`
- **projection candidates**: for associated types

**Winnowing:** If multiple candidates match, use `evaluate_candidate` to filter out candidates whose where-clauses cannot be satisfied, and `candidate_should_be_dropped_in_favor_of` to prefer more specific impls. Example:

```rust
impl<T: Copy> Get for T { ... }
impl<T: Get> Get for Box<T> { ... }
```

For `Box<u16>: Get`, both match superficially. Winnowing evaluates `Box<u16>: Copy` (fails, since `Box<T>` is not `Copy`) and keeps only the `Box<T>` impl.

If after winnowing exactly one candidate remains: confirmed. If zero: error. If more than one: ambiguity error.

---

### 2.4 Confirmation and Nested Obligations

Once a candidate is selected, *confirmation* unifies the trait's output type parameters (associated types) with the values implied by the impl, potentially yielding type errors. It also enqueues the impl's where-clause obligations as new obligations on the fulfillment worklist.

Example:
```rust
impl<T: Clone> Clone for Vec<T> { ... }
```
Confirming `Vec<u8>: Clone` with this impl adds the nested obligation `u8: Clone`.

**Selection during codegen:** During type checking, rustc tracks *which* impl would be selected but doesn't commit. At codegen, after all types are concrete, trait selection runs again to determine the actual vtable or monomorphized version to use.

---

### 2.5 Chalk's Logic-Programming Approach

[rustc-dev-guide]

Chalk recasts the Rust trait system as a logic program. Each trait, impl, and where clause is lowered into a Horn clause (or a slight generalization). The solver then executes SLD resolution (like Prolog) to prove goals.

**Lowering example:**

```rust
trait Clone { }
impl Clone for usize { }
impl<T: Clone> Clone for Vec<T> { }
```

becomes (Prolog-like notation):

```prolog
Clone(usize).
Clone(Vec(?T)) :- Clone(?T).
```

A query "does `Vec<Vec<usize>>` implement `Clone`?" becomes: prove `Clone(Vec(Vec(usize)))`, which resolves recursively.

**Why not pure Horn clauses:** Rust requires extensions:
- Negative reasoning (`!Send`)
- Higher-ranked predicates (`for<'a> Fn(&'a T)`)
- Associated type projections (`<T as Iterator>::Item = U`)
- Coinductive reasoning for `auto` traits and cycles

Chalk uses a language called Rust Chalk IR that handles these. The solver is a variant of SLD resolution with tabling (memoization) to handle cycles.

**The coherence check maps to logic:** The orphan rule ensures that for any given type `T` and trait `Tr`, there is at most one "program clause" proving `Tr(T)` that is globally visible. Without this, the logic program would be ambiguous (multiple proofs for the same fact, leading to different codegen decisions in different crates).

---

### 2.6 Coherence and the Orphan Rules

[rustc-dev-guide]

**What coherence means:** At most one impl of a given trait for a given type must be visible in any compilation context. This guarantees:
1. Trait dispatch is unambiguous (exactly one dictionary to choose)
2. Trait implementations are *globally consistent* — the same `T: Trait` impl is used everywhere, so compiled code from different crates can interoperate
3. Specialization (if permitted) follows a well-defined priority order

**The orphan rule:** An impl `impl Trait for Type` is legal only if at least one of `Trait` or `Type` is defined in the current crate. More precisely, rustc uses the "orphan check" which requires that either:
- The `Trait` is local (defined in current crate), or
- The `Self` type is "local" according to a precise structural criterion (the first non-generic, non-local type parameter position must be a local type)

**What breaks without orphan rules:** Imagine two crates both define `impl Display for Vec<String>`. Any program using both crates would have two conflicting impls with no way to choose. Downstream code cannot compile deterministically.

**Negative impls (`!Send`):** Opt out of auto-traits. `impl !Send for MyType {}` permanently marks `MyType` as not `Send`. The solver must handle these as definitive negative information, not just failure-to-prove. Rustc represents this as a "negative coherence" check.

---

### 2.7 Higher-Ranked Trait Bounds (HRTB)

[rustc-dev-guide]

`for<'a> Fn(&'a T)` means "this type implements `Fn` for every possible lifetime `'a`". This is needed when a callback must work with borrows of arbitrary lifetime:

```rust
fn apply<F>(f: F, s: &str) -> usize
where
    F: for<'a> Fn(&'a str) -> usize
{ f(s) }
```

**Implementation:** HRTB is handled by *skolemization* during candidate matching. The `'a` placeholder is replaced with a fresh "universally quantified" lifetime region that cannot unify with any specific lifetime. If the bound holds with the skolem, it holds for all lifetimes.

In Chalk terms, `for<'a> Fn(&'a T)` is a higher-ranked clause, not a Horn clause. The solver must handle binders (`for<'a>`) within predicates.

**Common HRTB patterns:**
- `for<'a> Fn(&'a T)` — closures that borrow their argument
- `for<'a> Future<Output = &'a T>` — async code returning borrows
- Implied by closure syntax in many cases (rustc inserts HRTB automatically for `Fn*` bounds)

---

## PART III: SUBTYPING AND VARIANCE — IMPLEMENTATION

### 3.1 Subtype Checking Algorithm

[HM-algo]

Subtyping is a *separate* relation from unification. Where unification requires `τ₁ = τ₂`, subtyping asks `τ₁ <: τ₂` ("τ₁ is a subtype of τ₂"). Implement as a recursive structural check:

```
function is_subtype(s, t):
  match (s, t) with

  | (TBase a, TBase b) ->
      -- Check subtype relation in the base type hierarchy
      subtype_of_base(a, b)       -- e.g., Int <: Number

  | (TArrow(s1, s2), TArrow(t1, t2)) ->
      -- Functions: contravariant in argument, covariant in result
      is_subtype(t1, s1) &&       -- note: reversed (contravariant)
      is_subtype(s2, t2)

  | (TApp(c1, args1), TApp(c2, args2)) when c1 = c2 ->
      -- Must respect variance of each parameter
      List.for_all2 (fun arg_s arg_t ->
        match variance_of_param(c1, pos) with
        | Covariant     -> is_subtype(arg_s, arg_t)
        | Contravariant -> is_subtype(arg_t, arg_s)
        | Invariant     -> arg_s = arg_t    -- must be equal
        | Bivariant     -> true             -- always ok
      ) args1 args2

  | (TVar α, t) | (t, TVar α) ->
      -- In systems with both inference and subtyping:
      -- generate constraint α <: t or t <: α
      add_constraint(...)

  | _ -> false
```

For languages with structural subtyping (TypeScript, Go interfaces, OCaml row polymorphism), the rule for records/objects is:
- `{x: A, y: B, z: C} <: {x: A, y: B}` — more fields is a subtype
- Check each required field is present with a subtype

For nominal subtyping (Java, Rust, C++):
- Only declared inheritance relationships count
- `Dog <: Animal` only if explicitly stated

---

### 3.2 Variance Inference

[rustc-dev-guide]

Variance is inferred for data types (structs, enums) by analyzing how type parameters are used in the type definition. The algorithm from "Taming the Wildcards" (PLDI 2011, Altidor et al.) used in rustc:

**Variance lattice:**
```
     * (bivariant — top)
    / \
   +   -
    \ /
     o (invariant — bottom)
```

**Constraint generation:** For each occurrence of a type parameter `X` in a field type:
- Covariant position (plain `X` in output): `V(X) <= +`
- Contravariant position (`X` in function argument): `V(X) <= -`
- Invariant position (`X` inside `&mut X`, `Cell<X>`): `V(X) <= o`
- Composed: if `X` occurs in `F<G<X>>` where `V_F(param) = -` and `V_G(param) = -`, then `X`'s contribution is `- × - = +`

**The variance transform:** `V3 = V1.xform(V2)` where `V1` is the definition-site variance of `F`'s parameter and `V2` is how `X` appears in the argument position:
```
+.xform(v) = v      -- covariant: pass through
-.xform(v) = -v     -- contravariant: flip
o.xform(v) = o      -- invariant: always invariant
*.xform(v) = *      -- bivariant: always bivariant
```

**Iteration to fixed point:** Mutual recursion between type definitions may require multiple passes. Start all parameters at `*` (bivariant, least constrained). Apply constraints. Iterate until stable. The solution always exists; the worst case is all-invariant.

**Example:**
```rust
enum Option<A> { Some(A), None }
// A appears in covariant position → V(A) = +

struct Fn1<B, C> { f: fn(B) -> C }
// B appears in contravariant (argument) position → V(B) = -
// C appears in covariant (result) position → V(C) = +

struct Cell<D> { value: UnsafeCell<D> }
// UnsafeCell is invariant in its argument → V(D) = o
```

---

### 3.3 PhantomData and Variance

[rustc-dev-guide]

When a struct uses a type parameter only through a raw pointer or unsafe code, the compiler cannot infer variance from usage. `PhantomData<T>` is the mechanism to declare variance explicitly:

```rust
struct MyVec<T> {
    ptr: *mut T,      -- raw pointer: doesn't count for variance inference
    len: usize,
    _marker: PhantomData<T>,   -- declare: MyVec is covariant in T
}
```

Without `PhantomData<T>`, rustc would make `T` invariant (safest default for raw pointers). With `PhantomData<T>`, it's covariant.

**Variants for different variance declarations:**
- `PhantomData<T>` — covariant in `T` (as if field of type `T`)
- `PhantomData<fn(T)>` — contravariant in `T` (as if consuming `T`)
- `PhantomData<fn(T) -> T>` — invariant in `T`
- `PhantomData<*mut T>` — invariant in `T` (raw pointer semantics)

**Why correctness matters:** Getting variance wrong enables unsound subtyping. If `Box<T>` were contravariant (wrong), you could coerce `Box<String>` to `Box<&str>` and then write a `String` into it through the coerced pointer, corrupting memory.

---

### 3.4 Function Type Variance

The function type rule is a foundational result (Cardelli 1985):

```
τ₁ → τ₂ is:
  - contravariant in τ₁ (the argument type)
  - covariant in τ₂ (the return type)
```

**Intuition for contravariance in arguments:**
- Suppose `Cat <: Animal`
- A function `Animal → String` is *more* useful than `Cat → String` because it accepts more inputs
- Therefore `(Animal → String) <: (Cat → String)` — the argument relationship is reversed
- This is contravariance: the subtype relation flips for argument positions

**Intuition for covariance in results:**
- A function returning `Cat` can be used anywhere a function returning `Animal` is expected (because `Cat <: Animal`)
- The result position preserves the subtype direction

**Java's covariant array flaw:**
```java
String[] strings = new String[3];
Object[] objects = strings;     // Java allows: String[] <: Object[]
objects[0] = new Integer(42);   // compiles! but throws ArrayStoreException at runtime
```
Arrays are mutable, so they should be invariant. Java made them covariant for convenience (pre-generics), inserting runtime checks to compensate. This is the canonical example of why variance matters for correctness.

---

## PART IV: TYPE REPRESENTATION IN THE COMPILER

### 4.1 Type Data Structures

Types are algebraic data types (sums of products):

```rust
// Typical type representation
enum Ty {
    Bool,
    Int(IntWidth),           // i8, i16, i32, i64, …
    Float(FloatWidth),
    Ref(Lifetime, Mutability, Box<Ty>),  // &'a mut T
    Ptr(Mutability, Box<Ty>),            // *mut T
    Tuple(Vec<Ty>),
    Array(Box<Ty>, usize),               // [T; N]
    Slice(Box<Ty>),                      // [T]
    FnPtr(Vec<Ty>, Box<Ty>),            // fn(A, B) -> C
    Adt(DefId, Substitutions),          // struct/enum with generic args
    TyVar(TyVarId),                     // inference variable
    Param(ParamIdx),                    // generic parameter T
    Alias(AliasKind, Box<Ty>, Substitutions), // type aliases, projections
    Error,                              // error sentinel
    Never,                              // ! type
    // … more variants
}
```

**Key design point:** Include an `Error` variant. When a type error is detected, return `Ty::Error` rather than panicking. All operations on `Error` should propagate `Error` without further errors — this suppresses cascading spurious errors from a single source mistake.

---

### 4.2 Type Interning

**The problem:** Types appear in millions of places in a large program. Storing them by value wastes memory and makes equality checks O(size of type).

**The solution:** Intern all types. An interning table stores one canonical copy of each unique type. All references to that type are thin pointers to the canonical copy.

```rust
// rustc-style: types are interned behind a thin pointer
type Ty<'tcx> = &'tcx TyS<'tcx>;  // 8 bytes on 64-bit

struct Interned<T>(NonNull<T>);

impl PartialEq for Ty<'_> {
    fn eq(&self, other: &Self) -> bool {
        // Pointer equality — O(1)!
        std::ptr::eq(*self as *const _, *other as *const _)
    }
}
```

**Interning procedure:**
1. Construct type term (e.g., `TyS::Ref(lifetime, mutability, inner)`)
2. Look up in hash set (structural equality for the lookup)
3. If found: return existing pointer
4. If not found: allocate in arena, insert into hash set, return pointer

**Benefits:**
- O(1) type equality (just pointer comparison)
- Canonical representation enables O(1) hashing
- Memory sharing: `Option<i32>` is stored once even if referenced 10,000 times
- Cache-friendly: small pointers fit in registers

**Tradeoff:** Types are immutable once interned. Substitution (replacing type parameters) must create new interned types rather than modifying in place.

---

### 4.3 Type Normalization

Many type systems include *type aliases* and *associated type projections* that must be resolved to canonical form.

**When to normalize:**
- Before unification: `type Foo = Bar` — `Foo` and `Bar` must compare equal
- Before subtype checks
- Before code generation (codegen needs concrete types)
- When displaying types in error messages (usually want to show user-visible names, not expanded forms)

**Normalization strategies:**

1. **Eager (expand immediately):** When a type alias is created, expand it. Simple to implement; no lazy evaluation. Downside: error messages show expanded types instead of user-declared names.

2. **Lazy (expand on demand):** Keep aliases unexpanded in the type representation. Only expand when needed (unification, codegen). Preserve user-facing names for error messages.

3. **Canonical form:** Define a total order on types and always produce the lexicographically least representation. Useful for caching and memoization.

**Projection normalization:** In Rust, `<T as Iterator>::Item` must be normalized. The solver queries: given `T: Iterator`, what is `T::Item`? This requires running the trait solver as part of normalization — they are mutually recursive.

**Normalization in rustc:** Types can be in "unnormalized" or "normalized" form. `normalize(ty, param_env)` reduces projections. Types returned from query results are generally normalized; types constructed during type-checking may not be. The compiler tracks which context requires which normalization level.

---

### 4.4 Type Aliases: Expand Eagerly vs. Lazily

**Expand eagerly (Pascal, C typedef, most ML `type` aliases):**
- Simple: substitution during type checking
- Error messages use expanded types
- Cheap to implement
- Fine for simple aliases, bad for deep aliases that lose meaning

**Keep lazily (Rust `type` aliases, TypeScript types):**
- Preserve human-readable names in error messages
- Require normalization pass before matching
- Allow recursive mentions (via nominal indirection)
- Lazy expansion in Rust: `type Result<T> = std::result::Result<T, MyError>` — expanded only when checked for compatibility

**Type alias impl trait (TAIT) in Rust:** `type Foo = impl Display` — the concrete type is *inferred* not declared. This requires an opaque type that is later revealed. The compiler tracks "opaque type definitions" separately and normalizes them only in their defining scope.

---

## PART V: ERROR MESSAGES FOR TYPE ERRORS

### 5.1 Producing "Expected X, Found Y"

The basic pattern:

```rust
fn check_expected_type(expected: Ty, found: Ty, span: Span) {
    if !types_match(expected, found) {
        emit_error(span,
            format!("expected `{}`, found `{}`",
                    display_type(expected),
                    display_type(found)));
    }
}
```

**Making messages useful:**
- Show the types in the way users declared them, not in internal form
- If both types look identical in display but differ internally (e.g., different lifetimes, different source modules), add disambiguation: `expected 'std::string::String', found 'my_crate::String'`
- For generic mismatches, show which generic parameter differs: `expected Vec<i32>, found Vec<String>` highlights that the element type differs

---

### 5.2 Tracking Which Constraint Caused the Failure

During unification, type errors can be traced back to their origin. Two techniques:

**Constraint tagging:** When adding a constraint, attach a `Cause` that records why it was generated:
```rust
struct Obligation {
    predicate: Predicate,
    cause: ObligationCause,   // where did this come from?
}

enum ObligationCause {
    FunctionArgument { call_site: Span, arg_index: usize },
    ReturnType { fn_def: DefId },
    LetBinding { binding_span: Span },
    MethodCall { method: Symbol, receiver_span: Span },
    // ...
}
```

**Error propagation in unification:** When unification fails, instead of just returning an error, return an error with context:
```
UnificationError {
  left: Ty,
  right: Ty,
  path: Vec<TypeErrorContext>,  // stack of "while unifying X with Y in context Z"
}
```

This allows printing layered error messages: "expected `i32`, found `String` (because function `foo` returns `i32` and you called it here)".

---

### 5.3 Type Variable Naming for Humans

Internally, type variables have opaque numeric IDs: `α₁`, `α₂`, `α₃`, …

For error messages, use a naming pass that assigns human-readable names. Strategy:

1. Collect all free type variables in the reported error
2. Assign single-letter names in order of first appearance: `T`, `U`, `V`, then `T1`, `U1`, …
3. For well-known patterns, use conventional names: type parameters on `Iterator` → `Item`, on `Option` → `T`, etc.
4. Alternatively, inherit the name from the source location: if a variable originated from a function parameter named `value`, call it `value_type` in the error

```rust
// rustc's approach: name type vars from their "origin"
enum TyVarOrigin {
    TypeParameterDefinition(Symbol, DefId),  // named by user
    AutoDeref(Span),                          // compiler-generated
    Coercion(Span),
    // ...
}
```

---

### 5.4 Avoiding Cascading Errors

A single type error at position X can cause dozens of spurious errors downstream. Mitigation strategies:

**Error sentinels:** When a type error occurs, produce `Ty::Error` (or equivalent). Define all operations on `Error`:
```
Error == any_type          -- equality always succeeds
Error <: any_type          -- subtyping always succeeds
any_type <: Error
unify(Error, t) = t        -- unification always succeeds
```
This lets type checking continue through error regions without cascading.

**Error deduplication:** Track which spans have already emitted errors. If a type error originates from an unresolved import or undefined name, suppress errors that are clearly downstream.

**Two-phase checking:** Check structural wellformedness first (arity, name resolution, basic syntax). Only run type inference after structural checking passes cleanly. Structural errors almost always cause spurious type errors.

**Limit error count:** rustc limits the number of type errors reported per function/module to avoid overwhelming the user with noise. Only the first N errors are shown; a note says "and N more errors".

---

## PART VI: SWIFT GENERICS AND PROTOCOL WITNESSES

[Swift]

### 6.1 Swift's Generics Overview

Swift uses a system similar to Haskell typeclasses / Rust traits, called *protocols*. Generic functions specify type constraints using protocols:

```swift
func sort<T: Comparable>(_ array: [T]) -> [T] { ... }
```

Swift uses two strategies for generic code, depending on context:

**Monomorphization (specialization):** The default for performance-critical paths. The compiler generates a separate concrete version for each type `T`. Zero runtime overhead, but increases binary size.

**Protocol witness tables (existentials):** When the concrete type is not known at compile time (e.g., stored in a collection, behind a `any Protocol` value). The protocol witness table (PWT) is a vtable-like structure.

---

### 6.2 Protocol Witness Tables

[Swift]

A protocol witness table is a per-(Type, Protocol) pair data structure. It contains:
- Function pointers for each protocol requirement
- Size/alignment metadata
- Protocol conformance records

```swift
protocol Drawable {
    func draw()
    func area() -> Double
}

// For `Circle: Drawable`, the witness table is:
// WitnessTable_Circle_Drawable = {
//   draw:  &Circle.draw,
//   area:  &Circle.area
// }
```

**Existential boxes:** A value of type `any Drawable` is an *existential box* containing:
1. An inline value buffer (usually 3 words; larger values are heap-allocated)
2. A pointer to the type's metatype (for type identity)
3. A pointer to the protocol witness table

```
┌──────────────────────────────────────┐
│ value (3 words inline or heap ptr)   │
│ metatype pointer                     │
│ witness table pointer                │
└──────────────────────────────────────┘
```

**Associated types:** Protocols with associated types (`associatedtype Element`) cannot be used as existentials directly (before Swift 5.7). The reason: the witness table for a protocol with associated types has entries that depend on the concrete type, and the compiler cannot lay out the table without knowing `Element`. Swift 5.7+ introduces *primary associated types* and `any Collection<Element: Equatable>` syntax.

---

### 6.3 Opaque Return Types (`some Protocol`)

[Swift]

`some Drawable` (opaque return type) is the dual of `any Drawable` (existential):
- `some Drawable` = "a specific, fixed type implementing `Drawable`, known at compile time but hidden from the caller"
- `any Drawable` = "any type implementing `Drawable`, resolved at runtime via PWT"

```swift
// Opaque: compiler knows the concrete type, optimizes it
func makeDrawable() -> some Drawable { return Circle(radius: 1.0) }

// Existential: runtime dispatch via witness table
func makeDrawable() -> any Drawable { return Circle(radius: 1.0) }
```

**Performance implication:** `some Drawable` is monomorphized; the caller gets a concrete type with zero overhead. `any Drawable` boxes the value and dispatches through the witness table.

---

## PART VII: DECISION GUIDE — WHICH TYPE SYSTEM TO IMPLEMENT

### 7.1 No Type System (Pure Dynamic)

**When this is right:**
- Scripting or glue languages where flexibility > correctness
- Rapid prototyping, DSLs where type safety adds friction
- Languages targeting novice programmers (type errors are confusing)
- Lox (Crafting Interpreters), Python, JavaScript, Lua, Ruby

**Cost:** Runtime overhead for tag checking; no compile-time error detection; harder to optimize; debugging type errors at runtime.

**Implementation:** Add a type tag to every value (enum tag or NaN-boxing). Check tags at operations. Throw runtime `TypeError` on mismatch.

---

### 7.2 Simple Static (Declarations-First)

**When this is right:**
- C, Pascal, FORTRAN-style languages
- Single-pass compilation is a requirement
- All types declared explicitly; no inference needed
- Tight integration with hardware/OS types

**Implementation:** Parse declarations first, build symbol table with types, then process expressions using declared types. Type check each expression against declared types. No inference required. O(N) time.

**Limitation:** No polymorphism (without generics). Every type is explicitly named. Code duplication for "the same function at different types".

---

### 7.3 HM with Let-Polymorphism — The ML Sweet Spot

**When this is right:**
- Pure or mostly-pure functional languages
- ML, OCaml, Standard ML, Haskell (base), Elm
- You want polymorphism without explicit type annotations
- Type inference should "just work" for most code

**The payoff:** Entire programs can be written without type annotations; the compiler infers principal types. Polymorphism is free — `length : ∀α. List α → Int` works for any element type.

**The cost:** Algorithm W with Rémy generalization is O(N·α(N)) — nearly linear in practice but the algorithm has some subtlety. The value restriction must be enforced.

**Implementation path:**
1. Represent types as described in §1.1
2. Implement unification with occurs check (§1.4)
3. Implement Rémy level-based generalization (§1.6)
4. Enforce value restriction (§1.7)
5. Intern types for efficiency (§4.2)

---

### 7.4 HM + Typeclasses/Traits

**When this is right:**
- Haskell, Rust, Scala
- You need ad-hoc polymorphism: different behavior per type, resolved at compile time
- `show :: Show a => a -> String` — print anything printable
- `sort :: Ord a => [a] -> [a]` — sort anything orderable

**The additional machinery:**
- Constraint solving / obligation fulfillment (§2.2–2.4)
- Coherence and orphan rules (§2.6)
- Dictionary passing or monomorphization (§2.3)
- HRTB for higher-ranked bounds (§2.7)

**Cost:** Significantly more complex type checker. Coherence checking is a whole-program analysis. Error messages for failed trait bounds can be deeply confusing.

**Decision point — dictionary vs. monomorphize:**
- Dictionary passing (Haskell default): smaller binary, runtime dispatch overhead
- Monomorphization (Rust default): larger binary, zero dispatch overhead, enables inlining

---

### 7.5 Bidirectional Type Checking + Higher-Kinded Types

**When this is right:**
- Languages with sophisticated type-level abstractions
- Functor, Monad, Applicative (`fmap :: Functor f => (a -> b) -> f a -> f b`)
- Languages where inference would otherwise be undecidable

**Bidirectional type checking:** Two modes:
- Check mode: "check that this expression has type T" (given T)
- Synth mode: "synthesize the type of this expression" (produce T)

Annotations (`expr: Type`) switch from synth to check mode. Lambdas require check mode to avoid undecidability. Better error messages because the "expected type" is known.

**Higher-kinded types (HKT):** Type constructors as parameters. `class Functor f where fmap :: (a -> b) -> f a -> f b` — `f` is not a type but a type constructor (`* → *`). Requires kind system: types have kinds (`*` for concrete types, `* → *` for constructors, etc.).

**Implementation cost:** Kind inference, kind unification, HKT makes type errors harder to diagnose.

---

### 7.6 Dependent Types

**When this is right:**
- Proof assistants (Coq, Lean, Agda, Idris)
- Safety-critical code where proofs are required
- Type-level programming that needs values in types

**What you get:** Types that depend on values. `Vec n T` is a vector of exactly `n` elements of type `T`. `n` is a value (natural number) in the type. The type checker proves arithmetic facts.

**The costs:**
- Typechecking is undecidable in general (requires theorem proving)
- Compilation is much slower
- Programmer must write proof terms
- Error messages are often incomprehensible
- Separate termination checker needed (to ensure type checking halts)

**Do not choose this** unless your primary use case is formal verification or you have years to invest in the type checker implementation.

---

## PART VIII: INTERNING, SUBSTITUTION, AND TYPE FOLDING

### 8.1 Type Substitution

Substitution is the operation of replacing type parameters with concrete types. It is the workhorse of generics:

```
subst([T → i32, U → String], Vec<T>) = Vec<i32>
subst([T → i32, U → String], fn(T, U) -> T) = fn(i32, String) -> i32
```

**Implementation with interned types:**

```rust
fn subst(ty: Ty, substs: &Substitutions) -> Ty {
    match ty {
        Ty::Param(idx) => substs[idx],   // look up replacement
        Ty::Bool | Ty::Int(_) | Ty::Float(_) => ty,   // no params, return as-is
        Ty::Ref(lt, m, inner) =>
            intern(Ty::Ref(lt, m, subst(inner, substs))),
        Ty::Tuple(fields) =>
            intern(Ty::Tuple(fields.iter().map(|f| subst(f, substs)).collect())),
        Ty::Adt(did, old_substs) =>
            intern(Ty::Adt(did, old_substs.iter().map(|t| subst(t, substs)).collect())),
        // ... all type variants
    }
}
```

**Optimization:** Before recursing, check if the type contains any of the parameters being substituted. If not, return it unchanged (no copy needed). Track this with a "has free type params" flag on each interned type.

---

### 8.2 TypeFolder Pattern (rustc approach)

Rustc uses a `TypeFolder` trait to factor out all type traversal logic:

```rust
trait TypeFolder {
    fn fold_ty(&mut self, ty: Ty) -> Ty;
    fn fold_region(&mut self, r: Region) -> Region;
    fn fold_const(&mut self, c: Const) -> Const;
    // ... default impls call super_fold_*
}

// Substitution is just a TypeFolder
struct SubstFolder<'a> { substs: &'a Substitutions }
impl TypeFolder for SubstFolder<'_> {
    fn fold_ty(&mut self, ty: Ty) -> Ty {
        if let Ty::Param(idx) = ty { self.substs[idx] }
        else { ty.super_fold_with(self) }
    }
}
```

This pattern means: implement `TypeFolder` once per operation (substitution, normalization, error recovery, collecting free variables), and all type variants are handled uniformly.

---

## PART IX: QUICK REFERENCE — SOURCES AND TAGS

| Tag | Source |
|-----|--------|
| `[HM-algo]` | Hindley (1969), Milner (1978), Damas-Milner (1982), Algorithm W |
| `[okmij.org]` | Oleg Kiselyov, "Efficient and Insightful Generalization" (2013), https://okmij.org/ftp/ML/generalization.html — documents Rémy 1988 algorithm |
| `[rustc-dev-guide]` | Rust Compiler Development Guide — traits/resolution.html, traits/chalk.html, variance.html |
| `[Swift]` | The Swift Programming Language (6.x), docs.swift.org/swift-book — Generics chapter |
| `[TAPL]` | Pierce, "Types and Programming Languages" (MIT Press, 2002) — foundational type theory |
| `[EaC]` | Cooper & Torczon, "Engineering a Compiler" (3rd ed.) — §4 Type Systems |
| `[CI]` | Nystrom, "Crafting Interpreters" — dynamic checking, tree-walk runtime |

---

## PART X: KEY PITFALLS SUMMARY

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Missing occurs check | Infinite type / infinite loop during unification | Add occurs check in unify before binding |
| Unsound generalization | Type variable from outer scope quantified inside inner let | Use Rémy levels: only quantify if `level > current_level` |
| Missing value restriction | `ref []` has type `∀α. α list ref` — can be used at incompatible types | Restrict generalization to nonexpansive (syntactic value) RHS |
| Mutable container covariance | Java `String[]` assigned to `Object[]`, then non-String stored | Invariant variance for mutable containers |
| Orphan rule violation | Two crates define same impl, downstream ambiguous | Enforce orphan rule: impl must be in crate of trait or type |
| Eager type expansion in errors | Error messages show expanded types, not user names | Normalize lazily; keep alias names for display |
| Cascading errors from single failure | 100 errors from one undefined variable | Use `Ty::Error` sentinel that suppresses downstream errors |
| No interning | O(N) type equality checks bottleneck compilation | Intern all types; use pointer equality |
