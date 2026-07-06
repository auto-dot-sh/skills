# Agent fleet (Chief of Staff Engineers)

The flagship: an organized fleet of agents on long-horizon engineering work, and the most central workflow running on the auto repo itself. A human tags **@chief** in Slack with a list of tasks; the chief splits the list into well-scoped tasks, dispatches one **staff-engineer** run per task, shepherds every run until its PR has green CI and a clean review verdict, escalates only decisions that belong to a human, and delivers one collated packet back in the thread when the batch is done.

```
.auto/
  fragments/environments/agent-runtime.yaml # reusable sandbox runtime
  agents/chief-of-staff.yaml       # concurrency: 1; persona, standing orders, mentions, replies, heartbeat
  agents/staff-engineer.yaml       # spawn-only: standing orders, woken by PR events on its owned artifact
```

## Create from the managed template

This archetype is published as the managed template `@auto/agent-fleet`. The default way to install it is one thin agent file per agent in the user's repo — the template carries the prompts, triggers, tools, runtime, and identity with its avatar baked in, so no assets are copied:

```yaml
# .auto/agents/chief-of-staff.yaml
imports:
  - "@auto/agent-fleet@latest/agents/chief-of-staff.yaml"
variables:
  repoFullName: acme/widgets
  githubConnection: github-acme
  slackConnection: slack
```

```yaml
# .auto/agents/staff-engineer.yaml
imports:
  - "@auto/agent-fleet@latest/agents/staff-engineer.yaml"
variables:
  repoFullName: acme/widgets
  githubConnection: github-acme
  slackConnection: slack
```

The same variables block is shared across both agent files for symmetry; each agent file substitutes only the variables it references and ignores the rest. Set the variables to the user's repo, connection names, and channel. Override any field by declaring it in the importing file (local fields win; triggers merge by their authoring `name:`), and use `remove: { triggers: [...], tools: [...] }` to drop inherited entries. The directory below is the source this template was derived from.

## How it works

This example composes nearly the whole trigger/routing vocabulary:

- **One-slot orchestrator**: the chief declares `concurrency: 1` and every trigger routes plain `deliver` (mentions and subscribed replies with `onUnmatched: spawn`) — mentions, subscribed thread replies, reactions, and a 15-minute heartbeat all land in the _one_ live chief run, which can track multiple batches from different threads simultaneously.
- **Division of labor by construction**: the chief's mount is read-only (it scopes tasks and answers questions but cannot write code); the staff engineer's mount can write and open PRs. The chief's tools are delegation and communication only. The staff engineer's GitHub API access is the brokered `github` tool catalog with its five tools named explicitly (read, comment, open/update PRs — never merge); this is deliberate so the example works without assuming a preauthenticated `gh` CLI in the sandbox.
- **Dispatch protocol**: one `auto.sessions.spawn` per task with an idempotency key (thread id + task slug) so retries never double-spawn. The spawn message is a structured brief: slug, statement, acceptance criteria, constraints, the originating thread, the chief's run id, and the reporting protocol.
- **Milestone reporting**: staff engineers report `started` / `pr-opened` / `fixing-ci` / `blocked` / `ready` to the chief's run via `auto.sessions.message`. Blocked early beats stuck silently.
- **Event-driven waiting, no polling**: after opening its PR, a staff engineer binds the PR to its session (`auto.bind`) and _ends its turn_. Check failures, aggregate CI success, review comments, and merge conflicts on that PR wake it via `bind` routing on the `github.pull_request` target. The PR-conversation trigger deliberately has no `$.github.auto.authored: false` filter — the staff engineer must receive the pr-review agent's comments, which are also auto-authored — so its own PR comments cost one no-op wake-up each; don't "fix" the filter or it goes deaf to its reviewer. The chief's heartbeat sweeps the fleet for stalls and respawns dead sessions.
- **Crash recovery**: if the chief's live session dies, the next mention spawns a fresh run whose first duty is to rebuild batch state from `auto.sessions.list`, introspection, and thread history.
- **Humans stay in the loop where it matters**: ambiguities are raised as crisp questions with recommendations; merging is always a human decision; the final packet links every PR with verification and residual risks.

## Composes with the code-review example

The staff engineer's definition of done includes "the pr-review check concluded thumbs-up" — that reviewer is the `code-review` example. Deploy both for the full loop (fleet writes, reviewer gates). Without it, drop the pr-review clauses from the staff-engineer prompt and the aggregate-CI trigger message.

## Customize

- Replace `acme/widgets`, `github-acme`, `slack`, and the staff-engineer mount's `commitAuthor` (your GitHub App's bot login and noreply email, used as fallback author, committer, and the agent co-author trailer); set the aggregate check name in the staff-engineer triggers (`All checks` here) to the repo's actual roll-up check, or remove that filter to react to every check.
- Tune the heartbeat cadence to batch volume; 15 minutes suits an active repo.
- Point both agent prompts at the repo's real contribution docs and test commands.

## Smoke test

After the PR merges and GitHub Sync applies the resources, tag the chief with one trivial task ("add a TODO note to README"). Confirm the intake reaction and roster reply, watch the staff-engineer session with `mcp__auto__auto_sessions_list` / `mcp__auto__auto_sessions_conversation`, and confirm the packet arrives in the thread once the PR is green.
