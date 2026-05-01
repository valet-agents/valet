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
valet agents drafts checkout <draft_id>
```

- `<draft_id>` comes from the first message's JSON payload.

This prints a freshly-minted, short-lived code.storage clone URL on
stdout (metadata goes to stderr). Pipe the URL into `git clone`:

```
url=$(valet agents drafts checkout <draft_id>)
git clone "$url" ./<target_agent_name>
cd ./<target_agent_name>
```

The URL is scoped to the ephemeral branch namespace and capped at
session length. If it expires mid-session, re-run `checkout` to mint
a new one and update the remote: `git remote set-url origin "$url"`.

### Pushing changes to the draft branch

After editing files in the checkout, ship them to the draft branch
with:

```
valet agents drafts push <draft_id>
```

The CLI walks the working directory and calls the server's
`PushDraftFiles` RPC. The server commits the changed files on the
draft branch. The dashboard's draft view refreshes so the user can
see what you wrote.

Plain `git push` against the cloned URL does **not** work — Pierre's
ephemeral endpoint can't enumerate ancestors across namespaces, so
the local pack builder fails. Always use `drafts push`.

### Showing what's changed

For uncommitted edits in your working tree:

```
git status
```

For the server-truth state of the draft branch (what the dashboard
sees):

```
valet agents drafts info <draft_id>
```

### Publishing

```
valet agents drafts publish <draft_id>
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
valet agents drafts discard <draft_id>
```

Deletes the draft branch and closes the draft. Use only when the
user has explicitly asked to abandon the draft. Confirm first.

### Listing and inspecting drafts

You rarely need these; the first message tells you your draft_id
directly. Useful if the user references a draft by name or if you
need to sanity-check state.

```
valet agents drafts [--agent <name>]   # bare group lists drafts
valet agents drafts info <draft_id>    # detail for one draft
```

## Catalog queries

Use these before committing to a `catalog:` reference in a user's
`valet.yaml`. If a catalog entry doesn't exist, tell the user rather
than writing a broken reference.

```
valet connectors catalog              # list catalog connectors
valet connectors catalog get <name>   # read one connector's details
valet channels catalog                # list catalog channels
valet channels catalog get <name>     # read one channel's details
```

`catalog get` prints the catalog entry's transport, required secret
slots, and slot documentation. Use this to decide what secret slots
to expose in the user's manifest (and what description text to give
them — the dashboard's wizard uses those descriptions when it asks
the user for values).

## Commit log

```
valet agents log [<name>] [--limit N]
```

Shows the agent's commit history on `main` (most recent first).
Useful if the user asks "what changed" or "when did I last deploy."

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
