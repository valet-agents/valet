---
name: authoring
description: Reference for writing a target agent's files — `SOUL.md`, `valet.yaml`, and `channels/<name>.md`. Loaded by every scenario whenever a turn writes or edits draft files. This is a reference, not a flow: it owns no session and runs no CLI lifecycle commands.
---

# Authoring reference

The conventions below are the source of truth for any file the concierge writes into a draft.

The flow rules (`Edit` not `Write`, validate-before-push, one push per turn) live in `skills/iterate/SKILL.md`. The interview ordering lives in `skills/interview/SKILL.md`. This file is about *file shape* only.

## What each file is for

- **`valet.yaml`** — the manifest. User-visible marketing copy (`story.*`) plus the connector / channel declarations. Everything the user sees on the customize page comes from this file.
- **`SOUL.md`** — the target agent's own prompt. Identity, configuration values, purpose, workflow, guardrails. The user does not see it unless they open the source.
- **`channels/<name>.md`** — per-channel preprocessor prompts (e.g. Quick Filter for Slack, cursor logic for heartbeat / cron). Not shown on the customize page.

## The Configuration — edit before deploy convention

The seed `SOUL.md` templates open with a `## Configuration — edit before deploy` section that holds the runtime values the agent reads each fire — output channel, criteria, schedule, source-of-truth system. The rest of `SOUL.md` references those values **by name** rather than hard-coding them:

> *"…and post to the **Output destination** above."*

Why: when the user wants to change a value, the edit is a single bullet, not a hunt through prose. When the concierge edits a value (e.g. swapping the destination channel), the rest of the file is automatically consistent.

When you add a new runtime value to an agent, put it in the Configuration section and reference it by **bolded name** elsewhere — never duplicate the value inline. Drift between Configuration and the rest of `SOUL.md` is a real bug.

## The `<EDIT — …>` marker convention

Anywhere a value still needs to be filled in by the user, write:

```
<EDIT — short example or hint>
```

The em-dash separates the marker from a concrete example. Examples:

- `<EDIT — e.g. you@example.com>`
- `<EDIT — 2-3 sentences naming what this agent does>`
- `<EDIT — concrete step naming a specific tool>`

The interview drives the concierge to find the next `<EDIT` and ask about it. The invariant: **any remaining `<EDIT` marker anywhere in the draft forces `concierge.status: drafting` in `valet.yaml`.**

Don't use `<EDIT>` for output-format slots (`<name>`, `<id>`, `<repo>`) — those are runtime tokens the agent fills at execution time, not human-fill placeholders. They use bare angle brackets without `EDIT —`.

## SOUL.md

The target agent's identity and behavior. Required.

```markdown
# <Agent Title>

## Configuration — edit before deploy

The agent reads these values from this section every fire. …

- **<Value name>:** `<value or <EDIT — hint>>`
- **<Value name>:** `…`

## Purpose

<2-3 sentences naming what this agent does and why. Name the specific
tools, inputs, and outputs.>

## Workflow

<Archetype-shaped. See seeds/ for the four shapes.>

## Guardrails

### Always
- <Required outcome. "Always" guardrails encode what done looks like.>

### Never
- <Bounded constraint.>
```

Other sections are optional and only needed when the archetype requires them:

- **Cursor** — required for `stateful` and `monitor` archetypes. Names where state lives and how to read it.
- **Personality** — optional. Weight varies by archetype: maximal for stateful conversational agents, modest for transform, low for investigation.
- **Webhook Scope Rule** — required when any channel is webhook-driven. Scopes the agent's actions to the identifiers in the payload (see Channel files below).

### Synthesis rules

- **Purpose:** specific *what* + *why*. Name inputs, outputs, and tools. *"Monitors YouTube channel X for new episodes, downloads transcripts, and posts digests to #channel on Slack."* Not *"Processes data."*
- **Workflow:** concrete numbered steps with actual tool names. For investigation and stateful archetypes the "steps" are scaffolding for an open-ended loop or state machine; for transform and monitor they're a linear pipeline.
- **Guardrails — Always:** positive patterns the agent must follow *every* fire. This is the right place for "End by posting to **Output destination**" — *required outcomes are guardrails, not workflow steps.*
- **Guardrails — Never:** constraints the agent must avoid. Bounded; specific.
- **Placeholders:** for user-specific values (IDs, URLs, channel names) use `<EDIT — hint>`; for runtime-filled output slots use bare `<name>`.

### The target runtime is ephemeral

The agent you are authoring runs in a container with **no persistent filesystem.** It resets on every run; the agent cannot edit its own `SOUL.md` or skills; there is no key-value store. So never write a workflow that tells the agent to "remember" a value in a local file (`MEMORY.md`, scratch JSON, anything on disk). It silently vanishes before the next run.

Consequences:

- **Identity values the agent needs** (a spreadsheet to log to, a channel to post in, a repo to watch) must be pinned in the Configuration section of `SOUL.md` — as a concrete value the user gives you, or as an `<EDIT — hint>` for them to fill in. Never have the agent create the resource on first run and "remember" the ID; on the next run it will have forgotten and create a duplicate.
- **Cursors that prevent repeated work** (don't double-post, don't re-log a row) must derive from the system the agent acts on, not from local state. Read the destination to find where it left off — the latest row already in the sheet, the last message in the channel, a label or tag on already-handled items. This is the cursor logic the `stateful` and `monitor` seed templates already make explicit.

### Runtime values go in SOUL.md, not the manifest

A user request like *"set the REPOS env var"* resolves to a SOUL.md Configuration edit naming the repo concretely, never a manifest `env:` entry. The manifest has **no** env/vars/config/settings block — see Manifest schema gotchas below.

### Command connector references

When the agent uses a **command connector**, the workflow must reference the **connector name** as the command — not the npm package name or `npx` invocation. The connector name is the only name on the agent's PATH.

- ✅ `Run agentmail inboxes list`
- ❌ `Run npx agentmail-cli inboxes list` (bypasses secret injection)
- ❌ `Run agentmail-cli inboxes list` (command not found)

## valet.yaml

The manifest. Drives the dashboard's customize page and configure-flow wizard.

```yaml
name: <agent-name>
display_name: <Human-Readable Name>
description: >-
  <One sentence shown in the dashboard during setup>
category: <category>
story:
  hero: "<one short line, names the agent>"
  subheadline: "<one concrete sentence — services and reward>"
  steps:
    - role: trigger
      title: "<25–50 chars>"
      body: "<80–130 chars>"
    - role: action
      title: "<25–50 chars>"
      body: "<80–130 chars>"
    - role: outcome
      title: "<25–50 chars>"
      body: "<80–130 chars>"
connectors:
  - catalog: <catalog-entry-name>
    description: >-
      <Agent-specific context for this connector>
channels:
  - catalog: <catalog-entry-name>
    description: >-
      <Agent-specific context for this channel>
    events:
      - <event_type>

concierge:
  status: drafting   # the concierge owns this — see SOUL.md
```

Rules:

- `name` must match the agent name used to create the agent.
- Every `catalog:` value must come from `valet connectors catalog` / `valet channels catalog`. Don't invent entries; see `skills/tool-discovery/SKILL.md`.
- `story` must have exactly three steps, in order: `trigger`, `action`, `outcome`.
- A step's `catalog:` (when set) must match a `catalog:` on this manifest's `connectors:` or `channels:`. Leave it empty to render the agent monogram (useful for the middle "the agent thinks" step).
- `concierge.status` is `drafting` while the interview is in progress, `ready` when no `<EDIT` markers remain. The concierge owns this field; don't ask the user about it.
- Omit `connectors:` / `channels:` arrays entirely when the agent has none. Omit optional fields (`description`, `events`, `slot_descriptions`, `ui`) when catalog defaults suffice.

### Length targets

The hard caps are enforced by the validator. The sweet spots are what renders well in the wizard — target those, not the caps. Only push toward a cap when the extra characters carry real information.

| Field             | Sweet spot    | Hard cap |
| ----------------- | ------------- | -------- |
| `hero`            | 45–75 chars   | 80       |
| `subheadline`     | 110–170 chars | 200      |
| step `title`      | 25–50 chars   | 60       |
| step `body`       | 80–130 chars  | 140      |
| ui `headline`     | 30–55 chars   | —        |
| ui `blurb`        | 80–140 chars  | —        |
| ui `done_note`    | 20–45 chars   | —        |

### Voice and style

**Always:**

- Present tense, active voice. *"Posts the briefing"*, not *"Will post the briefing"* or *"The briefing is posted"*.
- Name the agent at least once in the hero — the user is meeting it for the first time.
- Name concrete things: services (*Slack*, *GitHub*), artifacts (*#ai-news*, *pull request*), times (*8am*, *every Friday*).
- Address the reader as *you*. Never *"the user"*.

**Never:**

- Buzzwords: *seamless*, *leverages*, *empowers*, *intelligent*, *cutting-edge*, *streamlines*, *unlocks*, *powerful*, *robust*.
- Passive voice for the action step.
- Mechanism where a result would do. *"Authenticates via OAuth"* is mechanism; *"Uses OAuth — no API token needed"* is a result.

### Keep valet.yaml and README.md in sync

If the agent's `README.md` ships a tagline, it must match `subheadline` word-for-word. Same for the `README.md` title and `display_name`. Update one, update the other in the same push.

## Manifest schema gotchas

These are the validator errors that bite most often.

- **`field slots not found in type manifest.Connector`** — per-secret setup copy goes in `slot_descriptions:`, a flat map of `SECRET_NAME: "<where to get it>"`. It's not a nested `slots:` block.
- **`field env not found in type manifest.Manifest`** (also `vars`, `config`, `settings`) — the manifest has no env/vars/config/settings block. The only top-level keys are `name`, `display_name`, `description`, `category`, `author`, `story`, `example`, `connectors`, `channels`, `concierge`. Runtime values belong in `SOUL.md`'s Configuration section.
- **`every is required for heartbeat channels`** / **`schedule or cron is required for cron channels`** — inline channels pair the schedule field to the type. `type: heartbeat` requires `every:`; `type: cron` requires `cron:` or `schedule:`. Switch both fields together.
- **`body must be 140 characters or fewer`** (and the other length caps) — see the table above. Trim to the sweet spot, don't squeak under the cap.
- **catalog reference won't resolve** — every `catalog:` value, including a step's `catalog:`, must name a real catalog entry or a connector/channel declared on this manifest.
- **`concierge: status must be one of drafting, ready`** — only those two values are valid. Don't add ad-hoc statuses.

## Channel files

`channels/<name>.md` tells the agent how to handle an incoming message on that channel. Instructions TO the agent, written as direct imperatives.

### Webhook payload location (required for webhook channels)

Webhook-driven channel files **must** start with this verbatim block:

```
The JSON webhook payload is appended directly after these instructions
in the user message. Parse it inline — do not fetch, list, or search
for the payload elsewhere. Do NOT use tools to read the payload.
```

Without it, agents waste turns hunting for the payload with tool calls.

### Structure

1. **Payload location** — the instruction above (webhook only).
2. **What happened** — describe the event.
3. **What to extract** — which payload fields identify the work (IDs, refs).
4. **Scope boundary** — all actions are scoped to those identifiers; do not act on unrelated content.
5. **Steps** — step-by-step processing instructions.

For heartbeat / cron channels (no webhook payload), skip the payload-location instruction and describe the cursor logic instead (see the `monitor` and `stateful` seeds).

### Reinforce scope in SOUL.md

For webhook-driven agents, add to `SOUL.md`:

```markdown
## Webhook Scope Rule

When you receive a webhook, your scope of work is defined by the
identifiers in the payload. Use any tools to fully understand and
act on that specific content, but do not act on unrelated content.
```
