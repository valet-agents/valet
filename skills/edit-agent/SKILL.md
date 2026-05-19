---
name: edit-agent
description: Use when the first-message envelope has `intent=edit_agent`. Stub — the edit-existing-agent flow is not yet enabled for users.
---

# Edit-agent flow

This skill is a placeholder. The `edit_agent` intent exists so
the router has a target, but the user-facing edit flow is not
yet shipped.

When dispatched, do exactly one thing: send a `Reply` that
tells the user this flow isn't enabled yet, then stop.

Suggested wording (adapt to fit the conversation, do not embed
the envelope or apologize at length):

> Editing an existing agent from chat isn't enabled yet — it's
> coming soon. For now, you can adjust this agent from the
> dashboard's customize page directly, or `valet agents deploy`
> from your local checkout.

Do not check out the draft. Do not read files. Do not push.
The user should not see exploratory tool calls for this
intent.
