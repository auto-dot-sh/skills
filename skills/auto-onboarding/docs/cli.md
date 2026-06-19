# The `auto` CLI, as user-facing reference

Hosted onboarding agents operate Auto through Auto MCP tools, not through the
CLI. Use this page only as reference material when you need to explain how the
user can interact with Auto from their own terminal, or when a secret must be
entered without exposing it in Slack. For your own setup, inspection,
connection, resource validation, session inspection, and PR ownership work, use
`docs/auto-mcp.md` and the `mcp__auto__auto_*` tools instead.

Install: `npm install -g @autohq/cli` (Node 20+). Bare `auto` in a terminal opens the TUI dashboard; `auto --help` lists everything. Most commands accept `--json` or a global `--format` for machine-readable output, and `--api-url` to target a non-default API.

## Auth and accounts

```sh
auto auth login                  # sign in (browser flow; device code in headless contexts)
auto whoami                      # show the authenticated actor
auto auth status                 # token/account state
auto auth list                   # stored accounts
auto auth switch                 # pick the active account (interactive)
auto auth switch --user <email>  # switch without the prompt
auto auth logout
```

The CLI stores one account per email+server under `~/.auto/accounts/`, with the active one tracked by `activeAccount`. In CI, set `AUTO_API_TOKEN` to a service-account token instead of logging in (and `AUTO_API_BASE_URL` if not the default).

## Orgs and projects

```sh
auto orgs create             # create an organization (optionally with an initial project)
auto orgs list
auto orgs use <org>          # set the active organization
auto orgs invite <email>
auto projects create
auto projects list
auto projects use <project>  # set the active project — resources and applies are project-scoped
```

Hosted onboarding already starts after the user has an Auto account and project.
Explain these commands only when the user asks how to manage accounts, orgs, or
projects from their own terminal.

## Connections

```sh
auto connections list --available     # providers that can be connected
auto connect <provider>               # start a connection flow (opens browser; --no-browser prints the URL)
auto connections list                 # existing connections and their names
auto allow <provider> [project]   # share a connection with a project (defaults to the active project)
auto connections remove <provider>
```

Connect flows wait (up to ~5 minutes) for the user to finish in the browser;
`--no-wait` is fire-and-forget. During hosted onboarding, start and inspect
connection flows with Auto MCP tools instead; use these commands only to explain
the equivalent user-facing terminal workflow.

## Applying resources

```sh
auto apply --dry-run         # plan only — always run this first
auto apply                   # apply .auto/ (prunes omitted resources)
auto apply -f <file> --no-prune
auto agents connect <agent>     # realize an agent's chat identity (per-workspace bot)
```

During hosted onboarding, agent resource updates should go through PR merge and
GitHub Sync, not agent-run CLI applies. Use this section as background when
explaining what merge-to-apply replaces.

## Inspecting and editing resources

```sh
auto inspect <kind>/<name>   # show a resource spec, e.g. auto inspect agent/code-review
auto edit <kind>/<name>      # edit and re-apply
auto delete <kind>/<name>
```

## Running and interacting

```sh
auto start <agent> [-m <message>] [--attach]   # launch a run; --attach streams it
auto start <agent> --attach [-m <message>]
auto interactive <agent>                     # launch, stream, and type messages
auto attach <session-id>                           # stream an existing run
auto send <session-id> <message>                   # message a live run
auto console <session-id>                          # interactive console on a live run
auto sessions stop <session-id> [--reason <reason>]
```

## Observing sessions

```sh
auto sessions list [--agent <name>] [--status <s>] [--since <iso>] [--limit <n>]
auto sessions show <session-id>                  # lifecycle, timing, behavior summary
auto sessions conversation <session-id> [--tail <n>] [--full]   # snapshot; `auto attach` streams live
auto sessions search <session-id> <query...>     # grep a transcript
auto sessions tools <session-id> [--errors]      # paired tool calls/results with timing
auto sessions triggers <session-id>              # what spawned the session, events delivered
auto sessions artifacts <session-id>             # artifacts the session owns
auto sessions commands <session-id>              # inbound command history
auto sessions archive|unarchive <session-ids...>
```

This is the user's terminal debugging kit. During hosted onboarding, inspect
sessions with the matching Auto MCP tools (`mcp__auto__auto_sessions_list`,
`mcp__auto__auto_sessions_get`, `mcp__auto__auto_sessions_conversation`,
`mcp__auto__auto_sessions_triggers`, and
`mcp__auto__auto_sessions_tools`) instead of running CLI commands yourself.

## Secrets and service accounts

```sh
auto secrets set <name> | list | remove

auto service-account create <name> --preset applier    # or --preset read-only, or repeatable --scope
auto service-account update <name> --scope <scope>
auto service-account rotate <name>
```

When a workflow needs a secret value, ask the user to run the CLI from their own
terminal so the value never appears in Slack:

```sh
read -rsp "SENTRY_TOKEN: " SENTRY_TOKEN
printf %s "$SENTRY_TOKEN" | auto secrets set sentry-token --stdin
unset SENTRY_TOKEN
```

`service-account create` prints the token exactly once — hand it straight to a
CI secret if the user is setting up a separate CI workflow.

## Onboarding entry points

```sh
auto onboard            # human quickstart text
auto onboard --agent    # prints this skill's playbook for coding agents
```

Hosted onboarding agents already receive the mounted onboarding package and
should read it from `/workspace/auto-docs/` rather than asking the user to print
or install the skill.
