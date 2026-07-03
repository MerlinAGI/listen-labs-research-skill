# Worked examples

Realistic end-to-end flows showing which tools to call, in what order, and where to stop
and talk to the user. Tool calls are shown as `tool_name(args)`; responses are abbreviated
to the fields that drive the next decision.

---

## Example 1 — Create and launch a new study

> **User:** I want to find out if marketing managers would pay for an AI tool that writes
> ad copy. Can you set up interviews with about 25 of them and get it running?

The user gave goal, audience, and size — and "get it running" is an explicit launch
instruction, so a launch gate at the end is expected. Still confirm cost before spending.

**Turn 1 — org + seed**

```
list_creatable_orgs()
→ { orgs: [{ name: "Jarvis", role: "admin" }], total: 1 }        # one org: proceed

create_study(prompt: "Interview ~25 marketing managers who run paid ad campaigns.
  Goal: learn whether they would pay for an AI tool that writes ad copy — current
  workflow, pain points with copywriting, willingness to pay and price expectations.")
→ { studyId: "111…", chatId: "aaa…", onboardingStatus: "recruitment_type",
    agentQuestion: "How would you like to recruit?", nextActions: [
      { type: "button_press", buttonType: "recruitment_choice", actionTitle: "Use panel",
        data: { recruitmentType: "panel" } },
      { type: "button_press", buttonType: "recruitment_choice", actionTitle: "Bring my own",
        data: { recruitmentType: "self" } } ],
    state: { studyGuide: … } }
```

Render the study guide verbatim (even if partial), then relay the real decision to the
user: *"Listen can recruit the marketing managers from its panel (costs credits per
recruit) or you can share a link with your own contacts. Which do you want?"*

**Turn 2 — user says "use the panel"**

```
edit_study(studyId: "111…", chatId: "aaa…",
  buttonClick: { type: "button_press", buttonType: "recruitment_choice",
                 actionTitle: "Use panel", data: { recruitmentType: "panel" } })
→ { chatId: "aaa…", onboardingStatus: "recruitment_target", agentQuestion: "…", … }
```

The buttonClick is copied from `nextActions`, not hand-built. Continue through the stages
the same way — answer with `prompt` when the user already told you the answer (audience:
marketing managers, n=25; interview mode: ask the user, or default to `text` for a
willingness-to-pay study if they don't care). After the final turn, render the complete
study guide verbatim and ask the user to sign off.

**Turn 3 — launch gate**

```
get_study_state(studyId: "111…")
→ { launch: { isPublished: false, creditBalance: 500,
      recruitments: [{ type: "panel", cost: 375, eligibleAlone: true }],
      totalCost: 375, blockers: [] } }
```

Tell the user: *"Launching recruits 25 panelists for 375 credits; balance goes
500 → 125. Confirm launch?"* Only after a clear yes:

```
launch_study(studyId: "111…")
→ { published: true, launched: [ …panel recruitment… ],
    skippedInsufficientCredits: [], balanceBefore: 500, balanceAfter: 125 }
```

Report what launched and the new balance. Suggest checking back for responses later.

---

## Example 2 — The user says "set up a study" (create ≠ launch)

> **User:** Set up a study to test our new onboarding flow with existing users. I'll send
> it to our mailing list myself.

"Set up" means draft, and "I'll send it myself" means self-recruit. Do the creation flow
(Example 1, choosing `recruitmentType: self`), render the final guide, and **stop**:

> *"The study is drafted — here's the full guide above. It's not live yet. Say the word
> and I'll launch it, which activates your shareable interview link (responses bill per
> completion)."*

Do not call `launch_study` until they answer. When they do, the launch auto-creates and
activates the self-recruit link if onboarding didn't, and returns the link to share.

---

## Example 3 — Edit a live study, then publish

> **User:** On my LooksMax trust study, add a screener question that filters for people
> who've used a face-rating app before, and bump the target to 50.

```
list_studies(textHint: "LooksMax")
→ 3 matches                                   # ambiguous — ask which one

# user picks "LooksMax UX Trust & Usability Study"
get_study_state(studyId: "235…")
→ { …current guide + screener…, launch: { isPublished: true, hasDevChanges: false } }

edit_study(studyId: "235…",                    # no chatId → fresh direct-edit session
  prompt: "Add a screener question filtering for people who have used a face-rating
   app before (screen out those who haven't), and raise the recruitment target to 50.")
→ { chatId: "bbb…", state: { studyGuide: … } }
```

Render the updated guide + screener verbatim. The study is live and `hasDevChanges` is
now true, so respondents still see the old version — say so, and on the user's confirm:

```
publish_study(studyId: "235…")
→ { published: true, prodRevisionId: "…" }
```

If the target bump created a new unlaunched recruitment that needs credits, `publish_study`
won't start it — check `get_study_state`'s launch block and offer `launch_study` for that
part, with the usual cost preview.

---

## Example 4 — Pull findings with sourced quotes

> **User:** What did people say about pricing in the ad message study? Give me the
> highlights.

```
list_studies(textHint: "ad message")
→ [{ id: "287…", name: "StyleFits.ai Meta Ad Message Testing",
     response_count: 29, has_analysis: true }]

get_study_analysis(study_id: "287…")
```

Summarize the pricing-related findings, keeping every quote verbatim with its `[Source]`
link exactly as returned:

> Several respondents anchored on subscription fatigue — "I already pay for three AI
> tools, this would have to replace one" [Source](https://listenlabs.ai/response/…?message=12)

For a question the analysis doesn't cover, go to the raw transcripts instead — filtered,
not wholesale:

```
get_study_responses(study_id: "287…", question_numbers: [4, 5], max_responses: 15)
```

To zoom in on one interesting respondent (e.g. respondent 7):

```
get_response(study_id: "287…", readable_id: 7)
```

---

## Example 5 — Website usability test (screen share)

> **User:** We just shipped a redesigned checkout at shop.acme.com. Can you set up a
> usability test where ~15 people actually go through the site and think aloud?

Website and prototype tests hinge on two things the other examples don't exercise: a
**screen-share interview mode** and a **task-based study guide that names the URL**.
Put both in the seed prompt so the creation agent builds the right guide:

```
create_study(prompt: "Moderated usability test of the redesigned checkout flow at
  https://shop.acme.com. Interview ~15 online shoppers. Respondents share their screen,
  open the site, and complete tasks while thinking aloud: (1) find a product under $50,
  (2) add it to the cart, (3) go through checkout up to the payment step. Probe on
  confusion, hesitation, trust concerns, and anything they'd change.")
→ { studyId: "222…", chatId: "ccc…", … }
```

When onboarding reaches the interview-mode stage, pick a screen mode from `nextActions`:

```
edit_study(studyId: "222…", chatId: "ccc…",
  buttonClick: { type: "button_press", buttonType: "interview_mode",
                 actionTitle: "Audio + screen share",
                 data: { interviewMode: "audio_screen" } })
```

- `audio_screen` — voice + shared screen; the usual pick for website testing.
- `video_screen` — camera + screen, when the user also wants facial reactions.
- Plain `video` / `audio` / `text` modes can't see the site. If the user asked for a
  website or prototype test, don't let onboarding land on one of these — say why and
  pick a screen mode.

When you render the finished guide verbatim, check it like a researcher before asking
for sign-off: the site URL must appear in the instructions respondents actually see,
the tasks should be concrete and ordered, and there should be think-aloud prompting
("tell me what you're looking at / expecting"). Fix gaps with plain `edit_study`
prompts ("put the checkout URL in the intro block", "make task 2 more specific").

The launch gate is unchanged (Example 1): `get_study_state` cost preview → explicit
confirm → `launch_study`.

---

## Example 6 — Not connected yet

> **User:** Can you launch a Listen study for me?

No `create_study`/`launch_study` tools in the session → the MCP server isn't connected.
Follow [connect.md](connect.md): identify the user's client, give them the matching setup
steps (`https://listenlabs.ai/mcp`, OAuth), have them restart/enable the connector, then
verify with `list_creatable_orgs` and proceed with Example 1.
