# Research / optimization loop

A multi-agent autoresearch loop: give the **research-coordinator** a measurable objective ("get p95 API latency under 200ms", "cut sandbox startup time", "raise the benchmark score") and a budget, and it runs the experimental method on a fleet — proposing hypotheses, dispatching parallel **experimenter** sessions to test them in isolated sandboxes, collating measurements into a running lab log, and iterating round after round until the objective is met or the budget is spent. The shape comes from the optimization loops we run against our own infrastructure (sandbox startup benchmarking).

```
.auto/
  fragments/environments/agent-runtime.yaml # reusable sandbox runtime
  agents/research-coordinator.yaml  # concurrency: 1; persona, standing orders, mention to start, heartbeat to advance
  agents/experimenter.yaml          # spawn-only standing orders
```

## Create from the managed template

This archetype is published as the managed template `@auto/research-loop`. The default way to install it is one thin agent file per agent in the user's repo — the template carries the prompts, triggers, tools, runtime, and identity with its avatar baked in, so no assets are copied:

```yaml
# .auto/agents/research-coordinator.yaml
imports:
  - "@auto/research-loop@latest/agents/research-coordinator.yaml"
variables:
  repoFullName: acme/widgets
  slackConnection: slack
```

```yaml
# .auto/agents/experimenter.yaml
imports:
  - "@auto/research-loop@latest/agents/experimenter.yaml"
variables:
  repoFullName: acme/widgets
  slackConnection: slack
```

Set the variables to the user's repo, connection names, and channel. Override any field by declaring it in the importing file (local fields win; triggers merge by their authoring `name:`), and use `remove: { triggers: [...], tools: [...] }` to drop inherited entries. The directory below is the source this template was derived from.

## How it works

- **Long-horizon singleton**: like the agent fleet, the coordinator is a `concurrency: 1` agent (deliver + `onUnmatched: spawn`) woken by mentions, subscribed replies, and a heartbeat. A campaign survives across many wake-ups — and even across coordinator restarts, because the lab log lives in the Slack thread, not in the run's memory.
- **Rounds, not sprawl**: each round the coordinator proposes 2-4 falsifiable hypotheses, spawns one experimenter per hypothesis (idempotency-keyed on campaign + round + hypothesis slug), and waits for results to arrive via `auto.sessions.message`.
- **Experiments are isolated and complete**: each experimenter gets its own sandbox and checkout, implements exactly one variant, runs the measurement protocol from the brief (same warmup, same iterations, baseline + variant), and reports numbers — explicitly including null and negative results. Experimenters never open PRs during the loop.
- **The lab log is the artifact**: after each round the coordinator posts a structured update in the campaign thread — round number, each hypothesis with its measured effect, the running best configuration, and the next round's plan. Anyone can read the thread and know exactly where the search stands.
- **Convergence and handoff**: when the objective is met, the budget is exhausted, or two consecutive rounds yield no improvement, the coordinator closes the campaign: a final summary with the winning variant and the evidence, then — only on explicit human approval in the thread — instructs the winning experimenter's variant to be turned into a real PR.

## Customize

- Replace `acme/widgets`, `github-acme`, `slack`.
- The measurement protocol in the experimenter prompt is the part to make rigorous for the user's domain: what command produces the metric, how many iterations, what counts as noise. A loop is only as good as its measurements.
- Heartbeat cadence bounds how fast rounds advance; 10 minutes suits experiments that run in minutes, hourly suits long benchmarks.
- The same skeleton sessions non-code research: swap the mount and measurement for document corpora, eval suites, or prompt-optimization sessions.

## Smoke test

After the PR merges and GitHub Sync applies the resources, mention the coordinator with a toy objective and a budget of one round ("campaign: measure how long `npm install` takes cold vs warm-cached, 1 round, 2 hypotheses"). Confirm the campaign brief posts, experimenter sessions spawn and report, and the round summary lands in the thread.
