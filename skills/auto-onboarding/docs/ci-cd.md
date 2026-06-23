# GitHub Sync

The end state: `.auto/` is committed to the user's repo, pull requests show an
Auto Sync plan, and merging to the default branch applies the committed
resources automatically. The repo becomes the source of truth for the factory.

Use merge-to-apply for updating agent resources: open a PR with `.auto/`
changes, let the user merge it, then verify GitHub Sync applied the resources.
Use Auto MCP connection tools for updating provider grants and MCP tool
connections; those are platform connection state, not `.auto/` resource
changes.

## 1. Bind the repo to the project

Hosted onboarding starts with the user's GitHub repository already connected to
the Auto project. Identify it from the mounted checkout and
`git remote get-url origin`; do not ask the user which repo this is unless they
explicitly ask to switch repos. The GitHub Sync binding is what lets Auto
observe merged `.auto/` changes and apply them without storing a service-account
token in the repo.

Before opening a resource PR, validate the intended resources with
`mcp__auto__auto_resources_dry_run`.

When the user's repository has required GitHub Actions checks, ask whether those
workflow files should be watchdogged by GitHub Sync. A check-run trigger only
reacts after a check event exists; it does not imply that Auto should create or
re-run CI. The CI watchdog is an explicit Sync binding setting for workflows
that must produce a PR-head run and can be dispatched with GitHub's
`workflow_dispatch`.

Configure it when enabling or updating Sync with `mcp__auto__auto_sync_enable`:

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

Add one `workflows` entry for each required GitHub Actions workflow that Auto
should rescue when GitHub misses a PR-head run. Do not configure it for external
checks such as Vercel or CodeQL app checks, or for deploy/release workflows that
should not run on PR heads. The equivalent human CLI flow is
`auto sync enable github acme/widgets --connection github-acme --branch main --ci-watchdog-workflow ci.yml`.

## 2. Open the resource PR

Open a focused PR containing the `.auto/` files. Ask the user to review and
merge it when ready. Treat the sync plan comment/check as the source of truth for
what will change after merge.

Directory apply prunes omitted resources, so deleting a file from `.auto/` and
merging archives that resource. Say that out loud when the PR removes resources.

## 3. Verify Sync after merge

After merge, verify that GitHub Sync applied the resources by inspecting Auto
resource/session state, not GitHub Actions logs. For a smoke test, trigger the
new workflow and watch the resulting session until the user sees the expected
output.
