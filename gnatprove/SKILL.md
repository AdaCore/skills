---
name: gnatprove
description: This skill should be used when the user wants to "prove SPARK code", "run GNATprove", "investigate unproved checks", "write contracts or invariants", "add loop invariants", "write ghost code or lemmas", or is working on SPARK formal verification. Provides a full GNATprove proof campaign workflow.
license: Apache-2.0
metadata:
  version: "0.2.0"
---

# GNATprove SPARK Verification Skill

## Locating GNATprove

Unless the user has already told you how to invoke gnatprove, determine the right
invocation in this order:

1. **Check for an `alire.toml` in the project root.** Its presence means the project
   is an Alire crate; use `alr gnatprove -P <project.gpr> ...` rather than bare
   `gnatprove`. Alire computes and injects the correct environment (including
   `GPR_PROJECT_PATH` for multi-crate projects) before calling gnatprove, so this
   is the correct invocation for Alire-managed projects even when `gnatprove` is
   also on `PATH`.

2. **Check if `gnatprove` is already on `PATH`.** Run `which gnatprove`. If found,
   use it — but if proof runs fail with project-file resolution errors, the
   environment may be incomplete (see step 3).

3. **Ask the user.** SPARK Pro installations (commonly under `/usr/gnat` or
   `/opt/gnat`) typically require environment setup beyond a binary path. If
   gnatprove is not on `PATH` and there's no `alire.toml`, don't guess — ask the
   user how they want to configure the environment.

## Quick Start

To prove SPARK code, run GNATprove via Bash with the project's environment.

```bash
  gnatprove -P <project.gpr> -j0 --quiet <arguments> 2>&1 | tee gnatprove-run.txt
```

Always prefer to run with `--quiet` as this will suppress output
about analysis phases, which is mostly there to show users progress. If there
are no non-info messages output, the proof succeeded.

**Bash tool timeout**: GNATprove can take tens of minutes — there is no way to
predict duration in advance. Always set a large `timeout` on the Bash tool call:
- **Scoped runs** (`--limit-subp`, `--limit-line`): minimum 300000ms (5 min)
- **Whole-unit or whole-program runs**: 600000ms (10 min) or more

If gnatprove times out but produced output, work with that output — a partial
run is still useful and re-running would violate the no-redundant-runs rule.
Only re-run with a larger timeout if the run produced **no output at all**. Note
the timeout for future runs. Never rely on the default 120s timeout.

Piping through `tee` saves output to `gnatprove-run.txt` so you can re-filter
it (grep, etc.) without re-running. **Never re-run gnatprove solely to
re-examine output** — a re-run without a code or contract change produces the
same result at significant cost. For deeper investigation, read `gnatprove.out`:
the precise directory is project-dependent but generally
`objs/release/gnatprove/gnatprove.out`; refer to
[gnatprove-out.md](references/gnatprove-out.md) for details.

**Never run two gnatprove instances concurrently.** GNATprove assumes exclusive
ownership of its output directories; parallel runs corrupt results.

Common arguments:
- `-jN` where 0 means as many threads as possible and N>0 means use that many parallel threads
- `--level=1` (proof level 0-4; start low, increase only if needed)
- `--limit-subp=file.adb:NN` (prove one subprogram; NN = **declaration** line, not body)
- `--limit-line=file.adb:MM` (prove exactly one check; assumes assertions above)
- `--output=oneline` (one check per line; good for status assessment)
- `--mode=stone` (SPARK legality), `--mode=bronze` (flow), `--mode=all` (flow + proof, default)
- `-f` (force full reanalysis)

Note: with `--limit-subp` or `--limit-line`, the filename must NOT include the `src/`
prefix -- the GPR file handles source directories.

**Line numbers are not stable across edits.** Do not assume the previous `MM`
or `NN` remains valid after inserting, deleting, or moving
code/comments/contracts above the target. After any such change, you *must*
re-locate the target in the current file before the next run of:
  - `--limit-line=file.adb:MM` must use the check's current line number
  - `--limit-subp=file.adb:NN` must use the subprogram declaration's current
    line number

## Proof Status File

At the start of a proof campaign, create (or load) a proof status file to track
obligations and progress. Default location: `proof-status.md` alongside the code
being proved. This file is the persistent record of what's proved, what's in
progress, and what work items have been discovered.

See [workflow.md](references/workflow.md) for the format and rules for updating it.

## Workflow

### Step 1: Assess first

Run GNATprove with `--output=oneline` before reading any code in depth.

### Step 2: Triage — choose your path

**Quick-fix path** — use this if ALL of the following hold:
- All failures are in a single subprogram
- None involve loops, floating-point, or access types
- Each error message is self-explanatory (obvious overflow or missing range
  constraint where the fix is immediately clear)

If all three hold, work inline: read the subprogram, fix each check, re-verify
with `--limit-subp -f`, then review for antipatterns (Strategic Loop Step 4 in
[workflow.md](references/workflow.md)) and widen scope (Strategic Loop Step 5).
No proof-status.md or subagents needed.

**Escape hatch**: if any fix proves non-obvious, a loop invariant is needed, or
an unexpected check appears, switch to the full campaign path below.

**Full campaign** — use this if any failure spans multiple subprograms, involves
loops, floating-point, or access types, or is not immediately self-explanatory.
Load [workflow.md](references/workflow.md) and follow it from Step 1.

**You MUST invoke a subagent (Strategic Loop Step 3) for each subprogram.**
Tactical context (gnatprove output, line-by-line iteration) is noise once a
subprogram proves; keeping it out of the main context preserves your ability to
make strategic judgments — spotting subtype opportunities, choosing the next
leaf, reviewing for antipatterns. Use the subagent prompt template in workflow.md.

1. **Create proof status file** with top-level entry for the requested scope
2. **Decompose**: Add a status entry for each subprogram with unproved checks
3. **Start at a leaf**; invoke a subagent (Strategic Loop Step 3 in workflow.md)
4. **Review** each proved subprogram for antipatterns (Strategic Loop Step 4)
5. **Widen scope** with `-f` and finalize (Strategic Loop Step 5)
6. **Increase level** only up to 2; beyond 2 requires user permission

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
| Proof workflow | [workflow.md](references/workflow.md) | Multi-step proof campaigns (main agent) |
| Tactical loop | [workflow-subagent.md](references/workflow-subagent.md) | Per-subprogram proof steps (subagent) |
| Reading gnatprove.out | [gnatprove-out.md](references/gnatprove-out.md) | Understanding how to read the output file GNATprove writes |
| Overflow patterns | [overflow-patterns.md](references/overflow-patterns.md) | Index arithmetic, intermediate overflow |
| Investigating failures | [proof-debugging.md](references/proof-debugging.md) | Unproved checks |
| Writing contracts | [contracts.md](references/contracts.md) | Adding Pre/Post/Global/Depends |
| Loop invariants | [loops.md](references/loops.md) | Loops with proof obligations |
| Ghost code & lemmas | [ghost-code-and-lemmas.md](references/ghost-code-and-lemmas.md) | Need proof hints beyond Assert |
| Floating-point proofs | [floating-point.md](references/floating-point.md) | FP overflow, conversion, trig |
| Access types | [access-types.md](references/access-types.md) | Pointers, ownership, borrowing |
| Refactoring for proof | [refactoring-for-proof.md](references/refactoring-for-proof.md) | Code restructuring to simplify proof |
| CLI options | [command-reference.md](references/command-reference.md) | GNATprove flags and modes |
