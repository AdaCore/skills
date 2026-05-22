# Outputs and Crash Investigation

## What `gnatfuzz fuzz` displays

`gnatfuzz fuzz` orchestrates AFL++ (or libFuzzer) and shows two layers of
output:

1. **`gnatfuzz`'s own banner and stop-rule status.** Printed at startup
   and as criteria progress. Includes the elapsed time, anomalies count,
   GNATcoverage statement-coverage percentage, and a per-rule indicator
   for each criterion in `stop_criteria.xml`. Color-coded if your terminal
   supports ANSI (turn off with `--disable-styled-output`).

2. **AFL++'s status screen.** A live full-screen TUI maintained by AFL++
   itself. Documented at https://aflplus.plus/docs/status_screen/. Key
   fields:

   | Field | Meaning |
   |---|---|
   | `cycles done` | Times the fuzzer has fully processed the queue. |
   | `total execs` | Cumulative test executions. |
   | `exec speed` | Executions per second (current). |
   | `corpus count` | Inputs in the queue (starting corpus + discovered). |
   | `unique crashes` | Distinct saved crashes (not total crashing inputs). |
   | `unique hangs` | Distinct timeouts. |
   | `last new path` | Time since the most recent coverage increase. |
   | `stability` | Percentage of bitmap entries that behave reproducibly. |
   | `cur item` | The corpus entry currently being mutated. |

When running with `--cores N` and N > 1, AFL++ shows the master node's
screen; per-slave fuzzer states are written to disk under
`<harness>/fuzz_testing/session/fuzzer_output/gnatfuzz_<N>_slave/`.

If you tee the campaign output to a file (recommended), the AFL++ status
screen produces a flickering ANSI-escape blob in the file. The
authoritative post-run record is *not* the tee'd log; it's the contents of
`fuzz_testing/session/`. See § Session directory layout.

## Session directory layout

The first `gnatfuzz fuzz` invocation populates this tree under the harness:

```
<harness>/fuzz_testing/
├── user_configuration/
│   ├── user_configuration.adb
│   ├── user_configuration.ads
│   └── stop_criteria.xml
└── session/
    ├── fuzzer_output/
    │   ├── gnatfuzz_1_master/
    │   │   ├── crashes/      # crashes seen by node 1
    │   │   ├── hangs/        # hangs seen by node 1
    │   │   ├── queue/        # path-unique inputs seen by node 1
    │   │   └── fuzzer_stats  # AFL++ key=value metrics file
    │   └── gnatfuzz_<N>_slave/
    │       ├── crashes/
    │       ├── hangs/
    │       ├── queue/
    │       └── fuzzer_stats
    ├── CMPLOG_fuzzer_output/  # only when --engines includes cmplog
    │   └── ... (mirrors structure above)
    ├── results/               # synchronized aggregate across all nodes
    │   ├── crashes/           # all unique crashes from any node
    │   ├── hangs/             # all unique hangs from any node
    │   └── ending_corpus/     # final synchronized queue
    └── coverage_output/
        ├── index.html         # GNATcoverage HTML report entry point
        └── ...                # one HTML per source file with annotations
```

`results/` is the right place to look after a campaign ends — it
deduplicates across the parallel fuzzer nodes. `fuzzer_output/gnatfuzz_*/`
is the right place for *per-node* drill-down.

## fuzzer_stats — what to read after a campaign

`fuzzer_stats` is a key=value plain-text file written by AFL++ on each
node. Useful keys:

| Key | Meaning |
|---|---|
| `cycles_done` | Queue cycles completed. |
| `execs_done` | Total executions on this node. |
| `execs_per_sec` | Current throughput. |
| `corpus_count` | Inputs in this node's queue. |
| `unique_crashes` | Saved crashes on this node. |
| `unique_hangs` | Saved hangs on this node. |
| `stability` | Reproducibility percentage. <90% indicates the harness's `Setup`/`Teardown` does not fully reset state — see [fuzzing.md § AFL execution modes](../fuzzing/fuzzing.md#afl-execution-modes). |
| `last_path` | Wall time of the most recent path discovery (Unix epoch). |
| `last_crash` | Wall time of the most recent crash (Unix epoch). |

## Crash files

Each saved crash is a single binary file (TGen-marshalled input) with a
filename of the AFL++ form:

```
id:000000,sig:06,src:000000,time:64,execs:258,op:flip32,pos:4
```

Field decode:

| Field | Meaning |
|---|---|
| `id` | Zero-padded 6-digit crash counter, unique per node. |
| `sig` | Unix signal number that terminated the process. `06` = `SIGABRT` (Ada raises an unhandled exception → `Last_Chance_Handler` aborts), `11` = `SIGSEGV`, `04` = `SIGILL`, etc. |
| `src` | The `id` of the queue/corpus entry the mutation derived from. |
| `time` | Seconds since the campaign began. |
| `execs` | Cumulative executions on this node when the crash was saved. |
| `op` | The mutation operator that produced the crashing input (`flip1`, `flip32`, `arith8`, `havoc`, `splice`, `interest8`, etc.). |
| `pos` | Byte offset of the mutation in the source input. |

The colons make crash filenames awkward in shells — escape them or quote
the path. Examples below escape with backslashes.

## Replaying a crash

The harness build produces a verbose decoder binary under the AFL persist
build directory:

```
<harness>/build/obj-AFL_PERSIST/gnatfuzz_test_harness.verbose
```

Run it with the crash file as a positional argument. From the harness root:

```bash
./build/obj-AFL_PERSIST/gnatfuzz_test_harness.verbose \
   fuzz_testing/session/results/crashes/id\:000000\,sig\:06\,src\:000000\,time\:64\,execs\:258\,op\:flip32\,pos\:4
```

The verbose harness:

1. Decodes the binary input via TGen marshalling.
2. Prints the decoded parameter values as JSON.
3. Calls the target subprogram once.
4. Catches the unhandled exception (if any) and prints exception
   information as JSON.

Output has roughly this shape:

```json
{
  "Subprogram_Under_Test": "Pkg.Target_Subprogram",
  "Decoded_In_Parameters": {
    "Param_1": { "Type": "Integer", "Value": "-2147483648" },
    "Param_2": { "Type": "String",  "Value": "..." }
  },
  "Testcase_Exception": {
    "Name":        "CONSTRAINT_ERROR",
    "Message":     "pkg.adb:42 overflow check failed",
    "Information": "..."
  }
}
```

`Decoded_In_Parameters` is your bug report: those are the input values
that triggered the crash. `Testcase_Exception` tells you which check or
predicate failed and where.

The same binary decodes queue and hang files; only crash files have a
`Testcase_Exception` payload.

## Investigating crashes systematically

When a campaign saves multiple crashes, do not assume they are different
bugs. AFL++ deduplicates by **bitmap signature**, not by exception or
backtrace, so several `unique_crashes` may all surface the same root
cause. Workflow:

1. **Replay every saved crash through the verbose harness.** Group by
   `Testcase_Exception.Name` + `Message` first, then by
   `Decoded_In_Parameters` shape.

2. **Confirm the crash reproduces against the original (non-fuzzing)
   build of the project.** If it does not, the problem is in the harness
   or the wrapper, not the target subprogram. Common causes: wrapper
   precondition that the original subprogram never sees, harness
   `Setup`/`Teardown` leaving stale state, or a mismatch between the
   wrapper's parameter types and the original.

3. **Reduce the input** if the crash is real. Use the `Decoded_In_Parameters`
   to construct a minimal reproducer in plain Ada.

4. **Triage the underlying check.** A `Range_Check_Failed`,
   `Constraint_Error`, or `Predicate_Check_Failed` is usually a missing
   precondition, a too-loose subtype, or a missing input validation. A
   `SIGSEGV` from a C binding is a memory-safety bug in the bound code.

5. **Add a regression test** before fixing. The simplest test is a
   parameterized one that constructs the failing inputs from
   `Decoded_In_Parameters`.

## Hangs

`fuzz_testing/session/results/hangs/` collects test cases that exceed
AFL++'s per-test timeout (its default is dynamic; AFL++ adapts based on
observed execution times). Each hang file uses the same naming convention
as crashes, but with the `op` reflecting the mutation that produced the
slow input.

Replay a hang the same way as a crash. Hangs are usually evidence of:

* An infinite loop reachable from some input.
* A pathological algorithmic complexity in the target (or its bindings).
* `Setup`/`Teardown` failing to free a previous test's resources, so the
  next test runs against an over-large data structure.

`stability` < 100% combined with frequent hangs almost always means
`Setup`/`Teardown` is incomplete.

## Coverage report

`fuzz_testing/session/coverage_output/index.html` is a live GNATcoverage
HTML report, refreshed periodically during the campaign. Per-source
files (one HTML per `.adb` / `.ads`) annotate every line green
(covered), red (uncovered), or grey (not statement-bearing).

Coverage is **statement coverage** at the **module** level — every line
of every source file in the harness's `with`-closure is counted, not
just the target subprogram. Don't expect 100% on a real codebase.

Disable with `--no-GNATcov` on `gnatfuzz build` (and `gnatfuzz fuzz`) if
you don't need it; it imposes a measurable instrumentation overhead.

## Anti-patterns

* **Reading `unique_crashes` as the bug count.** It's the number of
  bitmap-distinct crashing inputs, not the number of distinct
  underlying defects. Replay and group.

* **Re-running the campaign because you didn't tee the output.** The
  authoritative artifacts are in `fuzz_testing/session/`. The terminal
  log is supplementary.

* **Editing files under `fuzz_testing/session/` during a campaign.**
  AFL++ owns this tree; concurrent writes corrupt its state.

* **Reading raw crash files in a text editor.** They're TGen-marshalled
  binary. Use the verbose harness.

* **Fixing a crash without reproducing the failure against the unfuzzed
  build.** A failure that only reproduces inside the fuzzing harness is
  often a harness bug, not a target bug.
