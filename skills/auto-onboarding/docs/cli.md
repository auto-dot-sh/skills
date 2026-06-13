# The `auto` CLI

Install: `npm install -g @autohq/cli` (Node 20+). Bare `auto` in a terminal opens the TUI dashboard; `auto --help` lists everything. Most commands accept `--json` or a global `--format` for machine-readable output, and `--api-url` to target a non-default API.

## Auth and profiles

```sh
auto auth login              # sign in (browser flow; device code in headless contexts)
auto whoami                  # show the authenticated actor
auto auth status             # token/profile state
auto auth list               # stored account profiles
auto auth switch <profile>   # change the active profile
auto auth logout
```

The CLI stores named account profiles; `--profile <name>` selects one per invocation. In CI, set `AUTO_API_TOKEN` to a service-account token instead of logging in (and `AUTO_API_BASE_URL` if not the default).

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

Ask the user what to name their org and project; don't invent names for them.

## Connections

```sh
auto connections list --available     # providers that can be connected
auto connect <provider>               # start a connection flow (opens browser; --no-browser prints the URL)
auto connections list                 # existing connections and their names
auto allow <provider> [project]   # share a connection with a project (defaults to the active project)
auto connections remove <provider>
```

Connect flows wait (up to ~5 minutes) for the user to finish in the browser; `--no-wait` is fire-and-forget.

## Applying resources

```sh
auto apply --dry-run         # plan only — always run this first
auto apply                   # apply .auto/ (prunes omitted resources)
auto apply -f <file> --no-prune
auto apply --connect         # then walk through OAuth for mcp_oauth tools
auto tools connect <tool>    # OAuth a remote MCP tool (--manual prints the URL)
auto sessions connect <session>   # realize a session's chat identity (per-workspace bot)
```

## Inspecting and editing resources

```sh
auto inspect <kind>/<name>   # show a resource spec, e.g. auto inspect session/code-review
auto edit <kind>/<name>      # edit and re-apply
auto delete <kind>/<name>
```

## Running and interacting

```sh
auto run <session> [-m <message>] [--attach]   # launch a run; --attach streams it
auto run-and-attach <session> [-m <message>]
auto interactive <session>                     # launch, stream, and type messages
auto attach <run-id>                           # stream an existing run
auto send <run-id> <message>                   # message a live run
auto console <run-id>                          # interactive console on a live run
auto runs stop <run-id> [--reason <reason>]
```

## Observing runs

```sh
auto runs list [--session <name>] [--status <s>] [--since <iso>] [--limit <n>]
auto runs show <run-id>                  # lifecycle, timing, behavior summary
auto runs conversation <run-id> [--tail <n>] [--full]   # snapshot; `auto attach` streams live
auto runs search <run-id> <query...>     # grep a transcript
auto runs tools <run-id> [--errors]      # paired tool calls/results with timing
auto runs triggers <run-id>              # what spawned the run, events delivered
auto runs artifacts <run-id>             # artifacts the run owns
auto runs commands <run-id>              # inbound command history
auto runs archive|unarchive <run-ids...>
```

This is your debugging kit during onboarding: `runs list` to confirm a trigger fired, `attach` to stream progress live (`runs conversation` for a snapshot), `runs triggers` when a trigger didn't match, `runs tools --errors` when a tool misbehaves.

## Secrets and service accounts

```sh
auto secrets set <name> | list | remove

auto service-account create <name> --preset applier    # or --preset read-only, or repeatable --scope
auto service-account update <name> --scope <scope>
auto service-account rotate <name>
```

`service-account create` prints the token exactly once — hand it straight to a CI secret (see `ci-cd.md`).

## Onboarding entry points

```sh
auto onboard            # human quickstart text
auto onboard --agent    # prints this skill's playbook for coding agents
```
