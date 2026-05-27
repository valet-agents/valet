# Valet Concierge Agent

This repo is the source for **@Valet**, the [Valet](https://valet.dev) concierge agent. It is an ordinary Valet agent — seeded from this repo as `catalog:valet` — auto-installed in every organization. Users see it in the dashboard as `@Valet` and talk to it to build and modify other agents.

@Valet is an interview-driven coding agent with a shell and the `valet` CLI. It runs inside the org's runtime and edits the user's draft agent in code.storage, turning a conversation into a deployable manifest one push at a time.

## How it works

Each user turn arrives with a structured YAML-frontmatter envelope that names the user's `intent`, the target agent, and the draft branch. `SOUL.md` is the router: it parses the envelope, dispatches to one of three scenarios, and lets the elicitation skills drive the rest of the session.

| `intent` + `seed_kind` | Scenario |
| ---------------------- | -------- |
| `create_agent` + `blank` | [`scenarios/create-from-prompt/`](scenarios/create-from-prompt/) — user described what they want; interview from stage 1 |
| `create_agent` + `catalog` or `github` | [`scenarios/create-from-manifest/`](scenarios/create-from-manifest/) — user picked a template; interview from stage 5 (personalize) |
| `edit_agent` (any seed) | [`scenarios/edit-existing/`](scenarios/edit-existing/) — classify the request, enter interview at the right stage |

Scenarios are thin — they pick the opening and hand off. The work lives in the skills:

| Skill | Role |
| ----- | ---- |
| [`skills/interview/`](skills/interview/) | The five-stage elicitation flow (Goals → Guardrails → Workflow → Tools → Personalize), archetype recognition, question pacing, finding the next `<EDIT — …>` marker |
| [`skills/iterate/`](skills/iterate/) | The per-turn work loop — checkout, edit, validate, push, publish — and the recovery rules |
| [`skills/tool-discovery/`](skills/tool-discovery/) | Bridge from goals to connectors and channels; favor already-installed org resources over catalog over custom |
| [`skills/authoring/`](skills/authoring/) | File conventions for the agents the concierge writes (`SOUL.md`, `valet.yaml`, `channels/*.md`) and the manifest-schema gotchas |

## The interview

Every agent design captures three load-bearing things:

- **Goals** — what the agent does, on what trigger, producing what artifact
- **Guardrails** — what it must always and never do (positive guardrails encode required outcomes)
- **Tools** — connectors and channels the agent uses

The concierge elicits these from the user one question per turn. Each answer becomes one push to the draft branch, so the user watches the customize pane evolve in real time. State lives in the draft files themselves — half-finished interviews carry `<EDIT — …>` markers that the concierge greps for at the top of every turn. There is no separate interview-state store.

## Archetypes and seed templates

The concierge recognises four agent archetypes from the user's first description and picks a matching seed template. The user never sees the archetype name — recognition is implicit.

| Archetype | Shape | Seed |
| --------- | ----- | ---- |
| `transform` | Single trigger → linear pipeline → bounded output (e.g. lead → qualify → Slack) | [`seeds/SOUL.transform.md`](seeds/SOUL.transform.md) |
| `investigation` | Single trigger → open-ended exploration → artifact (e.g. Sentry → debug → PR) | [`seeds/SOUL.investigation.md`](seeds/SOUL.investigation.md) |
| `stateful` | Multi-session per subject, cursor in external source of truth (e.g. onboarding) | [`seeds/SOUL.stateful.md`](seeds/SOUL.stateful.md) |
| `monitor` | Scheduled wake → check external state → maybe act (e.g. stale-PR nudge) | [`seeds/SOUL.monitor.md`](seeds/SOUL.monitor.md) |

The seed templates use a `## Configuration — edit before deploy` section at the top of `SOUL.md` that holds the runtime values the agent reads each fire. The rest of the file references those values by name. This mirrors the convention several catalog agents already use today.

## Ready-to-publish signal

When the interview is complete (no `<EDIT — …>` markers remain and `valet agents drafts validate` is clean), the concierge writes `concierge.status: ready` into `valet.yaml`. The dashboard reads that field to enable the Configure & Deploy CTA. While the interview is in progress the value is `concierge.status: drafting`. The invariant: any remaining `<EDIT` marker anywhere in the draft forces `drafting`.

The schema for the `concierge:` block lives in [`valet-manifest`](https://github.com/valetdotdev/ark/tree/main/valet-manifest).

## Project structure

```
SOUL.md                          # router + identity + envelope contract + operating principles + invariants
valet.yaml                       # the concierge's own manifest
README.md                        # this file

seeds/                           # archetype-shaped SOUL.md templates + valet.yaml template
  SOUL.transform.md
  SOUL.investigation.md
  SOUL.stateful.md
  SOUL.monitor.md
  valet.yaml.template

skills/                          # reusable behaviors loaded by every scenario
  interview/SKILL.md
  iterate/SKILL.md
  tool-discovery/SKILL.md
  authoring/SKILL.md

scenarios/                       # thin per-situation openers
  create-from-prompt/SCENARIO.md
  create-from-manifest/SCENARIO.md
  edit-existing/SCENARIO.md
```

The concierge declares no channels of its own — it talks to the Valet platform via the `valet` CLI from its shell, authenticated by a per-org JWT baked into the runtime container at boot.

## How updates roll out

A merged change to `main` in this repo is picked up by `valet-ops`:

- `valet-ops concierge deploy --org <name>` — deploy to one org (for pre-release testing).
- `valet-ops concierge roll` — deploy to every org.

Each org has its own code.storage copy of this repo and its own release. The rollout tooling opens a draft on each org's concierge, commits the updated files, and publishes — the same draft machinery a user would trigger through @Valet.

## References

- [Valet Concierge Agent proposal](https://github.com/valetdotdev/ark/blob/main/docs/proposals/valet-agent.md) — design doc this repo implements.
- [valet.md](https://valet.md) — the developer-facing skill for Claude Code. @Valet is the non-developer counterpart.
- [github.com/valet-agents](https://github.com/valet-agents) — catalog agents referenced by `catalog:<name>` in `valet.yaml` files.

## License

Apache 2.0 — see [LICENSE](LICENSE).
