# gnatfuzz

`gnatfuzz` is a coverage-guided fuzzer for Ada code. It runs in two project
contexts:

* **Against the user's project** (`-P user_project.gpr`): the `analyze` and
  `generate` subcommands. `analyze` scans the project for fuzzable
  subprograms; `generate` reads the project to emit a harness for one chosen
  subprogram.
* **Against a generated harness project** (`-P fuzz_test.gpr`): every other
  subcommand (`build`, `generate-corpus`, `minimize-corpus`, `fuzz`). The
  harness is a self-contained GPR project produced by `gnatfuzz generate`.

If the user asks about a use of `gnatfuzz` not documented in this skill,
*stop* and direct them to the `gnatfuzz` user's guide:
https://docs.adacore.com/live/wave/gnatdas/html/gnatdas_ug/gnatfuzz/gnatfuzz_part.html

Always tee `gnatfuzz` output to a file so you can re-examine it without
re-running:

```bash
gnatfuzz <subcommand> -P <project.gpr> ... 2>&1 | tee gnatfuzz-<subcommand>.txt
```

## End-to-end workflow

The standard pipeline runs five subcommands in order. The first two target
the **user's** project (`analyze` reads it; `generate` reads it to emit a
harness); the rest target the **harness** project produced by `generate`.

```bash
# 1. Identify fuzzable subprograms in the user project.
gnatfuzz analyze -P user_project.gpr

# 2. Generate a harness for one specific subprogram.
#    Use --analysis + --subprogram-id (preferred, robust to line shifts) or
#    -S source.adb -L NN (line-numbered, fragile).
gnatfuzz generate -P user_project.gpr \
    --analysis  obj/gnatfuzz/analyze.json \
    --subprogram-id 3 \
    -o fuzz_harness

# 3. Build the harness. Compiles fuzz_test.gpr with the relevant
#    instrumentation per --engines.
gnatfuzz build -P fuzz_harness/fuzz_test.gpr

# 4. Generate a starting corpus from the subprogram's parameter types.
gnatfuzz generate-corpus -P fuzz_harness/fuzz_test.gpr \
    -o fuzz_harness/starting_corpus

# 4a. (Optional) Minimize the corpus - particularly when there is a large
# initial corpus.
gnatfuzz minimize-corpus -P fuzz_harness/fuzz_test.gpr \
    -i fuzz_harness/starting_corpus \
    -o fuzz_harness/minimized_corpus


# 5. Run the campaign.
gnatfuzz fuzz -P fuzz_harness/fuzz_test.gpr \
    --corpus-path fuzz_harness/starting_corpus
```

`gnatfuzz fuzz` will run until its stopping criteria are met or you stop it
with Ctrl+C. See [fuzzing.md § Stopping criteria](../fuzzing/fuzzing.md#stopping-criteria).

## Project layout `gnatfuzz` produces

`gnatfuzz analyze` writes:

```
<obj>/gnatfuzz/analyze.json     # fuzzable + non-fuzzable subprograms with ids
```

`gnatfuzz generate -o <harness>` produces a harness project with this
structure:

```
<harness>/
├── fuzz_test.gpr                # harness GPR — pass to every other subcommand
├── fuzz_config.json             # campaign metadata
├── generated_src/               # generated harness code — DO NOT EDIT
│   ├── gnatfuzz_harness.adb
│   └── gnatfuzz_harness.ads
├── user_configuration/          # user-modifiable configuration
│   ├── user_configuration.ads
│   ├── user_configuration.adb   # User_Provided_Seeds, Setup, Teardown,
│   │                            # Deferred_Mode_Setup
│   └── stop_criteria.xml        # stopping criteria (see fuzzing.md)
└── build/                       # post-build artifacts (corpus encode/decode)
```

The first time you `build`, additional engine-specific subdirectories appear
under `<harness>/`:

```
<harness>/
├── obj-AFL_PERSIST/             # AFL++ persistent-mode build (default)
├── obj-LIBFUZZER/               # libFuzzer build (if --engines libfuzzer)
└── obj-CMPLOG/                  # CMPLOG build (if --engines cmplog)
```

The first `fuzz` invocation creates the session tree (see
[outputs.md § Session directory layout](../outputs/outputs.md#session-directory-layout)).

**Never edit `generated_src/`.** It is regenerated on every `gnatfuzz
generate`. Customizations belong in `user_configuration/`.

## Verifying the AFL++ environment

The default engines (`afl,cmplog`) run AFL++. AFL++ requires two `root`-owned
Linux configurations on every machine before running `gnatfuzz fuzz`:

* `/proc/sys/kernel/core_pattern` must be a literal core-file template, not a
  pipe to a userspace handler — otherwise AFL++ cannot distinguish a crash
  from a timeout.
* every CPU's `cpufreq` scaling governor should be `performance` — otherwise
  throughput and AFL++'s `stability` metric become noisy.

Both settings are world-readable, so probe them automatically before each
campaign and only involve the user if a check fails. Both revert on reboot,
so re-probe on a fresh session.

`libfuzzer` does not need either configuration; if you are running only
`--engines libfuzzer`, you can skip both probes. `symcc` cooperates with
AFL++ and so needs the same environment.

See the [User's Guide](https://docs.adacore.com/live/wave/gnatdas/html/gnatdas_ug/gnatfuzz/gnatfuzz_part.html#afl-requirements) for more information.

### Probe the environment

```bash
# 1. core_pattern: must NOT start with '|'. A leading pipe routes crashes
#    to a userspace handler, which AFL++ cannot distinguish from a timeout.
cat /proc/sys/kernel/core_pattern

# 2. cpufreq governor: every line should be 'performance'. The cpufreq tree
#    may not exist on virtualized hosts — that's fine, skip the check there.
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor 2>/dev/null \
    || echo "no cpufreq on this host"
```

| Probe | OK | Needs fix |
| --- | --- | --- |
| `core_pattern` | any value not starting with `\|` (e.g. `core`, `core.%e.%p`) | starts with `\|` (e.g. `\|/usr/share/apport/apport ...`) |
| cpufreq governor | every line is `performance`, *or* the path doesn't exist | any line other than `performance` |

### If a probe fails

***Do not run the fix yourself.*** Explain to the user why the change is
necessary, present the exact command, and ask them to run it under `sudo`.

If `core_pattern` starts with `|`, the user needs to run:

```bash
echo core | sudo tee /proc/sys/kernel/core_pattern
```

If any CPU is not on the `performance` governor, the user needs to run:

```bash
cd /sys/devices/system/cpu
echo performance | sudo tee cpu*/cpufreq/scaling_governor
```

If the user skips these, `gnatfuzz fuzz` will fail to start or will produce
unreliable `stability` numbers in the AFL++ status display.

## Verbosity

`-v` (info) and `-vv` (debug) sit on the **top-level** `gnatfuzz` command,
*before* the subcommand name:

```bash
gnatfuzz -vv analyze -P user_project.gpr   # debug
gnatfuzz -v  fuzz    -P fuzz_test.gpr      # info
```

`-v` increases what `gnatfuzz` itself prints (its orchestration, harness
generation, build invocations); it does not change what AFL++ prints.

## --version

```
$ gnatfuzz --version
GNATfuzz 27.0w (20260428)
```

`gnatfuzz`'s version banner follows the AdaCore version-string convention
(see [SKILL.md § Versions](../../SKILL.md#versions)). For matching toolchain
behavior, use a `gnat` and `gnatcov` of the same release; Alire or an
AdaCore-managed environment handles this for you.

## --help

`--help` is available on every subcommand and prints exactly the flags
accepted by that subcommand. Always check `gnatfuzz <subcommand> --help`
before you copy a command from elsewhere — flags evolve across releases.

```bash
gnatfuzz --help                    # top-level: lists subcommands
gnatfuzz <subcommand> --help       # per-subcommand options
```

## Common pitfalls

* **Passing the wrong GPR to a subcommand.** `analyze` and `generate` expect
  the user's project; `build`, `generate-corpus`, `minimize-corpus`, and
  `fuzz` expect the harness `fuzz_test.gpr`. The error message ("invalid
  project", "missing fuzz_config.json") is your cue to swap the `-P`
  argument.

* **Editing `generated_src/`.** Always overwritten on the next `generate`.
  Use `user_configuration/` instead.

* **Forgetting `build` after edits to `user_configuration/`.** `Setup`,
  `Teardown`, and `User_Provided_Seeds` are compiled into the harness
  binary; `gnatfuzz fuzz` does *not* automatically rebuild before running.

* **Two concurrent `fuzz` runs against the same harness.** Both will fight
  for `fuzz_testing/session/`. Use distinct harness directories if you want
  to fuzz multiple targets in parallel.

* **Stale `analyze.json`.** Source edits invalidate the `start_line` /
  `start_column` and may change `id`s. Re-run `analyze` after non-trivial
  edits to the user project.

For per-subcommand details and exact flag tables, see
[command-reference.md](command-reference.md).
