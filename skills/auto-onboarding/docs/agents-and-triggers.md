# Agents and triggers

An agent is the runnable unit of a workflow: its standing prompt, what it knows
at start, what it can touch, an optional chat identity, and the triggers that
start it or steer it.

## Agent spec

```yaml
name: pr-review
imports:
  - ../fragments/environments/agent-runtime.yaml      # reusable harness + environment fragment
systemPrompt: |                      # durable persona and rules
  You are the PR review agent for acme/widgets.
identity:                            # chat presence; always include one in onboarding-authored agents
  displayName: PR Review             # <=80 chars
  username: pr-review                # <=80 chars; @mention handle
  avatar:
    asset: .auto/assets/pr-reviewer.png
  description: Reviews each PR and posts a merge recommendation. # <=140 chars as Slack counts them
initialPrompt: |                     # optional, <=20k chars; event-payload templating
  Review pull request #{{github.pullRequest.number}} ...
env:                                 # optional env vars for the sandbox
  MY_API_KEY:
    $secret: my-api-key
mounts:                              # optional git checkouts
  - kind: git
    repository: acme/widgets
    mountPath: /workspace/widgets
    ref: main                        # or refs/pull/{{payload.github.pullRequest.number}}/head
    depth: 1
    auth:
      kind: githubApp
      capabilities:
        contents: read
        pullRequests: write
        issues: write
        checks: none
        actions: read
        workflows: none
workingDirectory: /workspace/widgets
tools:                               # alias -> inline local or remote MCP definition
  chat:
    kind: local
    implementation: chat
    auth:
      kind: connection
      provider: slack
      connection: slack
  notion:
    kind: mcp_remote
    url: https://mcp.notion.com/mcp
    auth:
      kind: mcp_oauth
      connection: notion
triggers: []                         # see below
```

Notes:

- **`initialPrompt` is per-run context; `systemPrompt` is standing orders.** Put the durable persona and rules in the agent's system prompt, and the event-specific task in the initial prompt. Triggered sessions render placeholders from the event payload by its top-level keys — `{{github.pullRequest.number}}`, `{{chat.channelId}}`, `{{message.text}}` — with no `payload.` prefix. (That prefix is only for mount `ref` templates, which render against the run-input wrapper.)
- **Mount capabilities are the permission boundary.** A `githubApp` mount mints installation tokens scoped to exactly the capabilities you declare (`none`/`read`/`write` for contents, pullRequests, issues, checks, actions, workflows). The brokered GitHub MCP tools and git pushes both run inside that scope. A reviewer that should never push gets `contents: read`.
- **`commitAuthor`** on a mount sets the bot author for commits the agent pushes.
- **Identity makes the agent a first-class chat persona.** With an `identity:`, GitHub Sync and the connected Slack workspace realize an @mentionable agent handle such as `@auto.pr-review`; mentions of that handle route to this agent.
- **Onboarding-authored agents should be Slack-capable by default.** Give each
  new agent a Slack-backed local `chat` tool. If Slack is only a backstop for
  discovery or smoke tests, add a direct mention trigger that handles clear
  role-appropriate requests or asks for missing context, and only falls back to
  a short hello/explanation when the mention is casual or unclear. If the agent
  is meant to be tagged in Slack during normal operations, its mention trigger
  should run that normal flow instead.

## Triggers

Each trigger names one or more events, an event source, optional filters, an optional message template, and a routing decision.

```yaml
triggers:
  - event: chat.message.mentioned
    connection: slack
    where:
      $.chat.provider: slack
      $.auto.authored: false
    message: |
      {{message.author.userName}} mentioned you on Slack:

      {{message.text}}

      Channel: {{chat.channelId}}
      Thread: {{chat.threadId}}

      Reply in that thread with chat.send. If the message contains a clear
      request that fits your normal role, handle it or ask for the missing
      required context. Otherwise, briefly explain what you do.
    routing:
      kind: spawn
  - events:                          # or singular: event: <name>
      - github.pull_request.opened
      - github.pull_request.synchronize
    connection: github-acme          # the provider connection that emits the events
    where:                           # filters; all must match
      $.github.repository.fullName: acme/widgets
    routing:
      kind: spawn
```

### Event sources

- **Provider connections** (`connection: <name>`): GitHub App installations, Slack/Linear/Telegram grants. The connection name comes from `mcp__auto__auto_connections_list`.
- **Custom webhooks** (`endpoint: <name>` + `auth`): the apply response returns an ingest URL per endpoint; POST JSON to it to emit the event.

  ```yaml
  - event: webhook.incident.opened
    endpoint: incident-webhook
    auth:
      kind: bearer_token             # or hmac_sha256, or none
      secretRef: incident-webhook-secret
    routing:
      kind: spawn
  ```

- **Heartbeats** (cron):

  ```yaml
  - kind: heartbeat
    cron: "0 8 * * *"
    timezone: America/Los_Angeles    # default UTC
    routing:
      kind: spawn
  ```

### Event catalog

| Family | Events |
| --- | --- |
| GitHub PRs | `github.pull_request.opened` / `.reopened` / `.synchronize` / `.edited`, `github.pull_request.merge_conflict` |
| GitHub conversation | `github.issue_comment.created` / `.edited`, `github.pull_request_review.submitted` / `.edited`, `github.pull_request_review_comment.created` / `.edited` |
| GitHub CI | `github.check_run.completed`, `github.push` |
| Linear | `linear.issue.created`, `linear.issue.updated` |
| Chat (Slack/Telegram/Linear) | `chat.message.mentioned`, `chat.message.direct`, `chat.message.subscribed`, `chat.reaction.added`, `chat.reaction.removed` |
| Custom | any `webhook.*` name you choose on an `endpoint:` trigger |
| Scheduled | `kind: heartbeat` with `cron` |

### Filters (`where:`)

Keys are payload paths (`$.github.repository.fullName`) or bare keys; values are either a scalar to match exactly or one operator object:

```yaml
where:
  $.github.repository.fullName: acme/widgets          # equality
  $.linear.issue.labelNames: { contains: auto-triage } # array membership
  $.auto.attributions: { exists: false }               # presence
  $.github.sender.login: { notIn: [some-bot] }         # / in: [...]
  $.github.auto.mentioned: { changedTo: true }         # edge-trigger on edits
```

Useful payload facts the platform adds under `$.auto.*`:

- `$.auto.authored` — the event was caused by an auto agent (filter `false` to avoid self-trigger loops).
- `$.auto.attributions` — the event is attributed to an existing run (a reply in a thread an agent participates in).

### Routing

```yaml
routing:
  kind: spawn                        # always start a new run

routing:
  kind: deliver                      # send the event into an existing run
  routeBy:
    kind: attributedSessions         # the session(s) attributed to this conversation
  onUnmatched: drop                  # or warn | error

routing:
  kind: deliverOrSpawn               # deliver to the singleton run, else spawn one
  routeBy:
    kind: singleton
```

`routeBy` options for `deliver`:

- `singleton` — the agent's one live run.
- `ownedArtifact` + `artifactType` (e.g. `github.pull_request`) — the run that recorded ownership of the artifact via `auto.artifacts.record`. This is how check failures, merge conflicts, and review comments on a PR route back to the run that owns it.
- `attributedSessions` — sessions attributed to the conversation (the agent's own messages and subscribed threads). This is how chat replies continue an existing conversation.
- `allLiveRuns` — broadcast to every live run of the agent.

Deliver triggers should carry a `message:` template (same `{{...}}` placeholders) framing the event for the in-flight agent — include `{{message.text}}` for chat events so the human's actual words arrive.

### The two attribution rules (validation will enforce them)

1. A chat-message `deliver`/`attributedSessions` trigger must filter `$.auto.authored: false`, or the agent will react to its own messages.
2. When the same chat event has both a `spawn` trigger and an `attributedSessions` deliver trigger, they must be mutually exclusive on `$.auto.attributions`: spawn requires `{ exists: false }`, deliver requires `{ exists: true }`. Fresh mentions start sessions; thread replies continue them.

The complement on the output side: agents posting to GitHub append a hidden attribution marker so the platform can recognize their own side effects:

```
<!-- auto:v=1 run_id=$AUTO_RUN_ID agent=$AUTO_AGENT_NAME -->
```

Build that instruction into any agent prompt that comments on GitHub.

### PR checks

Triggers on `github.pull_request.*` events may declare GitHub check runs the agent reports through the `checks.*` tools:

```yaml
checks:
  - name: pr-review
    displayName: Auto PR review
    description: Auto reviews this pull request.
    instructions: |
      Call checks.begin with { "name": "pr-review" } before anything else.
      Call checks.success / checks.failure with a summary when done.
    beginTimeout: { seconds: 1200, conclusion: failure }
    completeTimeout: { seconds: 1200, conclusion: failure }
```

The platform creates the check on the PR head; the agent moves it through begin → success/failure; the timeouts conclude it if the agent never does (`timeout` is an accepted alias for `beginTimeout` — use one or the other, not both). This is how an agent workflow becomes a required status check.
