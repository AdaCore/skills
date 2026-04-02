# Ghost Code, Lemmas, and Axioms

## Concepts

### Ghost code is proof-only

Ghost entities (`with Ghost`) exist solely for verification. They are stripped
during normal compilation -- zero runtime cost. Ghost code can read non-ghost
state but cannot affect it.

### Hint strength ordering

Always reach for the lightest tool first:

1. **`pragma Assert`** -- a breadcrumb. If discharged, the prover uses it as
   a known fact for subsequent checks. Try this first.
2. **Ghost lemma** (`is null` or with a body) -- a provable ghost procedure.
   GNATprove verifies the postcondition, then callers get it for free.
3. **Ghost axiom** (`with Import`) -- an assumed-true postcondition that
   GNATprove does **not** verify. This is an unchecked assumption that
   threatens soundness. **Never introduce without explicit user permission.**

### Expression functions vs. regular ghost functions

An **expression function** generates an implicit postcondition derived from its
expression body. Callers see this postcondition automatically -- no explicit
`Post` needed:

```ada
function Is_Sorted (A : Arr) return Boolean is
  (for all I in A'First .. A'Last - 1 => A(I) <= A(I+1))
with Ghost;
-- Implicit Post: Is_Sorted'Result = (for all I in ...)
```

A **regular function**'s body is opaque at call sites. The prover only sees
the explicit postcondition. Prefer the expression function form so the prover
gets the definition for free.

**Visibility caveat**: If an expression function is *defined* in a package body
(even if *declared* in the spec), the implicit postcondition is only available
within the same unit, not to external callers. Define expression functions in
the spec to make the implicit postcondition universally available.

### Connecting runtime code to ghost specs

When a runtime helper computes the same property as a ghost expression function,
bridge them with a conditional postcondition:

```ada
function Is_Palindrome_Rt (S : String) return Boolean with
  Post => (if Is_Palindrome_Rt'Result then Is_Palindrome (S));
```

The `if ... then` form is deliberate: you only need the True-implies-ghost
direction. Proving full equivalence is harder and usually unnecessary.

### SPARK lemma library

Standard lemmas in `SPARK.Lemmas.*` provide **relational** properties
(monotonicity, ordering). Call them at the proof point:

```ada
Lemma_Div_Is_Monotonic (Val1 => X, Val2 => Y, Denom => Z);
-- Prover now knows: X <= Y implies X/Z <= Y/Z (for Z > 0)
```

These are not for simple range/overflow facts -- use preconditions for those.

| Package | Domain |
|---------|--------|
| `SPARK.Lemmas.Integer_Arithmetic` | Signed integer |
| `SPARK.Lemmas.Long_Float_Arithmetic` | Long_Float monotonicity |
| `SPARK.Lemmas.Float_Arithmetic` | Float monotonicity |
| `SPARK.Lemmas.Mod32_Arithmetic` | Modular 32-bit |

## Patterns

### Null-body lemma (try first)

```ada
procedure Lemma_Product_Bounded (A : Positive_Float; B : Unit_Float)
with Ghost, Pre => B in 0.0 .. 1.0, Post => A * B <= Positive_Float'Last;
procedure Lemma_Product_Bounded (A : Positive_Float; B : Unit_Float) is null;
```

GNATprove often proves the postcondition from parameter subtypes alone.

### Inductive / recursive lemma

```ada
procedure Lemma_Mono (I : Natural) with Ghost,
  Pre => I >= 1, Post => F(I-1) < F(I),
  Subprogram_Variant => (Decreases => I);
procedure Lemma_Mono (I : Natural) is
begin
   if I > 1 then Lemma_Mono (I - 1); end if;
end;
```

### Case-splitting lemma loop

For proving a quantifier over a multi-region array:

```ada
procedure Lemma_Is_Palindrome (R : String; ...) with Ghost,
  Post => Is_Palindrome (R);
procedure Lemma_Is_Palindrome (R : String; ...) is
begin
   for I in 0 .. R'Length / 2 - 1 loop
      pragma Loop_Invariant
        (for all J in 0 .. I-1 => R(R'First+J) = R(R'Last-J));
      if I < Boundary then
         pragma Assert (...);  -- case 1
      else
         pragma Assert (...);  -- case 2
      end if;
      pragma Assert (R(R'First+I) = R(R'Last-I));
   end loop;
end;
```

### `Assert_And_Cut` (reduce proof context)

```ada
pragma Assert_And_Cut (Key_Property);
```

Verifies the property, then forgets all preceding context. Useful when
accumulated facts overwhelm the prover.

### Assertion policy (disable expensive ghost at runtime)

Since proofs guarantee correctness, runtime ghost execution is redundant.
Place directly in source files (not `.adc`):

In `.ads`:
```ada
pragma Assertion_Policy (Post => Ignore);
```

In `.adb`:
```ada
pragma Assertion_Policy (Ghost => Ignore, Loop_Invariant => Ignore,
                         Assert => Ignore, Post => Ignore);
```

Keep `Pre => Check` -- preconditions catch caller bugs and are cheap.

For proof-helper loops that shouldn't execute at runtime, move them
into a ghost procedure (erased entirely with `Ghost => Ignore`).

## Workflow: Status File Updates

- **Wrote a ghost lemma** → add "Prove Lemma_Name body" to Discovered Obligations
  (even `is null` lemmas generate a proof obligation that GNATprove must discharge)
- **Introduced an axiom** (with user permission) → add "AXIOM: Axiom_Name --
  assumed, not proved" to Discovered Obligations as a permanent note
- **Added Assert_And_Cut** → add "Verify Assert_And_Cut at file:NN" to
  Discovered Obligations (the assertion itself must prove)

## Gotchas

- **Naming convention**: `Lemma_<property>` (proved), `Axiom_<property>` (assumed).
  Makes proof status visible at a glance.
- **Ghost can read non-ghost but not vice versa.** Non-ghost code cannot
  reference ghost entities.
- **Ghost procedures must have no non-ghost outputs.** No writing to non-ghost
  globals, no `out` parameters of non-ghost type.
- **Axioms are unchecked.** An incorrect axiom makes the entire proof unsound.
  Always justify why the property is true but unprovable in SPARK.

## References

- [Ghost Code](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/source/specification_features.html#ghost-code) -- Ghost functions, variables, types, procedures, packages
- [SPARK Lemma Library](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/source/spark_libraries.html#spark-lemma-library) -- Distributed ghost lemma procedures for arithmetic
- [Pragma Assert_And_Cut](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/source/assertion_pragmas.html#pragma-assert-and-cut) -- Cutting proof context to simplify verification
- [Expression Functions](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/source/specification_features.html#expression-functions) -- Inlined specification functions usable in contracts
