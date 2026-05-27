---
name: create-from-prompt
description: Used when `intent=create_agent` and `seed_kind=blank`. The user opened a draft with a freeform description of what they want — or with no description at all. Enter the interview at stage 1 (Goals) and let it drive.
---

# Create from prompt

## Starting state

- `seed_kind=blank` in the envelope.
- The draft contains a minimal `valet.yaml` only.
- The envelope body is the user's description, or empty (the dashboard supports a no-prompt entry — see `HeroPrompt`).

## First turn

1. Read the envelope body.
2. **If the body is empty:** ask one open question (*"What do you want this agent to do?"*) before doing any file work. Don't checkout yet. End the turn.
3. **If the body has content:** infer archetype using the rules in `skills/interview/SKILL.md`. Checkout the draft.
4. Copy the matching archetype's seed `SOUL.md` (from `seeds/SOUL.<archetype>.md`) into the draft as `SOUL.md`, then copy `seeds/valet.yaml.template` over the minimal `valet.yaml`. Prefill any `<EDIT — …>` markers whose value is obvious from the user's description — names, services, channel hints. Don't guess specifics the user hasn't said.
5. Validate (`valet agents drafts validate <draft_id>`).
6. Push once: `Draft <archetype> agent from prompt`.
7. Reply: one sentence confirming the framing, then the first interview question targeting the most prominent remaining `<EDIT — …>`. Don't summarize what you just did — the customize pane shows it.

The framing-confirmation sentence is short and concrete:

> *"Sounds like a transform agent — fires when a PR opens, summarizes it for Slack. First question: should this fire on every PR, or only some (e.g., not drafts)?"*

Don't say *"I've copied a template and prefilled some values."* The user can see the draft.

## Subsequent turns

Hand to `skills/interview/SKILL.md`. The interview finds the next `<EDIT` marker and drives, turn by turn, until none remain. `skills/tool-discovery/SKILL.md` covers stage 4 (Tools). `skills/iterate/SKILL.md` is the per-turn work loop.

## If the archetype guess turns out wrong

If a few turns in the user's answers reveal a different shape (e.g. you started with `transform` and they describe a multi-session state machine), don't panic. Tell the user you'd like to switch templates, copy the new archetype seed over `SOUL.md` (preserving the Configuration values that still apply), and push once: `Recategorize as <new archetype>`. One reset is fine; two means you're guessing too aggressively.
