<p align="center">
  <img width="331" height="121" alt="image" src="https://github.com/user-attachments/assets/ac8c7561-22fc-4078-b5ec-eb2ed803c01a" />

</p>

<h1 align="center">Listen Labs Customer Research Skill</h1>

<p align="center">
  Run real customer research without leaving Claude — describe a goal in plain English,
  launch a study, and get sourced findings back.
  <br>
  <a href="https://listenlabs.ai">listenlabs.ai</a> ·
  <a href="https://docs.listenlabs.ai/mcp-docs">MCP docs</a> ·
  <a href="https://claude.ai/directory/connectors/listen-labs">Official Claude connector</a>
</p>

---

## About Listen Labs

[Listen Labs](https://listenlabs.ai) is an AI-moderated user research platform. An AI
interviewer conducts in-depth conversations with your customers — text, voice, video, or
screen share — at survey scale: it recruits participants from a professional panel (or
your own links), asks follow-up questions like a trained researcher, screens out bad-faith
respondents, and turns the transcripts into analysis reports where every insight is
backed by a sourced, clickable quote.

## What this skill does

This [Claude skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills) drives
the whole research loop through the [Listen MCP server](https://docs.listenlabs.ai/mcp-docs):
Claude walks the guided study setup, shows you the exact interview guide respondents will
see, gates every launch behind a cost preview, and later pulls sourced themes and quotes
out of the results. Things you can say:

- *"Interview 25 marketing managers about whether they'd pay for AI-written ad copy, and get it running."*
- *"Set up a study to test our new onboarding flow — I'll bring my own participants."*
- *"Run a usability test of our redesigned checkout at shop.acme.com — 15 people, screen share, thinking aloud."*
- *"Add a screener for prior AI-tool usage to my pricing study, then push it live."*
- *"What did respondents in the ad message study say about pricing? Quotes please."*

The skill is a single self-contained [SKILL.md](SKILL.md) — workflows, guardrails, and
worked examples inline, leaning on the Listen MCP tool schemas for parameter details, so
it works anywhere the skill file travels alone.

## Install

Two pieces: the skill, and the Listen MCP connection (the skill walks users through the
MCP part itself if it's missing).

**Claude.ai / Claude Desktop:** upload the skill in **Settings → Capabilities → Skills**,
and connect the official **Listen Labs** connector —
[claude.ai/directory/connectors/listen-labs](https://claude.ai/directory/connectors/listen-labs).

**Claude Code — personal (all projects):**

```bash
git clone https://github.com/listenlabs/listen-labs-research-skill ~/.claude/skills/listen-research
claude mcp add --transport http --scope user listenlabs https://listenlabs.ai/mcp
```

Then run `/mcp` in a session to complete the OAuth login.

**Claude Code — single project:** clone into `.claude/skills/listen-research` inside the
project instead, and drop `--scope user` from the `claude mcp add` command.

## Guardrails baked into the skill

- **Launching costs credits**, so the skill always previews cost + balance and requires
  an explicit go-ahead before `launch_study`. "Set up a study" drafts; only "launch it"
  launches.
- **The study guide is always shown verbatim** before sign-off — you approve exactly what
  respondents will see.
- **Quotes are always sourced** — every quote surfaced from transcripts or analyses keeps
  its `[Source]` deep link; unsourced claims are dropped.
