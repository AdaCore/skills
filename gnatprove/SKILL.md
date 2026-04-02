---
name: gnatprove
description: This skill should be used when the user wants to "prove SPARK code", "run GNATprove", "investigate unproved checks", "write contracts or invariants", "add loop invariants", "write ghost code or lemmas", or is working on SPARK formal verification. Provides a full GNATprove proof campaign workflow.
license: Apache-2.0
metadata:
  version: "0.1.0"
---

# GNATprove SPARK Verification Skill

## Quick Start

To prove SPARK code, run GNATprove via Bash with the project's environment.

```bash
  gnatprove -P <project.gpr> --quiet <arguments>
```

Always prefer to run with `--quiet` as this will suppress output
about analysis phases, which is mostly there to show users progress. If there
are no non-info messages output, the proof succeeded.

You can find more information about the last run by looking for `gnatprove.out`.
The precise directory under which it appears is project dependent, but it's
generally `objs/release/gnatprove/gnatprove.out` or similar;
refer to [gnatprove-out.md](references/gnatprove-out.md) for more information about how to
work with this file.

***Important***: running `gnatprove` can be very time-consuming. If you don't
find what you're looking for in its output, *prefer to analyze `gnatprove.out`
rather than re-running `gnatprove`*.

Common arguments:
- `-jN` where 0 means as many threads as possible and N>0 means use that many parallel threads
- `--level=1` (proof level 0-4; start low, increase only if needed)
- `--limit-subp=file.adb:NN` (prove one subprogram; NN = **declaration** line, not body)
- `--limit-line=file.adb:NN` (prove exactly one check; assumes assertions above)
- `--output=oneline` (one check per line; good for status assessment)
- `--mode=stone` (SPARK legality), `--mode=bronze` (flow), `--mode=all` (flow + proof, default)
- `-f` (force full reanalysis)

Note: with `--limit-subp` or `--limit-line`, the filename must NOT include the `src/`
prefix -- the GPR file handles source directories.

## Proof Status File

At the start of a proof campaign, create (or load) a proof status file to track
obligations and progress. Default location: `proof-status.md` alongside the code
being proved. This file is the persistent record of what's proved, what's in
progress, and what work items have been discovered.

See [workflow.md](references/workflow.md) for the format and rules for updating it.

## Workflow (summary; see [workflow.md](references/workflow.md) for details)

1. **Create proof status file** with top-level entry for the requested scope
2. **Assess first**: Run GNATprove with `--output=oneline` before deep-reading code
3. **Decompose**: Add a status entry for each subprogram with unproved checks
4. **Start at a leaf**; delegate Steps 3-6 to a subagent per subprogram
5. **Review** each proved subprogram for antipatterns (Step 7)
6. **Widen scope** with `-f` and finalize (Step 8)
7. **Increase level** only up to 2; beyond 2 requires user permission

## Before You Start (Mandatory)

Before writing any annotations or running any proofs beyond the initial assessment:

1. **Create `proof-status.md`** alongside the code being proved. This is NOT optional
   and is NOT replaced by any other document the user asks you to maintain. The format
   is in [workflow.md](references/workflow.md). Maintain both the proof status file and any
   user-requested work log.

2. **Identify subtype candidates.** Scan the failing checks for recurring range
   constraints (e.g., altitude bounds, FOV bounds, resolution bounds). If you see the
   same constraint appearing in 2+ preconditions, STOP and introduce a subtype before
   adding more preconditions. See the "Subtype Self-Check" in [contracts.md](references/contracts.md).

3. **Assess decomposition.** For any subprogram with >30 lines or >3 levels of nesting,
   consider breaking it up BEFORE annotating. See [refactoring-for-proof.md](references/refactoring-for-proof.md).

## Core Principles

- **NEVER use** `pragma Assume`, `pragma Annotate (GNATprove, False_Positive, ...)`,
  or `SPARK_Mode (Off)` on bodies without explicit user permission. Every time
  these have been suggested, a real fix was found instead. Always try: tighter
  preconditions, better type bounds, restructured code, ghost lemmas, loop
  invariants, or `pragma Assert` breadcrumbs first. Note: `SPARK_Mode (Off)` on
  a body means the body is entirely unverified — it is not a workaround, it is
  giving up. Most operations that seem unverifiable are not: `Ada.Strings.Unbounded.Length`
  and simple `:=` assignment are provable. Only heap-mutating operations may be
  genuinely problematic; restructure the code rather than disabling verification.

- **NEVER introduce clamps to make proofs easier.** A clamp (e.g.,
  `X := Real64'Min(X, MAX_VALUE)`) silently changes behavior for out-of-range
  inputs. This creates dead code that is unreachable in testing — anathema in
  certification work. Instead, use subtypes or preconditions to push the
  constraint to the caller, making the restriction visible and testable.
  See the "Clamping Anti-Pattern" in [contracts.md](references/contracts.md).

- **Hint strength ordering**: `pragma Assert` > ghost lemma (`is null`) >> ghost axiom (`Import`).
  Try the lightest tool first.

- **Types over contracts**: If a range constraint applies to a record field everywhere,
  encode it in the field's subtype rather than threading it through every Pre/Post/invariant.

## Reference Files

| Topic | File | When to read |
|-------|------|-------------|
| Proof workflow | [workflow.md](references/workflow.md) | Multi-step proof campaigns |
| Overflow patterns | [overflow-patterns.md](references/overflow-patterns.md) | Index arithmetic, intermediate overflow |
| Investigating failures | [proof-debugging.md](references/proof-debugging.md) | Unproved checks |
| Writing contracts | [contracts.md](references/contracts.md) | Adding Pre/Post/Global/Depends |
| Loop invariants | [loops.md](references/loops.md) | Loops with proof obligations |
| Ghost code & lemmas | [ghost-code-and-lemmas.md](references/ghost-code-and-lemmas.md) | Need proof hints beyond Assert |
| Floating-point proofs | [floating-point.md](references/floating-point.md) | FP overflow, conversion, trig |
| Access types | [access-types.md](references/access-types.md) | Pointers, ownership, borrowing |
| Refactoring for proof | [refactoring-for-proof.md](references/refactoring-for-proof.md) | Code restructuring to simplify proof |
| CLI options | [command-reference.md](references/command-reference.md) | GNATprove flags and modes |
