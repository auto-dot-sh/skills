# Issue triage + coder handoff

Two cooperating agents: a **triage** agent that wakes when an issue needs triage — deduping, prioritizing, categorizing, and asking for missing context — and a **coder** agent it spawns when an issue is implementation-ready, which opens a PR and reports back on the issue.

**GitHub Issues is the base**: the plain entrypoints track issues in the repo itself and need nothing beyond the GitHub connection the repo mount already requires. **Linear is the optional variant**: teams that track work in Linear import the `-linear` entrypoints instead, which swap Linear in as the issue source of truth and add the Linear connection requirement.

```
.auto/
  fragments/environments/agent-runtime.yaml # reusable sandbox runtime
  agents/issue-triage.yaml         # base: GitHub Issues, opened + label-triggered triage
  agents/issue-triage-slack.yaml   # base + Slack triage visibility
  agents/issue-coder.yaml          # base: spawn-only coder, reports on the GitHub issue
  agents/issue-coder-slack.yaml    # base + Slack coder
  agents/issue-triage-linear.yaml  # Linear variant: label-triggered triage
  agents/issue-coder-linear.yaml   # Linear variant: spawn-only coder, reports on the Linear issue
```

## Create from the managed template

This archetype is published as the managed template `@auto/issue-triage` (1.3.0 makes GitHub Issues the plain base and moves Linear to the `-linear` entrypoints; ≤1.2.0's plain entrypoints are Linear). The default way to install it is one thin agent file per agent in the user's repo — the template carries the prompts, triggers, tools, runtime, and identity with its avatar baked in, so no assets are copied.

### GitHub Issues (base — no Linear connection needed)

```yaml
# .auto/agents/issue-triage.yaml
imports:
  - "@auto/issue-triage@latest/agents/issue-triage.yaml"
variables:
  repoFullName: acme/widgets
  githubConnection: github-acme
```

```yaml
# .auto/agents/issue-coder.yaml
imports:
  - "@auto/issue-triage@latest/agents/issue-coder.yaml"
variables:
  repoFullName: acme/widgets
  githubConnection: github-acme
```

### Linear (optional variant)

```yaml
# .auto/agents/issue-triage.yaml
imports:
  - "@auto/issue-triage@latest/agents/issue-triage-linear.yaml"
variables:
  repoFullName: acme/widgets
  linearConnection: linear
```

```yaml
# .auto/agents/issue-coder.yaml
imports:
  - "@auto/issue-triage@latest/agents/issue-coder-linear.yaml"
variables:
  repoFullName: acme/widgets
  linearConnection: linear
```

For Slack visibility, import the `-slack` entrypoints instead — `agents/issue-triage-slack.yaml` / `agents/issue-coder-slack.yaml` over the GitHub base, or `agents/issue-triage-linear-slack.yaml` / `agents/issue-coder-linear-slack.yaml` over the Linear variant — and add `slackConnection` and `slackChannel` to the variables. Each `-slack` entrypoint imports its base and layers a Slack `@mention` trigger plus a ready-work note in the channel.

The same variables block is shared across both agent files for symmetry; each agent file substitutes only the variables it references and ignores the rest. Set the variables to the user's repo, connection names, and channel. Override any field by declaring it in the importing file (local fields win; triggers merge by their authoring `name:`), and use `remove: { triggers: [...], tools: [...] }` to drop inherited entries. The directory tree at the top of this file is the source this template was derived from (the Linear `-slack` entrypoints live only in the managed template).

## How it works

### GitHub Issues (base)

- **Every new issue**: `github.issue.opened` wakes triage for every new issue (filtered with `$.github.auto.authored: false` so issues the Auto app itself opens do not loop). This is the primary behavior — no label is required for the initial triage.
- **Label as re-triage token**: `github.issue.labeled` re-triages only when the `auto-triage` label is applied (`$.github.label.name: auto-triage`, also `$.github.auto.authored: false`). The triage agent removes the label with `issue_write` once it has acted, so re-labeling is how humans re-request triage.
- **Bind routing**: both GitHub triggers route through the `github.issue` binding with `onUnmatched: spawn`, mirroring `@auto/handoff@1.6.0`'s issue-origin pattern. A repeat event on an issue whose triage session is already live delivers to that session instead of spawning a duplicate.
- **`issue_read` / `issue_write` / `add_issue_comment`**: the triage agent inspects, updates (labels, state), and comments on issues through the github tool, backed by a GitHub App mount with `issues: write`. It never creates new labels — if the expected label does not exist it notes that in a comment and continues.
- **Agent-to-agent handoff**: for implementation-ready issues, triage calls `auto.sessions.spawn` with `agent: issue-coder` and a message carrying the issue number, title, URL, triage summary, acceptance criteria, and constraints. The coder opens a PR and comments back on the GitHub issue with the PR link, tests run, and residual risks.

> **Event names**: plain-issue comments are `github.issue.comment.*` (e.g. `github.issue.comment.created`); PR comments remain `github.issue_comment.*`. The authored flag lives at `$.github.auto.authored` on GitHub events — not `$.auto.authored`, which is chat-only.

### Linear (variant)

- **Label as request token**: the trigger fires on `linear.issue.created` with the label present, and on `linear.issue.updated` only when the label was just _added_ (`$.linear.updatedFrom.labelNames.added: { contains: ... }`). The triage agent removes the label once it has acted, so re-labeling is how humans re-request triage. There is no every-new-issue trigger in the Linear variant — the label is the request token.
- **`chat.issue.*`**: the triage agent reads and mutates issue metadata (state, assignee, labels) through the unified chat tool — and is forbidden from _creating_ metadata (labels, states, users) so it can't pollute the workspace.
- **Agent-to-agent handoff**: same shape as the base; the coder reports back on the Linear issue via `chat.send` instead of a GitHub issue comment.

## Customize

- Replace `acme/widgets`, `github-acme`, `linear`, `slack`, and `#dev`.
- Align the "implementation-ready" bar and the coder's PR conventions (branch naming, Review Map section) with how the team actually works.
- For direct coding handoffs from GitHub Issues (the human comments `/handoff` on a plain issue), the `handoff/` example already supports it: it keeps the `github.issue` binding, opens a PR, binds it as `github.pull_request`, and reports back on the issue.

## Smoke test

### GitHub Issues (base)

After the PR merges and GitHub Sync applies the resources, open a new test issue in the repo. Confirm a triage session spawns for the `github.issue.opened` event and the issue gets a triage comment. Then add the `auto-triage` label to an implementation-ready issue and watch the handoff spawn a coder session that opens a PR, comments back on the issue, and removes the label.

### Linear (variant)

After the PR merges and GitHub Sync applies the resources, add the `auto-triage` label to a test Linear issue. Confirm a triage session spawns, the issue gets a triage comment, and the label is removed. Then label an implementation-ready issue and watch the handoff spawn a coder session that opens a PR.
