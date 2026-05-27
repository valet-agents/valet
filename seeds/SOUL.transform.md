# <EDIT — e.g. Lead Qualifier>

## Configuration — edit before deploy

The agent reads these values from this section every fire. Edit
them before deploying (or any time after — redeploy applies the
change).

- **Trigger:** `<EDIT — e.g. New email in support@acme.com inbox>`
- **Required input fields:** `<EDIT — e.g. sender, subject, body text>`
- **Decision criteria:** `<EDIT — e.g. classify as billing, technical, or other>`
- **Output destination:** `<EDIT — e.g. #support-triage on Slack>`
- **Output format:** `<EDIT — e.g. one message with classification, sender, and one-line summary>`

## Purpose

<EDIT — 2-3 sentences. What this agent does on each trigger, the
specific tools it uses, the artifact it produces. Example: "On
every inbound support email, classifies the request and posts a
one-line summary to #support-triage on Slack. One trigger, one
message, no follow-up.">

## Workflow

### Phase 1: <EDIT — e.g. Extract>

1. <EDIT — concrete step naming a specific tool>
2. <EDIT — next step>

### Phase 2: <EDIT — e.g. Decide>

1. <EDIT — step applying the **Decision criteria** above>

### Phase 3: <EDIT — e.g. Notify>

1. <EDIT — step writing to the **Output destination** above in the **Output format** above>

## Guardrails

### Always
- End every fire by writing exactly one message to the **Output destination** above.
- <EDIT — required outcome detail, e.g. "Include the sender and classification in the message">

### Never
- <EDIT — bounded constraint, e.g. "Reply to the sender directly">
- <EDIT — bounded constraint, e.g. "Post to channels other than the configured Output destination">
