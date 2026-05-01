# Valet Concierge Agent

This repo is the source for **@Valet**, the [Valet](https://valet.dev) concierge agent. It is an ordinary Valet agent — seeded from this repo as `catalog:valet` — that is auto-installed in every organization. Users see it in the dashboard as `@Valet` and talk to it to build other agents.

@Valet is a coding agent with a shell and the `valet` CLI. Its tooling is the same tooling a developer gets when they install the [valet.md](https://valet.md) skill in Claude Code. It just runs in the organization's runtime instead of on a laptop.

## Scope

**v1: creating new agents.** A user opens the dashboard's "new agent" flow, picks a seed (blank, first-party template, or custom GitHub URL), and is handed off to @Valet in a chat. @Valet checks out an ephemeral draft branch of the new agent's source, reads the seed, converses with the user to iterate, and publishes when the user is ready. The dashboard's configure-flow wizard then handles OAuth / secret setup and deploy.

Not in v1:

- **Editing existing agents** in conversation. Deferred to v2.
- **Collecting secrets** in chat. Handled by the dashboard wizard after publish.
- **Running the final deploy.** @Valet's exit point is `valet agents drafts publish <draft_id>`; `DeployAgent` is run by the dashboard wizard after any pending install/attach work.
- **Editing itself.** The agent named `valet` is reserved and cannot be modified from a user session.

## Project structure

```
SOUL.md                        # identity, personality, workflow, guardrails
valet.yaml                     # agent manifest (no connectors, no channels)
skills/
  valet-cli/SKILL.md           # the `valet` CLI surface used by the concierge
  agent-authoring/SKILL.md     # how to compose SOUL.md, valet.yaml, skills, channels
```

The concierge declares no connectors and no channels. It talks to the Valet platform via the `valet` CLI from its shell, authenticated by a per-org JWT baked into the runtime container at boot.

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
