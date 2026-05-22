---
name: gnatfuzz
description: This skill should be used when the user wants to "fuzz Ada code", "run gnatfuzz", "generate a fuzzing harness", "investigate a fuzz crash", "write a fuzzing wrapper for an unsupported parameter type", "generate or minimize a corpus", or is otherwise running a fuzzing campaign on an Ada/SPARK project. Covers running gnatfuzz, designing fuzzable subprograms, managing corpora, and reading fuzzing outputs and crash files.
license: Apache-2.0
metadata:
  version: "0.1.0"
---

# gnatfuzz

`gnatfuzz` is AdaCore's coverage-guided fuzzer for Ada (and Ada-bound C) code.
It automates the production of a test harness for a target subprogram, the
generation of a starting corpus, and the orchestration of one or more fuzzing
engines (AFL++, CMPLOG, libFuzzer, SymCC) running in parallel against the
instrumented harness.

`gnatfuzz`'s scope is **unit-level vulnerability testing**: it fuzz-tests one
Ada subprogram at a time. Ada's runtime checks (overflow, range, array
bounds, division by zero, predicate violations) act as the oracle — a crash
during fuzzing usually means a constraint check, predicate, or other runtime
contract was violated by an input the fuzzer synthesized.

## Locating gnatfuzz

Unless the user has already told you how to invoke `gnatfuzz`, determine the
right invocation in this order:

1. **Check for an `alire.toml` in the project root.** Its presence means the
   project is an Alire crate. Invoke `gnatfuzz` via `alr exec -- gnatfuzz ...`
   so Alire injects the correct environment (`GPR_PROJECT_PATH`, `PATH`, etc.)
   before the tool runs. If you're not sure how the user installed `gnatfuzz`
   for this project, ask (step 3).

2. **Check if `gnatfuzz` is already on `PATH`.** Run `which gnatfuzz`. If
   found, use it — but if commands fail with project-file resolution errors,
   the environment may be incomplete (see step 3).

3. **Ask the user.** AdaCore installations (commonly under `/usr/gnat`,
   `/opt/gnat`, or a vendor-specific path) typically require environment
   setup beyond a binary on `PATH`. Don't guess — ask the user how they want
   to configure the environment.

## Quick Start

The typical `gnatfuzz` workflow on a fresh project is four commands. Run them
under the project's environment, teeing each invocation's output:

```bash
gnatfuzz analyze         -P <project.gpr> 2>&1 | tee gnatfuzz-analyze.txt
gnatfuzz generate        -P <project.gpr> --analysis <obj>/gnatfuzz/analyze.json --subprogram-id <ID> 2>&1 | tee gnatfuzz-generate.txt
gnatfuzz build           -P <harness>/fuzz_test.gpr 2>&1 | tee gnatfuzz-build.txt
gnatfuzz generate-corpus -P <harness>/fuzz_test.gpr -o <harness>/starting_corpus 2>&1 | tee gnatfuzz-corpus.txt
gnatfuzz fuzz            -P <harness>/fuzz_test.gpr --corpus-path <harness>/starting_corpus 2>&1 | tee gnatfuzz-fuzz.txt
```

`gnatfuzz analyze` and `gnatfuzz generate` run against the **user's
project**; the remaining subcommands (`build`, `generate-corpus`,
`minimize-corpus`, `fuzz`) run against the **harness project**
(`fuzz_test.gpr`) that `generate` produced.

`gnatfuzz fuzz` runs until its stopping criteria are met or until you stop it
manually with Ctrl+C. It can take from minutes to hours — see
[fuzzing.md § Stopping criteria](references/fuzzing/fuzzing.md#stopping-criteria).

**Bash tool timeout**: `gnatfuzz fuzz` is the long-running step — set the Bash
tool `timeout` accordingly, or, preferably, run `fuzz` in the background and
poll the session directory and `tee`-ed log file. See
[outputs.md](references/outputs/outputs.md) for what to monitor.

**Linux system configuration is mandatory before `gnatfuzz fuzz`**. AFL++
requires two `root`-owned changes (core-pattern + CPU governor) that persist
only until reboot. See [gnatfuzz.md § Verifying the AFL++ environment](references/gnatfuzz/gnatfuzz.md#verifying-the-afl-environment).

Common arguments (every subcommand accepts `-P` and most accept `-X`):

| Flag | Effect |
|------|--------|
| `-P, --project FILE` | Project file. For `analyze` and `generate`, the user's project; for everything else, the harness `fuzz_test.gpr` produced by `generate`. **Required** on every subcommand. |
| `-X NAME=VALUE` | Set an external scenario variable on the project. Repeatable. |
| `-v, -vv` | Verbosity at the top level: `-v` info, `-vv` debug. |
| `--help` | Per-subcommand help. Always available. |
| `--version` | Top-level only. Prints e.g. `GNATfuzz 27.0w (20260428)`. |
| `--disable-styled-output` | Strip ANSI escape sequences (use in non-TTY captures). |

## Version

`gnatfuzz` is only available as a Pro tool from AdaCore. Confirm the version
with `gnatfuzz --version`:

* **Pro.** The version sold by AdaCore with professional support (shipped
  as part of the GNATDAS suite). Prints a banner of the form
  `GNATfuzz NN.Nw (YYYYMMDD)`, where
  - `NN` is the release year (e.g., `27` ≈ 2026)
  - `N` is the release number (`0`: preview; `1`: release; `2`: stabilization)
  - `w` is optional and denotes a development *wavefront* (a continuously
    integrated build between numbered releases)
  - `YYYYMMDD` is the build date.

  E.g., `GNATfuzz 27.0w (20260428)` is a 27.0 development wavefront built on
  2026-04-28.

## Mandatory pre-work

Before running `gnatfuzz` on a new target, do these in order:

1. **Build the user project at least once with the standard toolchain**
   (`gprbuild -P <project.gpr>`). `gnatfuzz analyze` parses the project as
   GNAT sees it; an unbuildable project will not be analyzable.

2. **Run `gnatfuzz analyze`.** Read `<obj>/gnatfuzz/analyze.json`. For each
   target subprogram, check whether it appears under `fuzzable_subprograms`
   or `non_fuzzable_subprograms`, and which `id` it has — you'll need the
   `id` for `gnatfuzz generate --subprogram-id`. If the target lands in
   `non_fuzzable_subprograms`, read
   [fuzzing.md § What can be fuzzed](references/fuzzing/fuzzing.md#what-can-be-fuzzed)
   and write a wrapper before continuing.

3. **Verify Linux system configuration for AFL++** if you intend to use the
   default engines (`afl,cmplog`). See
   [gnatfuzz.md § Verifying the AFL++ environment](references/gnatfuzz/gnatfuzz.md#verifying-the-afl-environment).

4. **Decide an AFL execution mode.** The default is `afl_persist` (10,000
   tests per process; you must reset state in `Setup`/`Teardown`). If the
   target subprogram allocates global state that you cannot reset cheaply,
   pick `afl_defer` or `afl_plain` instead — see
   [fuzzing.md § AFL execution modes](references/fuzzing/fuzzing.md#afl-execution-modes).

## Core principles

* **`gnatfuzz` does not modify the user's source tree.** All artifacts live
  under `<obj>/gnatfuzz/` (for `analyze`) and the generated harness directory
  (for everything else). Treat the harness directory as build output: you can
  delete and regenerate it, but check in `user_configuration/` if you have
  customized seeds, setup/teardown, or `stop_criteria.xml`.

* **Never edit `generated_src/`.** It is regenerated on every `gnatfuzz
  generate`. Customizations belong in `user_configuration/`.

* **Never run two `gnatfuzz fuzz` campaigns concurrently against the same
  harness.** They share the `fuzz_testing/session/` directory.

* **Runtime checks are the oracle.** A "crash" in the AFL++ sense is any
  process termination by signal (or any unhandled Ada exception, since the
  harness lets exceptions propagate to terminate the process by default). A
  fuzzer crash is therefore evidence of a runtime-check violation, not
  evidence of memory unsafety per se. Inspect every saved crash with the
  verbose harness (see
  [outputs.md § Replaying a crash](references/outputs/outputs.md#replaying-a-crash))
  before declaring it a real bug.

* **Coverage is statement coverage at the *module* level.** GNATcoverage
  instruments the entire compilation module — the percentage in the TUI
  reflects everything in that module, not just the target subprogram. Don't
  expect to reach 100% on a real codebase.

## Use cases

There are four broad use cases for this skill.

1. **Working with `gnatfuzz`.** Invoking the tool, the project layout it
   expects, what each subcommand produces.
   Read [gnatfuzz.md](references/gnatfuzz/gnatfuzz.md) and
   [command-reference.md](references/gnatfuzz/command-reference.md).

2. **Fuzzing strategy.** What can and cannot be fuzzed, the included fuzzing
   engines, AFL execution modes, stopping criteria.
   Read [fuzzing.md](references/fuzzing/fuzzing.md).

3. **Inputs and corpora.** Generating a starting corpus, providing custom
   seeds, minimizing, encoding/decoding test cases.
   Read [corpus.md](references/corpus/corpus.md).

4. **Outputs and crash investigation.** What the campaign displays, the
   layout of `fuzz_testing/session/`, AFL++ crash filenames, replaying a
   crash with the verbose harness.
   Read [outputs.md](references/outputs/outputs.md).
