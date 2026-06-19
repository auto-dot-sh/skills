# Tools, connections, and the runtime surface

## Connections: who the platform may act as

A connection is a credential grant the platform holds and resolves at runtime — never a token in YAML.

Hosted onboarding starts after the user's GitHub repository and Slack workspace
are connected to the Auto project. Use the existing GitHub connection for mounts,
PRs, issues, checks, and repo events, and the existing Slack connection for the
onboarding thread and Slack-triggered workflows.

| Provider                                                                  | How it connects                   | What it powers                                                        |
| ------------------------------------------------------------------------- | --------------------------------- | --------------------------------------------------------------------- |
| `github`                                                                  | App installation on the org/repos | Git mounts, brokered GitHub MCP tools, PR/issue/check events          |
| `slack`                                                                   | Workspace OAuth                   | `chat` tools, mention/reply/reaction events, per-agent bot identities |
| `linear`                                                                  | Workspace OAuth                   | `chat` tools against issues, issue created/updated events             |
| `telegram`                                                                | Manager bot token                 | `chat` tools, mention/DM events, per-agent bots                       |
| Built-in MCP providers (`notion`, `vercel`, `sentry`, `datadog-us5`, ...) | MCP OAuth                         | Connection-backed MCP tools                                           |

Auto MCP flow for additional providers:

1. `mcp__auto__auto_connections_providers_list` shows what can be connected.
2. `mcp__auto__auto_connections_start` starts a provider consent flow and returns
   an authorization URL for the user.
3. `mcp__auto__auto_connections_list` confirms the connection is available to
   the project.

Trigger `connection:` fields and chat-tool `auth.connection` fields reference these grants by name.

## Tools

### Connection-backed MCP tools

For built-in hosted MCP providers, connect first with the Auto MCP connection
tools and keep agent YAML at the connection level:

```yaml
tools:
  notion:
    kind: connection
    provider: notion
    connection: notion
```

Auto resolves the connection, injects the hosted MCP server and OAuth bearer
token at runtime, and the agent sees `mcp__notion__*`.

### Raw remote MCP tools

Any MCP server reachable over HTTPS:

```yaml
kind: agent
metadata:
  name: notion-agent
spec:
  harness: claude-code
  environment: agent-runtime
  tools:
    notion:
      kind: mcp_remote
      description: Notion pages, databases, and workspace search.
      url: https://mcp.notion.com/mcp
      transport: streamable_http
      auth:
        kind: mcp_oauth
        connection: notion-workspace
```

Auth options: `mcp_oauth` (named project-scoped MCP OAuth credential; create or refresh it with `mcp__auto__auto_agent_tools_connect`), `bearer` (`token: { $secret: name }`), or `none`. Prefer `kind: connection` for built-in hosted MCP providers.

For onboarding, connect OAuth-backed remote MCP tools before opening the PR that
adds the full agent. Draft the full agent tool configuration first, then call
`mcp__auto__auto_agent_tools_connect` for the proposed agent/tool source. A
typical raw MCP OAuth tool block looks like this:

```yaml
tools:
  notion:
    kind: mcp_remote
    description: Notion pages, databases, and workspace search.
    url: https://mcp.notion.com/mcp
    transport: streamable_http
    auth:
      kind: mcp_oauth
      connection: notion
```

Concrete flow:

1. Draft the full agent resource locally, including `tools.notion`.
2. Call `mcp__auto__auto_agent_tools_connect` for that proposed agent/tool
   source, for example the proposed agent `notion-reporter` and tool `notion`.
3. If the result reports an authorization URL, send a brief Slack explainer,
   then send the URL by itself in a separate Slack message.
4. Verify the connect result reports a live connection.
5. Stage, validate, commit, and open the PR containing the full agent resource.

Real examples from production use: Linear (`https://mcp.linear.app/mcp`), Notion (`https://mcp.notion.com/mcp`), Vercel (`https://mcp.vercel.com`), Datadog (`https://mcp.us5.datadoghq.com/...`).

### Local tools

Platform-implemented tools are declared inline under `agent.spec.tools`:

```yaml
tools:
  slack:
    kind: local
    implementation: chat # chat | auto | ping
    auth:
      kind: connection
      provider: slack
      connection: slack # the connection name from Auto MCP connection listing
```

- **`chat`** — the unified messaging surface (below). Agents can bind several provider connections under one alias with `auth.kind: connections`.
- **`auto`** — the platform-coordination surface (below). Needs no auth.
- **`ping`** — connectivity smoke test.

### Granting tools to an agent

```yaml
tools:
  chat:
    kind: local
    implementation: chat
    auth:
      kind: connections # multiple providers under one alias
      connections:
        - provider: slack
          connection: slack
        - provider: linear
          connection: linear
  notion:
    kind: connection
    provider: notion
    connection: notion
  auto:
    kind: local
    implementation: auto
  github: # inline-only; auth comes from the agent's mounts
    kind: github
    tools: [pull_request_read, add_issue_comment] # omit for the curated default set
```

The alias (the YAML key) is what the agent sees; `workspace` is reserved. The `github` kind exposes a curated GitHub MCP catalog brokered through the agent's mount credentials — name tools explicitly to narrow (a reviewer gets read + comment, never merge).

## What the agent sees at runtime

### `chat.*` — unified messaging

One interface across Slack, Linear, and Telegram; every call takes a `target` of provider + destination:

- `chat.send` — send a message (channel, optional thread). Returns `messageId`/`threadId` — save the threadId to keep replies threaded.
- `chat.history` — read recent messages in a channel or thread.
- `chat.search` — find channels/messages.
- `chat.react` — add an emoji reaction to a message (cheap acknowledgement).
- `chat.issue.get` / `chat.issue.update` — read and mutate issue metadata (Linear): state, assignee, labels.

Slack notes worth baking into agent prompts: pass channel names like `"#dev"` directly; Slack renders mrkdwn links (`<https://url|text>`), not Markdown.

### `auto.*` — platform coordination

- `auto.run.get` — the current run's own identity and scope (who am I, which agent/project).
- `auto.agents.list` — list the project's agents (discover who to hand work to).
- `auto.sessions.spawn` — start a run of another agent with a message (agent-to-agent handoff).
- `auto.sessions.list` / `auto.sessions.get` / `auto.sessions.conversation` / `auto.sessions.search` / `auto.sessions.tools` / `auto.sessions.summary` / `auto.sessions.commands` / `auto.sessions.triggers` / `auto.sessions.artifacts` — inspect sibling sessions (this is how a self-improvement agent reads the system).
- `auto.sessions.message` — send a message into another live run.
- `auto.chat.subscribe` — subscribe this session to a chat thread so future replies route back to it (pairs with `attributedSessions` delivery). In onboarding, call this once immediately after the first reply in the thread.
- `auto.artifacts.record` / `auto.artifacts.list` / `auto.artifacts.release` — record, list, and release ownership of an artifact such as a PR (pairs with `ownedArtifact` delivery).

### `checks.*` — GitHub check runs

Available when the spawning trigger declared `checks:`. `checks.list` shows the run's managed checks with status and timeouts; `checks.begin`, then `checks.success` or `checks.failure` with `{ name, summary, text }`.

## Secrets

Secrets are entered by the user through the Auto CLI, never pasted into chat or
committed to YAML. Give the user a command like this and reference only the
secret name afterward:

```bash
read -rsp "SENTRY_TOKEN: " SENTRY_TOKEN
printf %s "$SENTRY_TOKEN" | auto secrets set sentry-token --stdin
unset SENTRY_TOKEN
```

Reference secrets as `{ $secret: <name> }` in env maps and tool auth, and as
`secretRef: <name>` in webhook trigger auth. Add `optional: true` on env
references that should degrade rather than block sessions when missing.
