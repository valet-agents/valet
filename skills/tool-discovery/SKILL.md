---
name: tool-discovery
description: Bridge from goals/guardrails/workflow to actual connectors and channels. Favor already-installed org resources; concierge picks and surfaces; user can override. No catalog match → tell the user and point them at support@valet.dev.
---

# Tool discovery

## When to invoke

After interview stages 1–3 (Goals, Guardrails, Workflow) are settled. The more the concierge knows about what the agent does, the better the tool match. Asking "which Slack channel?" before the user has agreed on what the agent posts wastes a turn.

## Discovery order

Always look at what's already set up at the org first — every existing connector and channel the user can reuse is one less thing they have to configure.

1. **Already-installed org connectors** — `valet connectors --org <org>`
2. **Already-installed org channels** — `valet channels --org <org>`
3. **Catalog connectors** — `valet connectors catalog`
4. **Catalog channels** — `valet channels catalog`

Run these once per session; the conversation history caches them. Re-run only if you have reason to think the org's setup changed mid-session.

## Picking

The concierge picks. The user sees the choice in the customize pane and can ask for a change.

Tie-break order:

1. **Already-installed org resource** that matches the goal — pick this unambiguously.
2. **Catalog entry** that matches — pick this and reply with a one-line rationale ("Using the GitHub catalog connector — you'll connect it during configure").
3. **Closest catalog entry** if nothing matches exactly — propose it; ask the user to confirm.
4. **No match** — see below.

Match the user's vocabulary. If they said "Slack," propose Slack, even if "Teams" exists in the catalog. Match the trigger shape: webhook for event-driven, cron for scheduled, heartbeat for polling.

## Writing the choice into the draft

When a tool is picked:

- Add the catalog entry to `valet.yaml` under `connectors:` or `channels:` (see `skills/authoring/SKILL.md` for the manifest shape and required fields).
- If the choice has agent-specific configuration the user must personalize (a channel name, a webhook target, a schedule), add `<EDIT — …>` markers for those values either in the manifest or in the Configuration section of `SOUL.md`. The next interview turn will pick them up.
- If a channel needs a channel file (`channels/<name>.md`), create it from the relevant scaffold and add `<EDIT — …>` markers for the per-agent values inside.

Push with a message naming the choice: `Use existing GitHub connector` or `Add Slack channel from catalog`.

## When no catalog entry matches

Don't invent a `catalog:` reference — it won't resolve and validate will fail. Instead:

- Tell the user, plainly, that what they're asking for isn't in the catalog today.
- Offer the closest thing that does exist, if one does. "There's no Notion connector yet; the catalog has Linear and Airtable, which often cover the same use case — would either work?"
- If there's truly no alternative, point the user at **support@valet.dev** so the team can prioritise adding it. Don't promise a timeline.

The user can either change tack (pick a tool the catalog has) or pause the interview and come back once the catalog covers their need. Both are acceptable outcomes — better than shipping a manifest with a phantom catalog reference.

## Heartbeat and cron channels

These are inline channels (no catalog entry needed). The manifest declares them with `type: heartbeat` + `every: <interval>` or `type: cron` + `cron: "<expr>"` or `schedule: "<human>"`. See `skills/authoring/SKILL.md` for the schema gotchas.

For monitor-archetype agents the schedule is core configuration — put it in the **Configuration — edit before deploy** section of `SOUL.md` (the seed template already does this) so the user can change cadence later without editing the manifest by hand.
