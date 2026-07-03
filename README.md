# listen-research — a Claude skill for Listen Labs

A [Claude skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills) that lets
Claude create, edit, **launch**, and analyze [Listen Labs](https://listenlabs.ai)
user-interview studies through the [Listen MCP server](https://docs.listenlabs.ai/mcp-docs).

Describe a research goal in plain English and Claude will walk the guided study setup,
show you the exact interview guide respondents will see, gate the launch behind a cost
preview, and later pull sourced findings out of the transcripts and analysis.

## What's in here

```
SKILL.md                  # the skill: workflows, guardrails, state discipline
references/
├── connect.md            # how to connect the Listen MCP server, per client
├── tools.md              # full reference for all 11 MCP tools
└── examples.md           # end-to-end worked flows (create→launch, edit→publish, analysis)
evals/evals.json          # test prompts for iterating on the skill
```

## Install

The skill needs two things: this folder in your skills directory, and the Listen MCP
server connected (the skill itself walks users through the MCP part if it's missing).

**Claude Code — personal (all projects):**

```bash
git clone https://github.com/listenlabs/listen-labs-research-skill ~/.claude/skills/listen-research
claude mcp add --transport http --scope user listenlabs https://listenlabs.ai/mcp
```

Then run `/mcp` in a session to complete the OAuth login.

**Claude Code — single project:** clone into `.claude/skills/listen-research` inside the
project instead, and drop `--scope user` from the `claude mcp add` command.

**Claude.ai / Claude Desktop:** upload the skill in **Settings → Capabilities → Skills**,
and connect the **Listen Labs** connector under **Settings → Connectors** (it's in the
official connector directory).

## Try it

- *"Interview 25 marketing managers about whether they'd pay for AI-written ad copy, and get it running."*
- *"Set up a study to test our new onboarding flow — I'll bring my own participants."*
- *"Run a usability test of our redesigned checkout at shop.acme.com — 15 people, screen share, thinking aloud."*
- *"Add a screener for prior AI-tool usage to my pricing study, then push it live."*
- *"What did respondents in the ad message study say about pricing? Quotes please."*

## Guardrails baked into the skill

- **Launching costs credits**, so the skill always previews cost + balance via
  `get_study_state` and requires an explicit go-ahead before `launch_study`. "Set up a
  study" drafts; only "launch it" launches.
- **The study guide is always shown verbatim** before sign-off — you approve exactly what
  respondents will see.
- **Quotes are always sourced** — every quote surfaced from transcripts or analyses keeps
  its `[Source]` deep link; unsourced claims are dropped.
