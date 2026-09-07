# AGENTS.md

Guidance for AI coding agents working in or with this repository.

## What this repo is

The official Listen Labs agent skill: instructions that teach an AI agent to run customer research (create, launch, and analyze AI-moderated user-interview studies) through the Listen Labs MCP server.

## Layout

- `SKILL.md`: the skill, kept at the repo root so `npx skills add MerlinAGI/listen-labs-research-skill` works.
- `skills/listen-research/SKILL.md`: the same skill at the Agent Plugins fixed discovery location. Keep both copies identical when editing.
- `plugin.json` and `mcp.json`: Agent Plugins v1 manifest and MCP server configuration, per https://agent-plugins.org/specification.
- `assets/`: images used by the skill and README.

## Using Listen Labs from an agent

- MCP endpoint: https://listenlabs.ai/mcp (OAuth 2.0, streamable HTTP; clients discover the auth endpoints automatically)
- Documentation: https://docs.listenlabs.com/ (agent index at https://docs.listenlabs.com/llms.txt)
- Site agent index: https://listenlabs.com/llms.txt
- Credentials guide: https://listenlabs.com/auth.md

## Rules

- Never commit credentials or API keys. The MCP server authenticates through OAuth in the connecting client.
- Keep the SKILL.md frontmatter valid YAML. The `description` field decides when agents invoke the skill, so keep it specific about the jobs the skill handles.
