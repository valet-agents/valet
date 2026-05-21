# Valet Concierge Agent

This repo is the source for **@Valet**, the [Valet](https://valet.dev) concierge agent. It is an ordinary Valet agent — seeded from this repo as `catalog:valet` — that is auto-installed in every organization. Users see it in the dashboard as `@Valet` and talk to it to build and modify other agents.

@Valet is a coding agent with a shell and the `valet` CLI. It runs inside the org's runtime and edits the user's draft agent in code.storage.

## How it works

Each user turn arrives with a structured envelope (a fenced ```json block following a short human preamble) that names the intent, the target agent, the draft branch, and an optional fast-path `hint` from the dashboard's suggestion chips. `SOUL.md` is a thin workflow router: it parses the envelope, branches on `intent`, and dispatches to a per-flow skill that owns the rest of the session.

| `intent`       | Skill                                        | Status                       |
| -------------- | -------------------------------------------- | ---------------------------- |
| `create_agent` | [`skills/create-agent/`](skills/create-agent/) | Live. Blank, catalog, and GitHub seeds. |
| `edit_agent`   | [`skills/edit-agent/`](skills/edit-agent/)     | Stub. Not yet shipped to users. |

The skill is where the workflow lives — checkout, edit, push, publish for `create_agent`; a single "coming soon" reply for `edit_agent`. `SOUL.md` carries only the routing logic and the always-on rules (commit discipline, one-push-per-turn batching, `Edit`-not-`Write` for existing files, no `git push`, no `valet auth login`, etc.) that every skill inherits.

## Project structure

```
SOUL.md                          # router + always-on rules
valet.yaml                       # agent manifest (declares the `valet` CLI connector)
skills/
  create-agent/SKILL.md          # create-agent flow (all seed kinds)
  edit-agent/SKILL.md            # edit-agent flow (stub — not yet enabled)
  authoring/SKILL.md             # shared file-authoring reference (SOUL.md, valet.yaml, channels)
```

The `authoring` skill is a reference, not a flow: it owns no
session and runs no CLI commands. Both `create-agent` and
`edit-agent` load it when a turn writes or edits a target
agent's files, so the SOUL.md / valet.yaml / channel-file
conventions and the manifest-schema gotchas live in exactly one
place.

The concierge declares no channels. It talks to the Valet platform via the `valet` CLI from its shell, authenticated by a per-org JWT baked into the runtime container at boot.

## How updates roll out

A merged change to `main` in this repo is picked up by `valet-ops`:

- `valet-ops concierge deploy --org <name>` — deploy to one org (for pre-release testing).
- `valet-ops concierge roll` — deploy to every org.

Each org has its own code.storage copy of this repo and its own release. The rollout tooling opens a draft on each org's concierge, commits the updated files, and publishes — the same draft machinery a user would trigger through @Valet.

## References

- [Valet Concierge Agent proposal](https://github.com/valetdotdev/ark/blob/main/docs/proposals/valet-agent.md) — the design doc this repo implements.
- [valet.md](https://valet.md) — the developer-facing skill for Claude Code. @Valet is the non-developer counterpart.
- [github.com/valet-agents](https://github.com/valet-agents) — catalog agents referenced by `catalog:<name>` in `valet.yaml` files.

## License

Apache 2.0 — see [LICENSE](LICENSE).
