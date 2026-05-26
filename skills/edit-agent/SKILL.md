---
name: edit-agent
description: Use when the first-message envelope has `intent=edit_agent`. The user clicked Edit on an existing agent and wants to either tweak observed behavior or give the agent a new capability. Read the agent's files to plan the change, then iterate via the same edit→validate→push loop as create-agent.
---

# Edit-agent flow

You are answering a user who just clicked Edit on an existing
agent. The customize page is on screen with the agent's current
manifest in the right pane — connectors, channels, schedule,
prompt summary — derived from `valet.yaml`. The agent already
exists, is deployed, and may be running real traffic. The user
wants to change something about it.

Editing is the user revisiting the goals and guardrails they
expressed during onboarding. Two flavors cover almost everything:

1. **Tweak behavior** — the user describes a problem they're
   observing ("it keeps replying in markdown when I asked for
   plain text", "it forgets the budget cap I set last week") and
   expects you to figure out which file controls that and edit
   it. Behavior usually lives in `SOUL.md`.
2. **Add (or swap, or remove) a capability** — framed as "learn
   a new skill", "also let it do X", or "stop using Slack". This
   usually touches `valet.yaml` (catalog references) and may add
   or remove a channel file.

The user thinks in outcomes ("be less verbose", "watch the
#deals channel too"), not in files. Translate the outcome into
the smallest file change that produces it.

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

The user built this agent. They already know what it does. Do
not open the chat with "Your agent does X" or list its
connectors. Answer the question they actually asked. The
customize pane shows the agent's shape; you don't need to
repeat it.

## How to talk while you work

Status lines and the `Reply` tool work as described in the
always-on rules in `SOUL.md` — status lines are terse progress
indicators; the final user-facing response always goes through
`Reply`. In `Reply`, use plain language, no platform jargon the
first time it appears, concrete proposals over open
brainstorming, and one clarifying question at a time when a
question is needed at all.

## Turn 1 — branch on hint, then read

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

For hints that need file work, check out the draft first per
the **Reading and editing draft files** procedure in `SOUL.md`,
then read only the one file the hint names.

### No hint: read what the user's request implies

Without a hint, the user's first message describes either a
behavior to fix or a capability to change. Pick what to read
based on what they said:

- **Behavior tweak** ("it keeps doing X", "stop doing Y",
  "remember to Z", "be more concise") — usually `SOUL.md`.
  Sometimes a channel file when the behavior is channel-specific
  ("on Slack it should…").
- **Capability change** ("add", "also", "use", "swap out",
  "stop using") — `valet.yaml` for the catalog reference, plus
  the channel file you'll add or the existing one you'll remove.
  Consult `valet connectors catalog` / `valet channels catalog`
  to confirm the catalog entry exists before referencing it.
- **Schedule change** ("run it daily", "every Monday", "stop
  running on weekends") — the channel file that owns the
  schedule (`channels/heartbeat.md`, `channels/cron.md`, or the
  inline `type: heartbeat` / `type: cron` block in `valet.yaml`).
- **Marketing copy edit** ("the description should say…",
  "rename it to…", "change the hero") — `valet.yaml` only.

Do **not** preemptively read every file. The agent has been
running; the user knows what they want. Read what the request
implies, propose a change, and ask one clarifying question only
when the request is genuinely ambiguous about which file or
which value.

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

For an edit-mode draft the working tree is a copy of the live
agent's `main` branch — every file already exists. Use `Edit`
for changes, never `Write`. `Write` over an existing file
produces a whole-file diff in the dashboard's draft view.

## Subsequent turns — iterate

Ask → propose → edit → validate → push. Small, reviewable steps
the user can follow in the dashboard's draft view. The user is
likely to react to each change ("that's not what I meant", "yes
but also…"); keep each turn focused on one concern.

Within a single turn:

1. Check out the draft if you haven't this turn —
   `cd "$(valet agents drafts checkout <draft_id>)"` — then read
   whatever you need to plan the change.
2. `Edit` every file the turn touches, into the working tree.
   Reach for `Write` only when you're creating a genuinely new
   file (e.g. adding a new channel file the agent didn't have
   before).
3. Run `valet agents drafts validate <draft_id>` and fix any
   reported errors (per the validate-before-push rule in
   `SOUL.md`).
4. Run `valet agents drafts push <draft_id> -m "<message>"`
   exactly once.

The `Edit`-not-`Write` rule, the commit-message format, and the
one-push-per-turn batching are all in the always-on rules in
`SOUL.md`. Follow them here without restating.

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
- Need the envelope context back after a long turn or a
  container recycle? Re-read the first-message envelope — it's
  in the session history. `target_agent_id`, `target_agent_name`,
  and `draft_id` all live there; there is no separate lookup
  command.

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
"apply", "deploy"), run `valet agents drafts publish <draft_id>`.
The command prints JSON:

```json
{
  "status": "deployable" | "needs_configuration",
  "pending_install": [{ "kind": "...", "catalog_name": "..." }],
  "pending_attach":  [{ "kind": "...", "catalog_name": "..." }]
}
```

Then say so to the user:

- `"deployable"` — "I've published your changes. The dashboard
  will roll out the new version."
- `"needs_configuration"` — list the pending items by name in
  plain language and point the user back to the dashboard,
  where the configure wizard will walk them through OAuth /
  secret setup before the rollout finishes.

You do not run `DeployAgent`. You do not collect secrets.
Publish is your exit point; the dashboard takes it from there.
A successful publish on an edit-mode draft produces a new
release of the existing agent — the previous release stays in
history.

## Discarding

If the user says they want to abandon their changes ("never
mind", "revert", "throw this away"), confirm first, then run
`valet agents drafts discard <draft_id>`. The live agent is
unchanged; only the draft branch is dropped. Don't discard
silently — the user may have invested several turns in the
conversation, and the confirmation prevents an accidental
loss.

## Authoring reference

When you write or edit any of the target agent's files —
`SOUL.md`, `valet.yaml`, or `channels/<name>.md` — the
conventions and the manifest-schema gotchas live in
`skills/authoring/SKILL.md`. Read it before producing file
content. Always prefer surgical `Edit` calls over re-`Write`ing,
and validate before every push.
