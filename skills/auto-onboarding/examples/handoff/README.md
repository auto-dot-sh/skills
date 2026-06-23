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

## How it works

- **Explicit handoff**: a GitHub comment/review that mentions the agent, or a
  Slack mention such as `@auto.handoff`, starts a new session with
  `routing: spawn`.
- **Ownership**: once it confirms the handoff is intentional, the agent records
  the pull request as a `github.pull_request` artifact. Later PR comments,
  reviews, failing checks, all-checks success, and merge conflicts deliver back
  to the same session with `ownedArtifact`.
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
small PR. Comment `/handoff please take this PR, address review comments and
CI failures, and report back when it is ready for final review.`

Confirm: a handoff session appears, the agent acknowledges on GitHub and in
Slack, records PR ownership, reacts to a follow-up PR comment or failing check,
and waits for the latest PR-reviewer feedback before final-ready status.
