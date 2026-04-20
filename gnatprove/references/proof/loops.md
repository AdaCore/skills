# Proving Loops

## Concepts

### Loops are cut points

After a loop, GNATprove knows only that modified variables satisfy their type
constraints. Every other property is forgotten unless stated in a `Loop_Invariant`.
This is fundamental: the invariant is the **only** way to carry knowledge across
iterations and out of the loop.

"Modified" includes both direct assignment and modification through a procedure
call with an `out` or `in out` parameter. If a loop body calls a procedure that
takes a record `in out`, GNATprove forgets everything about that record unless:
- The **callee** has a postcondition preserving or establishing the property, AND
- A **`Loop_Invariant`** restates it

Either alone fails. The callee postcondition tells GNATprove the property holds
after each call. The invariant carries it across iterations.

### The invariant is an induction hypothesis

GNATprove proves a loop invariant the same way you prove induction:
1. **INIT**: The invariant holds on the first iteration (base case)
2. **PRESERVE**: If it held on iteration N, it holds on N+1 (inductive step)

Two derived obligations follow:
3. **INSIDE**: The invariant (plus the loop condition) proves checks inside the body
4. **AFTER**: The invariant (plus the negated loop condition) proves checks after the loop

A common failure: the invariant is true but not **inductive** -- it doesn't reference
the loop index or otherwise tie itself to the loop's progress.

### Termination

`for` loops terminate by construction. `while` loops require a `Loop_Variant`:
a quantity that strictly decreases (or increases) each iteration.

Prefer `for` loops with `exit when` over `while` loops when the iteration count
has a known static bound. The `for` gives GNATprove the bound for free.

## Patterns

### Counting variable

```ada
pragma Loop_Invariant (Count <= I - A'First + 1);
pragma Loop_Invariant (Count <= A'Length);
```

### Quantifier (processed-so-far)

```ada
pragma Loop_Invariant (for all K in A'First .. I => A(K) = 0);
```

### Witness / existential

```ada
pragma Loop_Invariant
  (for all K in A'First .. I =>
     (for some J in A'First .. K => Result(K) = Input(J)));
```

### Conditional update (modified-if, else-unchanged)

```ada
pragma Loop_Invariant
  (for all K in A'First .. I =>
     A(K) = (if Condition(K) then New_Value else A'Loop_Entry(K)));
```

### Processed / unprocessed partition

```ada
pragma Loop_Invariant
  (for all K in A'First .. I - 1 => A(K) = New_Value);
pragma Loop_Invariant
  (for all K in I .. A'Last => A(K) = A'Loop_Entry(K));
```

Use when the postcondition promises the unprocessed portion is untouched.

### Relaxed initialization tracking

```ada
Result : Integer_Array (A'Range) with Relaxed_Initialization;
-- in loop:
pragma Loop_Invariant (Result (A'First .. I)'Initialized);
```

Use `(others => 0)` only for buffers where you can't predict which indices
will be written.

### Buffer + slice (variable-length result)

```ada
Buffer : Integer_Array (1 .. Input'Length) := (others => 0);
Count  : Natural := 0;
-- in loop:
   Count := Count + 1;
   Buffer (Count) := Input (I);
   pragma Loop_Invariant (Count <= I - Input'First + 1);
   pragma Loop_Invariant (Count <= Input'Length);
-- after loop:
return Buffer (1 .. Count);
```

### Two-direction variant

```ada
pragma Loop_Variant (Decreases => Remaining, Increases => Divisor);
```

Useful when a loop progresses by either shrinking a value or advancing an index.

## Workflow

### Writing invariants

1. **Comment**: Write a plain-English comment explaining what's true at the
   invariant point and why
2. **Formalize**: Translate the comment into Ada boolean expressions
3. **Reconcile**: If the formal version diverges from the comment, update the
   comment -- the comment was wrong, not the formalization

The invariant must capture exactly what has been accomplished so far -- typically
a quantifier over the range `First .. I` or a bound on an accumulator.

### Nested loops

Work **innermost to outermost**. The inner loop's invariant must restate any
property about data it modifies. The outer loop's invariant must account for
the inner loop's cumulative effect.

### Status file updates

- **Added a loop invariant** → add "Prove loop invariant init + preservation at
  file:NN" to Discovered Obligations
- **Added callee postcondition to support invariant** → add "Re-verify callers
  of Callee_Name" to Discovered Obligations

### Debugging broken invariants

1. **Test at runtime**: Compile with `-gnata` and run -- if it fails, it's wrong
2. **Check inductiveness**: Does it reference the loop index?
3. **Check quantifier bounds**: Off-by-one is the most common invariant bug
4. **Step back**: Re-examine the plain-English comment from step 1
5. **Increase level to 2**: Only after confirming the invariant is correct

## Gotchas

- **Placement**: Multiple invariants must be grouped (no intervening statements,
  except `Loop_Variant` and `Annotate` pragmas may be interleaved).
- **Automatic unrolling**: GNATprove unrolls simple for-loops (< 20 iterations,
  no invariant, only scalar local variables). Suppress with
  `pragma Loop_Invariant (True)` or `--no-loop-unrolling`. Note: the
  "scalar locals only" restriction applies to **unrolling**, not to
  `Loop_Invariant` itself -- non-scalar variables are fine in loops with
  explicit invariants.

## References

- [How to Write Loop Invariants](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/source/how_to_write_loop_invariants.html) -- Four properties, proving across iterations, examples
- [Loop Invariants](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/source/assertion_pragmas.html#loop-invariants) -- Language reference for pragma Loop_Invariant
- [Loop Variants](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/source/assertion_pragmas.html#loop-variants) -- Language reference for pragma Loop_Variant
- [Attribute Loop_Entry](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/source/specification_features.html#attribute-loop-entry) -- Referencing values from before loop entry
