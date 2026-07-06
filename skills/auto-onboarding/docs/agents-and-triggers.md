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
displayTitle: "Review PR #{{github.pullRequest.number}}: {{github.pullRequest.title}}" # optional template, or infer
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

- **`initialPrompt` is per-run context; `systemPrompt` is standing orders.** Put the durable persona and rules in the agent's system prompt, and the event-specific task in the initial prompt. Triggered sessions render `initialPrompt` and `displayTitle` placeholders from the event payload by its top-level keys — `{{github.pullRequest.number}}`, `{{chat.channelId}}`, `{{message.text}}` — with no `payload.` prefix. (That prefix is only for mount `ref` templates, which render against the run-input wrapper.)
- **`displayTitle` controls session list titles.** Set it to a deterministic template when the event has stable identifying fields, or to `infer` when the generated-title path is better.
- **Mount capabilities are the permission boundary.** A `githubApp` mount mints installation tokens scoped to exactly the capabilities you declare (`none`/`read`/`write` for contents, pullRequests, issues, checks, actions, workflows, secrets). The brokered GitHub MCP tools and git pushes both run inside that scope. A reviewer that should never push gets `contents: read`. `secrets: write` unlocks the `actions_secret_write` tool for provisioning GitHub Actions repository/environment secrets (and `secrets: read` unlocks `actions_secret_list`); like merge, these stay invisible without the grant.
- **`commitAuthor`** on a mount sets the fallback git author and committer for
  commits the agent pushes. When a session requester resolves to a GitHub
  identity, the requester becomes the primary author and `commitAuthor` becomes
  the agent `Co-authored-by` trailer.
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
      $.auto.authored: false   # chat events: top-level $.auto.authored (GitHub uses $.github.auto.authored)
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

- **Provider connections** (`connection: <name>`): GitHub App installations, Slack/Linear/Telegram grants. The connection name comes from `mcp__auto__auto_connections_list`. Add `optional: true` to a connection-bound trigger to skip it (instead of failing the apply) while its connection has no active grant — the apply succeeds, emits a non-blocking info notice naming the connection to set up, and activates the trigger on the next apply/sync once the connection exists.
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
| GitHub issues | `github.issue.opened` / `.edited` |
| GitHub issue comments | `github.issue.comment.created` / `.edited` |
| GitHub PR conversation | `github.issue_comment.created` / `.edited`, `github.pull_request_review.submitted` / `.edited`, `github.pull_request_review_comment.created` / `.edited` |
| GitHub CI | `github.check_run.completed`, `github.push` |
| Linear | `linear.issue.created`, `linear.issue.updated` |
| Chat (Slack/Telegram/Linear) | `chat.message.mentioned`, `chat.message.direct`, `chat.message.subscribed`, `chat.reaction.added`, `chat.reaction.removed` |
| Custom | any `webhook.*` name you choose on an `endpoint:` trigger |
| Scheduled | `kind: heartbeat` with `cron` |

PR comments use `github.issue_comment.*`; plain issue comments use `github.issue.comment.*`.
GitHub issue payloads expose `{{github.issue.number}}`,
`{{github.issue.htmlUrl}}`, and `{{github.issue.title}}` for prompt and
trigger message templates.

**Self-trigger loops on GitHub.** GitHub events do **not** carry a top-level
`$.auto.authored` flag — that path exists only on chat events. On GitHub
events the platform nests the same flag under `$.github.auto.authored`, so a
`where: { $.auto.authored: false }` filter on a `github.issue.*` /
`github.issue_comment.*` trigger never matches and silently drops real events.
Filter `$.github.auto.authored: false` to skip events your own agent caused.
To also suppress third-party bot noise (status bots, CI bots) while still
matching human and Auto-authored events, filter
`$.github.auto.externalBot: false`.

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

Useful payload facts the platform adds under `$.auto.*` (chat events) and
`$.github.auto.*` (GitHub events):

- `$.auto.authored` (chat) / `$.github.auto.authored` (GitHub) — the event was caused by an auto agent. Filter `false` to avoid self-trigger loops. A GitHub trigger filtered on the top-level `$.auto.authored` never matches and silently drops real events.
- `$.github.auto.externalBot` (GitHub) — the author is a bot other than Auto's own GitHub App. Filter `false` to drop third-party bot noise (status bots re-editing their comments, CI bots) while still matching human and Auto-authored events.
- `$.auto.attributions` — the event is attributed to an existing run (a reply in a thread an agent participates in).

### Routing

```yaml
routing:
  kind: spawn                        # always start a new run

routing:
  kind: deliver                      # send the event into an existing run
  routeBy:
    kind: attributedSessions         # the session(s) attributed to this conversation
  onUnmatched: drop                  # or warn | error | spawn

routing:
  kind: deliver                      # slot-member delivery for a concurrency-capped agent
  onUnmatched: spawn                 # spawn the member when none is live

routing:
  kind: bind                         # continue the session bound to the event's target
  target: github.pull_request        # or github.issue | slack.thread | agent.singleton
  onUnmatched: drop                  # or warn | error | spawn
```

An agent can cap its live sessions with the `concurrency: 1` spec field (only
`1` is supported). A `deliver` trigger with no `routeBy` resolves the capped
agent's one slot member, and `onUnmatched: spawn` turns an unmatched delivery
into a fresh session carrying the event — legal on any deliver/bind trigger,
bounded agent or not. `replace: auto` (with an `onReplace` rebuild prompt)
opts the agent into automatic replacement on spec drift or failure.
(`deliverOrSpawn`, `routeBy: singleton`, and `routeBy: ownedArtifact` are
legacy sugar — for `deliver` + `onUnmatched: spawn`, a routeBy-less `deliver`,
and `bind` over the same target respectively; parsing normalizes them and they
must not be authored in new specs.)

`bind` resolves the event's target (`github.pull_request`, `github.issue`,
`slack.thread`, `agent.singleton`) to the one session bound to it, written via
`auto.bind` or bind-at-spawn. This is how check failures, merge conflicts, and
review comments on a PR route back to the run that owns it, and how follow-up
comments on an issue route back to the run that owns it.

`routeBy` options for `deliver`:

- omitted — the capped agent's one live slot member.
- `attributedSessions` — sessions attributed to the conversation (the agent's own messages and subscribed threads). This is how chat replies continue an existing conversation.
- `allLiveSessions` — broadcast to every live session of the agent.

Deliver triggers should carry a `message:` template (same `{{...}}` placeholders) framing the event for the in-flight agent — include `{{message.text}}` for chat events so the human's actual words arrive.

### The two attribution rules (validation will enforce them)

1. A chat-message `deliver`/`attributedSessions` trigger must filter `$.auto.authored: false`, or the agent will react to its own messages. (Chat events only — GitHub events use `$.github.auto.authored: false` for the same purpose; the top-level path does not exist on GitHub payloads.)
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
