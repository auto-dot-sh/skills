# Self-improvement agent

An introspector that periodically reviews real evidence from the project and
reports concrete improvements. It can inspect PR feedback, read-only operational
or product data sources, and the project's own Auto session history. Findings
can target either the user's application itself or the Auto agents, prompts,
triggers, and processes that support it.

```
.auto/
  fragments/environments/agent-runtime.yaml
  agents/introspector.yaml
```

## How it works

- **Heartbeat sweep**: every 2 hours (tune the cadence to run volume), `routing: spawn`.
- **Session history**: `auto.sessions.list` to triage recent sessions, `auto.sessions.conversation` / `auto.sessions.tools` / `auto.sessions.triggers` to deep-dive, `auto.sessions.search` to find patterns.
- **PR feedback**: use the GitHub tools to inspect recent PRs, especially review comments, expressed preferences, repeated friction, unresolved blockers, and CI failures.
- **Read-only data sources**: connect logs, metrics, traces, incidents, docs, support, or analytics MCP tools when the user's environment has them. Keep them read-only unless the user explicitly asks for an agent that changes external systems.
- **Stateful across sweeps without state**: each sweep starts by finding its _own previous report_ (`auto.sessions.list` for its own agent, read the last run's final message) and only reports what changed since. Closed loops get closures; recurring problems get escalated, not re-announced.
- **Bounded depth**: at most three deep-dives per sweep — one well-evidenced diagnosis beats many shallow ones.
- **Conservative cadence, high-signal reports**: run often enough to catch useful patterns without becoming noise. Lead with high-leverage improvements the agent has strong evidence for, especially changes that can be automated going forward.
- **Concrete proposals**: every finding names the evidence, the affected surface, and the recommended change. The change may be application code/tests/docs or Auto resources/prompts/triggers/process.
- **Self-scrutiny included**: its own past sessions are in scope.

## Customize

- Replace `slack`, `github-acme`, `acme/widgets`, and `#dev`; pick the sweep cadence.
- Keep the example identity unless the user wants a different persona:
  `identity.username: introspector` with `identity.avatar.asset:
  .auto/assets/introspector.png`.
- Tailor "what counts as actionable" to the user's goals. A code-review-heavy team may care most about review feedback, expressed preferences, and missing tests; an ops-heavy team may care more about incident patterns, logs, and flaky handoffs.
- Add only read-only MCP tools for external data sources during onboarding. If the user later wants the agent to create issues or PRs from findings, make that an explicit follow-up workflow.

## Smoke test

Spawn the introspector with `mcp__auto__auto_sessions_spawn` and the message "Do a manual sweep of the last 24 hours: inspect sessions, recent PR feedback, and any connected read-only data sources, then report the highest-leverage concrete improvements you have evidence for." Confirm it reads real evidence, picks sensible deep-dives, and posts a finding thread with concrete recommendations or explains why the highest-confidence opportunities should wait for more data.
