# Code review agent

A tailored PR reviewer: on every PR open, reopen, and push, it reviews the diff against the repo's own conventions, posts exactly one PR comment with a merge recommendation, reports a GitHub check, and threads a short verdict in Slack. Later PR comments and reviews route back to the owning session so the reviewer can incorporate material follow-up without reacting to its own comments.

```
.auto/
  fragments/environments/agent-runtime.yaml # node24 sandbox
  agents/pr-review.yaml            # persona, standing orders, triggers, and scoped GitHub access
```

## Create from the managed template

This archetype is published as the managed template `@auto/code-review`. The default way to install it is a thin agent file in the user's repo — the template carries the prompts, triggers, tools, runtime, and identity with its avatar baked in, so no assets are copied:

```yaml
# .auto/agents/pr-review.yaml
imports:
  - "@auto/code-review@latest/agents/pr-review.yaml"
variables:
  repoFullName: acme/widgets
  githubConnection: github-acme
  slackConnection: slack
  slackChannel: "#dev"
```

Set the variables to the user's repo, connection names, and channel. Override any field by declaring it in the importing file (local fields win; triggers merge by their authoring `name:`), and use `remove: { triggers: [...], tools: [...] }` to drop inherited entries. The directory below is the source this template was derived from.

## How it works

- **Trigger**: `github.pull_request.opened|reopened|synchronize` on the user's repo, `routing: spawn` — every PR head gets a fresh review run. PR issue comments, PR reviews, and review comments with `$.auto.authored: false` route back through `ownedArtifact`.
- **Mount**: the PR head itself (`ref: refs/pull/{{payload.github.pullRequest.number}}/head`, `depth: 1`) with read-only contents — the reviewer can run targeted tests but **cannot push**. PR comments are allowed via `pullRequests: write` / `issues: write`.
- **GitHub tools**: narrowed to `pull_request_read` + `add_issue_comment`. No approve, no merge, by construction.
- **Check run**: the trigger declares a `pr-review` check; the agent moves it begin → success/failure so the review can gate merges.
- **Artifact ownership**: the reviewer records the PR as a `github.pull_request` artifact before review work, so later conversation events can route back to the same session.
- **Slack**: one top-level message per PR in `#dev`, verdicts threaded under it on subsequent pushes.

## CI watchdog

If the repo has required GitHub Actions workflows, configure GitHub Sync to
watchdog those workflow files separately from the review trigger with
`mcp__auto__auto_sync_enable`:

```json
{
  "kind": "github",
  "connection": "github-acme",
  "repo": "acme/widgets",
  "branch": "main",
  "ciWatchdog": {
    "workflows": [{ "workflowId": "ci.yml" }]
  }
}
```

This is only for required GitHub Actions workflows that support
`workflow_dispatch`. It is not inferred from `github.check_run.completed`
triggers, because those triggers are passive event routing and may refer to
external checks or job names that Auto cannot safely dispatch.

## Customize

- Replace `acme/widgets`, `github-acme`, and the `#dev` channel.
- Keep the example identity unless the user wants a different persona:
  `identity.username: pr-review` with `identity.avatar.asset:
  .auto/assets/pr-reviewer.png`.
- Point the agent prompt at the repo's real convention docs (`CONTRIBUTING.md`, style guides) — this is what makes the review _tailored_.
- Adjust validation commands in the prompt to the repo's actual test/typecheck invocations.
- Add or remove review principles based on the user's expressed preferences, but keep the default posture practical: correctness first, real tests at provider boundaries, strong types at ingress/egress, and simplicity over performative complexity.
- If the user wants the check to be required, have them mark `Auto PR review` as a required status check in branch protection.

## Smoke test

After the PR merges and GitHub Sync applies the resources, open a trivial PR. Confirm: a session appears via `mcp__auto__auto_sessions_list`, the check shows on the PR, a review comment lands, and a Slack thread appears in `#dev`.
