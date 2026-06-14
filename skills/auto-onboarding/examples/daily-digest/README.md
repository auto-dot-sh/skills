# Daily ship digest

A scheduled reporter: every morning it reads the last 24 hours of merged PRs and direct-to-main commits, checks whether they deployed cleanly, and posts a digest summary to Slack.

```
.auto/
  environments/agent-runtime.yaml
  profiles/analyst.yaml
  sessions/ship-digest.yaml
```

## How it works

- **Heartbeat trigger**: `kind: heartbeat` with `cron: "0 8 * * *"` and an explicit `timezone`. The prompt anchors the reporting window on `{{payload.heartbeat.scheduledAt}}` so the window is exact even if the run starts late.
- **Read-only by construction**: the mount grants `contents: read` only, the GitHub tools are the read/search set, and the instructions forbid running tests or builds — a reporter can't break anything.
- **Deeper history**: heartbeat analysis over a time window needs more git history than the default shallow clone, hence `depth: 300` on the mount.
- **Output discipline**: exactly one Slack message — a one-sentence summary with the digest threaded beneath it, so the channel stays scannable.

## Customize

- Replace `acme/widgets`, `github-acme`, `slack`, `#dev`; pick the cron and timezone.
- The digest sections (Shipped / Follow-ups / Quality watch / In flight) are a starting point — tune them to what the team wants to see.
- To publish the full digest somewhere richer than a thread (Notion, a wiki), add a remote MCP tool and have the Slack message carry the link instead. The production version of this workflow does exactly that.

## Smoke test

Cron triggers are awkward to wait for, so smoke test by launching manually: `auto run ship-digest --attach`. A manual run has no heartbeat payload, so expect the agent to treat "now" as the window end — fine for a smoke test. Confirm the digest message lands in the channel.
