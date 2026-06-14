# The resource model and `auto apply`

## Directory layout

`auto apply` (with no flags) reads `./.auto` and treats it as the authoritative declaration of the project's resources:

```
.auto/
  environments/   # kind: environment
  identities/     # kind: identity
  sessions/       # kind: session
  assets/         # avatar images referenced by session identities
```

Each YAML file under a kind directory must contain resources of that kind (a file may hold multiple `---`-separated documents). Resources are applied in dependency order: environments, then identities, then sessions.

## The resource envelope

```yaml
kind: <environment | identity | session>
metadata:
  name: my-resource        # required: 1-128 chars of [A-Za-z0-9_.-]
  labels:                  # optional string map
    purpose: example
  annotations: {}          # optional string map
spec:
  # kind-specific, validated strictly — unknown fields are rejected
```

Names are the reference currency: sessions reference identities and environments by name, while inline tools and triggers reference connections by name.

## Apply semantics

```sh
auto apply --dry-run        # print the plan (create/update/unchanged/archive) without changing anything
auto apply                  # make the platform match .auto/ — omitted resources are pruned
auto apply --no-prune       # apply without archiving omitted resources
auto apply -f path.yaml     # apply a single file (use with --no-prune for narrow experiments)
auto apply --directory dir  # apply a different directory
auto apply --json           # machine-readable response
```

Habits that keep applies safe:

- **Always dry-run first.** The plan output shows exactly what will be created, updated, or archived. Show it to the user before a real apply.
- **Directory apply prunes.** If a resource exists on the platform but not in `.auto/`, a full-directory apply archives it. Use `-f <file> --no-prune` when experimenting so you don't clobber the rest of the project.
- **Apply is idempotent.** Re-applying an unchanged directory reports every resource `unchanged`.

The apply response also includes **trigger receipts** for webhook triggers — each `endpoint:` trigger gets an ingest URL you can POST events to (see `sessions-and-triggers.md`).

## Secrets

Secrets live on the platform, never in YAML. Set them with the CLI:

```sh
auto secrets set my-webhook-secret      # prompts for the value; never echoes it
auto secrets list
auto secrets remove my-webhook-secret
```

Resources reference them with `$secret`:

```yaml
# in an environment or session spec.env:
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

An identity resource may declare an avatar:

```yaml
kind: identity
metadata:
  name: pr-review
spec:
  displayName: PR Review
  avatar:
    asset: .auto/assets/pr-review.png
```

Rules: the path must be relative, under `.auto/assets/`, end in `.png`/`.jpg`/`.jpeg`, be ≤2MB, and be a square between 512x512 and 2000x2000 pixels (Slack's app-icon constraints — the strictest provider). `auto apply` validates, uploads, and content-addresses the image; the platform serves it publicly and stamps agent messages with it.
