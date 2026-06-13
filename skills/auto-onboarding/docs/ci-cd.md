# CI/CD: apply on merge

The end state: `.auto/` is committed to the user's repo, pull requests show an apply *plan*, and merging to the default branch *applies* it. The repo becomes the source of truth for the factory.

## 1. Create service accounts

```sh
auto service-account create ci-apply --preset applier
auto service-account create ci-dry-run --preset read-only   # optional, for plan-on-PR
```

Each prints its token exactly once. Have the *user* run these commands in their own terminal and add the tokens as GitHub Actions secrets — `AUTO_APPLY_TOKEN` and (optionally) `AUTO_DRY_RUN_TOKEN` — in the repo's Settings → Secrets and variables → Actions. Tokens must never be pasted into the conversation; an agent running the create command puts the token in its own transcript. Use two accounts so the PR-facing job holds only read-only credentials.

## 2. Add the workflow

`.github/workflows/auto-apply.yml`:

```yaml
name: Auto apply

on:
  pull_request:
    paths: ['.auto/**']
  push:
    branches: [main]
    paths: ['.auto/**']

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '24'
      - run: npm install -g @autohq/cli
      - run: auto apply --directory .auto --dry-run
        env:
          AUTO_API_TOKEN: ${{ secrets.AUTO_DRY_RUN_TOKEN }}

  apply:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: read
    concurrency:
      group: auto-apply
      cancel-in-progress: false
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '24'
      - run: npm install -g @autohq/cli
      - run: auto apply --directory .auto
        env:
          AUTO_API_TOKEN: ${{ secrets.AUTO_APPLY_TOKEN }}
```

Notes:

- `branches: [main]` is a placeholder — swap in the repo's actual default branch before copying.
- The `paths` filter keeps the workflow quiet on unrelated changes; drop it if the user wants the plan on every PR. Don't mark the `plan` job as a required status check while the filter is in place — PRs that don't touch `.auto/` would hang on a check that never reports.
- If the user skips the read-only account, also drop the `plan` job — otherwise every `.auto/`-touching PR fails on the missing `AUTO_DRY_RUN_TOKEN` secret.
- The apply job's `concurrency` group serializes applies so two merges can't race.
- Remember directory apply **prunes**: deleting a file from `.auto/` and merging archives that resource. That's the desired GitOps behavior, but say it out loud to the user.
- Security hardening for busy repos (fork PRs, environment-scoped secrets, deployment branch policies) is worth a follow-up conversation but not required for a first setup.

## 3. Ship it

Open a PR containing `.auto/` and the workflow, have the user add the secrets and merge, then verify the apply job succeeded in the Actions tab (and `auto inspect` shows the expected resources). From this point on, prefer PRs over direct `auto apply` — model the workflow you just installed.
