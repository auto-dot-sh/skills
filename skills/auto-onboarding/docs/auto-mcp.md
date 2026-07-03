# Auto MCP Tools

Hosted onboarding agents operate Auto through MCP tools. The user already has
an Auto project with a GitHub repository connected and a Slack workspace
connected before the onboarding conversation starts.

Use these tools as the operator surface during onboarding:

| Need | Auto MCP examples |
| --- | --- |
| See available providers | `mcp__auto__auto_connections_providers_list` |
| List existing provider grants | `mcp__auto__auto_connections_list` |
| Start an additional provider consent flow | `mcp__auto__auto_connections_start` |
| List or enable Sync bindings | `mcp__auto__auto_sync_list`, `mcp__auto__auto_sync_enable` |
| Validate drafted `.auto/` resources | `mcp__auto__auto_resources_dry_run` |
| Connect a remote MCP OAuth tool from proposed agent source | `mcp__auto__auto_agent_tools_connect` |
| Spawn a manual smoke-test session | `mcp__auto__auto_sessions_spawn` |
| Inspect sessions | `mcp__auto__auto_sessions_list`, `mcp__auto__auto_sessions_get`, `mcp__auto__auto_sessions_conversation` |
| Debug session tools and routing | `mcp__auto__auto_sessions_tools`, `mcp__auto__auto_sessions_triggers`, `mcp__auto__auto_sessions_bindings` |
| Send or subscribe chat | `mcp__auto__chat_send`, `mcp__auto__auto_chat_subscribe` |
| Bind a PR to the session | `mcp__auto__auto_bind` |

## Connected by default

Treat the onboarding repo and Slack workspace as present. Identify the repo from
the mounted checkout and `git remote get-url origin` instead of asking the user
for it. Confirm the Slack destination with the user when it matters to the
workflow, but spend the setup beat on the workflow itself unless an MCP lookup
shows a missing grant.

For a GitHub-backed workflow, use the mounted repo and GitHub MCP tools to make
branches, commits, comments, and PRs. For Slack output, use the connected Slack
chat tool and the onboarding thread as the first verification surface.

## Additional provider flows

When the chosen workflow needs a provider beyond the existing GitHub and Slack
connections:

1. Call `mcp__auto__auto_connections_providers_list` to confirm the provider is
   available.
2. Call `mcp__auto__auto_connections_start` for the provider and project.
3. Send a brief chat message explaining what the authorization grants, then send
   the returned authorization URL by itself in a separate chat message.
4. Poll or re-list with `mcp__auto__auto_connections_list` until the grant is
   available for the project.

Do not ask the user to paste OAuth codes or tokens into chat. The MCP flow owns
the credential exchange and status reporting.

## Resource validation

Use `mcp__auto__auto_resources_dry_run` before opening or updating a PR with
`.auto/` changes. Summarize the create/update/archive plan for the user, then
ship the resource change through GitHub Sync by opening a PR and asking the user
to merge when ready.

GitHub Sync applies committed resources after merge. Inspect Auto resource and
session state through MCP after merge instead of relying on terminal output or
GitHub Actions logs.

## Sync bindings

Use `mcp__auto__auto_sync_list` and `mcp__auto__auto_sync_enable` for GitHub
Sync setup from hosted agents. Both tools use a provider discriminator so the
shape can grow beyond GitHub later.

List GitHub Sync bindings:

```json
{ "kind": "github" }
```

Enable or update a GitHub Sync binding:

```json
{
  "kind": "github",
  "connection": "github-acme",
  "repo": "acme/widgets",
  "branch": "main"
}
```

To configure the CI watchdog for required GitHub Actions workflows, include
`ciWatchdog`:

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

Use `ciWatchdog` only for required Actions workflows that support
`workflow_dispatch`. Check triggers are passive event routing; the watchdog
actively dispatches a missing workflow run.

## Remote MCP OAuth tools

For raw remote MCP tools with `auth.kind: mcp_oauth`, connect the tool before
opening the PR that adds the full agent resource. Draft the agent tool
configuration, then call `mcp__auto__auto_agent_tools_connect` for the proposed
agent/tool source. A typical tool block looks like this:

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
3. If the result reports an authorization URL, first send a short explainer in
   Slack, then send the URL by itself in a separate Slack message.
4. Verify the connect result reports a live connection.
5. Stage, validate, commit, and open the PR containing the full agent resource.

## Secrets

Secrets live on the platform, never in YAML or chat. If a workflow needs one,
direct the user to enter it from their own terminal with the Auto CLI and
reference it by name:

```bash
read -rsp "SENTRY_TOKEN: " SENTRY_TOKEN
printf %s "$SENTRY_TOKEN" | auto secrets set sentry-token --stdin
unset SENTRY_TOKEN
```

```yaml
env:
  SOME_API_KEY:
    $secret: some-api-key
    optional: true
```

Use `optional: true` for soft dependencies that should degrade rather than block
session launches when missing.
