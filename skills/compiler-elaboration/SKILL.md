---
name: compiler-elaboration
description: Dependent type elaboration expert — covers dependent type theory, universe hierarchies, elaboration from surface syntax to core calculus, and proof-relevant type checking (Lean4/Agda style). Use when implementing a dependently-typed language or proof assistant.
---

# Compiler Elaboration (Dependent Types)

## Overview

Elaboration is the process of translating a **surface term** (what the programmer writes) into a **core term** (a fully explicit, type-annotated term in a dependent core calculus). The core calculus has no implicit arguments, no syntactic sugar, and a straightforward type checking algorithm (type checking in the core is decidable and mechanical). The elaborator bridges the gap between the user-facing language and the formally verified core.

This is the central pass in dependently typed languages and proof assistants (Lean 4, Agda, Idris, Coq/Rocq).

---

## What Elaboration Is

**Surface term**: what the user writes. May contain:
- Omitted arguments (implicit/instance arguments)
- Unresolved overloading
- Syntactic sugar (`do` notation, `match` patterns, `where` clauses)
- Underscores `_` (holes to be filled by inference)
- Type ascriptions that need checking

**Core term**: fully explicit. Every argument is present. Every type annotation is filled in. The type checker for the core needs no heuristics — it is a simple recursive descent type checker.

**Elaboration = type-directed translation**: the expected type at each node guides how ambiguities are resolved. The expected type flows inward (top-down, like inherited attributes) while inferred types flow outward (bottom-up, like synthesized attributes).

---

## Universe Levels

Dependent type theories use a **universe hierarchy** to avoid Russell's paradox (`Type : Type` is inconsistent). Instead:

- `Type 0` (often written `Prop` or `Set`) contains ordinary propositions/small types
- `Type 1` contains `Type 0`, and so on
- `Type i : Type (i+1)` for all `i`

**Universe polymorphism**: rather than fixing a level, functions can be polymorphic over universe levels. [Lean4] uses `Sort u` for universe-polymorphic definitions; `Prop = Sort 0`, `Type u = Sort (u+1)`. [Agda] uses explicit `Level` parameters with `lzero`, `lsuc`, and `_⊔_` (max).

**Universe constraints**: the elaborator accumulates constraints between universe level variables and solves them. [Lean4] uses `Level.imax u v` for Pi types (handles impredicativity: `imax u 0 = 0`).

**Typical universe rules**:
- `Pi (x : A) B : Sort (imax u v)` where `A : Sort u`, `B : Sort v`
- `Sigma (x : A) B : Sort (max u v)`
- `Prop` is impredicative: `Pi (x : A) P : Prop` whenever `P : Prop`

See `references/elaboration-reference.md` Section 4 for full details including Agda `Level` arithmetic and `Setω`.

---

## Implicit Argument Inference

**Implicit arguments** are enclosed in `{...}` (Lean 4/Agda) or `[...]` (instance arguments in Lean 4), `{{...}}` (instance arguments in Agda). They are omitted from the surface syntax and must be inferred by the elaborator.

**Inference mechanism**:
1. At each application site, insert a *metavariable* (unification variable) `?M` for each omitted implicit argument
2. Unify the expected type with the actual type of the application — this constrains `?M`
3. Solving the unification problems fills in the metavariables

**Insertion heuristics**: insert implicit arguments eagerly as long as the result type doesn't match the expected type without insertion. Lean 4's strict implicits `⦃⦄` are only inserted when the next explicit argument follows.

**Instance arguments** (`[TC A]` in Lean 4, `{{TC A}}` in Agda): resolved by *backward chaining* typeclass search rather than by unification.

See `references/elaboration-reference.md` Section 3 (Implicit Arguments) and Section 6 (Instance Search) for full details.

---

## Unification and Constraint Solving

Elaboration requires **higher-order unification**, which is undecidable in general. In practice, elaborators implement:

**Pattern unification** (Miller, 1991): a decidable fragment. A flex-rigid equation `?M x₁ ... xₙ = t` where `x₁ ... xₙ` are distinct bound variables has a unique solution if `t` is a term whose free variables are a subset of `{x₁ ... xₙ}`. The solution is `?M := λ x₁ ... xₙ. t`.

**Postponing constraints**: when a unification problem cannot be solved immediately (because it is flex-flex or the rigid head is not yet known), it is added to a constraint queue. After other parts of the term are elaborated and more metavariables are filled in, the postponed constraints are retried. [Lean4]: elaborators throw `Except.postpone`; synthetic metavariables carry solving strategies (`typeClass`, `coe`, `tactic`, `postponed`).

**Metavariable dependency tracking**: metavariables have a local context (the variables that were in scope when the hole was created). A solution to a metavariable must be a closed term relative to its local context.

**Definitional equality**: in dependent type theory, types are equal up to β, η, δ, ι reduction. The unifier must reduce terms to WHNF before comparing heads. [Lean4]: `MetaM.isDefEq` checks equality and assigns metavariables. `MetaM.whnf` computes weak head normal form. Transparency mode controls δ-unfolding.

See `references/elaboration-reference.md` Section 5 for full pattern unification algorithm and flex-flex handling.

---

## Elaboration Algorithm Sketch

```
elab : Context → SurfaceTerm → ExpectedType? → (CoreTerm × CoreType)

elab ctx (App f arg) expectedTy =
  (fCore, fTy) ← elab ctx f None
  fTy' ← whnf fTy         -- weak head normal form
  match fTy' with
  | Pi x A B →
      argCore ← check ctx arg A
      return (App fCore argCore, subst x argCore B)
  | _ →
      -- try inserting implicit arg
      ...

check : Context → SurfaceTerm → CoreType → CoreTerm
check ctx t ty =
  (tCore, inferredTy) ← infer ctx t
  unless (defEq ctx ty inferredTy) do
    throw TypeError
  return tCore
```

---

## Elaboration Passes in Lean 4

[Lean4] Lean 4's elaborator is structured as a monad stack:
```
CoreM → MetaM → TermElabM → TacticM → CommandElabM
```

Key types: `Lean.Expr` (core AST), `Lean.Level` (universe levels), `Lean.MVarId`, `Lean.LocalContext`, `Lean.MetavarContext`.

Key sub-passes:
1. **Pre-elaboration**: expand macros (`Lean.Macro`), desugar syntax
2. **Term elaboration**: bidirectional type checking — `TermElabM.elabTerm : Syntax → Option Expr → TermElabM Expr`
3. **Instance synthesis**: `Meta.synthInstance : Expr → MetaM (Option Expr)`
4. **Defeq checking**: `MetaM.isDefEq : Expr → Expr → MetaM Bool` (assigns metavariables)
5. **WHNF**: `MetaM.whnf : Expr → MetaM Expr`
6. **Universe level solving**: solve accumulated `Level` constraints

See `references/elaboration-reference.md` Section 8 for full Lean 4 architecture with function signatures.

---

## Agda Implementation Notes

Key Agda-specific elaboration concepts:
- **Dot patterns**: `.t` in a pattern means `t` is *forced* (determined by unification from other arguments, not by matching)
- **With abstraction**: `with e` replaces all occurrences of `e` in the goal with a fresh variable, enabling case-split on `e`
- **Mutual blocks**: `mutual` keyword — multiple definitions elaborated together with shared context
- **Abstract blocks**: `abstract` hides definition bodies from outside the block
- **Instance overlap**: `OVERLAPPABLE`, `OVERLAPPING`, `OVERLAPS`, `INCOHERENT` pragmas control which instance wins when multiple candidates exist

See `references/elaboration-reference.md` Section 9 for full Agda notes with code examples.

---

## Deep Reference

For comprehensive coverage of all topics (bidirectional type checking rules, universe level arithmetic, pattern unification algorithm, instance search, Lean 4 monad architecture, Agda implicit/instance argument details, and implementation decision guide), see:

**`references/elaboration-reference.md`** — the primary deep-dive reference for this skill.

Sections covered in the reference:
1. What Elaboration Is (surface vs core, why separate elaborator)
2. Bidirectional Type Checking (inference/checking modes, formal rules, algorithm sketch)
3. Implicit Arguments (metavariables, unification, Agda `{A}` vs `{{inst}}`, Lean 4 `{α}` vs `[inst]`)
4. Universe Levels (`Sort`/`Type`/`Prop` hierarchy, Russell's paradox, `Level` arithmetic, `imax`, `Setω`)
5. Metavariables and Unification (pattern unification / Miller's algorithm, flex-flex, postponement, transparency modes)
6. Instance Search (backward chaining, coherence, overlap pragmas, debugging)
7. The Elaboration Algorithm (phases 0–3 with pseudocode)
8. Lean 4 Architecture (MetaM / TermElabM / TacticM stack, key functions)
9. Agda Notes (dot patterns, with abstraction, tactic arguments)
10. Implementation Decision Guide (HM vs bidirectional vs full elaboration, cost estimates, minimal subset)

---

## Sources Used

- Lean 4 Metaprogramming Book (leanprover-community.github.io) — MetaM, TermElabM, Expr constructors
- Agda docs: `implicit-arguments`, `universe-levels`, `instance-arguments` (agda.readthedocs.io)
- Dunfield & Krishnaswami (2019), arXiv:1908.05839 — bidirectional typing survey
- Ulf Norell's thesis (2007) — Agda's original elaboration design
- Andreas Abel — NbE for definitional equality
- Miller (1991) — pattern unification algorithm

---

## Implementation Checklist

- [ ] Define core calculus (MLTT: Pi, Sigma, Id, universe hierarchy)
- [ ] Define surface syntax with implicit arguments and holes
- [ ] Implement metavariable/unification variable infrastructure
- [ ] Implement weak head normal form (WHNF) reduction
- [ ] Implement definitional equality check (with β, η, δ reduction)
- [ ] Implement pattern unification (Miller's fragment)
- [ ] Implement constraint queue for postponed unification problems
- [ ] Implement implicit argument insertion heuristics
- [ ] Implement universe level variables and constraint solving
- [ ] Implement instance/typeclass search
- [ ] Implement surface-to-core desugaring (do-notation, match, where)
- [ ] Produce useful elaboration error messages (show the inferred vs. expected type, the hole context)

---

## Navigation

**Trace Up ↑** — skills that route here
- `compiler-type-system` — elaboration is the full-powered version of type checking for dependent types
- `compiler-expert` — routes here on dependent types / universe levels / proof assistant questions

**Trace Down ↓** — next skills after elaboration
- `compiler-mir` — the elaborated core term is lowered to a control-flow IR for evaluation/compilation

**Related →** — closely related skills
- `compiler-type-system` — elaboration subsumes type checking in the dependent-type setting
- `compiler-comptime` — dependent types and full comptime evaluation converge at the limit
