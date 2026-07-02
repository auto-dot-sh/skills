# Daily ship digest

A scheduled reporter: every morning it reads the last 24 hours of merged PRs and direct-to-main commits, checks whether they deployed cleanly, and posts a digest summary to Slack.

```
.auto/
  fragments/environments/agent-runtime.yaml
  agents/ship-digest.yaml
```

## Create from the managed template

This archetype is published as the managed template `@auto/daily-digest`. The default way to install it is a thin agent file in the user's repo — the template carries the prompts, triggers, tools, runtime, and identity with its avatar baked in, so no assets are copied:

```yaml
# .auto/agents/ship-digest.yaml
imports:
  - "@auto/daily-digest@latest/agents/ship-digest.yaml"
variables:
  repoFullName: acme/widgets
  slackConnection: slack
  slackChannel: "#dev"
```

Set the variables to the user's repo, connection names, and channel. Override any field by declaring it in the importing file (local fields win; triggers merge by their authoring `name:`), and use `remove: { triggers: [...], tools: [...] }` to drop inherited entries. The directory below is the source this template was derived from.

## How it works

- **Heartbeat trigger**: `kind: heartbeat` with `cron: "0 8 * * *"` and an explicit `timezone`. The prompt anchors the reporting window on `{{heartbeat.scheduledAt}}` so the window is exact even if the run starts late.
- **Read-only by construction**: the mount grants `contents: read` only, the GitHub tools are the read/search set, and the instructions forbid running tests or builds — a reporter can't break anything.
- **Deeper history**: heartbeat analysis over a time window needs more git history than the default shallow clone, hence `depth: 300` on the mount.
- **Output discipline**: exactly one Slack message — a one-sentence summary with the digest threaded beneath it, so the channel stays scannable.

## Customize

- Replace `acme/widgets`, `github-acme`, `slack`, `#dev`; pick the cron and timezone.
- The digest sections (Shipped / Follow-ups / Quality watch / In flight) are a starting point — tune them to what the team wants to see.
- To publish the full digest somewhere richer than a thread (Notion, a wiki), add a remote MCP tool and have the Slack message carry the link instead. The production version of this workflow does exactly that.

## Smoke test

Cron triggers are awkward to wait for, so smoke test by spawning the agent with `mcp__auto__auto_sessions_spawn` and a message asking for an immediate digest. A manual run has no heartbeat payload, so expect the agent to treat "now" as the window end — fine for a smoke test. Confirm the digest message lands in the channel.
