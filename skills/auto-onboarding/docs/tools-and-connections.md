# Tools, connections, and the runtime surface

## Connections: who the platform may act as

A connection is a credential grant the platform holds and resolves at runtime — never a token in YAML.

| Provider | How it connects | What it powers |
| --- | --- | --- |
| `github` | App installation on the org/repos | Git mounts, brokered GitHub MCP tools, PR/issue/check events |
| `slack` | Workspace OAuth | `chat` tools, mention/reply/reaction events, per-agent bot identities |
| `linear` | Workspace OAuth | `chat` tools against issues, issue created/updated events |
| `telegram` | Manager bot token | `chat` tools, mention/DM events, per-agent bots |

CLI flow:

```sh
auto connections list --available    # what can be connected
auto connect <provider>              # start the flow (opens a browser; --no-browser to print the URL)
auto connections list                # what is connected, with connection names
auto allow <provider> [project]      # share an org-level connection with a project
```

Trigger `connection:` fields and chat-tool `auth.connection` fields reference these grants by name.

## Tools

### Remote MCP tools

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

Auth options: `mcp_oauth` (named project-scoped MCP OAuth credential; create or refresh it with `auto apply --connect`), `provider_oauth` (reference an existing provider connection by name), `bearer` (`token: { $secret: name }`), or `none`.

Real examples from production use: Linear (`https://mcp.linear.app/mcp`), Notion (`https://mcp.notion.com/mcp`), Vercel (`https://mcp.vercel.com`), Datadog (`https://mcp.us5.datadoghq.com/...`).

### Local tools

Platform-implemented tools are declared inline under `agent.spec.tools`:

```yaml
tools:
  slack:
    kind: local
    implementation: chat       # chat | auto | ping
    auth:
      kind: connection
      provider: slack
      connection: slack        # the connection name from `auto connections list`
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
      kind: connections               # multiple providers under one alias
      connections:
        - provider: slack
          connection: slack
        - provider: linear
          connection: linear
  notion:
    kind: mcp_remote
    url: https://mcp.notion.com/mcp
    transport: streamable_http
    auth:
      kind: mcp_oauth
      connection: notion-workspace
  auto:
    kind: local
    implementation: auto
  github:                             # inline-only; auth comes from the agent's mounts
    kind: github
    tools: [pull_request_read, add_issue_comment]   # omit for the curated default set
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
- `auto.sessions.list` — list the project's agents (discover who to hand work to).
- `auto.runs.spawn` — start a run of another agent with a message (agent-to-agent handoff).
- `auto.runs.list` / `auto.runs.get` / `auto.runs.conversation` / `auto.runs.search` / `auto.runs.tools` / `auto.runs.summary` / `auto.runs.commands` / `auto.runs.triggers` / `auto.runs.artifacts` — inspect sibling runs (this is how a self-improvement agent reads the system).
- `auto.runs.message` — send a message into another live run.
- `auto.chat.subscribe` — subscribe this run to a chat thread so future replies route back to it (pairs with `attributedRuns` delivery).
- `auto.artifacts.record` / `auto.artifacts.list` / `auto.artifacts.release` — record, list, and release ownership of an artifact such as a PR (pairs with `ownedArtifact` delivery).

### `checks.*` — GitHub check runs

Available when the spawning trigger declared `checks:`. `checks.list` shows the run's managed checks with status and timeouts; `checks.begin`, then `checks.success` or `checks.failure` with `{ name, summary, text }`.

## Secrets

```sh
auto secrets set <name>      # prompted, never echoed
auto secrets list
auto secrets remove <name>
```

Referenced as `{ $secret: <name> }` in env maps and tool auth, and as `secretRef: <name>` in webhook trigger auth. Add `optional: true` on env references that should degrade rather than block runs when missing.
