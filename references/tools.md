# Listen MCP tool reference

Every tool the Listen Labs MCP server exposes, ordered by lifecycle. Tool names may carry
an MCP prefix depending on how the connector was named (e.g. `mcp__listenlabs__create_study`);
the base names below are stable.

## Contents

- [Discovery: list_creatable_orgs, list_studies, search_across_studies](#discovery)
- [Creation: create_study](#create_study)
- [Editing: edit_study](#edit_study)
- [Inspection: get_study_state](#get_study_state)
- [Going live: publish_study, launch_study](#going-live)
- [Results: get_study_analysis, get_study_responses, get_response](#results)

---

## Discovery

### list_creatable_orgs

Organizations the user can create studies in.

- **Params** (all optional): `textHint` (case-insensitive substring on org name),
  `cursor`, `limit` (default 10, max 25).
- **Returns**: `{ orgs: [{ id, name, role }], total, hasMore, nextCursor? }`.
- Call it before `create_study` whenever the user hasn't named a workspace. One org →
  proceed silently. Multiple → ask the user to pick.

### list_studies

Paginated list of studies the user can access (50 per page).

- **Params** (all optional): `textHint` (case-insensitive substring on study title,
  matches published or draft title), `status` (`open` | `closed`), `cursor`.
- **Returns**: `{ studies: [{ id, name, status, created_at, response_count, has_analysis }], total_count, next_cursor }`.
- When the user names a study, always pass `textHint` instead of paging blindly. If the
  hint returns 0 studies, ask for a different name — don't dump the full list.
- `total_count` respects the filter, so it answers "how many studies do I have?".
- Keep `textHint`/`status` identical across pages of one search.

### search_across_studies

Keyword search across study titles, goals, and analyses. Use for "have we ever researched
X?" questions that span studies.

- **Params**: `query` (required), `max_results` (default 10).

---

## Creation

### create_study

Starts a **guided creation conversation** with the platform's creation agent. This call
seeds the study; the agent then walks stages: `study_goals` → `recruitment_type` →
`recruitment_target` → `interview_mode` → `study_guide`.

- **Params**: `prompt` (required — rich plain-language description: goal, who to
  interview, what to learn), `orgName` or `orgId` (required whenever the user belongs to
  more than one org). **Omit `chatId`** — the server mints one.
- **Returns**: `studyId`, `chatId`, summary, `onboardingStatus`, optional `agentQuestion`,
  `suggestions`, `nextActions` (structured button events for the next turn), and the full
  study `state` including `state.studyGuide`.
- Persist `studyId` and `chatId`. All follow-up turns go through `edit_study` with both.
- Calling `create_study` twice creates two studies. To fix or continue one you just made,
  use `edit_study`.
- **Render `state.studyGuide` verbatim to the user after every call** — every block, every
  question, every answer option, in order. If the guide is empty because onboarding is
  early, say so explicitly.

The richer the initial prompt, the fewer onboarding turns. Compare:

- Weak: `"a study about our app"`
- Strong: `"Interview 25 US-based marketing managers who run paid social campaigns.
  Goal: find out whether they'd pay for AI-generated ad copy, what they use today,
  and what price feels fair. Text interviews are fine."`

---

## Editing

### edit_study

Natural-language edits via the creation agent. The server picks a mode from the chat's
history:

1. **Direct-edit mode** — fresh `chatId` (or omitted) on an existing study. Send a plain
   `prompt`: "remove Q3", "switch the panel to healthcare professionals and bump to 200
   people", "add a screener question that filters for parents". No `buttonClick`.
2. **Onboarding mode** — `chatId` carried over from `create_study` this session. The agent
   walks the remaining stages; respond with `buttonClick` values copied from the previous
   turn's `nextActions`, or `prompt` to answer the agent's question in free text.

- **Params**: `studyId` (required), then **either** `prompt` **or** `buttonClick` — never
  both in one call. `chatId`: omit on the first turn of a new edit session; pass the
  previous turn's echoed `chatId` to continue a conversation.
- `buttonClick` shape: `{ type: "button_press", buttonType, actionTitle, data? }` where
  `buttonType` ∈ `next_step | recruitment_choice | interview_mode | skip_tutorial`.
  Copy these from `nextActions` verbatim rather than constructing them — the server
  defines what's clickable at each stage.
  - `data.recruitmentType`: `self` (user brings participants) | `panel` (Listen recruits).
  - `data.interviewMode`: `video` | `video_screen` | `audio` | `audio_text` |
    `audio_screen` | `text`.
- Call `get_study_state` first when you don't have a fresh snapshot of the study.
- Edits to an **already-launched** study land in a dev revision that respondents don't see
  until `publish_study` (or the next `launch_study`) promotes it. Always tell the user
  when their edit is sitting unpublished.

---

## Inspection

### get_study_state

Slim snapshot of a study: title, audience, study guide outline with questions, screener,
recruitment setup — plus a `launch` block:

- `isPublished`, `hasDevChanges` (unpublished edits exist)
- organization credit balance
- per-recruitment cost and eligibility (`alreadyLaunched`, `eligibleAlone`)
- total cost to launch everything unlaunched
- `blockers` (e.g. `no_recruitments`, `study_busy`)

**Params**: `studyId`.

Use it (a) before editing, so you know what's there; (b) before launching, to preview
costs and blockers; (c) whenever the user asks "what's the status of my study?".

---

## Going live

`publish_study` and `launch_study` do different jobs — don't conflate them:

| | publish_study | launch_study |
|---|---|---|
| Promotes dev revision to prod | ✅ | ✅ (automatically, if needed) |
| Starts recruitment | ❌ | ✅ |
| Spends credits | ❌ | ✅ |
| Typical use | Push edits to a live study | Go live / start collecting responses |

### publish_study

- **Params**: `studyId`.
- **Returns**: `published` (boolean), `prodRevisionId`, or a note when dev and prod were
  already identical (no-op).
- Needed after `edit_study` on a launched study, otherwise respondents keep seeing the
  old guide.

### launch_study

Publishes the dev revision (if needed) and starts **every unlaunched recruitment that
fits the credit balance**, greedily in dashboard order — each recruitment whose cost fits
the running balance launches and deducts; the rest are skipped and reported.

- **Params**: `studyId`.
- **Returns**: `published`, `launched`, `skippedInsufficientCredits`,
  `skippedAlreadyLaunched`, `balanceBefore`, `balanceAfter`, `totalCostLaunched`, failures.
- Self-recruit links bill per response and always launch if unlaunched. Panel
  recruitments need their upfront cost to fit the balance. Custom-link recruitments
  (external-panel dashboards) are **not** launched by this tool.
- If the user chose "I'll bring my own participants" but never created a link, this tool
  creates and activates one automatically — so a draft can go live in one step.
- Idempotent-ish: safe to re-call; already-launched recruitments are skipped, not
  relaunched.
- **Always** precede with `get_study_state` + explicit user confirmation (see SKILL.md —
  this spends real credits). Credits can only be added in the Listen dashboard, not via
  MCP.

---

## Results

All three results tools enforce the same grounding contract: quotes are returned verbatim
with `[Source]` links (`https://listenlabs.ai/response/<id>?message=<n>`). When you
surface a quote, copy it and its link exactly; never merge, reorder, paraphrase, or reuse
links; never state findings that aren't in the returned content. A quote without a
`[Source]` link in the output is unusable — drop it.

### get_study_analysis

The AI-generated analysis report as markdown, with sourced quotes and appended sections
for the real values behind named metrics, charts, tables, and reels.

- **Params**: `study_id`.
- Only useful when `list_studies` shows `has_analysis: true` for that study.

### get_study_responses

Paginated respondent transcripts for a study.

- **Params**: `study_id` (required), `max_responses` (default 10, max 50),
  `question_numbers` (1-indexed filter), `readable_ids` (filter to specific respondents),
  `cursor`.
- Filter aggressively — pulling every transcript for a 200-person study blows context for
  no reason. If the user cares about one question, pass `question_numbers`.

### get_response

Deep-dive on a single respondent's interview: structured transcript with question
tracking, input types, multiple-choice data, per-item `source_url` deep links, plus
`url_params` (custom recruitment-link parameters) and `linked_data` (panel attributes
like Age, Country) when present.

- **Params**: `study_id`, `readable_id` (respondent number from `get_study_responses`),
  `cursor` for long transcripts.
