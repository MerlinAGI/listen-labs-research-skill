---
name: listen-research
description: >-
  Run real user research from Claude via the Listen Labs MCP server: create,
  edit, launch, and analyze AI-moderated user-interview studies. Use this
  whenever the user wants to launch a study, interview users or customers,
  recruit participants or a panel, validate an idea, test messaging / pricing /
  concepts / prototypes with real people, add screener questions, or pull
  findings, themes, quotes, or transcripts from an existing study — even if
  they never say "Listen Labs". Also use it when the user asks to connect
  Listen Labs to Claude or asks what their Listen MCP connection can do.
---

# Listen Labs Research

Listen Labs runs AI-moderated user interviews: describe a research goal, a study gets
built (interview guide + screener + recruitment), real respondents are recruited and
interviewed, and an analysis report is generated from the transcripts. Everything below
runs through the Listen Labs MCP server.

## Step 0 — check the connection

You need the Listen MCP tools: `create_study`, `edit_study`, `launch_study`,
`publish_study`, `get_study_state`, `list_studies`, `list_creatable_orgs`,
`get_study_analysis`, `get_study_responses`, `get_response`, `search_across_studies`.
They may sit under an MCP server prefix, and their schemas may need to be loaded before
first use, depending on the client.

If none are available, help the user connect, then continue with their request:

- **Claude.ai / Claude Desktop** — connect the official connector:
  <https://claude.ai/directory/connectors/listen-labs> (or **Settings → Connectors →
  Browse connectors** → "Listen Labs"). Sign in with OAuth, then enable the connector in
  the chat's tools menu.
- **Claude Code** — `claude mcp add --transport http listenlabs https://listenlabs.ai/mcp`,
  then `/mcp` to sign in.
- **Other MCP clients** — add remote server `https://listenlabs.ai/mcp` (OAuth).
  Docs: <https://docs.listenlabs.ai/mcp-docs>.

The user needs a Listen Labs account with an organization. Verify the connection with
`list_creatable_orgs` — a successful reply also shows which workspace(s) they can use.

## Using the tools

The MCP tool descriptions are the authoritative reference — they document every
parameter, pagination rule, and output requirement. Read a tool's schema before calling
it — parameter casing varies by tool (`studyId` vs `study_id`), and `get_response` needs
a `readable_id` that comes from `get_study_responses`. Quick map:

| Tool | What it's for |
|---|---|
| `list_creatable_orgs` | Workspaces the user can create studies in |
| `list_studies` | Find studies — pass `textHint` when the user names one |
| `search_across_studies` | Keyword search across studies |
| `create_study` | Seed a new study and start the guided setup conversation |
| `edit_study` | Continue setup, or edit any study in natural language |
| `get_study_state` | Snapshot: guide, screener, recruitment, launch readiness + costs |
| `publish_study` | Make edits visible to respondents — doesn't start recruitment |
| `launch_study` | Publish + start recruitment — **spends credits** |
| `get_study_analysis` | AI analysis report, once available |
| `get_study_responses` | Interview transcripts, paginated and filterable |
| `get_response` | Deep-dive on one respondent |

## Workflow 1 — create a study

1. Make sure you know what the user wants to learn, who to talk to, and roughly how many
   people. If they already told you, confirm in one line — don't re-interview them.
2. Call `list_creatable_orgs`. One org → proceed. Several → ask which, and pass
   `orgName` to `create_study`. Never guess.
3. Call `create_study` with a rich prompt — goal, audience, what to learn. Rich seeds
   skip setup turns: *"Interview 25 US marketing managers who run paid social. Goal:
   would they pay for AI-generated ad copy — current workflow, pain points, price
   expectations."* beats *"a study about our app"*.
4. Keep the returned `studyId` and `chatId` and continue every setup turn through
   `edit_study` with both. Answer the setup agent's structured choices with the
   `nextActions` buttons echoed by the previous turn — copy them verbatim, don't
   hand-craft them. If a button choice doesn't take (the agent re-asks and nothing
   changed), state the choice as a plain `prompt` instead. Relay real decisions (panel
   vs. own participants, interview format, target size) to the user; answer mechanical
   steps yourself when the user already gave you the answer.
5. **After every turn, show the entire study guide verbatim** — every section, question,
   and answer option, in order. Never summarize: the user is signing off on exactly what
   respondents will see. If the guide is still empty, say so.
6. Iterate with `edit_study` until they're happy. Creating is not launching — "set up a
   study" means draft it.

For website / app / prototype tests, pick a screen-share interview format during setup
(respondents share their screen while completing tasks — the other formats can't observe
the product) and make sure the URL and concrete, ordered tasks appear in the guide
respondents see.

## Workflow 2 — launch (spends real money — always gate)

1. Call `get_study_state` and read the launch info: costs, credit balance, what will
   launch and what would be skipped, any blockers.
2. Show the user the cost and balance impact, and get an **explicit go-ahead**. Never
   launch on your own initiative — only an unambiguous "launch it / go live / start
   recruiting" counts.
3. Call `launch_study`, then report what launched, what was skipped and why, and the new
   balance. If credits are short, the user adds them in the Listen dashboard — that
   can't be done over MCP. Re-calling is safe: already-running recruitment isn't
   restarted.

Panel recruitment (Listen finds participants) has an upfront cost; a self-recruit link
(user brings participants) bills per response.

## Workflow 3 — edit an existing study

1. Find it with `list_studies` + `textHint`; ask the user to pick if several match.
2. Call `get_study_state` so you and the user see what's there now.
3. Send a plain-language `edit_study` prompt ("add a screener question for parents",
   "bump the target to 50"). Omit `chatId` on the first turn; reuse the echoed one for
   follow-ups.
4. Show the updated guide verbatim, same rule as creation.
5. **If the study is already live**, edits are not visible to respondents until the study
   is re-published. Tell the user, and call `publish_study` when they confirm. Use
   `launch_study` instead only when they also want to start new recruitment.

## Workflow 4 — read results

- `list_studies` shows status, response counts, and whether analysis is ready.
- `get_study_analysis` for the report; `get_study_responses` for transcripts — filter by
  question numbers or respondents instead of pulling everything; `get_response` to zoom
  in on one person; `search_across_studies` for "did we ever research X?".
- **Grounding is non-negotiable**: every quote you surface must be verbatim and
  immediately followed by its exact `[Source]` link copied from the tool output (for
  `get_response`, build the link from the item's `source_url`). Never
  paraphrase, merge, or re-attribute quotes; if a claim isn't in the returned content,
  don't make it. If the data doesn't answer the question (e.g. no pricing question was
  asked), say so rather than stretching — and offer a follow-up study.

## Ground rules

- Track `studyId` and the latest `chatId` across turns. Each `edit_study` call takes
  *either* a prompt *or* a button click, never both.
- Never re-call `create_study` to fix a study you just made — that creates a second
  study. Edit the one you have.
- Call `get_study_state` before mutating any study you didn't just create.
