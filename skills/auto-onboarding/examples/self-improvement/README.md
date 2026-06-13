# Self-improvement loop

The workflow Beat 8 of the onboarding installs: an introspector that periodically sweeps the project's own runs for failures, anomalies, and drift, and reports actionable findings — with proposed fixes to sessions, prompts, and triggers — to the team's channel. The auto system watching the auto system.

```
.auto/
  environments/agent-runtime.yaml
  profiles/introspector.yaml
  tools/slack-chat.yaml
  sessions/introspector.yaml
```

## How it works

- **Heartbeat sweep**: every 2 hours (tune the cadence to run volume), `routing: spawn`.
- **The `auto` tool is the whole game**: `auto.runs.list` to triage recent runs, `auto.runs.conversation` / `auto.runs.tools` / `auto.runs.triggers` to deep-dive, `auto.runs.search` to find patterns. No mount needed — the system itself is the subject.
- **Stateful across sweeps without state**: each sweep starts by finding its *own previous report* (`auto.runs.list` on its own session, read the last run's final message) and only reports what changed since. Closed loops get closures; recurring problems get escalated, not re-announced.
- **Bounded depth**: at most three deep-dives per sweep — one well-evidenced diagnosis beats many shallow ones.
- **Quiet by default**: nothing actionable means no Slack message at all. When there are findings: one short top-level line, full detail threaded.
- **Self-scrutiny included**: its own past runs are in scope.

## Customize

- Replace `slack`, `#dev`; pick the sweep cadence.
- Tailor "what counts as actionable" to the user's workflows — for a code-review-only system, prompt-quality drift matters more than queue times.
- Since this is installed after CI/CD (Beat 8), deliver it as a PR and let the user merge — don't `auto apply` it directly.

## Smoke test

`auto run introspector --attach -m "Do a manual sweep of the last 24 hours and report what you find."` Confirm it lists runs, picks sensible deep-dives, and either posts a finding thread or correctly reports nothing actionable.
