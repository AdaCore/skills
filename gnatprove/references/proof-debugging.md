# Investigating Unproved Checks

## Concepts

### Six root causes

Every unproved check falls into one of these categories:

| # | Cause | Fix |
|---|-------|-----|
| 1 | Bug in implementation | Fix the code |
| 2 | Assertion too strong | Weaken or correct it |
| 3 | Missing annotation | Add precondition, postcondition, or loop invariant |
| 4 | GNATprove modeling limitation | Use a lemma or introduce an axiom **only with user permission** |
| 5 | Insufficient proof time | Raise `--level` or `--steps` |
| 6 | Prover can't handle the logic | Custom lemma or restructure code |

Causes 1-3 account for the vast majority of proof failures above `--level=2`.

Cause 4 can often been detected by using `--info` and analyzing messages that
SPARK provides indicating imprecise modeling.

Causes 5-6 are rarely permanent at `--level=2` -- exhaust all other
possibilities before concluding the prover is the problem.

The most common mistake is declaring "the SMT solver can't prove this" too
early. In practice, nearly every failure is: wrong code, missing information,
overwhelming context, or a non-inductive invariant. "The prover is too weak"
is the rarest cause.

### `--limit-line` as a diagnostic tool

`--limit-line=file.adb:NN` proves **exactly** line NN, assuming all assertions
above it (even unproved ones). This is powerful for what-if exploration:

1. Insert `pragma Assert (P)` above the failing check
2. Run `--limit-line` on the check
3. If it proves, P is sufficient -- go prove P separately
4. If not, P isn't enough -- try a different property

This avoids re-proving the whole subprogram during exploration. You **must**
separately prove any assertions you insert.

### GNATprove's "possible fix" suggestions

These are hand-coded heuristics aimed at novice users. Treat them as
directional hints, not commands. They're often right about *what's missing*
but may not suggest the best form of the fix.

## Decision Tree

### Step 1: Is the code correct?

Compile with `-gnata` and run tests. If the check fires at runtime,
it's a code bug.

### Step 2: Missing annotations?

Look for GNATprove hints like "subprogram should mention Var in a precondition."
Add the suggested contract (adjusting form as needed).

### Step 3: Assert breadcrumbs

Binary search for where the proof breaks. Place assertions between where a
property should hold and where GNATprove loses it. If the midpoint proves,
the problem is later.

**Once the missing fact is identified**: if it is an arithmetic ordering or
monotonicity property (e.g. `A * B ≤ C`, `X / Y ≤ X`, monotonicity of a
computed value), jump to Step 5a — the SPARK Lemma Library may have it, and
a lemma call often makes raising the level unnecessary.

### Step 3a: Need a bound on an input?

If the fix requires bounding a parameter's value, add a precondition on the
current subprogram *or* introduce a subtype — do NOT add a runtime guard
`(if X > Bound then return)`.
Guards are defensive code, which is a SPARK antipattern (see
[contracts.md](contracts.md)). Push the constraint to the caller via Pre. If
this creates a cascade of preconditions, consider whether the bound belongs in
a subtype instead.

### Step 4: Frame condition?

If a property was established but a call with `in out` makes GNATprove forget
it, the callee needs a frame postcondition. See [contracts.md](contracts.md).

### Step 5: Increase level

Raise `--level` (0 -> 1 -> 2). Beyond 2 requires user permission. Use
`--steps=N` for reproducibility.

### 6: Lemmas

**Before writing a custom lemma, check whether `SPARK.Lemmas.*` has what you need.**

First, verify the SPARK.Lemmas is available in the project:

1. Read the top-level GPR file (the one passed to gnatprove with `-P`).
2. Collect all `with "..."` imports (recursively).
3. For each imported GPR file, check whether it contains `extends "sparklib_internal"`.

If no imported GPR extends `sparklib_internal`: **stop and ask the user to add the sparklib dependency before continuing.** Do not write a custom lemma as a workaround for a missing library import.

If available: find the relevant package in [ghost-code-and-lemmas.md](ghost-code-and-lemmas.md) and call the lemma at the proof point. If `SPARK.Lemmas.*` has nothing applicable, write a custom lemma.


### Step 7: Last resort (requires user permission)

`pragma Assume` or `pragma Annotate (GNATprove, False_Positive, ...)`.
Document the justification.

## Workflow: Status File Updates

When investigating a failing check:
- **Added Assert breadcrumb** → add "Prove Assert at file:NN" to Discovered Obligations
- **Added/modified a precondition** → add "Re-verify callers of Subprogram" to Discovered Obligations
- **Added frame postcondition** → add "Re-verify callers of Callee" to Discovered Obligations
- **Wrote a ghost lemma** → add "Prove Lemma_Name body" to Discovered Obligations
- **Check now proves** → mark it done in the status file
- **Increased proof level** → note the level in the status entry

## Message Reference

### Severity levels

GNATprove messages fall into four separate categories:

- **Errors**: SPARK violations, legality issues. Must fix.
- **Check messages** (`high` / `medium` / `low`): Unproved checks, ranked by priority.
- **Warnings**: Suspicious but non-blocking situations.
- **Info**: Proved checks (visible with `--report=all`).

### Common check categories

| Check | Meaning |
|-------|---------|
| `overflow check` | Arithmetic exceeds type bounds |
| `range check` | Value outside subtype constraint |
| `index check` | Array index out of bounds |
| `division check` | Division by zero |
| `precondition` | Caller doesn't establish callee's Pre |
| `postcondition` | Body doesn't establish Post |
| `loop invariant init` | Invariant false on first iteration |
| `loop invariant preservation` | Invariant not maintained across iteration |
| `discriminant check` | Wrong variant accessed |

### Counterexample interpretation

Counterexamples may not represent feasible paths. "Impossible" values
usually indicate a missing precondition, not a real bug. Use
`--counterexamples=on` (default at level >= 2).

## References

- [How to Investigate Unproved Checks](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/source/how_to_investigate_unproved_checks.html) -- Incorrect code, unprovable properties, prover shortcomings
- [Understanding Counterexamples](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/gnatprove.html#understanding-counterexamples) -- Interpreting counterexample output
- [Categories of Messages](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/gnatprove.html#categories-of-messages) -- Info, warning, check, and error message types
