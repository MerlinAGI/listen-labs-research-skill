# Connecting the Listen Labs MCP server

Read this when the Listen MCP tools (`create_study`, `launch_study`, `list_studies`, …)
are not available in the session. Walk the user through the setup for whichever client
they're in, then have them retry their request.

## What the user needs

- A **Listen Labs account** with access to at least one organization
  (sign up at [listenlabs.ai](https://listenlabs.ai) if they don't have one).
- An MCP client that supports **remote servers with OAuth** (Claude.ai, Claude Desktop,
  Claude Code, ChatGPT, and Codex all qualify).

## Server details

| | |
|---|---|
| Server URL | `https://listenlabs.ai/mcp` |
| Alternate URL | `https://mcp.listenlabs.ai/mcp` |
| Transport | Streamable HTTP (stateless) |
| Auth | OAuth — 1-hour access token (auto-refreshed), 30-day refresh token |
| Official docs | <https://docs.listenlabs.ai/mcp-docs> |

## Setup by client

### Claude.ai (web) and Claude Desktop

Listen Labs is an **official connector** in the Claude directory, so no custom-URL setup
is needed:

1. Open the directory listing — <https://claude.ai/directory/connectors/listen-labs> —
   and click **Connect**. (Equivalent path: **Settings → Connectors → Browse connectors**,
   search **"Listen Labs"**, **Connect**.)
2. Log in to Listen Labs in the OAuth window and approve access.
3. In a new or existing chat, make sure the Listen Labs connector is enabled in the
   tools/search-and-tools menu.

Only fall back to **Add custom connector** with the server URL `https://listenlabs.ai/mcp`
if the directory listing isn't available in the user's org.

### Claude Code (CLI)

```bash
claude mcp add --transport http listenlabs https://listenlabs.ai/mcp
```

Then inside a Claude Code session, run `/mcp`, select **listenlabs**, and complete the
OAuth login in the browser window that opens. Add `--scope user` to the command above if
the user wants the server available in every project rather than just the current one.

### Codex (CLI)

```bash
codex mcp add listenlabs --url https://listenlabs.ai/mcp
```

Then authenticate when prompted.

### ChatGPT

Official app-store listing:
<https://chatgpt.com/apps/listen-labs/asdk_app_6a0765f330f08191a2e5d95f075948a9> — or
search for **"Listen Labs"** in the **Apps** section and connect from there.

### Other MCP clients

Any client that supports remote streamable-HTTP servers with OAuth works: add a server
with URL `https://listenlabs.ai/mcp` and complete the OAuth flow.

## After connecting

Verify the connection by calling `list_creatable_orgs` or `list_studies` — a successful
response confirms auth and shows which organization(s) the user can work in.

## Troubleshooting

- **Tools still missing after setup** — the session usually needs to be restarted (or the
  connector toggled on) for tools to register. In Claude Code, check `/mcp` for the
  server's status.
- **Auth errors / 401s after it previously worked** — the 30-day refresh token likely
  expired or access was revoked. Reconnect: re-run the OAuth flow from the connector
  settings (Claude.ai) or `/mcp` (Claude Code).
- **User can see no studies / can't create** — permissions mirror the Listen Labs web app
  exactly. If they can't do it at listenlabs.ai, they can't do it over MCP; they need an
  org admin to grant access.
- **Revoking access** — remove the server/connector from the client; tokens are
  invalidated.
