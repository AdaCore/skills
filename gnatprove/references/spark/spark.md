# Writing SPARK

This section covers the SPARK language — contracts, types, ghost code, and access
types — and the coding principles that make SPARK code provable.

## Core Principles

### Types over contracts

If a range constraint applies to a value everywhere — in a record field, every
caller, every use — encode it in the subtype rather than threading it through every
`Pre`/`Post`/invariant. Subtypes are checked at every assignment; contracts are
only checked at call boundaries. Fewer contracts means fewer proof obligations.

### Preconditions over defensive code

In non-SPARK code, the standard practice is defensive programming: guard against
bad inputs inside the body, then raise or return an error. **This is an antipattern
in SPARK.** Defensive guards create branches that must be proved and error paths
callers must handle. Use preconditions to exclude invalid inputs instead — the body
then has clean, straight-line logic. Callers prove the precondition holds before
calling, often from type bounds, earlier checks, or loop conditions.

### Forbidden without explicit user permission

| What | Why not |
|------|---------|
| `SPARK_Mode (Off)` on a body | The body is entirely unverified — not a workaround, giving up. Most operations that *seem* unverifiable are not; heap-mutating operations are the rare genuine exception. |
| Value clamps (e.g. `X := T'Min(X, MAX)`) | Silently changes behavior for out-of-range inputs. Creates dead code unreachable in testing — anathema in certification work. Use subtypes or preconditions to push the constraint to the caller instead. |

## Reference Files

| File | When to read |
|------|--------------|
| [contracts.md](contracts.md) | `Pre`, `Post`, `Global`, `Depends`, `Contract_Cases`; frame postconditions; the clamping anti-pattern |
| [ghost-code-and-lemmas.md](ghost-code-and-lemmas.md) | Ghost variables and functions; `pragma Assert`; lemmas vs. axioms; hint strength ordering |
| [access-types.md](access-types.md) | Ownership model; borrowers and observers; null exclusion; allocators |
