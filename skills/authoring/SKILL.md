---
name: authoring
description: Reference for writing a target agent's files — SOUL.md, valet.yaml, and channel files. Loaded by create-agent and edit-agent whenever a turn writes or edits draft files. This is a reference, not a flow: it owns no session and never runs CLI lifecycle commands.
---

# Authoring reference

The conventions below are the source of truth for any file you
write into a draft. They tell you *how to write* — they do not
authorize you to rewrite the seed. For `catalog` and `github`
seeds the draft already has working files; treat them as
existing and prefer surgical `Edit` calls over re-`Write`ing.
The `Edit`-not-`Write` rule and the validate-before-push rule
in `SOUL.md` apply to everything here.

## What each file is for

- **`valet.yaml`** — user-visible marketing copy (`story.*`)
  plus the manifest of connectors and channels. Anything the
  user can see on the customize page lives here.
- **`SOUL.md`** — the target agent's own prompt. Identity,
  personality, workflow, guardrails. The user does not see it
  unless they open the source.
- **`channels/<name>.md`** — per-channel preprocessor prompts
  (Quick Filter for Slack, cursor logic for heartbeat / cron).
  Also unseen on the customize page.

A frequent failure mode: the same value (a character cap, a
threshold, a channel name, a schedule) appears in both the
marketing copy and the SOUL spec. When the user asks to change
it, edit *both* in the same push. Drift between
`story.steps[].body` and the SOUL workflow is a real bug — the
user will not notice an edit they cannot see. Before pushing,
if your turn changes a number, a named target, a schedule, or a
tone descriptor, run `grep -rn "<old value>" .` from the draft
root. Every hit is a candidate edit.

## SOUL.md

The target agent's identity and behavior. Required.

```markdown
# <Agent Title>

## Purpose

<2-3 sentences: what this agent does and why. Name the specific
tools, inputs, and outputs.>

## Personality

<3-4 traits matching the agent's domain. Skip for simple utility agents.>

- **<Trait>**: <Description>

## Workflow

### Phase 1: <Phase Name>

1. <Concrete step referencing specific tool names>
2. <Next step>

### Phase 2: <Phase Name>

1. <Steps>

## Guardrails

### Always
- <Positive constraint>

### Never
- <Negative constraint>
```

Synthesis rules:

- **Purpose**: specific what + why. Name inputs, outputs, and
  tools. Good: "Monitors YouTube channel X for new episodes,
  downloads transcripts, and posts digests to #channel on
  Slack." Bad: "Processes data."
- **Workflow**: concrete numbered steps with actual tool names.
  Group into phases by logical purpose.
- **Guardrails Always**: positive patterns the agent must
  follow consistently.
- **Guardrails Never**: constraints the agent must avoid.
- **Placeholders**: replace user-specific values (IDs, URLs,
  keys) with `<placeholder-name>`.

### Runtime values go in SOUL.md, not the manifest

When a user wants the agent to use a specific value — a repo to
watch, a default channel, a threshold, a timezone — write that
value into `SOUL.md`. The manifest has **no** env/settings
block (see "Manifest schema gotchas"), so there is nowhere in
`valet.yaml` to put it. A request phrased as "set the REPOS env
var" still resolves to a SOUL.md edit naming the repo
concretely, never a manifest `env:` entry.

### Command connector references

When the agent uses a **command connector**, the workflow must
reference the **connector name** as the command — not the npm
package name or `npx` invocation. The connector name is the only
name on the agent's PATH. Good: `Run agentmail inboxes list`.
Bad: `Run npx agentmail-cli inboxes list` (bypasses secret
injection) or `Run agentmail-cli inboxes list` (command not
found).

### Common mistakes

- Empty or vague Purpose — always name specific inputs, tools,
  and outputs.
- Missing Workflow — Purpose without steps leaves the agent
  guessing.
- Hardcoded values that should be `<placeholder>`s.
- No scope boundary for webhook agents (see "Channel files").
- Wrong command name for a command connector — it must match the
  CLI command (e.g. `agentmail`, not `agentmail-cli`).

## valet.yaml

The manifest. Drives the dashboard's customize page and
configure-flow wizard.

```yaml
name: <agent-name>
display_name: <Human-Readable Name>
description: >-
  <What the agent does — shown in the dashboard during setup>
category: <category>
story:
  hero: "<one short line, names the agent>"
  subheadline: "<one concrete sentence — services and reward>"
  steps:
    - role: trigger
      title: "<25–50 chars>"
      body: "<80–130 chars>"
      catalog: <optional catalog ref>
    - role: action
      title: "..."
      body: "..."
    - role: outcome
      title: "..."
      body: "..."
connectors:
  - catalog: <catalog-entry-name>
    description: >-
      <Agent-specific context for this connector>
    slot_descriptions:
      <SECRET_NAME>: "<where the user gets this credential>"
    ui:
      headline: "<verb + service imperative>"
      blurb: "<one paragraph: this agent's use of the service>"
      done_note: "<one line: what the user sees after deploy>"
channels:
  - catalog: <catalog-entry-name>
    description: >-
      <Agent-specific context for this channel>
    events:
      - <event_type>
```

Rules:

- `name` must match the agent name used to create the agent.
- Every `catalog:` value must come from
  `valet connectors catalog` / `valet channels catalog`. Don't
  invent entries; if the user wants something not in the
  catalog, say so and offer what exists.
- `story` must have exactly three steps in order: `trigger`,
  `action`, `outcome`. Nothing else.
- A step's `catalog:` (when set) must match a `catalog:` on one
  of this manifest's `connectors` or `channels`. Leave it empty
  to render the agent monogram (useful for the middle "the agent
  thinks" step).
- Omit `connectors` / `channels` arrays entirely when the agent
  has none. Omit optional fields (`description`, `events`,
  `slot_descriptions`, `ui`) when catalog defaults suffice.

### Length targets

The hard caps are enforced by `valet manifest validate`. The
sweet spots are what renders well in the wizard — target those,
not the caps. Only push toward a cap when the extra characters
carry real information (a channel name, a specific time, a named
artifact). Never pad.

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

- Present tense, active voice. *"Posts the briefing"*, not
  *"Will post the briefing"* or *"The briefing is posted"*.
- Name the agent at least once in the hero — the user is meeting
  it for the first time.
- Name concrete things: services (*Slack*, *GitHub*), artifacts
  (*#ai-news*, *pull request*), times (*8am*, *every Friday*).
  Never *"messaging platforms"* or *"on a schedule"*.
- Address the reader as *you*. Never *"the user"*.

**Never:**

- Buzzwords: *seamless*, *leverages*, *empowers*, *intelligent*,
  *cutting-edge*, *streamlines*, *unlocks*, *powerful*, *robust*.
- Passive voice for the action step.
- Mechanism where a result would do. *"Authenticates via OAuth"*
  is mechanism; *"Uses OAuth — no API token needed"* is a
  result. *"Parses the payload"* is mechanism; *"Reads the PR"*
  is a result.
- Copy that could belong to a different agent. If you can swap
  the agent's name out without the sentence breaking, it's
  generic — add the hook.

### Per-field tone

- **Hero** is imperative or present-indicative and names the
  agent: *"Let's get AskADev answering questions for your
  team."* — not *"AskADev is an AI agent that answers
  questions."*
- **Subheadline** is one concrete sentence naming the services
  and the reward. No lists, no semicolons.
- **Trigger title** starts with the user or the channel event:
  *"Your team asks…"*, *"A PR is opened."*, *"8am hits."*
- **Action title** uses active voice and names the agent:
  *"AskADev reads your code."*
- **Outcome title** names the concrete artifact the user sees:
  *"Replies in-thread, linking to the file."*

### Good vs. bad

| Field | ❌ Bad | ✅ Good |
|-------|-------|--------|
| hero | *"AskADev is an AI-powered Slack assistant that answers code questions."* | *"A Slack bot that reads your code before it answers."* |
| subheadline | *"Uses GitHub MCP and Slack MCP to provide intelligent responses."* | *"Ask about a GitHub repo in Slack. AskADev researches the code and commit history, then replies in-thread."* |
| trigger title | *"Webhook event received"* | *"A PR is opened."* |
| action title | *"Diff analysis"* | *"Code Reviewer reads the diff."* |
| outcome title | *"Review submitted"* | *"Inline comments — or an approve."* |
| ui blurb | *"Connect your GitHub account to give the agent access."* | *"AskADev reads your code when it answers — like a new hire would."* |
| ui done_note | *"Successfully connected"* | *"Listening in #engineering"* |

### Per-service `ui:` block

Each `connectors[]` / `channels[]` entry can carry a `ui:` block
that overrides generic catalog copy:

- `headline` — verb + service imperative: *"Let AskADev hear
  your team in Slack."*
- `blurb` — one paragraph on *this specific agent's* use of the
  service. Not a generic Slack/GitHub explainer — that's what
  the catalog `description` is for.
- `done_note` — the one line shown on the deploy screen once the
  service connects: *"Listening in #engineering"*,
  *"Connected to valetdotdev/ark"*.

Do **not** put permissions, restrictions, safety chips, "why",
verb, or minutes in `ui:` — those are catalog-owned fields the
Valet team fills in.

### Keep valet.yaml and README.md in sync

The agent's `README.md` tagline and the `valet.yaml`
`subheadline` are the same copy on two surfaces — keep them
word-for-word identical. Same for the `README.md` title and
`display_name`. Update one, update the other in the same push.

## Manifest schema gotchas

These are the validator errors that bite most often. Each maps
to the exact message `valet agents drafts validate` /
`valet manifest validate` prints. When you see the message, this
is the fix.

- **`field slots not found in type manifest.Connector`** —
  per-secret setup copy goes in `slot_descriptions:`, a **flat
  map** of `SECRET_NAME: "<where to get it>"`. It is *not* a
  nested `slots:` block with `description:` children. If a seed
  uses `slots:`, rename and flatten it to `slot_descriptions:` —
  never delete the credential text, or the configure wizard
  loses the user's setup instructions.
- **`field env not found in type manifest.Manifest`** (also
  `vars`, `config`, `settings`) — the manifest has **no**
  env/vars/config/settings block. The only top-level keys are
  `name`, `display_name`, `description`, `category`, `author`,
  `story`, `example`, `connectors`, `channels`. Runtime values
  the agent needs belong in `SOUL.md` (see "Runtime values go in
  SOUL.md").
- **`every is required for heartbeat channels`** /
  **`schedule or cron is required for cron channels`** — inline
  channels pair the schedule field to the type. `type: heartbeat`
  requires `every:` (e.g. `every: 24h`); `type: cron` requires
  `cron:` or `schedule:`. They are not interchangeable — switch
  both together in the same edit.
- **`body must be 140 characters or fewer`** (and the other
  length caps) — see the length-targets table above. Trim to the
  sweet spot, don't just squeak under the cap.
- **catalog reference won't resolve** — every `catalog:` value,
  including a step's `catalog:`, must name a real catalog entry
  or a connector/channel declared in this same manifest.

Per the validate-before-push rule in `SOUL.md`, run
`valet agents drafts validate <draft-id>` after editing
`valet.yaml` and fix every reported error before pushing.

## Channel files

`channels/<name>.md` tells the agent how to handle an incoming
message on that channel. Instructions TO the agent, written as
direct imperatives.

For webhook-driven channels, the file **must** start with:

```
The JSON webhook payload is appended directly after these instructions
in the user message. Parse it inline — do not fetch, list, or search
for the payload elsewhere. Do NOT use tools to read the payload.
```

Without this, agents waste turns hunting for the payload with
tool calls.

Structure:

1. **Payload location** — the instruction above.
2. **What happened** — describe the event.
3. **What to extract** — which payload fields identify the work
   (IDs, refs).
4. **Scope boundary** — all actions are scoped to those
   identifiers; do not act on unrelated content.
5. **Steps** — step-by-step processing instructions.

For heartbeat / cron channels (no webhook payload), skip the
payload-location instruction and describe the cursor logic
instead.

For webhook-driven agents, reinforce the boundary in `SOUL.md`:

```markdown
## Webhook Scope Rule

When you receive a webhook, your scope of work is defined by the
identifiers in the payload. Use any tools to fully understand and
act on that specific content, but do not act on unrelated content.
```
