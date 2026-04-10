# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A collection of **agent skills** for AdaCore tools — curated, expert-level instruction sets that agents invoke via slash commands (e.g., `/gnatprove`). Skills are pure Markdown; there is no build system, no tests, and no compilation. The repository is installed into agent platforms (such as Claude Code) and the Markdown files are served to agents on demand.

The design rationale (from `README.md`): MCP servers struggle with AdaCore tool environments, so skills provide agents with comprehensive CLI-oriented instructions instead of wrapping tools in protocol servers.

## Repository Structure

```
skills/
├── gnatprove/
│   ├── SKILL.md            # Skill entry point: metadata, quick start, core principles
│   ├── README.md           # User-facing description
│   └── references/         # topic-specific reference guides
└── README.md
```

Each skill lives in its own top-level directory. `SKILL.md` is the mandatory entry point; the `references/` subdirectory holds deeper dives that `SKILL.md` links to.

## Adding a New Skill

1. Create a top-level directory named after the tool (e.g., `alire/`).
2. Add `SKILL.md` with YAML frontmatter (name, description, license, version) and sections matching the pattern in `gnatprove/SKILL.md`:
   - Quick start / key commands
   - Mandatory pre-work
   - Core principles / anti-patterns
   - Reference file roadmap (what to read and when)
3. Add focused reference guides under `references/` — one file per topic area.

## GNATprove Skill Architecture

The GNATprove skill encodes an 8-step iterative proof campaign (see `gnatprove/references/workflow.md`). Key design decisions baked into the skill:

- **Proof Status File** (`proof-status.md`): the persistent source of truth, with sections: Not Started / In Progress / Reviewed / Proved and Finalized / Discovered Obligations.
- **Scope narrowing strategy**: campaign → unit → subprogram → check (narrow), then widen back.
- **Leaf-first ordering**: prove functions with no callees before their callers.
- **Hint strength ordering**: `pragma Assert` > ghost lemma > ghost axiom (lightest tool first).
- **Forbidden without explicit user permission**: `pragma Assume`, `pragma Annotate GNATprove False_Positive`, `SPARK_Mode (Off)`, clamping values to satisfy proofs.

Reference files and their purposes:

| File | When to consult |
|------|----------------|
| `workflow.md` | Structuring a proof campaign end-to-end |
| `workflow-subagent.md` | Instructions for a subagent proving one subprogram |
| `contracts.md` | Writing Pre/Post/Global/Depends/Contract_Cases |
| `command-reference.md` | GNATprove CLI flags, proof levels, timeouts |
| `gnatprove-out.md` | Interpreting proof output without re-running |
| `proof-debugging.md` | Decision tree for unproved checks |
| `loops.md` | Loop invariants and termination proofs |
| `ghost-code-and-lemmas.md` | Ghost variables, lemmas, axioms |
| `refactoring-for-proof.md` | Restructuring code to improve provability |
| `overflow-patterns.md` | Integer overflow and intermediate values |
| `floating-point.md` | Floating-point proof techniques |
| `access-types.md` | Pointers, ownership, borrowing in SPARK |
