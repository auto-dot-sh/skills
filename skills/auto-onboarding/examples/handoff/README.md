# Handoff coding agent

A handoff coder takes ownership of a pull request or implementation request,
keeps the PR and Slack thread updated, reacts to CI/review/blockers, and
reports back when the work is ready for final human review. This example is
modeled on the handoff agent used in the Auto repository.

```
.auto/
  fragments/environments/agent-runtime.yaml # node24 sandbox
  agents/handoff.yaml              # persona, ownership protocol, triggers, and write access
```

## Create from the managed template

This archetype is published as the managed template `@auto/handoff`. The default way to install it is a thin agent file in the user's repo — the template carries the prompts, triggers, tools, runtime, and identity with its avatar baked in, so no assets are copied:

```yaml
# .auto/agents/handoff.yaml
imports:
  - "@auto/handoff@latest/agents/handoff.yaml"
variables:
  repoFullName: acme/widgets
  githubConnection: github-acme
  slackConnection: slack
  slackChannel: "#dev"
```

Set the variables to the user's repo, connection names, and channel. Override any field by declaring it in the importing file (local fields win; triggers merge by their authoring `name:`), and use `remove: { triggers: [...], tools: [...] }` to drop inherited entries. The directory below is the source this template was derived from.

## How it works

- **Explicit handoff**: a GitHub PR comment/review, GitHub issue body, or
  GitHub issue comment that mentions the agent starts or continues a handoff.
  Plain issue handoffs use `github.issue.opened` and
  `github.issue.comment.created`; PR comments keep the older
  `github.issue_comment.*` event names. A Slack mention such as
  `@auto.handoff` can also start a new session.
- **Ownership**: once it confirms the handoff is intentional, the agent keeps
  the spawn-claimed `github.issue` binding for issue-origin work, opens or
  identifies the implementation PR, and binds that PR as `github.pull_request`.
  Later issue comments, PR comments, reviews, failing checks, all-checks
  success, and merge conflicts deliver back to the same session with `bind`
  routing on the matching target.
- **Issue status surface**: when work starts from a GitHub issue, the agent
  acknowledges on the issue, opens a PR when implementation begins, links the
  PR back to the issue, and reports final status on the issue as well as the
  PR.
- **Slack thread**: the agent acknowledges the handoff in the originating Slack
  thread when one exists. For GitHub-only handoffs, it establishes or reuses a
  PR thread in `#dev`, subscribes to it, and sends future status there.
- **Event-driven waiting**: after pushing a fix or reaching a wait point, the
  session leaves a concise status update and ends. Auto wakes it back up for
  CI failures, review comments, PR-reviewer feedback, merge conflicts, and
  subscribed Slack replies.
- **PR-review loop**: when CI passes, the handoff waits for the latest
  PR-reviewer feedback on the current head before saying the PR is ready for
  final review. This works best after installing the `code-review/` example.
- **Write access**: the mount can push commits, create/update PRs, read
  Actions checks, and write PR comments. It does not merge unless a human
  explicitly asks and the PR is ready.

## Customize

- Replace `acme/widgets`, `github-acme`, `slack`, and `#dev`.
- Keep the example identity unless the user wants a different persona:
  `identity.username: handoff` with `identity.avatar.asset:
  .auto/assets/handoff.png`.
- Update the prompt to name the repo's real setup docs, test commands, review
  requirements, and branch policy.
- If you have a PR reviewer agent, keep the readiness gate that waits for its
  latest feedback. If you do not, replace that section with your team's normal
  review requirement.
- Add provider tools only when the handoff needs them to do real work, such as
  Linear for issue context or an observability MCP tool for bug fixes. Prefer
  connection-backed or read-only tools unless the workflow truly needs write
  access.
- Keep accidental-handoff detection. Documentation and examples often mention
  agent handles; the handoff should not take ownership just because it was
  quoted.

## Smoke test

After the PR merges and GitHub Sync applies the resources, open or reuse a
small GitHub issue. Comment `/handoff please implement this issue, open a PR,
and report back here when it is ready for final review.`

Confirm: a handoff session appears, the agent acknowledges on the issue and in
Slack when configured, holds the `github.issue` binding, opens and binds a PR,
reacts to a follow-up issue comment or failing PR check, and reports final
ready status back on the issue.
