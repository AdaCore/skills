# Fuzzing with `gnatfuzz`

`gnatfuzz` performs **coverage-guided** fuzzing: each test case is executed
through an *instrumented* build of the target subprogram. Coverage feedback
from the instrumentation feeds back into the fuzzer's mutation engine,
driving it toward inputs that exercise new paths.

The harness — produced by `gnatfuzz generate` — wraps the target subprogram
in a process that:

1. Reads a binary blob from `stdin` (or AFL++'s shared-memory equivalent).
2. Decodes the blob into the subprogram's parameter values via TGen
   marshalling.
3. Calls the target subprogram once.
4. Returns or terminates by signal/unhandled exception.

Ada's runtime checks (`Constraint_Error`, `Range_Check_Failed`,
`Predicate_Check_Failed`, etc.) act as the **oracle**: an unhandled
exception terminates the process, and `gnatfuzz` records the input as a saved
crash. Memory-safety bugs in C bindings additionally surface as `SIGSEGV`,
`SIGBUS`, etc.

## What can be fuzzed

`gnatfuzz analyze` divides every subprogram in the project into two buckets
in `analyze.json`:

* `fuzzable_subprograms` — both a harness and a starting corpus can be
  auto-generated.
* `non_fuzzable_subprograms` — at least one of those two cannot be
  auto-generated.

`gnatfuzz` can auto-generate a **harness** for a subprogram unless one of
the following is true (these rules apply to both `in` and `in out` mode
parameters except where noted):

1. A parameter is of an **access type** or contains a sub-component of an
   access type.
2. A parameter is a **subprogram access type** (e.g.,
   `access function (...) return ...`).
3. A parameter is a **limited type**.
4. An **`out`** mode parameter is a discriminated or variant record whose
   discriminant has no default value.
5. A parameter is a **null record**, derives from a null record, or
   contains a null-record sub-component.
6. A parameter is a **tagged type**.
7. A parameter is an **array type with more than 1000 components**.
8. A parameter is a **composite type** built from any of the above.
9. A parameter type carries a **`Static_Predicate` aspect**.

`gnatfuzz` can auto-generate a **starting corpus** for a subprogram unless:

1. A harness cannot be generated (any rule above).
2. A parameter is an **anonymous type** or composed of one.
3. The subprogram is a **generic procedure or function instantiation**.

If your target lands in `non_fuzzable_subprograms`, you have three options:

1. **Write a wrapper subprogram** that accepts fuzzable types and converts
   them to the original subprogram's parameter shape. Put the wrapper in a
   child package of the unit under test, then run `analyze` again and
   target the wrapper. This is the documented approach for access types,
   tagged types, and global/private state. See [§ Wrapper patterns](#wrapper-patterns).

2. **Provide manual seeds via `User_Provided_Seeds`** in
   `user_configuration/user_configuration.adb`, plus a *dummy corpus* of
   random bytes. TGen marshalling will progressively mutate dummy bytes
   into valid values during the campaign. See
   [corpus.md](../corpus/corpus.md).

3. **Skip fuzzing this subprogram.** Some constructs (e.g., generic
   instantiations referenced by other code) are easier to fuzz indirectly
   via a calling subprogram that *is* fuzzable.

### Wrapper patterns

Write a child package of the unit under test that re-exports the target as a
fuzzable subprogram.

For access-typed parameters, allocate inside the wrapper:

```ada
package Original_Pkg.Wrappers is
   function Fuzzable_Wrapper (Data : Concrete_Type) return Boolean is
   begin
      return Original_Subprogram (new Concrete_Type'(Data));
   end Fuzzable_Wrapper;
end Original_Pkg.Wrappers;
```

For subprograms whose behavior depends on hidden global or private state,
expose the state as an explicit parameter:

```ada
procedure Process_With_State
  (Data      : T;
   State_Var : State_Type)
is
begin
   Hidden_State := State_Var;
   Original_Process (Data);
end Process_With_State;
```

Tagged types, limited types, and null-record-derived types all use the same
indirection.

## Included fuzzing engines

`gnatfuzz` orchestrates four engines. `--engines` (a comma-separated list)
on `build` and `fuzz` selects which to compile and run. The default on
Linux is `afl,cmplog`; the default on Windows (where only libFuzzer is
available) is `libfuzzer`.

### AFL++ (`afl`)

Linux only. `gnatfuzz`'s primary engine. Instruments the harness via the
**AFL-GCC-fast** compiler plugin, which adds branch-coverage tracepoints
maintained in a shared-memory bitmap. AFL++ uses a genetic algorithm to
mutate inputs whose execution touches new bitmap entries.

Mutations include bit/byte flips, arithmetic increments, "interesting"
value substitution, splice (cross-pollination between corpus entries), and
havoc-style random destruction. AFL++ runs an initial deterministic
phase (sequential bit flips) on every corpus entry; skip it with
`--no-deterministic-phase`.

### CMPLOG / RedQueen (`cmplog`)

Linux only. An AFL++ plugin that observes comparisons and value
transformations during execution, then steers mutations to satisfy them.
Particularly effective when the target compares against constants or
checksums (it will discover them rather than brute-force them). Runs
alongside AFL++ — that's why the default is `afl,cmplog`. Has its own
`obj-CMPLOG/` build and `<harness>/fuzz_testing/session/CMPLOG_fuzzer_output/`.

### libFuzzer (`libfuzzer`)

Linux and Windows (Windows is beta). Built via GNAT LLVM (LLVM-based Ada
front end) using `-fsanitize=fuzzer`. In-process fuzzer: each test runs in
the same process as the fuzzer, which is faster than AFL++'s fork/exec but
puts more responsibility on the harness to leave global state in a clean
state. No AFL++ system requirements (`/proc/sys/kernel/core_pattern`, CPU
governor) when running libFuzzer alone.

### SymCC (`symcc`)

Linux only. Beta. Symbolic-execution engine that complements AFL++ by
solving path conditions to generate inputs that reach previously
unexplored branches. Runs alongside AFL++.

## AFL execution modes

`--afl-mode` (set at `gnatfuzz build` time, not at `fuzz` time) chooses how
AFL++ executes each test case. The choice is a tradeoff between throughput
and isolation.

| Mode | Behavior | When to use |
|---|---|---|
| `afl_plain` | A new process is spawned for every test. Memory is fully reset between tests. Highest overhead. | When the target leaks state in ways `Setup`/`Teardown` cannot reset, or when reproducibility is paramount. |
| `afl_persist` (**default**) | Up to 10,000 tests run in one persistent process before respawning. Highest throughput. | Default. The harness's `Setup` and `Teardown` are responsible for resetting any mutable state between tests. |
| `afl_defer` | Forks a new process per test, but defers the fork past large one-time initialization (`Deferred_Mode_Setup`). Cheaper than `afl_plain` for targets with expensive init. | Targets that need expensive one-time initialization (e.g., load a config file) but cannot tolerate state carry-over between tests. |
| `afl_defer_and_persist` | Hybrid: forks a fresh process every ~10,000 tests, after deferred initialization. | Targets with expensive init *and* mostly-resettable state. |

If you do not provide adequate `Setup`/`Teardown` for `afl_persist` and the
target has hidden mutable state, AFL++ will report low **stability**
(percentage of bitmap entries that behave reproducibly across re-execution
of the same input). Stability under 90% is the canonical signal that you
need a stronger reset (move to `afl_defer` or fix the harness).

## Stopping criteria

A `gnatfuzz fuzz` campaign runs until **any** of these happen:

1. The stopping criteria defined in `stop_criteria.xml` evaluate to true.
2. The user sends Ctrl+C (or otherwise terminates the process).
3. `--ignore-stop-criteria` was passed; only Ctrl+C will stop the campaign.

`stop_criteria.xml` lives at
`<harness>/fuzz_testing/user_configuration/stop_criteria.xml` and is
generated by `gnatfuzz generate`. You can edit it in place, or pass an
override via `--stop-criteria <file>`.

### XML schema (by example)

The default `stop_criteria.xml` shipped by `gnatfuzz`:

```xml
<GNATfuzz>
    <stdout_colours>
       <banner_border>Green</banner_border>
       <banner_text>Red</banner_text>
       <section_header>Cyan</section_header>
       <criteria_met>Green</criteria_met>
       <general_info>Grey</general_info>
       <stop_rule_info>Blue</stop_rule_info>
       <node_status>Grey</node_status>
       <warning_text>Red</warning_text>
       <error_text>Red</error_text>
    </stdout_colours>

    <afl_stop_criteria>
        <rule_grouper type="or">
            <rule>
                <description>Stop if the stability of any node drops below 90%</description>
                <state>stability</state>
                <operator>lt</operator>
                <value>90</value>
                <all_sessions>false</all_sessions>
            </rule>
           <rule_grouper type="and">
               <rule>
                   <description>Ensure all test cases on every node's queue have been processed at least twice</description>
                   <state>cycles_done</state>
                   <operator>gt</operator>
                   <value>1</value>
                   <all_sessions>true</all_sessions>
               </rule>
               <rule>
                   <description>Ensure that nodes have no pending favourites</description>
                   <state>pending_favs</state>
                   <operator>eq</operator>
                   <value>0</value>
                   <all_sessions>true</all_sessions>
               </rule>
               <rule>
                   <description>Stop if GNATcoverage has detected no coverage increase for 2 minutes</description>
                   <state>last_coverage</state>
                   <operator>gt</operator>
                   <value>120</value>
                   <all_sessions>false</all_sessions>
               </rule>
          </rule_grouper>
       </rule_grouper>
    </afl_stop_criteria>
</GNATfuzz>
```

Element reference:

| Element | Purpose |
|---|---|
| `<afl_stop_criteria>` | Container for AFL++ stopping rules. Required. |
| `<rule_grouper type="and"\|"or">` | Combines child rules and groupers. Nestable. |
| `<rule>` | A single criterion. |
| `<description>` | Free-text label. Printed in the campaign's stop-rule status display. Optional but recommended. |
| `<state>` | The metric being tested. See state list below. |
| `<operator>` | Comparison operator: `eq`, `gt`, `lt`. |
| `<value>` | Threshold value (integer). |
| `<all_sessions>` | `true`: rule must hold across **all** parallel fuzzing nodes; `false`: rule fires when **any** node satisfies it. |
| `<display_formuliac>` | `true`/`false`. Show the rule as a formula in the status display. Optional. |
| `<display_node_status>` | `true`/`false`. Show per-node status for this rule. Optional. |
| `<stdout_colours>` | Optional. Per-element ANSI colors for the campaign's stop-rule banner. |

States observed in the shipped templates:

| `<state>` | Meaning |
|---|---|
| `stability` | AFL++ stability percentage (100 = fully reproducible). |
| `cycles_done` | Number of times AFL++ has fully processed the test queue. |
| `pending_favs` | Pending "favorite" test cases not yet explored. |
| `last_coverage` | Seconds since GNATcoverage last reported a coverage increase. |
| `coverage` | GNATcoverage statement-coverage percentage. |
| `elapsed_time` | Seconds since the campaign started. |
| `saved_crashes` | Crashes saved on a single node. |
| `global_saved_crashes` | Crashes saved across all nodes. |

The default rules read in English: *"Stop if any node's stability drops
below 90%, OR (every node has cycled the queue more than once AND every
node has zero pending favorites AND no node has found new coverage in the
last 120 seconds)."*

To run for a fixed wall-clock time, change the rule grouper to `or` and
add an `elapsed_time` rule:

```xml
<rule>
    <description>Stop after 30 minutes</description>
    <state>elapsed_time</state>
    <operator>gt</operator>
    <value>1800</value>
    <all_sessions>true</all_sessions>
</rule>
```

To stop the moment a crash is found:

```xml
<rule>
    <state>global_saved_crashes</state>
    <operator>gt</operator>
    <value>0</value>
    <all_sessions>true</all_sessions>
</rule>
```

The default rules give a "good first snapshot" — adequate for small
subprograms but offer **no guarantee** that the campaign reached its full
potential. For real targets, expect to extend the time budget (raise the
`last_coverage` threshold, or wrap an `or` grouper around an
`elapsed_time` rule).

## Anti-patterns

* **Treating a crash as a confirmed bug.** A crash is a *candidate*. Always
  replay it through the verbose harness ([outputs.md § Replaying a
  crash](../outputs/outputs.md#replaying-a-crash)) to confirm the input
  parameters and the exception, and to rule out a harness/fuzzer artifact
  (e.g., the input violated a wrapper precondition that the original
  subprogram never sees).

* **Running `afl_persist` against a target with non-resettable state.** If
  stability is consistently <90%, switch to `afl_defer` (or fix
  `Setup`/`Teardown`) instead of pushing through.

* **Setting `--cores` higher than the physical core count.** AFL++ uses one
  core per fuzzing node; oversubscription doesn't add throughput and
  destabilizes the stability metric.

* **Editing `stop_criteria.xml` mid-campaign.** Changes are read at startup,
  not on every loop. Stop the campaign, edit, restart.

* **Coverage chasing on a real codebase.** Coverage is *module-level*
  statement coverage; sub-100% on a real module is normal. The default
  rules use a *plateau* signal (`last_coverage` exceeds threshold), not a
  100%-coverage signal, for exactly this reason.
