# Issue triage + coder handoff

Two cooperating agents: a **triage** agent that wakes when a Linear issue gets the `auto-triage` label — deduping, prioritizing, categorizing, and asking for missing context — and a **coder** agent it spawns when an issue is implementation-ready, which opens a PR and reports back on the issue.

```
.auto/
  environments/agent-runtime.yaml
  profiles/triage.yaml              # triage judgment + metadata hygiene rules
  profiles/coder.yaml               # implementation standing orders
  agents/issue-triage.yaml        # label-triggered triage
  agents/issue-coder.yaml         # spawn-only implementation agent
```

## How it works

- **Label as request token**: the trigger fires on `linear.issue.created` with the label present, and on `linear.issue.updated` only when the label was just *added* (`$.linear.updatedFrom.labelNames.added: { contains: ... }`). The triage agent removes the label once it has acted, so re-labeling is how humans re-request triage.
- **`chat.issue.*`**: the triage agent reads and mutates issue metadata (state, assignee, labels) through the unified chat tool — and is forbidden from *creating* metadata (labels, states, users) so it can't pollute the workspace.
- **Agent-to-agent handoff**: for implementation-ready issues, triage calls `auto.runs.spawn` with `session: issue-coder` and a message carrying the issue context, acceptance criteria, and constraints. The coder agent has **no triggers** — it only runs when spawned (or via `auto run issue-coder`).
- **Slack visibility**: handoffs get a brief top-level note in `#dev` with details threaded.

## Customize

- Replace `acme/widgets`, `github-acme`, `linear`, `slack`, `#dev`, and the label name.
- Align the "implementation-ready" bar and the coder's PR conventions (branch naming, Review Map section) with how the team actually works.
- If the team uses GitHub Issues instead of Linear, the same shape works with `github.issue_comment` / label events and the `github` tool.

## Smoke test

Apply, then add the `auto-triage` label to a test issue. Confirm a triage run spawns, the issue gets a triage comment, and the label is removed. Then label a genuinely ready issue and watch the handoff spawn a coder run that opens a PR.
