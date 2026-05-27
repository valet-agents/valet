---
name: create-from-manifest
description: Used when `intent=create_agent` and `seed_kind` is `catalog` or `github`. The shape is decided by the seed; personalization is the work. Enter the interview at stage 5 (Personalize).
---

# Create from manifest

## Starting state

- `seed_kind=catalog` (seed_source names a catalog entry) or `seed_kind=github` (seed_source is a public repo URL).
- The draft already has a populated `valet.yaml` + `SOUL.md` + possibly `channels/*.md` from the seed.
- The envelope body may be empty (the user clicked a template chip from the dashboard).

## First turn

1. Checkout the draft.
2. Read `valet.yaml` and `SOUL.md`. Read the **Configuration — edit before deploy** section if present.
3. **For github seeds, validate first.** Public repos can ship malformed manifests; if `valet agents drafts validate <draft_id>` errors, surface the validation errors to the user as the first reply and stop. Don't proceed until the seed is valid (the user may want to switch sources).
4. **Identify what needs personalization:**
   - Run `grep -rn '<EDIT' .` from the checkout root. Every hit is a personalization site.
   - For catalog seeds that don't yet use the `<EDIT — …>` convention (most catalog agents today), look for placeholder values in the Configuration section: bare-looking emails (`you@example.com`), `your-X` strings, sentinel channel names (`#channel`). Don't rewrite the seed into `<EDIT>` form en masse — just identify what needs the user's input.
5. **Set `concierge.status: drafting`** in `valet.yaml` if the seed didn't already (catalog seeds may not carry the block).
6. **If the envelope body has content:** treat it as the user's first personalization input. Apply what you can directly — fill the obvious EDITs — and reply with one targeted follow-up question.
7. **If the body is empty:** reply with one sentence confirming what the template does (the user clicked a chip — they may not have read the marketing copy), then ask the first personalization question targeting the most prominent remaining EDIT.
8. Validate. Push once: `Personalize <agent-name> from <catalog|github> seed`.

The confirmation sentence is concrete and short:

> *"This agent watches GitHub PRs and posts summaries to Slack. Quick question to get it pointed at your setup: which repo should it watch?"*

## Subsequent turns

Hand to `skills/interview/SKILL.md`, entering at stage 5 (Personalize). The interview drives until no `<EDIT` markers (or unfilled placeholders) remain.

The user may loop back to stages 2–3 — *"actually, I want it to only post on weekdays"* is a guardrail change. That's fine; the interview handles non-strict ordering. Edit `SOUL.md` Guardrails as needed; the Configuration value the guardrail references comes along for the ride.

## Deploy-as-is seeds

Some catalog agents have no personalization sites — they're complete by design (e.g., a stateless echo agent). For those, after step 4 above:

- Run `valet agents drafts validate <draft_id>` to confirm cleanliness.
- Set `concierge.status: ready` directly.
- Push: `Mark draft ready for configure`.
- Reply: *"This template is ready to deploy as-is — Configure & Deploy is open on your right. Anything you'd like to tweak first?"*

The user can still ask for changes; that re-enters the interview normally. But the default is "you're done."
