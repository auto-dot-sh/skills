# GitHub Sync

The end state: `.auto/` is committed to the user's repo, pull requests show an
Auto Sync plan, and merging to the default branch applies the committed
resources automatically. The repo becomes the source of truth for the factory.

Do not add a hand-written GitHub Actions workflow for `auto apply` during normal
onboarding. GitHub Sync is the default deployment path.

## 1. Bind the repo to the project

Use the product setup flow or CLI surface that creates a GitHub Sync binding for
the user's project and repository. The binding is what lets Auto observe merged
`.auto/` changes and apply them without storing a service-account token in the
repo.

Before opening a resource PR, validate the intended resources with
`auto apply --dry-run` or the local dry-run tool exposed to the onboarding
agent.

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

## Legacy manual apply workflows

Only use a GitHub Actions `auto apply` workflow if current product behavior or
the user explicitly requires a legacy setup. In that case, use service accounts
rather than user tokens, keep PR planning read-only, and never ask the user to
paste tokens into chat.
