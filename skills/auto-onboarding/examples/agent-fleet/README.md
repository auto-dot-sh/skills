# Agent fleet (Chief of Staff Engineers)

The flagship: an organized fleet of agents on long-horizon engineering work, and the most load-bearing workflow running on the auto repo itself. A human tags **@chief** in Slack with a list of tasks; the chief splits the list into well-scoped tasks, dispatches one **staff-engineer** run per task, shepherds every run until its PR has green CI and a clean review verdict, escalates only the decisions that genuinely belong to a human, and delivers one collated packet back in the thread when the batch is done.

```
.auto/
  environments/agent-runtime.yaml
  profiles/chief-of-staff.yaml      # the orchestrator's standing orders
  profiles/staff-engineer.yaml      # the worker's standing orders
  agents/chief-of-staff.yaml      # singleton: mentions, thread replies, reactions, heartbeat
  agents/staff-engineer.yaml      # spawn-only: woken by PR events on its owned artifact
```

## How it works

This example composes nearly the whole trigger/routing vocabulary:

- **Singleton orchestrator**: every chief trigger routes `deliverOrSpawn` / `deliver` with `routeBy: singleton` — mentions, subscribed thread replies, reactions, and a 15-minute heartbeat all land in the *one* live chief run, which can track multiple batches from different threads simultaneously.
- **Division of labor by construction**: the chief's mount is read-only (it scopes tasks and answers questions but cannot write code); the staff engineer's mount can write and open PRs. The chief's tools are delegation and communication only. The staff engineer's GitHub API access is the brokered `github` tool catalog with its five tools named explicitly (read, comment, open/update PRs — never merge); this is deliberate so the example works without assuming a preauthenticated `gh` CLI in the sandbox.
- **Dispatch protocol**: one `auto.runs.spawn` per task with an idempotency key (thread id + task slug) so retries never double-spawn. The spawn message is a structured brief: slug, statement, acceptance criteria, constraints, the originating thread, the chief's run id, and the reporting protocol.
- **Milestone reporting**: staff engineers report `started` / `pr-opened` / `fixing-ci` / `blocked` / `ready` to the chief's run via `auto.runs.message`. Blocked early beats stuck silently.
- **Event-driven waiting, no polling**: after opening its PR, a staff engineer records ownership (`auto.artifacts.record`) and *ends its turn*. Check failures, aggregate CI success, review comments, and merge conflicts on that PR wake it via `ownedArtifact` delivery. The PR-conversation trigger deliberately has no `$.auto.authored: false` filter — the staff engineer must receive the pr-review agent's comments, which are also auto-authored — so its own PR comments cost one no-op wake-up each; don't "fix" the filter or it goes deaf to its reviewer. The chief's heartbeat sweeps the fleet for stalls and respawns dead runs.
- **Crash recovery**: if the chief singleton dies, the next mention spawns a fresh run whose first duty is to rebuild batch state from `auto.runs.list`, introspection, and thread history.
- **Humans stay in the loop where it matters**: ambiguities are raised as crisp questions with recommendations; merging is always a human decision; the final packet links every PR with verification and residual risks.

## Composes with the code-review example

The staff engineer's definition of done includes "the pr-review check concluded thumbs-up" — that reviewer is the `code-review` example. Deploy both for the full loop (fleet writes, reviewer gates). Without it, drop the pr-review clauses from the staff-engineer profile and the aggregate-CI trigger message.

## Customize

- Replace `acme/widgets`, `github-acme`, `slack`, and the staff-engineer mount's `commitAuthor` (your GitHub App's bot login and noreply email, so pushed commits attribute correctly); set the aggregate check name in the staff-engineer triggers (`All checks` here) to the repo's actual roll-up check, or remove that filter to react to every check.
- Tune the heartbeat cadence to batch volume; 15 minutes suits an active repo.
- Point both profiles at the repo's real contribution docs and test commands.

## Smoke test

Apply, run `auto agents connect chief-of-staff` to realize the bot, then tag it with one trivial task ("add a TODO note to README"). Confirm the intake reaction and roster reply, watch the staff-engineer run with `auto runs list` / `auto runs conversation`, and confirm the packet arrives in the thread once the PR is green.
