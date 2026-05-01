---
name: agent-authoring
description: >-
  Compose the files that make up a Valet agent — `SOUL.md`,
  `valet.yaml`, `skills/**`, `channels/**` — following the patterns
  used by first-party agents. Use when the user wants to create a new
  agent and you need to translate their intent into concrete source.
---

# Agent Authoring

An agent is a set of files in a git repo:

```
SOUL.md              # identity and behavior (required)
valet.yaml           # connectors, channels, metadata (required)
skills/
  <skill-name>/
    SKILL.md         # a named procedure the agent can invoke
channels/
  <binding-name>.md  # per-channel instructions (one per binding
                     # that needs specific guidance)
```

Nothing is auto-generated. You are writing these files directly into
the draft checkout and pushing the working directory to the draft
branch with `valet agents drafts push`. Be deliberate — every line in
these files becomes part of the agent's prompt.

## `SOUL.md` — identity and behavior

The agent's top-level prompt. Structure:

```
# <Agent Name>

## Purpose
One short paragraph: what the agent does, for whom, and when.
No marketing language.

## Personality
3–5 bulleted traits, each with a one-line explanation. These
shape tone and style, not behavior.

## Workflow
The step-by-step the agent follows. For agents with a single
trigger (e.g. a daily cron), list the phases plainly. For agents
with multiple triggers (e.g. interactive Slack + cron), split
by trigger and give each its own phased workflow.

## Guardrails
### Always
- <rule>
- <rule>

### Never
- <rule>
- <rule>
```

### Purpose — concrete, not abstract

Good: "Every morning, research AI news from the past 24 hours and
post a briefing with the top 5 stories to #ai-news."

Bad: "Keep the team informed about AI."

### Personality — traits with consequences

Pick traits that change what the agent will do, not decorative ones.
"Concise: distill complex developments into clear, scannable
summaries" changes output. "Friendly" on its own doesn't.

### Workflow — what, in what order, with what tools

Write the workflow as the agent will execute it. Reference specific
tool names when the agent will use them (`slack_post_message`,
`search`, `fetch`). If the agent has more than one trigger, give
each one its own workflow section — readers (and the model itself)
get confused when interactive and scheduled behaviors are blended.

### Guardrails — Always / Never

Name invariants explicitly. "Always include a source link for every
claim." "Never post more than once per cron trigger." "Always use
#ai-news for daily briefings; never fall back to a different
channel."

## `valet.yaml` — configuration

Minimal shape:

```yaml
name: <agent-name>
display_name: <Human-Readable Name>
description: >-
  One or two sentences describing what the agent does. Appears in
  the dashboard agent list.
category: <category>
```

`category` is a grouping label for the dashboard. Use the closest
fit from agents already in the catalog (research, development,
utilities, operations, …) rather than inventing a new one.

### Connectors

Connectors give the agent tools from a Model Context Protocol (MCP)
server. Declare them as catalog references:

```yaml
connectors:
  - catalog: slack
    description: >-
      Slack MCP server for posting the daily briefing to #ai-news
    slot_descriptions:
      SLACK_BOT_TOKEN: Slack bot token (xoxb-...) with chat:write
      SLACK_TEAM_ID: Slack workspace team ID (starts with T)
```

- `catalog:` must match an entry from `valet connectors catalog`.
  Run that command to confirm before committing.
- `description:` is shown in the dashboard during setup. Say
  concretely why *this agent* needs this connector.
- `slot_descriptions:` give per-secret help text for the wizard's
  setup UI. Describe what the secret is, where to get it, and any
  required scopes or prerequisites.

### Channels

Channels are how messages enter the agent. Three forms appear in
existing agents:

**Inline cron** (schedule):

```yaml
channels:
  - type: cron
    cron: "0 8 * * *"
    description: Triggers the daily briefing at 8am
```

**Inline webhook** (generic HTTPS intake):

```yaml
channels:
  - type: webhook
    description: Receives JSON payloads from an external system
```

**Catalog channel** (Slack, Telegram, Sentry-webhook, etc. — first-
party integrations with their own transport and auth):

```yaml
channels:
  - catalog: slack
    description: Interactive channel for questions in Slack
  - catalog: telegram
    description: Inbound Telegram messages
  - catalog: sentry-webhook
    description: Receives Sentry issue and event alert webhooks
    events:
      - issue
      - event_alert
    slot_descriptions:
      SENTRY_CLIENT_SECRET: Sentry internal integration client secret
```

For catalog channels that need secrets or event filters, run
`valet channels catalog get <name>` to see what the catalog entry
expects.

## `skills/` — named procedures

Skills are for agents whose behavior has distinct, self-contained
procedures. A small agent that just posts a message to Slack does
not need a skill — its entire behavior fits in `SOUL.md`. A
research agent with different workflows for daily briefings vs.
interactive questions might put each in a skill and orchestrate
them from `SOUL.md`.

Skill file shape:

```markdown
---
name: <skill-name>
description: >-
  When to use this skill, in one sentence.
---

# <Skill Name>

<The procedure, step by step. Reference specific tool names.>
```

Keep the decision to add a skill proportional to the agent's
complexity. Many perfectly good agents have no skills at all.

## `channels/` — per-binding instructions

When a channel binding needs its own instructions separate from the
agent's overall behavior, write them at
`channels/<binding-name>.md`. The binding name is the channel's
`as` (or the channel `name` if `as` is omitted), not the channel
`type`.

### Webhook payload convention

Webhooks inject the JSON payload inline in the user message, right
after the channel instructions. Channel prompts for webhooks almost
always need to state this, because models tend to reach for file-
reading tools to "find" the payload. Example:

```markdown
# <Channel Name>

IMPORTANT: The JSON webhook payload appears directly below these
instructions in this same message. It is NOT in a file. Do NOT
use Read, Bash, find, or any tool to locate it. Just look below
— the JSON is right here in the text you are reading.

## Steps
1. Parse the JSON payload from the user message.
2. Extract the `<field>` field.
3. <action>.
```

### Slack channel convention

Interactive Slack bindings (where the user talks to the agent in
Slack) benefit from a note about how to deliver replies — use the
`Reply` tool or the Slack messaging tools, and be judicious about
when to respond so the agent doesn't echo every channel message.

## Putting it together: a minimal agent

A concrete starting point the concierge can adapt for "post a daily
summary to Slack" prompts:

`SOUL.md`:

```markdown
# Daily Digest

## Purpose
Every morning at 8am, post yesterday's <topic> summary to
#<channel> in Slack.

## Personality
- Concise: 5 bullets, no preamble.
- Sourced: every bullet links to the underlying source.

## Workflow
1. Gather the items for yesterday using <tool>.
2. Format as a Slack `mrkdwn` message with 5 bullets.
3. Post with `slack_post_message` to #<channel>.

## Guardrails
### Always
- Post exactly once per trigger.
- Include a source link in every bullet.

### Never
- Post to channels other than #<channel>.
- Retry on failure — log and stop.
```

`valet.yaml`:

```yaml
name: <name>
display_name: <Display Name>
description: Daily <topic> summary to #<channel>
category: research
connectors:
  - catalog: slack
    description: Posts the daily digest to #<channel>
    slot_descriptions:
      SLACK_BOT_TOKEN: Slack bot token (xoxb-...) with chat:write
      SLACK_TEAM_ID: Slack workspace team ID
channels:
  - type: cron
    cron: "0 8 * * *"
    description: Triggers the daily digest at 8am
```

No skills, no channel prompts — the cron trigger carries no payload
the agent needs to parse, and the entire behavior fits in `SOUL.md`.

## Writing posture

- **Push in small steps.** One logical change per `valet agents
  drafts push` ("add valet.yaml scaffold", "write SOUL.md workflow
  section", "add Slack connector"). The user watches the draft
  evolve in the dashboard.
- **Narrate as you go.** When you push a change, tell the user in
  plain language what you just did and why.
- **Ask when the user's intent is genuinely ambiguous, not to
  collect a checklist.** "Which Slack channel should the briefing
  go to?" is worth asking. "What should we name the variable?" is
  not.
- **Keep the first version small.** Publish a working minimum, then
  iterate. It's much better to publish something deployable and let
  the user ask for more than to build a sprawling agent up-front
  that the user then edits down.

## Catalog awareness

Before writing a `catalog:` reference in `valet.yaml`, confirm the
entry exists:

```
valet connectors catalog
valet connectors catalog get <name>
valet channels catalog
valet channels catalog get <name>
```

If the user wants a connector or channel that isn't in the catalog,
say so. Do not write a `catalog:` reference to something you haven't
confirmed exists — publish will fail and the user will not know why.
