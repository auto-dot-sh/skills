# Outbound lead engine (agentic BDR)

The non-engineering archetype: an agentic BDR that turns raw signups and prospect lists into researched dossiers and ready-to-send outreach. A new lead arrives by webhook (signup form, CRM automation, enrichment pipeline, or a `curl`); the agent researches the person and company, scores the fit, drafts personalized outreach, and posts the package to `#sales` for a human to approve — then revises in-thread based on feedback. The human always pulls the trigger on sending.

```
.auto/
  fragments/environments/agent-runtime.yaml
  agents/lead-researcher.yaml
```

## How it works

- **Webhook intake**: `event: webhook.lead.created` on `endpoint: lead-webhook` with bearer-token auth. Wire any lead source that can POST JSON at the ingest URL from the apply receipt. The example payload shape is `{ "name", "email", "company", "source", "notes" }` — adapt the prompt to the real fields.
- **No repo mount**: this workflow has nothing to do with code. The agent works from the lead payload, public research from its sandbox, and whatever tools you grant — a good reminder that auto agents are general agents, not just coding agents.
- **Human-in-the-loop by design**: the agent's hard limit is that the agent never contacts a prospect. Its output is a dossier + draft package in `#sales`; approval, edits, and sending stay with the team. Thread feedback routes back to the same run (`attributedRuns` + `auto.chat.subscribe`), so "make it shorter and mention the SOC2 page" produces a revision in seconds.
- **Honest scoring**: the agent prompt demands evidence-based fit scoring and explicitly allows "bad fit, skip" as a recommendation — an outbound engine that can say no is the one sales teams learn to trust.

## Customize

- Replace `slack` and `#sales`; map the webhook payload to the real lead source.
- Fresh @mentions of the bot outside a lead thread are dropped by design — the chat triggers require `$.auto.attributions: { exists: true }` and only `deliver`, so the engine is webhook-driven, not mention-driven. Add a mention trigger with `routing: { kind: spawn }` if you want the bot conversational on its own.
- Add a CRM as a remote MCP tool (`kind: mcp_remote` pointing at the CRM's MCP server) and extend the agent prompt to log dossiers and dispositions there — see `docs/tools-and-connections.md`.
- Encode the team's actual ICP (ideal customer profile) and messaging guidelines in the agent prompt; that's what makes drafts sound like the company instead of a template.
- For batch prospecting, add a heartbeat trigger that processes a queue (a spreadsheet, a CRM view) on a daily cadence.

## Smoke test

```sh
auto secrets set lead-webhook-secret
auto apply        # note the ingest URL in the trigger receipt
curl -X POST <ingest-url> \
  -H "Authorization: Bearer <secret>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Jane Doe","email":"jane@example.com","company":"Example Corp","source":"demo-request","notes":"asked about enterprise pricing"}'
```

Confirm a run spawns and a dossier + draft thread appears in `#sales`, then reply with feedback and confirm a revision arrives.
