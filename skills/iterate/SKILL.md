---
name: iterate
description: The work loop — checkout, edit, validate, push, publish. Every scenario uses this; it's the single source of truth for the command sequencing the concierge runs every turn.
---

# Iterate

## Per turn

When a turn writes or edits any draft files, run this sequence:

1. **Land in the checkout.** Run `cd "$(valet agents drafts checkout <draft_id>)"` if you haven't this turn. The command is idempotent — re-running refreshes the working tree and prints its path. The `cd "$(…)"` lands you inside that path with the draft's files (`SOUL.md`, `valet.yaml`, `channels/`) already checked out. Never run `git clone`, `git fetch`, or `git checkout` by hand.
2. **Read what the answer touches.** Don't preemptively read the whole tree — read the file(s) the question's answer maps to.
3. **`Edit`, don't `Write`, files that already exist.** `Write` is for the first creation of a genuinely new file. Re-`Write`ing an existing file makes the dashboard's diff view show every line as changed.
4. **Validate.** Run `valet agents drafts validate <draft_id>`. Fix every error before pushing — never push an unparseable manifest.
5. **Push exactly once.** `valet agents drafts push <draft_id> -m "<message>"`. The server commits the whole working tree as a single commit. Two pushes in one turn produce two dashboard rows and can briefly leave the draft inconsistent (the server treats any path absent from a push as a deletion).

One push per user answer is the interview's heartbeat — the user watches the customize pane evolve answer by answer.

## Commit messages

Single line, imperative voice, ≤72 characters, no trailing period, no body. The dashboard renders it as the change label.

Name the change, not the file:

- ✅ `Set goal: classify and route inbound support email`
- ✅ `Use existing GitHub connector and Slack channel`
- ✅ `Mark draft ready for configure`
- ❌ `Edit SOUL.md`
- ❌ `Update Purpose section`

Pick the dominant change when a push covers two related edits; don't list both.

## Recovery

- **Mangled `Edit`** (wrong `old_string`, broken indentation): restore with `git checkout -- <path>` from inside the checkout and redo the `Edit`. Never `Write` over an existing file to clean up a botched edit.
- **`checkout` fails**: retry once — first failures are sometimes transient. If it fails again, stop. `ReportError` with the verbatim error (tags like `["checkout", "git-fetch-128"]`) and `Reply` to the user with what happened. Container state can be corrupt in ways only the operator can reset.
- **`validate` reports `valet.yaml not found`**: this should only happen on a pre-concierge agent. Out of scope for the current flow — `ReportError` and tell the user this agent needs to be re-created via the concierge.
- **Three identical failures**: any command failing the same way three times means the turn is cooked. Stop. `ReportError` + `Reply` + end the turn. Don't iterate through variations of a failing approach.

## Per session

- **Publish.** When the interview is done (no `<EDIT` markers, `concierge.status: ready`, validate clean), the user advances via the dashboard's Configure & Deploy CTA. The concierge itself runs `valet agents drafts publish <draft_id>` only when the user explicitly asks to ship from chat ("publish", "ship it", "looks good").
- **Discard.** If the user wants to abandon the draft, confirm first, then `valet agents drafts discard <draft_id>`. Don't discard silently — a multi-turn interview is a real investment.

## Forbidden raw git

Inside the checkout, only three git verbs are allowed:

- `git status` — see uncommitted edits.
- `git show HEAD:<path>` — read the last server-side commit (handy when comparing your edit to the prior good state).
- `git checkout -- <path>` — restore a file you mangled with `Edit`.

Everything else (`clone`, `commit`, `push`, `fetch`, `init`, `add`, `remote`, `branch`, `reset`) is the `valet agents drafts` subcommands' job. Wrapping `git` with a shell script to spy on what `checkout` does counts as forbidden raw git.
