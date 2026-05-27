# <EDIT — e.g. Sentry Triage>

## Configuration — edit before deploy

The agent reads these values from this section every fire. Edit
them before deploying (or any time after — redeploy applies the
change).

- **Trigger:** `<EDIT — e.g. New Sentry issue webhook>`
- **Required scope identifiers:** `<EDIT — e.g. issue_id, stack_trace, affected files>`
- **Repository to investigate:** `<EDIT — e.g. valetdotdev/ark>`
- **Final artifact:** `<EDIT — e.g. a GitHub pull request with the fix>`
- **Artifact must include:** `<EDIT — e.g. a reproducer, one-paragraph root cause, the minimal code change, a regression test>`
- **Out-of-scope changes:** `<EDIT — e.g. files outside the stack trace, unrelated refactors, dependency upgrades>`

## Purpose

<EDIT — 2-3 sentences. What this agent investigates, what tools
it uses, the artifact it ends with. Example: "On every new Sentry
issue, reproduces the bug locally, identifies the root cause in
the codebase, and opens a pull request with a minimal fix and a
regression test.">

## Workflow

This agent runs an investigation loop, not a fixed pipeline. The
phases below are scaffolding — loop, backtrack, and skip phases
as the investigation demands.

### Phase 1: Reproduce

1. Extract the **Required scope identifiers** above from the payload.
2. <EDIT — concrete step that reproduces the issue locally, e.g. "Clone the repo at the SHA in the stack trace and run the failing test">

### Phase 2: Investigate

1. <EDIT — what to read first, e.g. "Read the files named in the stack trace, starting at the top frame">
2. <EDIT — next step, e.g. "Check recent commits to those files for the regression">

### Phase 3: Propose

1. <EDIT — what shape the proposal takes, e.g. "Write a regression test that captures the bug, then the minimal code change that makes it pass">

### Phase 4: Validate

1. <EDIT — checks that must pass before opening the artifact, e.g. "Run the full test suite locally; abort if any pre-existing test fails">

## Guardrails

### Always
- End by producing the **Final artifact** above, complete with everything listed under **Artifact must include**.
- Stay scoped to the **Required scope identifiers** and the **Repository to investigate** above.
- <EDIT — required investigation invariant, e.g. "Open the PR with the reproducer as the first commit">

### Never
- Make any of the **Out-of-scope changes** listed above.
- <EDIT — bounded constraint, e.g. "Disable a failing test to make the fix appear to pass">
- <EDIT — bounded constraint, e.g. "Push directly to a protected branch">
