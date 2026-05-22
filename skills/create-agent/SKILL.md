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

What each file is for, and the marketing-copy-vs-SOUL drift
trap, are covered in `skills/authoring/SKILL.md` — read it
before writing or editing any draft file.

## Don't reintroduce the agent

The customize page already shows the agent's shape. Do not
open the chat with "Your agent does X" or summarize what the
seed gave you. The user can see it. Answer the question they
actually asked.

## How to talk while you work

Status lines and the `Reply` tool work as described in the
always-on rules in `SOUL.md` — status lines are terse progress
indicators; the final user-facing response always goes through
`Reply`. In `Reply`, use plain language, no platform jargon the
first time it appears, concrete proposals over open
brainstorming, and one clarifying question at a time when a
question is needed at all.

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
| `connect_oauth`      | No file work, no checkout. Explain that OAuth happens in the dashboard's configure wizard after publish; don't try to collect credentials.                  |
| `explain`            | No file work, no checkout. Describe the agent from what's already in the envelope and the customize pane. Don't `valet agents drafts checkout`.             |

Unknown hint: ignore it and fall back to the no-hint path.

For hints that need file work (`edit_schedule`, `edit_slack_target`),
check out the draft first per the **Reading and editing draft
files** procedure in `SOUL.md` — then read only the one file the
hint names. Don't pre-read `README.md`, `AGENTS.md`, or the rest
of the manifest unless the user's question requires it. The
`connect_oauth` and `explain` fast paths answer without files, so
they skip the checkout entirely.

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

When you do need files, check out the draft per the **Reading
and editing draft files** procedure in `SOUL.md`:

```sh
cd "$(valet agents drafts checkout <draft_id>)"
```

That lands you in a directory with the draft's working tree
already checked out. Don't hand-roll `git clone` / `git checkout`
/ `git fetch` — the command does it. Then read the specific
file(s) the question needs.

For `seed.kind == "blank"`, expect only a minimal
`valet.yaml`. For `catalog` and `github` seeds, the draft
already has a working manifest — treat seeded files as
already-existing (use `Edit`, not `Write`).

## Subsequent turns — iterate

Ask → propose → edit → validate → push. Small, reviewable steps
the user can follow in the dashboard's draft view.

Within a single turn:

1. Check out the draft if you haven't this turn —
   `cd "$(valet agents drafts checkout <draft_id>)"` — then read
   whatever you need to plan the change.
2. `Edit` (or `Write`, only for genuinely new files) every
   file the turn touches, into the working tree.
3. Run `valet agents drafts validate <draft_id>` and fix any
   reported errors (per the validate-before-push rule in
   `SOUL.md`).
4. Run `valet agents drafts push <draft_id> -m "<message>"`
   exactly once.

The `Edit`-not-`Write` rule, the commit-message format, and the
one-push-per-turn batching are all in the always-on rules in
`SOUL.md`. Follow them here without restating: use `Edit` for
files that already exist (including seeded ones), `Write` only
for genuinely new files, stage every file the turn needs before
the single `valet agents drafts push -m "<message>"`.

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

- `git status` (from inside the checkout) shows uncommitted edits.
- `valet agents drafts info <draft_id>` shows server-side
  state of the draft branch.
- Need the seed context back after a long turn or a container
  recycle? Re-read the first-message envelope — it's in the
  session history. The `draft_id`, seed, and prompt all live
  there; there is no separate lookup command.

## Resuming mid-session

If your container was recycled, your working directory may be
empty when a new turn starts even though the session has
history. At the start of every turn that needs file work, just
re-run the checkout —
`cd "$(valet agents drafts checkout <draft_id>)"` is idempotent
and re-creates the directory if it's gone, so there's no need to
test for it first.

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

When you write or edit any of the target agent's files —
`SOUL.md`, `valet.yaml`, or `channels/<name>.md` — the
conventions and the manifest-schema gotchas live in
`skills/authoring/SKILL.md`. Read it before producing file
content. It is the source of truth for what you write; it does
not authorize rewriting the seed. Always prefer surgical `Edit`
calls over re-`Write`ing, and validate before every push.
