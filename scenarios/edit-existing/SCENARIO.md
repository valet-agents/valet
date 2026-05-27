---
name: edit-existing
description: Used when `intent=edit_agent`. The user clicked Edit on a deployed agent and wants to change something. Classify the request, enter the interview at the right stage.
---

# Edit existing

## Starting state

- `intent=edit_agent` in the envelope.
- The draft is a working copy of the live agent's `main` branch — every file already exists.
- The envelope body describes what to change. May be terse (*"be less verbose"*) or specific (*"add Slack"*).
- The agent is deployed and may be running real traffic. Edits are reviewable, not catastrophic — but the user has built habits around the agent's current behavior, so treat changes as deliberate, not exploratory.

## Out of scope

This scenario assumes the draft was produced by — or is structurally compatible with — the concierge's seed templates. Specifically:

- `valet.yaml` exists and parses.
- `SOUL.md` exists.

If those preconditions don't hold (an old pre-concierge agent), `valet agents drafts validate <draft_id>` will surface the gap. The current flow does **not** auto-migrate such agents — `ReportError` with the validation failure and reply to the user that this agent needs to be re-created through the concierge before it can be edited here. Old agents are out of scope on purpose; auto-migration is a separate body of work.

## First turn

1. Read the envelope body and **classify** the request:
   - **Behavior tweak** (*"be less verbose"*, *"stop replying in markdown"*, *"include the company name"*) → enter interview at stage 2 (Guardrails) or stage 3 (Workflow). The file you touch is `SOUL.md`.
   - **Capability change** (*"add Slack"*, *"also watch repo X"*, *"stop using GitHub"*) → enter at stage 4 (Tools). Touch `valet.yaml`; possibly add or remove `channels/<name>.md`.
   - **Personalization** (*"change the channel to #engineering"*, *"watch repo Y instead of X"*, *"run daily not hourly"*) → enter at stage 5 (Personalize). Touch the **Configuration** section of `SOUL.md` (and the manifest if a schedule is inline).
   - **Marketing copy** (*"rename it"*, *"change the hero"*, *"the description should say…"*) → `valet.yaml` only. No interview stage; just edit.
2. Checkout the draft. Read only the file the classification implies — don't pre-read the whole tree.
3. **Re-introduce `<EDIT — …>` if needed.** If the change introduces a new field the user hasn't specified yet (*"also watch a second repo"* → which repo?), write the field with an `<EDIT — hint>` marker. That sets `concierge.status: drafting` and the interview picks up the question on the next turn.
4. Set `concierge.status: drafting` in `valet.yaml` if any `<EDIT` markers exist after your edit (even ones already present from the prior session).
5. Propose the change concretely in your reply. Ask one clarifying question only when the request is genuinely ambiguous about which file or which value. Don't ask the user to spell out what you can reasonably propose.
6. Validate. Push once with a message naming the change: `Tighten verbosity guardrail`, `Add second repo to watch list`, `Rename to Lead Scorer`.

## Subsequent turns

Hand to `skills/interview/SKILL.md` at the classified stage. The user may pivot — *"actually, while we're at it, can it also…"* — re-classify and re-enter at the new stage.

When all `<EDIT` markers are clean and validate passes, transition `concierge.status: ready` and tell the user Configure & Deploy is open.

## When the user says "actually I want it to do something completely different"

That's a sign to suggest creating a new agent rather than editing this one. The deployed agent has users / triggers attached; tearing its identity down inside an edit-mode draft is risky. Reply with the suggestion ("Sounds like a different agent — want me to spin up a fresh one? We can keep this one running until you're happy with the new one"). Don't unilaterally pivot.
