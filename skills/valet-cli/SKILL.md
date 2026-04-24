---
name: valet-cli
description: >-
  Use the `valet` CLI from the runtime shell to check out drafts,
  inspect agent source, query the catalog, and publish drafts. Covers
  every CLI command the concierge uses during a create-agent session.
---

# Valet CLI

The `valet` CLI is your primary interface to the Valet platform. It
is pre-installed on the runtime container's PATH and pre-authenticated
via a baked-in per-org token. You do not log in, you do not hand out
tokens, and you do not run anything as a different identity.

## Orientation

- `valet --help` — top-level command list.
- `valet <cmd> --help` — per-command help.
- `valet version` — CLI version, useful when something unexpected
  happens and you want to rule out a version mismatch.

## Draft lifecycle

The CLI's draft commands are the machinery of every create-agent
session. A draft is an ephemeral branch on the target agent's
code.storage repo that you edit, then publish (merge to `main`) or
discard (delete).

### Checking out a draft

```
valet agents draft checkout <draft_id> [<path>]
```

- `<draft_id>` comes from the first message's JSON payload.
- `<path>` is optional; defaults to `./<target_agent_name>`.

This clones the draft branch into the given path. It also installs
a `prepare-commit-msg` git hook in the checkout that appends
`Valet-Source: concierge` to every commit message — you don't have
to do this yourself.

`cd` into the checkout directory before doing anything else.

### Committing and pushing

Inside the checkout, this is plain git. No Valet wrappers.

```
git add -A
git commit -m "<what you changed, in one line>"
git push
```

The `prepare-commit-msg` hook installed by `draft checkout` handles
the attribution trailer.

### Showing what's changed

```
git diff origin/main...HEAD       # diff vs the base
git log --oneline origin/main..   # commits on this draft
git status                        # working tree state
```

### Publishing

```
valet agents draft publish [<draft_id>]
```

Merges the draft into `main` and deletes the draft branch. Returns
JSON:

```json
{
  "status": "deployable" | "needs_configuration",
  "pending_install": [{ "kind": "connector|channel",
                        "catalog_name": "..." }],
  "pending_attach":  [{ "kind": "connector|channel",
                        "catalog_name": "..." }]
}
```

- `pending_install` — catalog services the user's org does not
  have yet. The dashboard wizard will set them up (OAuth, secrets).
- `pending_attach` — services the org has but aren't yet attached
  to this agent. The wizard handles attachment too.
- `status: "deployable"` — both lists are empty; the user can
  deploy immediately.

Publishing is your exit from active editing. You do not run
`DeployAgent`; the dashboard's configure-flow wizard does, after any
pending install/attach work is complete.

### Discarding

```
valet agents draft discard [<draft_id>]
```

Deletes the draft branch and closes the draft. Use only when the
user has explicitly asked to abandon the draft. Confirm first.

### Listing and inspecting drafts

You rarely need these; the first message tells you your draft_id
directly. Useful if the user references a draft by name or if you
need to sanity-check state.

```
valet agents drafts list [--agent <name>]
valet agents drafts show <draft_id>
```

## Catalog queries

Use these before committing to a `catalog:` reference in a user's
`valet.yaml`. If a catalog entry doesn't exist, tell the user rather
than writing a broken reference.

```
valet connectors catalog            # list catalog connectors
valet connectors describe <name>    # read one connector's details
valet channels catalog              # list catalog channels
valet channels describe <name>      # read one channel's details
```

`describe` prints the catalog entry's description, required secrets,
and any slot documentation. Use this to decide what secret slots to
expose in the user's manifest (and what description text to give
them — the dashboard's wizard uses those descriptions when it asks
the user for values).

## Commit log with attribution

```
valet agents log [<name>] [--limit N]
```

Shows the agent's commit history with the `Valet-Source` trailer
parsed out of each commit message. Sources are `concierge` (you),
`cli` (a developer on their laptop), or `system` (Valet's own ops
tooling). Useful if the user asks "what changed" or "when did I
last deploy."

## Important don'ts

- **Never** run `valet auth login` — authentication is baked in.
- **Never** run `valet agents create`, `valet agents deploy`, or
  `valet agents destroy`. Agent creation happens before the session
  reaches you; deploy happens after you publish, via the dashboard;
  destroy is not something you do on the user's behalf.
- **Never** pass `--agent <valet>` or otherwise target your own
  agent. Your job is the target agent identified in the first
  message's payload.
- **Never** pass `--attach-connector` or `--attach-channel` flags.
  Attachment is the dashboard wizard's responsibility after
  publish, not yours.

## Target-agent identification

Every command that can be ambiguous about which agent it's operating
on should target the user's agent, not yours. In practice, once you
are `cd`'d into the draft checkout, git commands operate on the
right repo automatically, and the draft commands take a `draft_id`
that already identifies the target agent. But if you ever construct
a command that takes `--agent <name>`, use `target_agent_name` from
the first message — **not** `$VALET_AGENT_ID`.
