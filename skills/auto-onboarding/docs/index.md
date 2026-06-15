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

1. **The declarative half.** Agent YAML in `.auto/agents/` is the source of truth. `auto apply` compiles inline identities and environments from those agents, then makes the platform match the directory. Merge-to-apply CI makes the repo the source of truth.
2. **The event half.** Provider connections (GitHub, Slack, Linear, Telegram), custom webhooks, and cron heartbeats emit events. Agent triggers match events and either **spawn** a new run or **deliver** the event to a live one. Runs are durable, observable executions of an agent.

## Agent YAML

Every authored workflow lives under `.auto/agents/`. Agents can import shared
fragments from `.auto/fragments/environments/` for reused runtime, mount, tool, or trigger
configuration. Avatar images referenced by inline identities live in
`.auto/assets/`.

An agent file uses the root-level facade format:

```yaml
name: pr-review
labels:
  purpose: pr-review
imports:
  - ../fragments/environments/agent-runtime.yaml
identity:
  displayName: PR Review
  username: pr-review
systemPrompt: |
  You are the code review agent for acme/widgets.
```

## Agents are the center of gravity

An agent answers five questions:

- **Who is the agent?** `systemPrompt` and optional inline `identity` fields that give it a name, avatar, and its own @mentionable presence in chat providers.
- **Where does it run?** `harness` plus an inline or imported `environment` definition: base image, setup, resources, and env vars.
- **What does it know at start?** `initialPrompt`, a template with access to the triggering event via `{{payload.*}}` placeholders.
- **What can it touch?** `mounts` (git checkouts with scoped GitHub App permissions) and `tools` (inline local or remote MCP tool definitions).
- **When does it run?** `triggers`: provider events, custom webhook endpoints, or cron heartbeats, each with `where:` filters and a routing decision.
- **How do events route?** `routing.kind: spawn` starts a fresh run; `deliver` sends the event into an existing run, selected by `routeBy` (`singleton`, `ownedArtifact`, `attributedRuns`, `allLiveRuns`).

`docs/sessions-and-triggers.md` covers all of this field by field.

## What an agent can do at runtime

Inside a run, the agent has its mounted repo, ordinary shell access in its sandbox, and the tools granted to it:

- **`chat.*`** — send/read messages and manage issues across the connected chat providers (Slack, Linear, Telegram) through one interface.
- **`auto.*`** — coordinate with the platform itself: spawn sibling agents, list and read other runs, subscribe to chat threads, record artifact ownership.
- **`checks.*`** — report a GitHub check run (begin/success/failure) when a PR trigger declared one.
- **GitHub MCP tools** — a curated, capability-scoped GitHub API surface brokered from the agent's mounts.
- **Remote MCP tools** — whatever inline remote MCP tools the agent declares (Notion, Datadog, ...).

`docs/tools-and-connections.md` has the full runtime surface.

## Operating it

The `auto` CLI is the operator surface: auth/account profiles, org/project management, provider connections, `auto apply`, run inspection (`auto runs list|show|conversation|tools|triggers`), live interaction (`auto run`, `auto send`, `auto attach`), secrets, and service accounts for CI. See `docs/cli.md` and `docs/ci-cd.md`.

## Where to go next

- `docs/resource-model.md` — the `.auto/` directory and apply semantics
- `docs/sessions-and-triggers.md` — the trigger/event/routing vocabulary
- `docs/environments-and-profiles.md` — sandboxes and reusable agents
- `docs/tools-and-connections.md` — capabilities, connections, secrets
- `docs/cli.md` — command reference
- `docs/ci-cd.md` — apply-on-merge
- `../examples/index.md` — prose outline of nine complete `.auto/` example directories you can copy
