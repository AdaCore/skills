# GNATprove Proof Workflow

## Overview: Scope-Based Progression

SPARK verification is fundamentally **modular**: GNATprove thinks about one subprogram
at a time, so you should too. Work at progressively narrower scope, then widen back out:

```
program → unit → subprogram → check (narrow down)
check → subprogram → unit → program (widen back, re-verify with -f)
```

***Important***: while you are working within a subprogram (on checks),
strongly prefer using a subagent, as described below in "Delegating Steps 3-6
to a Subagent".

## Proof Status File

### Purpose

The proof status file (`proof-status.md`, alongside the code being proved) is the
persistent record of a proof campaign. It tracks what's proved, what's in progress,
and what obligations have been discovered. The user can review it to understand
overall progress and proof state.

### Format

***Important***: preserve the headings and comments when writing this file.

```markdown
# Proof Status: <program or unit or subprogram>
<!-- Reflect the top-level goal given. Items in the list below are moved from
     Not Started to In Progress to Reviewed and finally to Proved and Finalized. -->

Summary of final proof status.

## Proved and Finalized
<!-- Before marking an item complete here, follow Step 8 in `workflow.md` in
     the /gnatprove Skill -->

- [ ] Program (level, mode)
  - [ ] Unit_Name (level, mode)
    - [x] Subprogram_Name (level, mode)

## Reviewed
<!-- Before marking an item complete here, review it following Step 7 in
     `workflow.md` in the /gnatprove Skill -->

- [ ] Program
  - [ ] Unit_Name
    - [x] Subprogram_Name

## In Progress
<!-- Strongly prefer using a subagent to execute Steps 3 - 6 from
     `workflow.md` in the /gnatprove Skill. The subagent should update this
     section as it works. -->

- [ ] Subprogram_Name (level, mode)
  - [x] overflow check line NN
  - [ ] range check line MM -- added Assert at line KK, needs proof
  - [ ] postcondition line PP

## Not Started
<!-- Whenever a subprogram is added (due to refactoring) or discovered (e.g.,
     in Steps 1-3 from `workflow.md` in the /gnatprove skill), it should be
     listed here so that it is not forgotten. -->

- [ ] Subprogram_Name

## Discovered Obligations
- [ ] Prove Assert breadcrumb at file.adb:KK (added to help line MM)
- [ ] Prove Lemma_Foo body (ghost-code-and-lemmas.md)
- [ ] Add frame postcondition to Callee for Field_X (contracts.md)
```

### When to update

| Event | Action |
|-------|--------|
| Initial assessment (Step 1) | Create file; add entry per subprogram with failures |
| Start working a subprogram | Move to "In Progress"; list its failing checks |
| Check proves | Mark check done |
| Add `pragma Assert` breadcrumb | Add to Discovered Obligations |
| Write a ghost lemma | Add lemma body proof to Discovered Obligations |
| Add/modify a contract | Add to Discovered Obligations (callers may need re-proof) |
| Subprogram fully proves with `--limit-subp -f` | Move to "Reviewed" |
| Step 7 review complete (antipatterns checked, warnings resolved) | Move to "Proved and Finalized" after re-running `--limit-subp -f` (Step 8) |
| Unit fully proves with `-u -f` | Mark unit entry in "Proved and Finalized" |

### Rules

- **Always create or load** the status file at the start of a proof campaign
- **Update after each meaningful step**, not after every GNATprove invocation
- **Keep it honest**: if a subprogram is "Proved" but you later change its callee's
  contract, move it back to "Not Started" (it needs re-verification)

## Step 1: Assess Current State

Before reading code in depth, run GNATprove to see where things stand.

```bash
gnatprove -P <project.gpr> -j0 --level=0 --output=oneline
```

- `--output=oneline` keeps context small; one failed check per line
- Start at level 0, then try level 1 if many timeouts
- Count failures (`| wc -l`) to gauge the scope of work
- If everything proves: you're done (don't waste time reading code)

If the user gave you a specific file or subprogram, scope the initial run:
- `-u file.adb` for a single unit
- `--limit-subp=file.adb:NN` for a single subprogram

**Status file**: Create the proof status file. Add an entry under "Not Started"
for each subprogram with unproved checks. If the scope is a single subprogram,
add it to "In Progress" and list its failing checks.

## Step 2: Choose Starting Point

**Start at a leaf in the call graph.** Leaf subprograms have no callees (or only
callees with already-proven contracts), so their proofs depend only on their own
code and their preconditions. This avoids cascading failures from unproven callee
contracts.

After proving a leaf, move to its callers or to another leaf. Continue until
the unit is done.

## Step 3: Read the Right Things

For the subprogram under proof, read:
- The **subprogram itself** (body + contracts)
- The **specifications** (not bodies) of its callees
- The **data types** involved

SPARK is modular: callee bodies are irrelevant for proof. Only the callee's
contract matters. If a callee has no contract, only its parameter types matter.

## Step 4: Consider Decomposition

Before diving into annotations, ask: **is this subprogram too monolithic?** A
long body with many local variables and branches drowns the prover in context.
Extracting a coherent block of logic into its own subprogram with a clear
contract hides the implementation detail from the caller's proof — the same
information-hiding principle that aids design also aids provability.

Each extracted call is a cut point (requires Pre/Post), but this cost is almost
always worth paying: the prover sees less, the contracts are reusable, and the
code is better. See [refactoring-for-proof.md](refactoring-for-proof.md) for
patterns and guidance.

## Step 5: Work Check-by-Check

Within a subprogram, use `--limit-line` to focus on one check at a time:

```bash
gnatprove ... --limit-subp=file.adb:NN --limit-line=file.adb:MM
```

**Important**: `--limit-line` proves **exactly** that line, not "up to" that line.
SPARK assumes all assertions above the target line (even unproved ones). This is
powerful for what-if exploration: you can insert `pragma Assert (P)` above a
failing check to test whether P would help, without yet proving P.

### Ordering within a subprogram

1. **Top-to-bottom**: because SPARK assumes assertions above the current check
2. **Overflows first**: if you can't prove absence of overflow, nothing else matters;
   overflow fixes often introduce changes that affect downstream checks

## Step 6: Fix, Prove, Repeat

For each failing check, follow the decision tree in
[proof-debugging.md](proof-debugging.md). The short version:

1. Is the code actually correct? (test at runtime with `-gnata`)
2. Missing annotation? (check GNATprove's suggestions)
3. Insert `pragma Assert` breadcrumbs to narrow down the gap
4. Frame condition missing? (add postcondition preserving unmodified fields)
5. Increase level (up to 2; beyond 2 requires user permission)
6. Prefer to ask the user before writing a lemma

**Status file**: Each fix that creates a new proof obligation gets an entry:
- Added `pragma Assert` at line KK → "Prove Assert at file:KK"
- Wrote ghost lemma `Lemma_Foo` → "Prove Lemma_Foo body"
- Added/changed a precondition → "Re-verify callers of Subprogram_Name"
- Added frame postcondition → "Re-verify callers of Callee_Name"

Mark the original check as done when it proves.

**Do not blame the prover prematurely.** Most proof failures are caused by:
- Wrong code (the check is correct; the code isn't)
- Missing information (needs a precondition, type bound, or intermediate assertion)
- Overwhelming context (extract a lemma to reduce context)
- Non-inductive property (needs to reference the loop index)

The prover being "too weak" is the rarest cause.

### Delegating Steps 3-6 to a Subagent

Steps 3-6 are iterative, context-heavy, and largely mechanical. Consider
delegating them to a subagent for each subprogram. This keeps the main
context clean for judgment-intensive work (choosing starting points,
reviewing results, widening scope).

The subagent prompt should include:
- The subprogram to prove (file, declaration line)
- Known bounds and contracts from already-proved callees
- Structural constraints ("use preconditions not guards", target proof level)
- Instruction to update proof-status.md

When the subagent finishes, proceed to Step 7 (Review) before widening
scope. The fresh perspective on the subagent's work is the primary
quality gate — review for antipatterns, missed subtypes, guards that
should be preconditions, etc.

Important: Only one subagent should run gnatprove at a time (it
assumes exclusive access to the GPR project). You can plan the next
subprogram or do non-proof work while a subagent is running.

## Step 7: Review Work for Antipatterns

Before widening the scope to re-verify and marking the subprogram or unit as
complete, carefully reread your now-proved implementation and make sure you are
following the guidance in (at least) [contracts.md](contracts.md) and
[refactoring-for-proof.md](refactoring-for-proof.md).

This is a good moment to take a step back, away from the pressure of
completing the current check NOW, to make sure that the quality of the code
you've written is as high as possible. It's worth repeating steps (as needed)
if the result is better, more maintainable code. In other words, proof is a
critical goal, but it's not the only goal!

Look for opportunities to improve types, contracts, code layout, factoring,
etc. Also, take this opportunity to resolve any warnings that may be issued
by the compiler or SPARK for the scope you're working on - these are also
worth fixing. In particular, check for recurring range constraints that should
be subtypes (see Subtype Self-Check in contracts.md).

## Step 8: Widen Scope and Re-Verify

After completing all checks in a subprogram via `--limit-line`, re-verify the
whole subprogram:

```bash
gnatprove ... --limit-subp=file.adb:NN -f
```

The `-f` forces full reanalysis. This catches checks you may have missed and
verifies that your `pragma Assert` breadcrumbs themselves prove.

**CRITICAL**: `--limit-line` is a diagnostic tool, not a proof. It assumes all
assertions above the target line are true — including ones you haven't proved
yet. A subprogram is NOT proved until `--limit-subp` succeeds with no
`medium` or `high` messages. Similarly, a unit is not proved until `-u`
succeeds.

If `--limit-subp` fails at the target level on checks that `--limit-line`
proved:
1. **First**, try `--limit-subp` at the **next higher level**. If it succeeds,
   report the level and ask the user whether to accept it or invest effort
   reducing the level.
2. If the next level also fails, the issue is real — likely context overload
   requiring decomposition ([refactoring-for-proof.md](refactoring-for-proof.md)),
   not just a timeout.

Do NOT declare a subprogram "proved" based solely on `--limit-line` results.

Continue widening:
- After all subprograms in a unit: `gnatprove ... -u file.adb -f`
- After all units: `gnatprove ... -f` (whole program)

Use `--output=oneline` at each widening step for a quick pass/fail summary.

**Status file**: Move subprograms to "Proved" as they pass full re-verification.
If widening reveals new failures (e.g. a callee contract change broke a caller),
move the caller back to "In Progress" and list the new failures.

## Step 9: Cleanup (Ask User)

After all checks pass, optionally:
- **Remove temporary Assert breadcrumbs**: These may no longer be needed, but
  removing them is expensive (must re-prove to confirm). Ask the user if
  proof minimization is desired.
- **Review preconditions**: Are any unreasonably restrictive? Could the
  subprogram handle a wider input range without complication?
- **Commit checkpoint**: Suggest committing proven code before moving on,
  especially if the proof campaign involved code changes.

## When to Convert Through Stone/Bronze First

- **User says "I have SPARK code"**: Jump straight to proof
- **User says "prove this Ada code" or "translate this C to SPARK"**: Work through
  stone (SPARK legality) and bronze (flow analysis) first to reduce noise. Offer
  to commit at each level transition.

## Proof Level Discipline

| Level | When to use |
|-------|------------|
| 0 | Initial assessment; quick iteration |
| 1 | Normal proof work |
| 2 | When level 1 times out on checks you believe are correct |
| 3-4 | **Only with user permission.** May experiment with `--limit-line` to test provability, but prefer finding a level-2 solution |

## References

- [Levels of SPARK Use](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/usage_scenarios.html#levels-of-software-assurance) -- Stone through Platinum assurance levels
- [Silver Level - Absence of Run-time Errors](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/usage_scenarios.html#silver-level-absence-of-run-time-errors-aorte) -- Proving AoRTE as the standard baseline
- [Gold Level - Key Integrity Properties](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/usage_scenarios.html#gold-level-proof-of-key-integrity-properties) -- Functional correctness beyond AoRTE
- [How to Speed Up GNATprove](https://docs.adacore.com/live/wave/spark2014/html/spark2014_ug/en/gnatprove.html#how-to-speed-up-a-run-of-gnatprove) -- Incremental proof, parallelism, caching
