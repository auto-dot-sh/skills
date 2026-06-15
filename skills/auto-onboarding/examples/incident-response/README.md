# Incident response agent

A first responder for production alerts: any system that can POST JSON (PagerDuty webhooks, Datadog monitors, Sentry alert rules, a `curl` in a runbook) sends an event to a custom webhook endpoint; the agent investigates against the codebase and recent changes, posts a structured triage to Slack, and stays in the thread for follow-up questions.

```
.auto/
  environments/agent-runtime.yaml
  profiles/responder.yaml
  agents/incident-response.yaml
```

## How it works

- **Custom webhook trigger**: `event: webhook.incident.opened` on `endpoint: incident-webhook` with `auth.kind: bearer_token`. The shared secret is created with `auto secrets set incident-webhook-secret` *before* applying. After `auto apply`, the trigger receipt in the apply output contains the **ingest URL**; wire the alerting system to POST there with `Authorization: Bearer <secret>`.
- **Payload is yours**: whatever JSON the alert source posts arrives as `{{payload.*}}`. The example assumes `{ "title": ..., "severity": ..., "service": ..., "description": ..., "link": ... }` — adapt the template to the real alert shape.
- **Investigation surface**: a read-only mount of the repo (to correlate the alert with recent commits) plus an optional Datadog MCP tool for logs and metrics.
- **Conversation**: the agent posts a triage thread in `#incidents`, then calls `auto.chat.subscribe` so responder questions in the thread route back to the same run (`attributedRuns` delivery with the mandatory `$.auto.authored: false` filter).

## Customize

- Replace `acme/widgets`, `github-acme`, `slack`, `#incidents`, and the inline Datadog tool's `auth.connection` name (`datadog-acme` is a placeholder for the MCP OAuth connection name).
- Match the prompt's payload fields to the actual alert JSON.
- Swap or drop the inline Datadog tool for the team's observability stack (any remote MCP server works; see `docs/tools-and-connections.md`).
- Severity routing (page a human for sev1, agent-only for sev3) is one `where:` filter away.
- Fresh @mentions of the bot outside a triage thread are dropped by design — the chat triggers require `$.auto.attributions: { exists: true }` and only `deliver`, so the responder is webhook-driven. Add a mention trigger with `routing: { kind: spawn }` if you want it conversational on its own.

## Smoke test

```sh
auto secrets set incident-webhook-secret
auto apply        # note the ingest URL in the trigger receipt
curl -X POST <ingest-url> \
  -H "Authorization: Bearer <secret>" \
  -H "Content-Type: application/json" \
  -d '{"title":"Smoke test alert","severity":"sev3","service":"widgets-api","description":"Test from onboarding","link":"https://example.com"}'
```

Confirm a run spawns and a triage thread appears in `#incidents`, then reply in the thread and confirm the agent answers.
