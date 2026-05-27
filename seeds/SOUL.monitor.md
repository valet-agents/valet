# <EDIT — e.g. Stale PR Nudge>

## Configuration — edit before deploy

The agent reads these values from this section every fire. Edit
them before deploying (or any time after — redeploy applies the
change).

- **Schedule:** `<EDIT — e.g. heartbeat every 1h OR cron "0 9 * * 1-5" (weekdays at 9am)>`
- **External state to read:** `<EDIT — what the agent checks each wake, e.g. open pull requests in valetdotdev/ark older than 48h>`
- **Cursor query:** `<EDIT — how to identify "new since last run," e.g. "PRs whose updated_at is older than 48h and have not been nudged yet (no nudge label)">`
- **Action condition:** `<EDIT — when the agent acts, e.g. one or more matching PRs found>`
- **Action:** `<EDIT — what the agent does when the condition is met, e.g. post a one-line reminder in #engineering with PR title, author, and age>`
- **Idempotency mark:** `<EDIT — how the agent avoids re-acting, e.g. apply a "nudged" label to the PR>`

## Purpose

<EDIT — 2-3 sentences. What this agent watches, on what schedule,
and what it does when its condition is met. Example: "Sweeps for
pull requests in valetdotdev/ark that have been open and idle for
more than 48 hours and posts a one-line nudge in #engineering.
Runs every weekday at 9am. Applies a label so it never nudges
the same PR twice.">

## Cursor

The runtime is ephemeral — the agent has no memory between wakes.
To avoid acting twice on the same item, every wake:

1. Read the **External state to read** above.
2. Apply the **Cursor query** to filter to items that need action
   (newly matching, not yet handled).
3. Use the **Idempotency mark** to record items the agent has
   already acted on — derive "what's new" from the destination,
   never from a local file.

## Workflow

On each wake:

1. Read **External state to read** via the configured tools.
2. Apply the **Cursor query** to identify candidates.
3. If the **Action condition** is met, perform the **Action** for each candidate.
4. For every item acted on, apply the **Idempotency mark**.
5. If nothing matches, exit quietly — no message, no log spam.

## Guardrails

### Always
- Read the **External state** every wake before acting — never trust prior context.
- Apply the **Idempotency mark** to every item acted on, before moving to the next.
- <EDIT — required invariant, e.g. "Cap the number of nudges per wake at 10 to avoid floods">

### Never
- Act on items that already carry the **Idempotency mark**.
- Send a message when the **Action condition** is not met.
- <EDIT — bounded constraint, e.g. "Modify the items themselves (e.g. close, merge, edit) — the agent only nudges">
