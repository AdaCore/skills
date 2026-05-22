# `gnatfuzz` Skill

An agent skill for running `gnatfuzz` — AdaCore's coverage-guided fuzzer for
Ada code — on an Ada or SPARK project.

## What this skill does

When invoked, it gives an agent a complete workflow for:

- Running `gnatfuzz` end-to-end on a target subprogram (`analyze` →
  `generate` → `build` → `generate-corpus` → `fuzz`)
- Deciding which subprograms can be auto-fuzzed and writing wrappers for
  those that cannot
- Choosing fuzzing engines (AFL++, CMPLOG, libFuzzer, SymCC) and AFL
  execution modes
- Producing, providing, minimizing, and decoding corpora
- Reading the campaign's terminal output, locating crashes and hangs, and
  replaying a crash with the verbose harness to obtain the failing inputs and
  exception information
- Tuning stopping criteria via `stop_criteria.xml`

The skill is structured as a primary entry point (`SKILL.md`) that links to
focused reference guides under `references/`.

## Toolchain location is intentionally not assumed

This skill does not hardcode a `gnatfuzz` binary path or assume any
particular environment setup. This is by design.

`gnatfuzz` is shipped only through AdaCore Pro toolchains. Nevertheless,
different Ada projects will require different environment configurations
(e.g., `GPR_PROJECT_PATH` and `PATH` set by `alr exec`, or by a user-supplied
environment script). The skill instructs the agent to detect the likely
invocation method (checking `PATH`, looking for `alire.toml`) and fall back to
asking the user when the setup isn't clear.

If you are extending or modifying this skill, preserve this property — don't
add hardcoded paths or environment assumptions.

## Scope

This skill covers `gnatfuzz` only. `gnatfuzz` is one component of a larger
AdaCore tool suite, but material on the broader suite is intentionally out
of scope here.

The skill documents the six core `gnatfuzz` subcommands: `analyze`,
`generate`, `build`, `generate-corpus`, `minimize-corpus`, and `fuzz`. It
mentions the `fuzz-everything` and `hot-spot` workflows for awareness only —
those are not covered in depth.

## Installation

Install into your agent platform per its skill installation instructions.
Once installed, invoke with `/gnatfuzz` (or your platform's equivalent).
