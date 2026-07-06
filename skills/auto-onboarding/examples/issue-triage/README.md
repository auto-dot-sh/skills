# Issue triage + coder handoff

Two cooperating agents: a **triage** agent that wakes when a Linear issue gets the `auto-triage` label — deduping, prioritizing, categorizing, and asking for missing context — and a **coder** agent it spawns when an issue is implementation-ready, which opens a PR and reports back on the issue.

```
.auto/
  fragments/environments/agent-runtime.yaml # reusable sandbox runtime
  agents/issue-triage.yaml         # label-triggered triage, persona, and standing orders
  agents/issue-coder.yaml          # spawn-only implementation agent and standing orders
```

## Create from the managed template

This archetype is published as the managed template `@auto/issue-triage`. The default way to install it is one thin agent file per agent in the user's repo — the template carries the prompts, triggers, tools, runtime, and identity with its avatar baked in, so no assets are copied:

```yaml
# .auto/agents/issue-triage.yaml
imports:
  - "@auto/issue-triage@latest/agents/issue-triage.yaml"
variables:
  repoFullName: acme/widgets
  linearConnection: linear
  slackConnection: slack
  slackChannel: "#dev"
```

```yaml
# .auto/agents/issue-coder.yaml
imports:
  - "@auto/issue-triage@latest/agents/issue-coder.yaml"
variables:
  repoFullName: acme/widgets
  linearConnection: linear
  slackConnection: slack
  slackChannel: "#dev"
```

The same variables block is shared across both agent files for symmetry; each agent file substitutes only the variables it references and ignores the rest. Set the variables to the user's repo, connection names, and channel. Override any field by declaring it in the importing file (local fields win; triggers merge by their authoring `name:`), and use `remove: { triggers: [...], tools: [...] }` to drop inherited entries. The directory below is the source this template was derived from.

## How it works

- **Label as request token**: the trigger fires on `linear.issue.created` with the label present, and on `linear.issue.updated` only when the label was just _added_ (`$.linear.updatedFrom.labelNames.added: { contains: ... }`). The triage agent removes the label once it has acted, so re-labeling is how humans re-request triage.
- **`chat.issue.*`**: the triage agent reads and mutates issue metadata (state, assignee, labels) through the unified chat tool — and is forbidden from _creating_ metadata (labels, states, users) so it can't pollute the workspace.
- **Agent-to-agent handoff**: for implementation-ready issues, triage calls `auto.sessions.spawn` with `agent: issue-coder` and a message carrying the issue context, acceptance criteria, and constraints. The coder's real work starts from that handoff; its Slack mention trigger is only a friendly "here is what I do" path.
- **Slack visibility**: handoffs get a brief top-level note in `#dev` with details threaded.

## Customize

- Replace `acme/widgets`, `github-acme`, `linear`, `slack`, `#dev`, and the label name.
- Align the "implementation-ready" bar and the coder's PR conventions (branch naming, Review Map section) with how the team actually works.
- If the team uses GitHub Issues instead of Linear, use the GitHub issue event
  family: `github.issue.opened` / `.edited` for issue metadata changes and
  `github.issue.comment.created` / `.edited` for plain issue comments. PR
  comments remain `github.issue_comment.*`; plain-issue comments are
  `github.issue.comment.*`. For direct coding handoffs from GitHub Issues, the
  `handoff/` example already supports `/handoff` on a plain issue, keeps the
  `github.issue` binding, opens a PR, binds it as `github.pull_request`, and
  reports back on the issue.

## Smoke test

After the PR merges and GitHub Sync applies the resources, add the `auto-triage` label to a test issue. Confirm a triage session spawns, the issue gets a triage comment, and the label is removed. Then label an implementation-ready issue and watch the handoff spawn a coder session that opens a PR.
