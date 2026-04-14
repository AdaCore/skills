# CLAUDE.md

## What This Repository Is

A collection of **agent skills** for AdaCore tools — curated, expert-level instruction sets that agents invoke via slash commands (e.g., `/gnatprove`).

## Repository Structure

```
skills/
├── gnatprove/
│   ├── SKILL.md
│   ├── README.md
│   └── references/
└── README.md
```

Each skill lives in its own top-level directory. `SKILL.md` is the mandatory
entry point; the `references/` subdirectory holds additional information that
`SKILL.md` links to.

When asked to work on a given skill, read the `README.md` in that skill's
top-level directory for more information.

## Adding a New Skill

1. Create a top-level directory named after the tool (e.g., `alire/`).
2. Add `SKILL.md` with YAML frontmatter (name, description, license, version) and sections matching the pattern in `gnatprove/SKILL.md`:
   - Quick start / key commands
   - Mandatory pre-work
   - Core principles / anti-patterns
   - Reference file roadmap (what to read and when)
3. Add `README.md` with a user-facing description and any guidance for agents modifying the skill.
4. Add focused reference guides under `references/` — one file per topic area.
