# skills

Skills represent usually codeless instructions that help agents accomplish
tasks with which they may be unfamiliar or less familiar out of the box. They
are similar to MCP servers in that they aid the agent with discovery and use of
new capabilities.

In our exploration and adoption of agentic tools, we have discovered that the
use of MCP servers for *toolchains* in particular is challenged by the
difficulty of ensuring that the right environment is passed to the tools that
are effectively wrapped by the MCP server. Fortunately, all of AdaCore's tools
and toolchains offer feature-complete command-line interfaces. So there's
really no reason not to present these to the agent directly, along with careful
instructions on how best to use our tools and toolchains. That way, you or your
agent can run whatever environment setup scripts may be needed prior to the
agent invoking our tools.

We hope you find these skills useful. Keep an eye on this repository; we'll be
adding new skills over time and evolving those that are here as we learn how to
tune them better for success with agents.

## Installation

You can install these skills two ways:

- **Plugin marketplace** (recommended): your agent registers this repo as a
  remote marketplace and installs the `adacore` plugin from it. Updates flow
  automatically when AdaCore ships a new plugin version upstream.
- **Manual**: download the skill folders you want from
  [`plugins/adacore/skills/`](plugins/adacore/skills) into your agent's
  project-local skills directory. Re-download to update.

When prompted by your agent for a marketplace, use the GitHub shorthand
`AdaCore/skills`. When prompted for a plugin name, use `adacore`. The
marketplace's catalog name is `adacore-skills` — some agents use the
`<plugin>@<marketplace>` form, e.g. `adacore@adacore-skills`.

| Agent                    | Plugin install                                                                                                | Manual install                                                                                |
| ------------------------ | ------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Claude Code              | [docs](https://code.claude.com/docs/en/discover-plugins#community-marketplace)                                | [docs](https://code.claude.com/docs/en/skills#where-skills-live)                              |
| OpenAI Codex             | [docs](https://developers.openai.com/codex/plugins/build#add-a-marketplace-from-the-cli)                      | [docs](https://developers.openai.com/codex/skills#where-to-save-skills)                       |
| GitHub Copilot (VS Code) | [docs](https://code.visualstudio.com/docs/copilot/customization/agent-plugins#_configure-plugin-marketplaces) | [docs](https://code.visualstudio.com/docs/copilot/customization/agent-skills#_create-a-skill) |
| Cursor                   | [docs](https://cursor.com/docs/plugins#add-a-team-marketplace) (admin only)                                   | [docs](https://cursor.com/docs/skills)                                                        |
| Mistral Vibe             | —                                                                                                             | [docs](https://docs.mistral.ai/mistral-vibe/agents-skills)                                    |
| OpenCode                 | —                                                                                                             | [docs](https://opencode.ai/docs/skills/)                                                      |
| Cline                    | —                                                                                                             | [docs](https://docs.cline.bot/customization/skills#where-skills-live)                         |
| JetBrains Junie          | —                                                                                                             | [docs](https://junie.jetbrains.com/docs/agent-skills.html#skill-location)                     |

If your agent isn't listed but follows the [Agent Skills open
standard](https://agentskills.io), check its own docs for the skills directory
path.

## Usage

Once installed, you can bring the skill into your conversation / prompt by
using the slash command associated with the skill. So, for example, if you
want to use the GNATprove skill to help with your SPARK coding and proofs,
you would include `/gnatprove` in your prompt. For example, you might say:

```md
Using /gnatprove, help me prove this program to SPARK silver.
```

The skill will handle the rest.
