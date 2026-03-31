# Writing Contracts

## Concepts

### Contracts are the interface between proofs

A **precondition** restricts the caller: it becomes a proof obligation at every
call site. A **postcondition** helps the caller: it becomes an assumption the
caller can rely on. This asymmetry matters -- overly tight preconditions burden
callers; generous postconditions help them.

SPARK is **modular**: when proving a caller, only the callee's contracts matter,
not its body. This means contracts are the sole communication channel between
subprogram proofs.

### Preconditions over defensive code

In non-SPARK code, the standard practice is defensive programming: guard against
bad inputs inside the body, then raise or return an error. **This is an antipattern
in SPARK.** Defensive guards create branches that must be proved; error paths that
callers must handle. Instead, use preconditions to exclude invalid inputs. The body
then has clean, straight-line logic.

The caller must ensure the precondition before calling. Often the caller can *prove*
it from earlier checks, type bounds, or loop conditions -- eliminating runtime guards
entirely. This is the SPARK payoff: proof replaces defensive code.

### Frame postconditions

When a procedure takes an `in out` record but only modifies some fields, GNATprove
assumes **any** field could change. Callers lose all knowledge about unmodified fields.
Fix: add `Post => R.Field = R.Field'Old` for every unmodified field. GNATprove
proves this trivially from the body.

### Types over contracts

If the same range constraint appears across multiple subprograms, encode it in
the **type**:

```ada
subtype Positive_Altitude is Real64 range 0.0 .. MAX_ALTITUDE;
```

Stronger types make proof easier (bounds are known automatically) at the cost of
proving every assignment fits the range. This is usually a good trade-off.

Use contract-level constraints only for properties that genuinely can't live in
the type: relational properties between fields, or constraints that hold
conditionally.

### Recursive / inductive postconditions

For functions that build a result element-by-element, define the output precisely
using inductive structure rather than loose characterizations:

```ada
Post =>
  Result'Length = Input'Length
  and then Result (Input'First) = Input (Input'First)
  and then (for all I in Input'Range =>
    (if I /= Input'First then
       Result (I) = Integer'Max (Result (I-1), Input (I))));
```

This fully specifies behavior and doubles as the loop invariant template:
maintain the same property for the range processed so far.

## Patterns

### Precondition

```ada
procedure P (X : Integer) with
  Pre => X >= 0 and then X <= Integer'Last - 1;
```

### Postcondition

```ada
function F (X : Integer) return Integer with
  Post => F'Result >= X;
```

`'Old` captures pre-call values; `'Result` is the return value.

### Contract Cases

```ada
procedure P (X : in out Integer) with
  Contract_Cases =>
    (X > 0  => X = X'Old + 1,
     X = 0  => X = 0,
     others => X = X'Old);
```

Guards must be mutually exclusive. When `others` is not used, they must also
be complete (cover all inputs). When `others` is present, only disjointness
is checked.

### Global / Depends

```ada
procedure P with
  Global => (Input => A, Output => B, In_Out => C, Proof_In => D);
procedure Q with
  Depends => (Y => X, Total =>+ Incr);
```

### Frame postcondition

```ada
procedure Update_GSD (R : in out Footprint) with
  Post => R.Position = R.Position'Old;
```

### Helper function for complex conditions

```ada
function Starts_With (S : String; Prefix : String) return Boolean with
  Pre => S'Length >= Prefix'Length;
```

Extract complex expressions into helpers with their own preconditions.
Simplifies call sites and isolates the proof.

## Workflow: Status File Updates

- **Added/tightened a precondition** → add "Re-verify callers of Subprogram_Name"
  to Discovered Obligations (callers now have a new proof obligation)
- **Added a postcondition** → the subprogram itself needs re-proof (the body must
  establish it). Mark the subprogram as "In Progress" if it was "Proved"
- **Added frame postcondition** → add "Re-verify callers of Callee_Name" (callers
  may now prove things they couldn't before -- check for cascading improvements)
- **Introduced a new subtype** → add "Check assignments to Subtype_Name" (every
  assignment must now prove it fits the range)

## Subtype Self-Check

Before adding a precondition to a subprogram, ask:

> Am I about to write the same range constraint that already appears (or will
> appear) in another subprogram's contract?

If yes, **STOP**. Introduce a subtype instead. The anti-pattern is threading
the same bound through multiple preconditions:

```ada
-- BAD: same constraint in 3 places, easy to get out of sync
procedure Compute_Slant (Alt : Real64) with Pre => Alt in 10.0 .. 1.0e6;
procedure Compute_GSD   (Alt : Real64) with Pre => Alt in 10.0 .. 1.0e6;
procedure Compute_Width (Alt : Real64) with Pre => Alt in 10.0 .. 1.0e6;

-- GOOD: constraint lives in the type, used everywhere
subtype Bounded_Altitude is Real64 range 10.0 .. 1.0e6;
procedure Compute_Slant (Alt : Bounded_Altitude);
procedure Compute_GSD   (Alt : Bounded_Altitude);
procedure Compute_Width (Alt : Bounded_Altitude);
```

With subtypes, the prover gets bounds for free on every use. The cost is that
every assignment to the subtype must be proved in-range -- but this cost is
paid once at the point where the value enters the system, rather than being
replicated at every call site.

## Clamping Anti-Pattern

**Do NOT introduce clamps to make proofs easier.** A clamp silently changes
behavior for out-of-range inputs:

```ada
-- BAD: silently changes behavior, creates untestable dead code
Altitude := Real64'Min (Raw_Altitude, MAX_ALTITUDE);
-- The "else" branch (Raw_Altitude > MAX_ALTITUDE) is dead code that
-- can never be tested. In certification, this is unacceptable.

-- GOOD: make the constraint visible as a precondition or subtype
subtype Bounded_Altitude is Real64 range 0.0 .. MAX_ALTITUDE;
procedure Process (Alt : Bounded_Altitude);
-- Callers must prove Alt is in range. The constraint is visible,
-- testable, and documented.
```

Why this matters: clamps create branches where one arm handles "impossible"
inputs. This dead code cannot be exercised in testing, violating coverage
requirements in certified systems. Worse, the clamp may mask a real error in
the calling code.

The parallel with `pragma Assume` is exact: both make the proof go through
by hiding a gap, rather than closing it. Use subtypes or preconditions to push
constraints to callers, where they become visible proof obligations.

## Gotchas

- **Use `and then` / `or else`** in contracts. Short-circuit prevents evaluation
  errors in the contract itself.
- **Overflow in contracts**: write `X <= T'Last - Y` not `X + Y <= T'Last`.
- **Type predicates** are useful for record-level invariants (relationships between
  fields) but are checked on every assignment -- use sparingly.
- **Postconditions with quantifiers** execute at runtime with `-gnata`. For large
  ranges, this is expensive. See assertion policy in
  [ghost-code-and-lemmas.md](ghost-code-and-lemmas.md).

## Preconditions vs. Code Restructuring

Prefer restructuring code over adding preconditions. Every precondition restricts
callers and imposes proof obligations on them.

**Add a precondition when** there's a clear semantic reason to exclude values:
- Division by zero
- Negative input to sqrt/log
- Values that are meaningless in the domain (negative altitude, etc.)

**Restructure instead when** the subprogram could reasonably handle the edge case:
- Empty or single-element array
- Zero-length string
- Boundary values of integer ranges

The goal: keep preconditions as wide as possible so callers are unconstrained.

## Type Strengthening vs. Contracts

If the same constraint appears across multiple subprograms, it likely belongs
in the **type**, not repeated in contracts:

```ada
-- Better: constraint lives in the type
subtype Positive_Altitude is Real64 range 0.0 .. MAX_ALTITUDE;

-- Worse: same constraint repeated in every Pre/Post
procedure P (Alt : Real64) with Pre => Alt in 0.0 .. MAX_ALTITUDE;
procedure Q (Alt : Real64) with Pre => Alt in 0.0 .. MAX_ALTITUDE;
```

Strengthen a type when there's a clear rationale: requirements, real-world
knowledge, or the semantics of consumers (sqrt needs non-negative input).

Stronger types make proof easier (GNATprove gets bounds for free) but every
assignment must be proven to fit the range. This is usually a good trade-off.

## References

- [Subprogram Contracts](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/source/subprogram_contracts.html) -- Pre, Post, Contract_Cases, data/flow dependencies
- [How to Write Subprogram Contracts](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/source/how_to_write_subprogram_contracts.html) -- Practical guidance for integrity and correctness
- [Generation of Dependency Contracts](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/source/how_to_write_subprogram_contracts.html#generation-of-dependency-contracts) -- Auto-generated Global and Depends
- [Writing Contracts for Functional Correctness](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/source/how_to_write_subprogram_contracts.html#writing-contracts-for-functional-correctness) -- Postconditions for full functional specs
