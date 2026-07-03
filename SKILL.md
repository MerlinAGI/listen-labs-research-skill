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

Listen Labs is a user-interview platform: you describe a research goal, its AI builds a
study (interview guide + screener + recruitment), real respondents are recruited and
interviewed by an AI moderator, and an analysis report is generated from the transcripts.
This skill drives that whole loop through the Listen MCP server.

## Step 0 — Check the connection

Every workflow below depends on the Listen MCP tools: `create_study`, `edit_study`,
`launch_study`, `publish_study`, `get_study_state`, `list_studies`, `list_creatable_orgs`,
`get_study_analysis`, `get_study_responses`, `get_response`, `search_across_studies`.
They are usually namespaced under an MCP server prefix (e.g. `mcp__listenlabs__create_study` —
the prefix varies by how the user named the connector). If your harness defers MCP tool
schemas, load them before calling.

If none of these tools are available, the Listen MCP server is not connected. Read
[references/connect.md](references/connect.md) and walk the user through connecting
(server URL: `https://listenlabs.ai/mcp`, OAuth login). Then continue with their request.

## The four workflows

Pick the workflow from what the user is trying to do. Full parameter details and gotchas
for every tool live in [references/tools.md](references/tools.md); realistic end-to-end
walkthroughs live in [references/examples.md](references/examples.md).

### 1. Create a new study

1. **Understand the goal first.** Before touching tools, make sure you know: what the user
   wants to learn, who they want to talk to, and roughly how many people. If they gave you
   this already, don't re-interview them — a one-line confirmation is enough.
2. **Resolve the organization.** Call `list_creatable_orgs`. One org → proceed. Multiple →
   ask which one, then pass `orgName` to `create_study`. Never guess the org.
3. **Seed the study** with `create_study`, passing a rich plain-language prompt (goal,
   audience, what to learn). The response returns a `studyId` and a server-minted `chatId` —
   keep both for the rest of the conversation.
4. **Walk the guided onboarding.** The creation agent moves through stages
   (goals → recruitment type → audience → interview mode → study guide). Continue each turn
   with `edit_study`, passing the same `studyId` + `chatId` and *either* a `prompt` (to
   answer/refine) *or* a `buttonClick` copied from the previous turn's `nextActions` —
   never hand-craft button payloads. Relay the agent's questions to the user when they're
   real decisions (panel vs. own participants, interview mode, target size); answer
   mechanical steps yourself when the user already told you the answer.
5. **Show the study guide verbatim.** After every `create_study`/`edit_study` response,
   render the entire `state.studyGuide` to the user: every block in order with its title,
   and every question and statement under it, full text and all answer options. Never
   summarize or truncate — the user is signing off on exactly what respondents will see.
   If the guide is still empty, say so.
6. **Iterate** with further `edit_study` prompts until the user is happy.

### 2. Launch (spends real money — always gate this)

`launch_study` publishes the study and starts recruitment, deducting credits from the
organization's balance. Treat it like a purchase:

1. Call `get_study_state` and read the `launch` block: published state, credit balance,
   per-recruitment cost and eligibility, total cost, and blockers.
2. Show the user what will happen: which recruitments will launch, what it costs, the
   balance before/after, and anything that will be skipped for insufficient credits.
3. **Get an explicit go-ahead.** Never call `launch_study` on your own initiative, as a
   side effect, or because the user said "set up" or "create" a study — those words mean
   draft it. Only an unambiguous instruction to launch/go live/start recruiting counts.
4. Launch, then report the result: what launched, what was skipped and why, and the new
   balance. If credits are short, the user must add them in the Listen dashboard — that
   can't be done over MCP.

`launch_study` is safe to re-call: already-launched recruitments are skipped, never
relaunched.

### 3. Edit an existing study

1. Find it: `list_studies` with `textHint` when the user names it (never page through
   everything when you have a name to filter on).
2. Snapshot it with `get_study_state` so you and the user see what's there now.
3. Send natural-language edits with `edit_study` (`studyId`, `prompt`, **no** `chatId` on
   the first turn — a fresh chat on an existing study is direct-edit mode). Reuse the
   returned `chatId` for follow-up edits in the same session.
4. Render the updated study guide verbatim, same rule as creation.
5. **If the study is already live**, edits sit in a dev revision that respondents don't
   see until you call `publish_study`. Tell the user this and publish when they confirm.
   (`launch_study` also auto-publishes, but use `publish_study` when recruitment is
   already running and you only need the content update.)

### 4. Read results

- `list_studies` → status, response counts, and whether analysis is ready (`has_analysis`).
- `get_study_analysis` → the AI-generated report (only when `has_analysis` is true).
- `get_study_responses` → paginated transcripts; filter with `question_numbers` or
  `readable_ids` instead of pulling everything.
- `get_response` → deep-dive on one respondent, including panel attributes and URL params.
- `search_across_studies` → keyword search over titles, goals, and analyses when the user
  asks "did we ever research X?".

**Grounding is non-negotiable.** When you surface findings from analyses or transcripts,
every quote must be verbatim and immediately followed by its exact `[Source]` link copied
from the tool output. Never invent, paraphrase, merge, or re-attribute quotes or numbers;
if a claim isn't in the returned content, don't make it. This matters because researchers
act on these reports — an unsourced quote is indistinguishable from a fabricated one.

## Conversation-state discipline

- Track `studyId` and the latest `chatId` across turns. `create_study` mints the `chatId`;
  every `edit_study` response echoes the one in use.
- One `edit_study` call carries *either* `prompt` *or* `buttonClick`, never both.
- Don't re-call `create_study` to fix a study you just created — that makes a second
  study. Use `edit_study` with the existing `studyId`.
- Before any mutation on a study you didn't just create, `get_study_state` first.

## Cost model (so you can advise the user)

- **Self-recruit** ("I'll bring my own participants"): a shareable interview link, billed
  per completed response. Cheap to launch; the user drives traffic.
- **Panel**: Listen recruits respondents matching the audience; upfront cost that must fit
  the credit balance at launch time.
- Recruitments launch greedily in dashboard order — each one that fits the remaining
  balance launches; the rest are reported as skipped so you can tell the user exactly
  what didn't run.
