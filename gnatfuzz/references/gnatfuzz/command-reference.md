# `gnatfuzz` Command Reference

Per-subcommand reference. All subcommands accept `--help`; run
`gnatfuzz <subcommand> --help` to see the exact flag list for the version of
`gnatfuzz` you have.

## Top-level

```
gnatfuzz [-v|-vv] [--version] <subcommand> [options]
```

Subcommands documented here:

| Subcommand | Purpose |
|---|---|
| `analyze` | Find fuzzable subprograms in the user project. |
| `generate` | Generate a harness project for one subprogram. |
| `build` | Compile the harness with the chosen engine instrumentation. |
| `generate-corpus` | Synthesize a starting corpus from parameter types. |
| `minimize-corpus` | Reduce a corpus to the inputs that exercise unique paths. |
| `fuzz` | Run the fuzzing campaign. |

Two additional workflows — `fuzz-everything` and `hot-spot` — exist but are
out of scope for this skill. Mention them only if the user asks.

The `import-tgen-vectors` utility is a beta feature for incremental
campaigns; see the user's guide.

## `gnatfuzz analyze`

Scans an Ada project to identify subprograms that `gnatfuzz` can
auto-fuzz, plus those it cannot.

```
gnatfuzz analyze -P <user_project.gpr> [options]
```

| Flag | Effect |
|---|---|
| `-P, --project FILE` | **Required.** The user's project GPR (not the harness). |
| `-X NAME=VALUE` | External scenario variable on the project. Repeatable. |
| `--no-subprojects` | Process only the root project; ignore `with`-ed subprojects. |
| `-S, --sources FILE` | Limit analysis to the given source file. For C, supply both header and source. |
| `-o, --output DIRECTORY` | Override where artifacts are written. Default: `<obj>/gnatfuzz/`. |

**Output**: `<obj>/gnatfuzz/analyze.json` with the structure:

```json
{
  "user_project": "<path>.gpr",
  "scenario_variables": [],
  "fuzzable_subprograms":     [ { "id": 1, "label": "...", "source_filename": "...", "start_line": 10, "start_column": 4, "instantiations": [...] }, ... ],
  "non_fuzzable_subprograms": [ ... ]
}
```

Use the `id` of a fuzzable entry as input to
`gnatfuzz generate --subprogram-id`.

The `instantiations` array under each entry lists generic instantiations of
that subprogram, each with their own `id`.

## `gnatfuzz generate`

Generates a self-contained harness project for one target subprogram.

```
gnatfuzz generate -P <user_project.gpr> [target-selection] [options]
```

Target selection (mutually exclusive groups):

* `--analysis FILE --subprogram-id ID` — preferred. Uses `analyze.json`;
  robust to line shifts.
* `-S, --source FILE -L, --line N` — fragile alternative; identifies the
  subprogram by source location.

| Flag | Effect |
|---|---|
| `-P, --project FILE` | **Required.** User's project GPR. |
| `--analysis FILE` | Path to `analyze.json`. |
| `--subprogram-id INT` | Id from `analyze.json`. Requires `--analysis`. |
| `-S, --source FILE` | Source file. Requires `-L`. Mutually exclusive with `--analysis`. |
| `-L, --line INT` | Subprogram declaration line (1-indexed). Requires `-S`. |
| `-X NAME=VALUE` | External scenario variable. Repeatable. |
| `-o, --output DIRECTORY` | Output directory for the harness. Default: `<obj>/gnatfuzz/harness`. |
| `--disable-styled-output` | No ANSI escapes. |

**Output**: a harness directory containing `fuzz_test.gpr`, `generated_src/`,
`user_configuration/`, etc. See
[gnatfuzz.md § Project layout](gnatfuzz.md#project-layout-gnatfuzz-produces).

`generate` is idempotent on `generated_src/` (regenerated each run) but does
*not* overwrite `user_configuration/` if it already contains user edits.

## `gnatfuzz build`

Compiles the harness for one or more fuzzing engines.

```
gnatfuzz build -P <fuzz_test.gpr> [options]
```

| Flag | Effect |
|---|---|
| `-P, --project FILE` | **Required.** The harness `fuzz_test.gpr`. |
| `-X NAME=VALUE` | External scenario variable. Repeatable. |
| `--engines SET` | Comma-separated engines: `afl`, `cmplog`, `libfuzzer`, `symcc`. Default: `afl,cmplog`. |
| `--afl-mode MODE` | `afl_persist` (default), `afl_defer`, `afl_defer_and_persist`, `afl_plain`. See [fuzzing.md](../fuzzing/fuzzing.md#afl-execution-modes). |
| `-j, --jobs INT` | Parallel build jobs. |
| `--no-GNATcov` | Skip GNATcoverage statement-coverage instrumentation. |

**Output**: per-engine `obj-*` subdirectories under the harness root (e.g.,
`obj-AFL_PERSIST/`, `obj-LIBFUZZER/`, `obj-CMPLOG/`). Each contains the
instrumented harness binary; `obj-AFL_PERSIST/` also contains the
`gnatfuzz_test_harness.verbose` binary used to replay crashes (see
[outputs.md § Replaying a crash](../outputs/outputs.md#replaying-a-crash)).

Run `build` after every change to `user_configuration/`. Building is
otherwise unnecessary between successive `fuzz` runs against the same
harness.

## `gnatfuzz generate-corpus`

Synthesizes a starting corpus from the parameter types of the target
subprogram.

```
gnatfuzz generate-corpus -P <fuzz_test.gpr> [-o DIR] [--strategy STRATEGY]
```

| Flag | Effect |
|---|---|
| `-P, --project FILE` | **Required.** The harness `fuzz_test.gpr`. |
| `-X NAME=VALUE` | External scenario variable. Repeatable. |
| `-o, --output DIRECTORY` | Output directory for corpus files. |
| `--strategy STRATEGY` | Comma-separated: `internal`, `user`, or both. Default: `internal,user`. |

The `--strategy` choices are:

* `internal` — auto-generate test cases by sampling each parameter type at
  `T'First`, a middle value, and `T'Last` (and combinations for arrays /
  records).
* `user` — invoke the `User_Provided_Seeds` procedure in
  `user_configuration/user_configuration.adb` to add user-supplied test
  cases.

Use `--strategy user` alone to generate a starting corpus consisting only of
manually provided seeds (e.g., when the parameter types are unsupported by
internal generation; see [corpus.md](../corpus/corpus.md)).

**Output**: binary-encoded files. User-provided seeds are named `User_1`,
`User_2`, etc. Seeds that come from automatic corpus generation are roughly the
fully qualified name plus a hash that is calculated by TGen (in order to handle
function overloads).. Use the `decode_test` utility (under `<harness>/build/`)
to inspect them — see
[corpus.md § Reading a corpus entry](../corpus/corpus.md#reading-a-corpus-entry).

## `gnatfuzz minimize-corpus`

Reduces a corpus to the smallest subset that still covers the same set of
unique execution paths.

```
gnatfuzz minimize-corpus -P <fuzz_test.gpr> [-i INDIR] [-o OUTDIR]
```

| Flag | Effect |
|---|---|
| `-P, --project FILE` | **Required.** The harness `fuzz_test.gpr`. |
| `-i, --input DIRECTORY` | Source corpus. Default: the `generate-corpus` output directory. |
| `-o, --output DIRECTORY` | Destination. Default: `<obj>/gnatfuzz/minimized_corpus`. |

Internally uses `afl-cmin` (for AFL++/CMPLOG) or libFuzzer's merge mode.

`gnatfuzz fuzz` runs corpus minimization automatically before the campaign
unless you pass `--no-cmin`. Calling `minimize-corpus` explicitly is useful
when you want the minimized corpus as a checked-in artifact, or when you
intend to fuzz repeatedly with the same starting set.

## `gnatfuzz fuzz`

Launches, monitors, and stops a fuzzing campaign.

```
gnatfuzz fuzz -P <fuzz_test.gpr> [options]
```

| Flag | Effect |
|---|---|
| `-P, --project FILE` | **Required.** The harness `fuzz_test.gpr`. |
| `-X NAME=VALUE` | External scenario variable. Repeatable. |
| `--engines SET` | Engines: `afl`, `cmplog`, `libfuzzer`, `symcc`. Default: `afl,cmplog` on Linux; `libfuzzer` on Windows. |
| `--cores INT` | Cores to use. `0` means auto (all available). Range: 0..256. Default: 0. |
| `--corpus-path DIR` | Starting corpus directory. Default: the auto-generated corpus location. |
| `--seed INT` | Fixed RNG seed for deterministic benchmarking. Default: random. |
| `--no-deterministic-phase` | Skip AFL++'s initial bit-flip phase. |
| `--stop-criteria FILE` | Path to a custom `stop_criteria.xml`. Default: `<harness>/fuzz_testing/user_configuration/stop_criteria.xml`. |
| `--ignore-stop-criteria` | Ignore the stop file; campaign runs until Ctrl+C. |
| `--no-GNATcov` | Skip GNATcoverage statement-coverage instrumentation. |
| `--tgen-gnattest-test-generator` | Beta. Watch the AFL queue and use TGen to emit GNATtest test cases. |
| `--tgen-serialized-test-dir DIR` | Output directory for the above. Requires `--tgen-gnattest-test-generator`. |
| `--disable-styled-output` | No ANSI escapes. |

`--no-cmin` (skip pre-campaign corpus minimization) is documented in the
user's guide but may not appear in `--help` on every release; verify with
`gnatfuzz fuzz --help` before relying on it.

`--afl-mode` is set at `build` time, not at `fuzz` time. To switch modes,
re-run `build` with the new mode and then `fuzz`.

`fuzz` does not produce a single "result" file; instead it populates
`<harness>/fuzz_testing/session/` with crashes, hangs, queue entries,
fuzzer stats, and a coverage report. See
[outputs.md](../outputs/outputs.md).
