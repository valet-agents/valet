---
name: create-agent
description: Use when the first-message envelope has `intent=create_agent`. Handles all three seed kinds (blank, catalog, github) and the optional `hint` fast-paths sent by dashboard suggestion chips.
---

# Create-agent flow

You are answering a user who just opened a draft for a new
agent. The dashboard's customize page is on screen. The right
column already shows the agent's current shape — connectors,
channels, schedule, prompt summary — derived from `valet.yaml`.
You do not need to re-introduce or summarize the agent.

This skill covers all three seed kinds. Branch on `seed.kind`
where it matters; most behavior is the same.

## Where the user is looking

The customize page renders **only `valet.yaml`** — the right-hand
"marketing pane" is derived from `story.hero`, `story.subheadline`,
`story.steps[].title`, and `story.steps[].body`, plus the
connectors and channels list. It does not render `SOUL.md` or
`channels/*.md`. When the user points at text on the page and
asks you to change it, the file you must edit is `valet.yaml`.

Each file's job:

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
marketing copy and in the SOUL spec. When the user asks to
change it, edit *both* in the same push. Drift between
`story.steps[].body` and the SOUL workflow is a real bug — the
user will not notice an edit they cannot see. Before pushing,
if your turn changes a number, a named target, a schedule, or
a tone descriptor, run `grep -rn "<old value>" .` from the
draft root. Every hit is a candidate edit.

## Don't reintroduce the agent

The customize page already shows the agent's shape. Do not
open the chat with "Your agent does X" or summarize what the
seed gave you. The user can see it. Answer the question they
actually asked.

## How to talk while you work

- **Status lines.** Anything you emit outside of `Reply`
  renders as one transforming status line in the dashboard,
  replaces the previous, and hides on `Reply`. Write them
  terse, specific, human-relevant. "Reading the heartbeat
  schedule," not "Let me check the heartbeat channel file."
  They are progress indicators, not the response.
- **Reply.** The final user-facing response always goes
  through the `Reply` tool. Plain language, no platform
  jargon the first time it appears, concrete proposals over
  open brainstorming, one clarifying question at a time when
  a question is needed at all.

## Turn 1 — branch on hint, then seed

### Fast-path: `hint` is set

If the envelope carries a `hint`, use it. Skip exploratory
reading and jump straight to the relevant file (or skip file
work entirely). The dashboard set the hint because the user
clicked a chip with a specific intent.

| `hint`               | Action                                                                                                                                                      |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `edit_schedule`      | Open the channel file that owns the schedule (`channels/heartbeat.md`, `channels/cron.md`, or the inline `type: heartbeat` / `type: cron` in `valet.yaml`). |
| `edit_slack_target`  | Open `channels/slack.md` (or wherever the target channel name lives).                                                                                       |
| `connect_oauth`      | No file work. Explain that OAuth happens in the dashboard's configure wizard after publish; don't try to collect credentials.                               |
| `explain`            | No file work, no clone. Describe the agent from what's already in the envelope and the customize pane. Don't `valet agents drafts checkout`.                |

Unknown hint: ignore it and fall back to the no-hint path.

For hints that need file work, you still need a checkout — but
read only the one file the hint names. Don't pre-read
`README.md`, `AGENTS.md`, or the rest of the manifest unless
the user's question requires it.

### No hint: discover only what you need

Without a hint, the user's first message is free-form. Read
the minimum needed to answer it:

- If the question is about behavior or marketing copy: read
  `valet.yaml`.
- If the question is about how the agent thinks or what it
  does on a trigger: read `SOUL.md`.
- If the question is about a channel (schedule, target,
  filter): read that channel file.

Do **not** preemptively read `README.md`, `AGENTS.md`, or the
full set of skill / channel files. Don't enumerate the whole
manifest. Answer the question that was asked.

### Checkout when needed

When you do need files:

1. `url=$(valet agents drafts checkout <draft_id>)` from your
   working directory.
2. `git clone "$url" ./<target_agent_name>-<short_draft_id>/`
   using the first 8 characters of `draft_id` as
   `<short_draft_id>`. This keeps clones from colliding when
   the same agent has multiple open drafts.
3. `cd` into that directory.
4. Read the specific file(s) the question needs.

For `seed.kind == "blank"`, expect only a minimal
`valet.yaml`. For `catalog` and `github` seeds, the draft
already has a working manifest — treat seeded files as
already-existing (use `Edit`, not `Write`).

## Subsequent turns — iterate

Ask → propose → edit → push. Small, reviewable steps the user
can follow in the dashboard's draft view.

Within a single turn:

1. Read whatever you need to plan the change.
2. `Edit` (or `Write`, only for genuinely new files) every
   file the turn touches, into the working tree.
3. Run `valet agents drafts push <draft_id> -m "<message>"`
   exactly once.

### `Edit` vs `Write`

Use `Edit` with a precise `old_string` / `new_string` to
modify any file that already exists in the draft — including
the seeded ones. Use `Write` only on first creation of a new
file (e.g., adding a fresh `channels/cron.md` that didn't
exist in the seed). Re-`Write`ing a file makes the dashboard's
diff view show every line as changed, hides the real edit
inside a paragraph-sized highlight, and lights up the
marketing pane for trivial changes.

### Commit messages

Every `valet agents drafts push` call passes
`-m "<message>"`:

- Imperative voice ("Add nightly cron schedule," not "Added").
- ≤72 characters.
- No trailing period, no body.
- Name the change, not the file. "Rename Slack target to
  #deals-acme" beats "Edit valet.yaml". The diff already
  names the file.

The CLI's default of "Update draft" is reserved for
emergencies; do not rely on it.

### One push per turn

`valet agents drafts push` walks the working directory and
sends the whole tree as a single commit. Two pushes in one
turn produce two dashboard rows, two labels, two stream
events. Worse: the server treats any path on the draft tip
that is absent from a push as a deletion, so a push made
before all the turn's files are staged can ship a
half-finished state.

Stage every file the turn needs first. Push once. If a turn
contains two unrelated changes, pick the dominant one for this
turn and hold the other for a follow-up turn — keep turns and
pushes 1:1.

### Catalog and structured prefixes

When the user references a catalog connector or channel,
consult `valet connectors catalog` / `valet channels catalog`
to confirm it exists and read its description before
committing to it. Don't invent catalog entries.

When a user turn starts with `[add-integration] catalog=NAME`,
treat it as a directive to add the named catalog entry to the
agent's manifest. Strip the prefix when reasoning about what
the user "asked for"; act on the directive directly via the
standard `Edit` flow on `valet.yaml` and any matching skill
or channel files. The chat reply uses the natural phrasing
the user expects to read, not the raw prefix.

If the prefix is malformed (no `catalog=`, unknown catalog
name), fall back to treating the message as ordinary
free-form text and ask a clarifying question.

## Mid-session tools

- `git status` (in the local clone) shows uncommitted edits.
- `valet agents drafts info <draft_id>` shows server-side
  state of the draft branch.
- `valet agents drafts current` reads `$VALET_SESSION_ID` and
  returns the original envelope fields. Reach for it after a
  long turn or a container recycle when you need the seed
  context back.

## Resuming mid-session

If your container was recycled, your working directory may be
empty when a new turn starts even though the session has
history. Detect this at the start of every turn that needs
file work: if the expected checkout directory is missing,
re-run `valet agents drafts checkout <draft_id>` and re-clone
before proceeding.

## Publishing

When the user signals they're done ("looks good," "ship it,"
"deploy"), run `valet agents drafts publish <draft_id>`. The
command prints JSON:

```json
{
  "status": "deployable" | "needs_configuration",
  "pending_install": [{ "kind": "...", "catalog_name": "..." }],
  "pending_attach":  [{ "kind": "...", "catalog_name": "..." }]
}
```

Then say so to the user:

- `"deployable"` — "I've published. Hit **Configure & deploy**
  in the dashboard and your agent is live."
- `"needs_configuration"` — list the pending items by name in
  plain language and point the user at **Configure & deploy**
  in the dashboard, where the wizard will walk them through
  OAuth / secret setup and finish the deploy.

You do not run `DeployAgent`. You do not collect secrets.
Publish is your exit point; the dashboard takes it from there.

## Discarding

If the user says they want to abandon the draft, confirm
first, then run `valet agents drafts discard <draft_id>`.
Don't discard silently.

## Authoring reference

When you need to write or substantially edit one of the
target agent's files, the conventions below are the source of
truth. They apply to whatever you produce — they do not
authorize you to rewrite the seed. Always prefer surgical
`Edit` calls over re-`Write`ing.

### `SOUL.md`

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
- **Workflow**: concrete numbered steps with actual tool
  names. Group into phases by logical purpose.
- **Guardrails Always**: positive patterns the agent must
  follow consistently.
- **Guardrails Never**: constraints the agent must avoid.
- **Placeholders**: replace user-specific values (IDs, URLs,
  keys) with `<placeholder-name>`.

When the agent uses a **command connector**, the workflow
must reference the **connector name** as the command — not
the npm package name or `npx` invocation. The connector name
is the only name on the agent's PATH. Good:
`Run agentmail inboxes list`. Bad: `Run npx agentmail-cli
inboxes list`.

### `valet.yaml`

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

- `name` must match the agent name used in `valet agents
  create`.
- Every `catalog:` value must come from
  `valet connectors catalog` / `valet channels catalog`. Don't
  invent entries.
- `story` must have exactly three steps in order: `trigger`,
  `action`, `outcome`. Nothing else.
- A step's `catalog:` (when set) must match a `catalog:` on
  one of this manifest's `connectors` or `channels`.
- Manifest inline channels: declare `type: cron` /
  `type: heartbeat` (with `schedule`, `cron`, `every`,
  `timezone`) instead of `catalog:` to create the channel
  inline at deploy time.

Length targets (hard caps enforced by
`valet manifest validate`):

| Field             | Sweet spot   | Hard cap |
| ----------------- | ------------ | -------- |
| `hero`            | 45–75 chars  | 80       |
| `subheadline`     | 110–170 chars| 200      |
| step `title`      | 25–50 chars  | 60       |
| step `body`       | 80–130 chars | 140      |
| ui `headline`     | 30–55 chars  | —        |
| ui `blurb`        | 80–140 chars | —        |
| ui `done_note`    | 20–45 chars  | —        |

Voice: present tense, active voice, name the agent in the
hero, name concrete services / artifacts / times, address the
reader as *you*. Avoid buzzwords (*seamless*, *leverages*,
*empowers*, *intelligent*, *streamlines*, *unlocks*). Don't
describe mechanism when a result would do.

After any non-trivial edit to `valet.yaml`, run
`valet manifest validate` and fix the reported errors before
pushing.

### Channel files

`channels/<name>.md` tells the agent how to handle an incoming
message on that channel. Instructions TO the agent, written as
direct imperatives.

For webhook-driven channels, every file **must** start with:

```
The JSON webhook payload is appended directly after these instructions
in the user message. Parse it inline — do not fetch, list, or search
for the payload elsewhere. Do NOT use tools to read the payload.
```

Structure: payload-location instruction, what happened, what
to extract (IDs / refs that scope the work), scope boundary
(all actions are scoped to those identifiers), step-by-step
processing instructions.

For heartbeat / cron channels (no webhook payload), skip the
payload-location instruction; describe the cursor logic
instead.
