# The Resource Model and GitHub Sync

## Directory layout

GitHub Sync reads the committed `.auto/` directory after merge and treats it as
the authoritative declaration of the project's resources:

```
.auto/
  agents/         # complete agent YAML files
  fragments/      # partial YAML imported by agents; not applied directly
  assets/         # avatar images referenced by agent identities
```

Each YAML file under `.auto/agents/` is an agent definition in the root-level
agent facade format. Fragments under `.auto/fragments/` are imported by agents
and are not applied by themselves.

## Agent YAML

```yaml
name: pr-review            # required: 1-128 chars of [A-Za-z0-9_.-]
labels:                    # optional string map
  purpose: pr-review
imports:
  - ../fragments/environments/agent-runtime.yaml
identity:                  # optional @mentionable persona
  displayName: PR Review
  username: pr-review
  avatar:
    asset: .auto/assets/pr-review.png
systemPrompt: |
  You are the code review agent for acme/widgets.
initialPrompt: |
  Review PR #{{github.pullRequest.number}}.
```

Imported fragments can define reusable fields like `harness`, `environment`,
`identity`, tools, mounts, or triggers. Inline identity and environment objects
are materialized into generated backend resources during apply, and the compiled
agent references those generated resources by name. Reused runtimes should live
in fragment imports rather than repeated in every agent.

## Sync semantics

Use `mcp__auto__auto_resources_dry_run` to validate drafted resources before a
PR. The dry-run result reports which resources will be created, updated,
unchanged, or archived when GitHub Sync applies the merged `.auto/` directory.

Habits that keep sync safe:

- **Always dry-run first.** Show the plan to the user before opening or updating the PR.
- **Merged directory state is authoritative.** If a resource exists on the platform but no longer exists in `.auto/`, GitHub Sync archives it. Say that out loud when a PR removes resources.
- **Sync is idempotent.** Re-syncing an unchanged directory reports every resource `unchanged`.

The dry-run/sync response also includes **trigger receipts** for webhook triggers
— each `endpoint:` trigger gets an ingest URL you can POST events to (see
`agents-and-triggers.md`).

## Secrets

Secrets live on the platform, never in YAML or chat. Have the user enter secret
values from their own terminal with the Auto CLI, then reference only the secret
name in resources:

```sh
read -rsp "SENTRY_TOKEN: " SENTRY_TOKEN
printf %s "$SENTRY_TOKEN" | auto secrets set sentry-token --stdin
unset SENTRY_TOKEN
```

Resources reference them with `$secret`:

```yaml
# in an agent env:
env:
  SOME_API_KEY:
    $secret: my-api-key
    optional: true   # missing secret omits the var instead of failing the run

# in a tool spec.auth:
auth:
  kind: bearer
  token:
    $secret: my-mcp-token
```

`optional: true` is for soft dependencies — removing the secret degrades the workflow rather than breaking run launches and apply validation.

## Avatar assets

An inline agent identity may declare an avatar. The authored identity is compiled
into a generated identity resource, and GitHub Sync validates the asset through
that generated resource:

```yaml
name: pr-review
identity:
  displayName: PR Review
  avatar:
    asset: .auto/assets/pr-review.png
```

Rules: the path must be relative, under `.auto/assets/`, end in `.png`/`.jpg`/`.jpeg`, be ≤2MB, and be a square between 512x512 and 2000x2000 pixels (Slack's app-icon constraints — the strictest provider). GitHub Sync validates, uploads, and content-addresses the image; the platform serves it publicly and stamps agent messages with it.
