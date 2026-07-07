# CLAUDE.md

## What This Repository Is

A **plugin marketplace** of agent skills for AdaCore tools. The whole
toolchain ships as a single `adacore` plugin that bundles five skills
(`alire`, `gnatdoc`, `gnatfuzz`, `gnatprove`, `gnattest`). Users install once and get
the full set of Ada/SPARK skills.

## Repository Structure

```
skills/
├── .claude-plugin/
│   └── marketplace.json          # Claude Code marketplace catalog
├── .agents/plugins/
│   └── marketplace.json          # OpenAI Codex marketplace catalog
├── .cursor-plugin/
│   └── marketplace.json          # Cursor marketplace catalog
├── plugins/
│   └── adacore/
│       ├── .claude-plugin/
│       │   └── plugin.json       # Per-plugin manifest (Claude Code)
│       ├── .codex-plugin/
│       │   └── plugin.json       # Per-plugin manifest (Codex; mirror of above)
│       ├── .cursor-plugin/
│       │   └── plugin.json       # Per-plugin manifest (Cursor; mirror of above)
│       └── skills/
│           └── <skill-name>/
│               ├── SKILL.md      # Mandatory entry point for the skill
│               ├── README.md     # User-facing overview (optional)
│               └── references/   # Topic-focused reference guides
├── CLAUDE.md
└── README.md
```

The three marketplaces and three per-plugin manifests exist because each
agent looks for its config at a different path. The `SKILL.md` content
itself follows the [Agent Skills open standard](https://agentskills.io) and
is shared across all agents — only the surrounding manifest files are
duplicated.

Source path conventions differ slightly between marketplace formats:
- Claude Code: `"source": "./plugins/adacore"`
- Codex: `"source": "./plugins/adacore"`
- Cursor: `"source": "plugins/adacore"` (no leading `./`)

Keep the three `plugin.json` files under `plugins/adacore/` in sync — they
are content-identical mirrors. The marketplace entries differ structurally:
Codex requires `policy.installation`, `policy.authentication`, and
`category` per entry; Cursor and Claude Code use `description`, `category`,
and `tags`.

GitHub Copilot is not listed above because it natively reads
`.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` — no
Copilot-specific files are needed. Users add the marketplace with
`copilot plugin marketplace add <owner>/<repo>`.

When asked to work on a given skill, read the skill's `README.md`
(e.g. `plugins/adacore/skills/gnatprove/README.md`) for more information.

## Adding a New Skill

1. Add the skill at `plugins/adacore/skills/<tool-name>/SKILL.md` with YAML
   frontmatter (name, description, license, version) and sections matching
   the pattern in `plugins/adacore/skills/gnatprove/SKILL.md`:
   - Quick start / key commands
   - Mandatory pre-work
   - Core principles / anti-patterns
   - Reference file roadmap (what to read and when)
2. Add focused reference guides under
   `plugins/adacore/skills/<tool-name>/references/` — one file per topic
   area.
3. Optionally add `plugins/adacore/skills/<tool-name>/README.md` with a
   user-facing description and any guidance for agents modifying the skill.
4. Update the `description` in all three `plugins/adacore/.*/plugin.json`
   files to mention the new tool.
5. Update the `description` and `tags` in the three marketplace catalogs
   (`.claude-plugin/marketplace.json`, `.agents/plugins/marketplace.json`,
   `.cursor-plugin/marketplace.json`) to mention the new tool.
6. Bump the plugin `version` in all three `plugin.json` files in lockstep so
   existing users receive the update.

## Versioning

The `adacore` plugin has a single `version` field that applies to the whole
bundle, declared in all three `plugin.json` files (kept in sync). Bump it
on every release; Claude Code, Codex, and Cursor only deliver updates to
users when the version string changes.

Individual `SKILL.md` files may also carry a `metadata.version` in their
frontmatter; these are advisory and document the maturity of an individual
skill's content. Plugin update detection is driven by the plugin-level
version only.

## Validating Changes

From the repo root, validate the Claude Code marketplace:

```bash
claude plugin validate .
```

To validate the plugin's manifest and skill frontmatter:

```bash
claude plugin validate ./plugins/adacore
```
