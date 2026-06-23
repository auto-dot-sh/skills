# Incident response agent

A first responder for production alerts: any system that can POST JSON (PagerDuty webhooks, Datadog monitors, Sentry alert rules, a `curl` in a runbook) sends an event to a custom webhook endpoint; the agent investigates against the codebase and recent changes, posts a structured triage to Slack, and stays in the thread for follow-up questions.

```
.auto/
  fragments/environments/agent-runtime.yaml
  agents/incident-response.yaml
```

## How it works

- **Custom webhook trigger**: `event: webhook.incident.opened` on `endpoint: incident-webhook` with `auth.kind: bearer_token`. Before the PR merges, have the user enter the shared secret from their own terminal, for example `read -rsp "INCIDENT_WEBHOOK_SECRET: " INCIDENT_WEBHOOK_SECRET; printf %s "$INCIDENT_WEBHOOK_SECRET" | auto secrets set incident-webhook-secret --stdin; unset INCIDENT_WEBHOOK_SECRET`. The GitHub Sync receipt contains the **ingest URL**; wire the alerting system to POST there with `Authorization: Bearer <secret>`.
- **Payload is yours**: the JSON the alert source posts is the event payload, and the trigger `message` template references its top-level keys directly (`{{title}}`, `{{severity}}`, … — no `payload.` prefix). The example assumes `{ "title": ..., "severity": ..., "service": ..., "description": ..., "link": ... }` — adapt the template to the real alert shape.
- **Investigation surface**: a read-only mount of the repo (to correlate the alert with recent commits) plus an optional Datadog MCP tool for logs and metrics.
- **Conversation**: the agent posts a triage thread in `#incidents`, then calls `auto.chat.subscribe` so responder questions in the thread route back to the same session (`attributedSessions` delivery with the mandatory `$.auto.authored: false` filter).

## Customize

- Replace `acme/widgets`, `github-acme`, `slack`, `#incidents`, and the inline Datadog tool's `auth.connection` name (`datadog-acme` is a placeholder for the MCP OAuth connection name).
- Match the prompt's payload fields to the actual alert JSON.
- Swap or drop the inline Datadog tool for the team's observability stack (any remote MCP server works; see `docs/tools-and-connections.md`).
- Severity routing (page a human for sev1, agent-only for sev3) is one `where:` filter away.
- Fresh @mentions of the bot outside a triage thread get a short hello that explains the responder's job. Real investigations still start from the incident webhook or from a user who provides alert details clearly.

## Smoke test

After the PR merges and GitHub Sync applies the resources, copy the webhook
ingest URL from the trigger receipt and POST a smoke payload to it:

```sh
curl -X POST <ingest-url> \
  -H "Authorization: Bearer <secret>" \
  -H "Content-Type: application/json" \
  -d '{"title":"Smoke test alert","severity":"sev3","service":"widgets-api","description":"Test from onboarding","link":"https://example.com"}'
```

Confirm a run spawns and a triage thread appears in `#incidents`, then reply in the thread and confirm the agent answers.
