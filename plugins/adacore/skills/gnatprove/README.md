# GNATprove Skill

An agent skill for running formal verification proof campaigns on Ada/SPARK code
using the GNATprove tool.

## What this skill does

When invoked, it gives an agent a complete workflow for:

- Running GNATprove at the right scope and proof level
- Tracking proof obligations in a persistent status file
- Investigating and resolving unproved checks
- Writing contracts, loop invariants, ghost code, and lemmas
- Refactoring code to improve provability

The skill is structured as a primary entry point (`SKILL.md`) that links to
focused reference guides under `references/`.

## Toolchain location is intentionally not assumed

This skill does not hardcode a gnatprove binary path or assume any particular
environment setup. This is by design.

SPARK users have meaningfully different setups: FSF SPARK via Alire, SPARK Pro
installed under `/usr/gnat` or `/opt/gnat`, project-specific toolchain scripts,
containers, and so on. Many of these require more than just a binary path — for
example, SPARK Pro needs `GPR_PROJECT_PATH` and related variables set
correctly, and Alire projects should be invoked via `alr gnatprove` so that
Alire can inject the right environment for multi-crate dependency resolution.

The skill instructs agents to detect the likely invocation method (checking
`PATH`, looking for `alire.toml`) and fall back to asking the user when the
setup isn't clear. This is intentional: one question up front is cheaper than a
failed proof run from a misconfigured environment.

If you are extending or modifying this skill, preserve this property — don't
add hardcoded paths or environment assumptions.

## Installation

Install into your agent platform per its skill installation instructions.
Once installed, invoke with `/gnatprove` (or your platform's equivalent).
