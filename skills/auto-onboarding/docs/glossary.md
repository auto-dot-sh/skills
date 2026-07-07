# Glossary

Canonical definitions for Auto-specific terms, written for someone using the
platform for the first time. This is the source of truth for product
vocabulary: UI copy, docs, and agent-facing prompts should define terms the
way this file does. Keep entries to a couple of sentences; deeper mechanics
belong in the other docs in this directory (`resource-model.md`,
`agents-and-triggers.md`, and friends).

## Tenancy

- **Organization** — The top-level account that owns everything else:
  projects, members, org-wide secrets, and connections. You join Auto as a
  member of one or more organizations.
- **Project** — A workspace inside an organization, tied to the repository
  (or repositories) it works on. Agents, sessions, and most day-to-day
  activity are scoped to a project.

## Agents and sessions

- **Agent** — An AI teammate defined by a YAML file in your repo. An agent
  definition names its identity, the environment it runs in, its prompts and
  tools, and the triggers that put it to work.
- **Session** — One unit of an agent doing work: a conversation, a triggered
  task, a scheduled run. Sessions start from the web app, the CLI, or a
  trigger, and each one runs in its own sandbox.
- **Harness** — The coding-agent runtime a session runs on under the hood
  (for example Claude Code or Codex). Agents declare which harness they use;
  most users never interact with it directly.
- **Trigger** — A rule that starts sessions automatically when something
  happens: a pull request opens, an issue gets a label, someone mentions the
  agent in chat, or a schedule fires.
- **Identity** — An agent's outward presence: its name, avatar, and
  description — how it appears when it comments on GitHub or speaks in Slack.

## Where agents run

- **Environment** — A resource describing the cloud workspace an agent runs
  in: base image, installed tooling, environment variables, secret
  references, and repository mounts.
- **Sandbox** — The isolated cloud machine created from an environment for a
  single session. It is provisioned when the session starts and torn down
  after; nothing runs on your own hardware.
- **Mount** — A git repository checked out into a sandbox so the agent can
  read and change its code.
- **Secret** — A named, encrypted value (an API key, a token) stored by
  Auto. Resources reference secrets by name (`$secret: my-key`); the value is
  injected only at runtime and never lives in your repo. Secrets can carry an
  optional expiry — an absolute deadline and/or an unused-for window — after
  which they behave as if they were never set.
- **Connection** — A link between your project and an external provider
  such as GitHub, Slack, or Linear. Connections are what let agents open pull
  requests, post messages, and read issues.

## Configuration as code

- **`.auto/` directory** — The directory in your repository that holds your
  project's resource definitions. Your repo is the source of truth: what is
  merged there is what runs.
- **Resource** — Any YAML definition under `.auto/`. The kinds are
  connection, identity, environment, agent, and config, and they apply in
  that order.
- **Apply** — Compiling the resources in `.auto/` and making them live in
  your project. Applies normally happen automatically via GitHub Sync.
- **GitHub Sync** — The automation that watches your repository and applies
  `.auto/` changes when they merge to the default branch, reporting the
  result as a check on the merge.
- **Managed Template** — A pre-built, Auto-maintained package of resources
  (named like `@auto/onboarding`) that you import into `.auto/` instead of
  writing everything from scratch. Importing at `@latest` tracks updates
  automatically; pinning a version (`@auto/onboarding@1.2.0`) opts out.
- **Fragment** — A local, reusable partial under `.auto/fragments/` that
  your agents import to share common configuration within one repo.

## Everything else

- **Software factory** — The working metaphor for the end state: your
  team's set of agents, triggers, and workflows, running as code owned by
  your repo — a production line for software changes rather than a single
  assistant.
- **`auto` CLI** — The command-line client (installed from npm) for working
  with sessions and resources from a terminal instead of the web app.
