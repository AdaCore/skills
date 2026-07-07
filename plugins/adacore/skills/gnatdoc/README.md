# `gnatdoc` Skill

An agent skill for running `gnatdoc` — AdaCore's documentation generator
for Ada — on an Ada or SPARK project.

## What this skill does

When invoked, it gives an agent a complete workflow for:

- Running `gnatdoc` against a GNAT project to produce HTML, ODF, RST, or
  XML documentation
- Determining the project's documentation `--style` (`leading`,
  `trailing`, `gnat`) from the existing sources, and agreeing the
  `--generate` scope (`public`, `private`, `body`) with the user
- Writing and fixing documentation comments using the supported tags
  (`@param`, `@return`, `@field`, `@enum`, `@exception`, `@formal`,
  `@exclude`, and their aliases), including the placement rules
  `gnatdoc` enforces
- Configuring `gnatdoc` declaratively via the `Documentation` package of
  the root project file (`Documentation_Pattern`, `Excluded_Project_Files`,
  `Output_Dir`, `Resources_Dir`, `Image_Dirs`)
- Customizing backend output by overriding the resource templates

The skill is structured as a primary entry point (`SKILL.md`) that links
to focused reference guides under `references/`.

## Toolchain location is intentionally not assumed

This skill does not hardcode a `gnatdoc` binary path or assume any
particular environment setup. This is by design.

`gnatdoc` ships as part of the GNAT Studio package, but different Ada
projects will require different environment configurations (e.g.,
`GPR_PROJECT_PATH` and `PATH` set by `alr exec`, or by a user-supplied
environment script). The skill instructs the agent to detect the likely
invocation method (checking `PATH`, looking for `alire.toml`) and fall
back to asking the user when the setup isn't clear.

If you are extending or modifying this skill, preserve this property —
don't add hardcoded paths or environment assumptions.

## Scope

This skill covers `gnatdoc` only. It documents:

- The command-line interface and its switches
- The HTML (with the `--html-oop-style` switch), ODF, RST, and XML
  backends, their resource directory layouts, and customization points
- The supported documentation tags and the placement/indentation rules
  for comment blocks
- The two effective documentation styles (`leading` and `gnat`/`trailing`,
  the latter two being equivalent spellings of the same internal style)
- The `Documentation` package attributes of GNAT project files

GNAT Studio's interactive use of `gnatdoc` (Analyze → Documentation →
Generate project) is intentionally out of scope — the skill targets
command-line and CI use.

## Installation

Install into your agent platform per its skill installation
instructions. Once installed, invoke with `/gnatdoc` (or your platform's
equivalent).
