# auto: the mental model

auto lets you program software factories the same way you program CI/CD: declare agents and triggers in YAML, apply them to the cloud, and let events do the rest.

## The loop

```
.auto/ YAML  --auto apply-->  deployed resources
external event (PR opened, issue labeled, @mention, cron, webhook)
        --> trigger match --> run spawned (or delivered to an existing run)
        --> agent works in a cloud sandbox with its tools
        --> side effects (PR comments, Slack messages, commits, pages)
        --> follow-up events route back to the same run
```

Two halves to internalize:

1. **The declarative half.** Everything is a resource in the `.auto/` directory of a repo. `auto apply` makes the platform match the directory. Merge-to-apply CI makes the repo the source of truth.
2. **The event half.** Provider connections (GitHub, Slack, Linear, Telegram), custom webhooks, and cron heartbeats emit events. Agent triggers match events and either **spawn** a new run or **deliver** the event to a live one. Runs are durable, observable executions of an agent.

## The four resource kinds

Applied in this order (dependencies point left):

| Kind | Directory | What it is |
| --- | --- | --- |
| `environment` | `.auto/environments/` | The sandbox: base image, extra image steps, setup commands (cacheable), CPU/memory, env vars. |
| `profile` | `.auto/profiles/` | A reusable agent: which harness (`claude-code`), which environment, and durable instructions — the agent's standing orders. |
| `tool` | `.auto/tools/` | A capability: a remote MCP server (Linear, Notion, Datadog, Vercel, ...) or a local implementation (`chat`, `auto`, `ping`) bound to a provider connection. |
| `agent` | `.auto/agents/` | The runnable workflow: binds a profile to an initial prompt, repo mounts, a tool set, an identity, and triggers. |

A fifth directory, `.auto/assets/`, holds avatar images referenced by agent identities.

Every resource file is YAML with the same envelope:

```yaml
kind: agent
metadata:
  name: pr-review
  labels:
    purpose: pr-review
spec:
  # kind-specific fields
```

## Agents are the center of gravity

An agent answers five questions:

- **Who is the agent?** `spec.profile` (instructions + environment), plus an optional `spec.identity` that gives it a name, avatar, and its own @mentionable presence in chat providers.
- **What does it know at start?** `spec.initialPrompt`, a template with access to the triggering event via `{{payload.*}}` placeholders.
- **What can it touch?** `spec.mounts` (git checkouts with scoped GitHub App permissions) and `spec.tools` (aliased tool references or inline definitions).
- **When does it run?** `spec.triggers`: provider events, custom webhook endpoints, or cron heartbeats, each with `where:` filters and a routing decision.
- **How do events route?** `routing.kind: spawn` starts a fresh run; `deliver` sends the event into an existing run, selected by `routeBy` (`singleton`, `ownedArtifact`, `attributedRuns`, `allLiveRuns`).

`docs/sessions-and-triggers.md` covers all of this field by field.

## What an agent can do at runtime

Inside a run, the agent has its mounted repo, ordinary shell access in its sandbox, and the tools granted to it:

- **`chat.*`** — send/read messages and manage issues across the connected chat providers (Slack, Linear, Telegram) through one interface.
- **`auto.*`** — coordinate with the platform itself: spawn sibling agents, list and read other runs, subscribe to chat threads, record artifact ownership.
- **`checks.*`** — report a GitHub check run (begin/success/failure) when a PR trigger declared one.
- **GitHub MCP tools** — a curated, capability-scoped GitHub API surface brokered from the agent's mounts.
- **Remote MCP tools** — whatever `tool` resources the agent references (Notion, Datadog, ...).

`docs/tools-and-connections.md` has the full runtime surface.

## Operating it

The `auto` CLI is the operator surface: auth and profiles, org/project management, provider connections, `auto apply`, run inspection (`auto runs list|show|conversation|tools|triggers`), live interaction (`auto run`, `auto send`, `auto attach`), secrets, and service accounts for CI. See `docs/cli.md` and `docs/ci-cd.md`.

## Where to go next

- `docs/resource-model.md` — the `.auto/` directory and apply semantics
- `docs/sessions-and-triggers.md` — the trigger/event/routing vocabulary
- `docs/environments-and-profiles.md` — sandboxes and reusable agents
- `docs/tools-and-connections.md` — capabilities, connections, secrets
- `docs/cli.md` — command reference
- `docs/ci-cd.md` — apply-on-merge
- `../examples/index.md` — prose outline of nine complete `.auto/` example directories you can copy
