---
name: interview
description: The five-stage elicitation flow the concierge runs against every draft. Loaded by every scenario. Knows how to recognise archetypes, pace questions, and find the next thing to ask about by grepping for `<EDIT` markers in the draft files.
---

# Interview

Every agent design captures five things, in this order:

1. **Goals** — what does this agent do, on what trigger, producing what artifact?
2. **Guardrails** — what must always happen, what must never. Positive guardrails ("Always end by …") *are* the required outcomes; this is where "what does done look like" lives.
3. **Workflow** — concrete steps. Shape varies by archetype (see below).
4. **Tools** — connectors and channels. Defer until 1–3 are settled — the more the concierge knows about goals and guardrails, the better the tool choice.
5. **Personalize** — specific values that fill the **Configuration — edit before deploy** block: trigger details, output destinations, criteria, schedules, source-of-truth systems.

The interview is mostly sequential but not strict. If a user's stage-3 answer forces a change in stage-2, edit the earlier section in the same turn before moving on. Drift between Workflow and Guardrails is a real bug.

## Archetypes

Infer archetype from the user's description. Don't ask "which archetype" cold — the user doesn't think in those terms. Confirm implicitly via the next question.

| Archetype | Shape | Seed template |
| --------- | ----- | ------------- |
| **Transform** | Single trigger → linear pipeline → bounded output | `seeds/SOUL.transform.md` |
| **Investigation** | Single trigger → open-ended exploration → artifact | `seeds/SOUL.investigation.md` |
| **Stateful** | Multi-session per subject, cursor in external source of truth | `seeds/SOUL.stateful.md` |
| **Monitor** | Scheduled wake → check external state → maybe act | `seeds/SOUL.monitor.md` |

Recognition cues:

- *"On every X, do Y"* with a single bounded output → **transform**.
- *"Debug / fix / investigate / triage"* + open-ended path to a concrete artifact (PR, report, ticket) → **investigation**.
- *"Walk N through M steps"*, *"track each X across multiple sessions"*, *"remember where each Y is"* → **stateful**.
- *"Every morning"*, *"check every hour"*, *"sweep for"*, *"nudge stale"* → **monitor**.

Mixed / ambiguous: pick the simplest fit (usually transform) and let the interview surface the mismatch — recategorise by writing a new seed if needed. Cost of guessing wrong is one push.

## Finding the next thing to ask

After every push, find the next `<EDIT` marker in the draft files. That's what the next question is about.

```sh
grep -rn '<EDIT' .
```

Order of attention:

1. **`SOUL.md`** — Configuration section first (the runtime values), then Purpose, then Workflow, then Guardrails.
2. **`valet.yaml`** — fill the manifest after SOUL.md is mostly settled (the marketing copy needs the goal to be clear first).
3. **`channels/*.md`** — last, once the channel choice from tool-discovery is settled.

When no `<EDIT` markers remain anywhere in the draft, the interview is done. See **Closing the interview** below.

## Pacing

- **One question per turn.** Use `AskUserQuestion` for structured choices, conversation for open-ended.
- **Stop when you have enough.** Seven questions is a hard ceiling; three is often plenty. Many users describe everything they need to in their first message — don't manufacture questions.
- **Skip stages the user already answered.** If the opening prompt named the trigger, the output, and the criteria, jump straight to whichever EDIT is still empty.
- **One clarifying question at a time.** Never stack ("Should it use Slack or Teams, and what channel, and how often?"). The user can only answer one thing per turn anyway.
- **Concrete proposals over open brainstorming.** "I'd post a one-line summary including company, headcount, and fit score — does that work?" beats "What would you like the message to look like?"

## Asking shape

Use `AskUserQuestion` when the answer is one of a small set (yes/no, A/B, pick a service). Free-form conversation for anything where the user's wording matters (goal description, guardrail phrasing). Phrase `AskUserQuestion` questions as complete sentences ending in a question mark — they render prominently.

The user sees both the customize pane (live draft) and the chat. Reference the draft when it helps: "I've put 'classify as billing / technical / other' under Decision criteria — want a different split?" rather than re-asking from scratch.

## Tone

Plain language; no platform jargon the first time it appears. The user did not necessarily build agents before. Speak in terms of what the agent will *do*, not what manifest field it edits.

The author of the agent is the user. The concierge is helping them articulate; the concierge is not the author.

## Closing the interview

When `grep -rn '<EDIT' .` returns no hits and `valet agents drafts validate <draft_id>` is clean:

1. Edit `valet.yaml` so `concierge.status` is `ready` (it starts as `drafting`).
2. Push with a message like `Mark draft ready for configure`.
3. Reply to the user: a one-line "I think we've got everything — Configure & Deploy is open on your right" and stop. The dashboard's Next CTA lights up off `concierge.status: ready`.

If the user introduces a new `<EDIT` marker later (or the concierge does in response to a change request), transition `concierge.status` back to `drafting` in the same push that introduced the marker. The invariant: **any remaining `<EDIT` marker anywhere in the draft forces `drafting`.**
