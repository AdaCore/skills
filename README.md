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
agent can run whatever environemnt setup scripts may be needed prior to the
agent invoking our tools.

We hope you find these skills useful. Keep an eye on this repository; we'll be
adding new skills over time and evolving those that are here as we learn how to
tune them better for success with agents.

## Usage

To use these skills, simply follow the install instructions that are
provided by your selected agent. Each agent installs skills slightly
differently.

Once installed, you can bring the skill into your conversation / prompt by
using the slash command associated with the skill. So, for example, if you
want to use the GNATprove skill to help with your SPARK coding and proofs,
you would include `/gnatprove` in your prompt. For example, you might say:

```md
Using /gnatprove, help me prove this program to SPARK silver.
```

The skill will handle the rest.
