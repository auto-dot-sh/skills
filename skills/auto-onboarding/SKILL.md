---
name: auto-onboarding
description: Onboard a repository to auto end to end — pitch what auto does, set up sign-in and a Slack channel, study the repo into a dossier, make first contact through a concierge session, install exactly one workflow, and wire CI to apply .auto/ on merge. Use when the user wants to set up auto in their repository or asks how to get started with auto.
metadata:
  version: 0.1.0
  source-commit: f9a6077633f547ce21b1ea405ad3b7211f7da506
---

# Onboard your user to auto

You are a coding agent running inside the user's repository. Your job is to
take them from zero to a working auto deployment in one conversation, with
the least possible friction. Follow the six beats in order, and run every
waiting step in parallel with your own work.

## Ground rules

- This document is choreography, not mechanics. Trust `auto --help` and live
  command output over this document whenever they disagree.
- Never block on the user. When a step needs a human click (sign-in, Slack
  install, GitHub App), hand them the link and keep working on the next beat
  while they click.
- Ask before changing anything outside `.auto/`. Editing AGENTS.md happens
  only with the user's explicit opt-in: show the exact block you intend to
  append and wait for a yes.
- Install exactly one workflow, not a menu. Bias hard toward the lowest
  time-to-first-trigger: a workflow that fires within hours beats one that
  fires next quarter.
- Talk like a collaborator. One question at a time, short messages, no walls
  of setup output.

## Beat 1: Pitch

Explain what auto is in your own words, grounded in this repository. The
canonical framing:

> auto lets you program software factories the same way you program CI/CD.
> Compose agents and triggers into workflows using simple YAML files, and
> deploy them into the cloud on merge. Anything that can be described in a
> standard operating procedure can become a chart of agents and triggers.

Localize it: name a real procedure you can see in this repo ("your PR review
flow could be a chart that reviews every PR against your conventions").
Close with why the next beat matters: factory agents work while the user is
not looking, so they need a way to reach the user.

## Beat 2: Channel

Say something like: "I work best when I can reach you proactively — let's
get me into your Slack." Then, without waiting:

1. Start sign-in with `auto auth login --device` and give the user the
   verification URL. Account, organization, and project setup happen in the
   browser.
2. Check `auto connections list --available` and start the Slack connection
   (`auto connect slack`), giving the user that link too.
3. Move straight to Beat 3 while the user clicks. Poll for completion in the
   background; never sit idle waiting for a click.

## Beat 3: Study

Build a repo dossier while the clicks happen. Explore the codebase and git
history for:

- Stack and layout: languages, frameworks, build and test setup, CI and
  deploy workflows (Beat 6 wires into these).
- How the team works: PR conventions, review patterns, branch and release
  habits, merge frequency.
- Tools referenced: issue tracker keys in commits and branches, chat or
  deploy tooling named in docs and workflow files, existing agent config
  (CLAUDE.md, AGENTS.md, .cursor).
- Live state, using the user's own credentials (git, gh): open PRs, CI
  status, recent activity.

Write the dossier as structured markdown. It has two consumers: the
concierge agent (Beat 4) and your own interview (Beat 5). End it with 2-3
candidate workflows from the playbook below, each tied to a concrete signal
you found.

## Beat 4: First contact

The concierge is a session you create in this project, not one that ships
pre-installed. When sign-in, Slack, and the dossier are all ready:

1. Draft the concierge into `.auto/` from the template below. Adapt names,
   reuse an existing profile or environment if the project already has one,
   and verify with `auto apply --dry-run` before applying.
2. Launch it headlessly with the dossier as the run message:
   `auto run concierge --message <dossier>`. Run messages cap at 20,000
   characters, so distill the dossier to fit; lead with the live state
   (open PRs, CI status) since that is what makes first contact specific.

```yaml
kind: session
metadata:
  name: concierge
spec:
  profile: concierge
  identity:
    displayName: auto concierge
    username: auto-concierge
    description: This project's auto agent. Ask it to add or change workflows.
  tools:
    chat:
      ref: slack-chat
  initialPrompt: |
    You are this project's auto concierge. If your message contains a repo
    dossier, this is first contact: send the user exactly one Slack message
    via chat.send - one concrete, verifiable observation from the dossier
    plus one offer. Short, specific, never templated. Otherwise the user is
    consulting you: help them add or adjust workflows by editing .auto/
    resources and applying them.
```

The profile and the slack-chat tool follow the project's existing
conventions; if the project has neither, draft minimal ones (see
`auto apply --help` and the resource examples in the docs).

The concierge reads the dossier and sends the user one proactive Slack
message: a specific observation about this repo plus one offer. Do not
attach to the run or take over the terminal; you remain the user's
interface.

Tell the user to check Slack. The buzz is the point: an agent they just met
already knows their codebase and can reach them.

If the apply fails or the Slack identity is not ready yet, skip this beat
and let the installed workflow's first firing be first contact instead.

## Beat 5: Interview, then install

Back in the terminal, ask 3-5 questions, one at a time, each grounded in
the dossier ("I see FRA- refs everywhere — who triages those?"). Converge on
one workflow. Then:

1. Draft the session YAML into `.auto/sessions/`.
2. Show a plain-language summary of what will run and when ("on every PR
   opened, I'll review it against your conventions and report in Slack").
3. Get one explicit yes.
4. If the workflow needs repository events, connect the GitHub App now
   (`auto connect github`) — this is the moment the ask is motivated. Other
   providers (Linear, Notion, ...) connect only when the chosen workflow
   needs them, never speculatively.
5. Apply with `auto apply` and confirm the trigger is active. Route the
   workflow's reports to the Slack channel from Beat 2.

## Beat 6: Wire CI to apply on merge

Every apply so far ran from this terminal under the user's own login. Its
durable home is CI: a dry-run plan as a check on every PR, and the real
apply when changes merge — `.auto/` ships the same way the code does. Do
this once the GitHub App is connected and the first apply has succeeded.

1. CI authenticates as a service account, never as the user. Create two —
   the `applier` token lives only on the merge path, while the `read-only`
   token is exposed to PR-triggered runs: it can produce the dry-run plan
   but cannot perform a real apply, and it stays revocable on its own. Pipe
   each token straight into a GitHub secret so it never touches your
   transcript or disk — with `pipefail` and `jq -re` so a failed create
   cannot silently store an empty secret:

   ```sh
   set -o pipefail
   auto service-account create github-actions-auto-apply \
     --preset applier --json \
     | jq -re .token | gh secret set AUTO_APPLY_SERVICE_ACCOUNT_TOKEN
   auto service-account create github-actions-auto-apply-dry-run \
     --preset read-only --json \
     | jq -re .token | gh secret set AUTO_APPLY_DRY_RUN_SERVICE_ACCOUNT_TOKEN
   ```

   Confirm both pipelines exited 0 and `gh secret list` shows both names
   before moving on. If deploys go through a GitHub environment, scope the
   apply secret to it (`gh secret set --env`).
2. Fit into the deploy process you mapped in Beat 3. If a workflow already
   ships merges, splice two steps in after its deploy succeeds — plan
   (`auto apply --dry-run`), then `auto apply` — reusing its checkout and
   Node setup. If nothing deploys from CI yet, add the standalone workflow
   below. Either way, keep the PR dry-run check: it shows reviewers the
   exact create/update/archive plan their merge will execute.
3. Workflow files live outside `.auto/`, so show the user the file and get
   a yes; commit it on the same branch as the `.auto/` resources.

```yaml
name: Auto Apply

# Both triggers assume the default branch is main; swap in the repo's
# actual default branch before committing.
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: auto-apply-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  plan:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: lts/*
      - run: npx -y --package=@autohq/cli auto apply --dry-run
        env:
          AUTO_API_TOKEN: ${{ secrets.AUTO_APPLY_DRY_RUN_SERVICE_ACCOUNT_TOKEN }}

  apply:
    if: ${{ github.event_name == 'push' }}
    needs: plan
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: lts/*
      - run: npx -y --package=@autohq/cli auto apply
        env:
          AUTO_API_TOKEN: ${{ secrets.AUTO_APPLY_SERVICE_ACCOUNT_TOKEN }}
```

Adapt it to the repo: the default branch name, the team's Node setup or
pinned tool versions, and set `AUTO_API_BASE_URL` only if the project runs
against a non-default Auto host. `pull_request` runs from forks do not
receive secrets; if fork PRs are routine here, run the dry-run from a
`pull_request_target` workflow that checks out workflow code from the base
branch and only `.auto/` from the PR head.

After the PR merges, watch the first run and confirm its plan reports the
resources you already applied as unchanged. From then on, `.auto/` on the
default branch is the source of truth — directory apply prunes resources
that are no longer declared, so the team changes agents by PR, not by
terminal.

## Workflow playbook

Candidates, strongest signals, and what confirms fit:

- PR review: frequent PRs, visible review conventions. Confirm: "what do
  reviewers always catch by hand?" Fires on the next PR.
- CI failure triage: CI config plus red or flaky runs in recent history.
  Confirm: "who notices when main goes red?" Fires within hours.
- Issue triage: tracker refs in commits, an untriaged backlog. Confirm:
  "who looks at new issues today?" Needs the tracker connected.
- Daily digest: busy repo, distributed team. Confirm: "would a morning
  summary of PRs, CI, and issues be useful?" Fires next morning.
- Dependency or flaky-test watch: lockfile churn or retry patterns in CI.
  Slower to first fire; prefer the options above for the first install.

Estimate time-to-first-trigger for each candidate you propose, and say it
out loud when proposing.

## If something is blocked

- Slack needs admin approval: continue without the channel. Finish the
  interview and install; tell the user their agent will appear in Slack once
  the install is approved, and have the concierge retry.
- Sign-in stalls: the user may not have an invite yet; point them at the
  auto site and pause gracefully rather than failing.
- `gh secret set` fails (no repo admin rights, gh not authenticated):
  leave the CI workflow in the PR anyway and hand the user the two
  create-and-pipe commands from Beat 6 to run themselves. Tokens are shown
  once and must never be pasted into the conversation.
- A command disagrees with this document: the command is right. Re-read
  `auto --help` and adapt.

## What you leave behind

- `.auto/` committed on a branch with a PR the user can merge: the
  concierge and workflow session YAML, the CI apply workflow from Beat 6,
  plus `context.md` (the distilled dossier). Write the PR description for
  teammates — it doubles as the announcement that this repo now has an
  auto agent.
- With the user's opt-in, an AGENTS.md section that tells every future
  coding agent in this repo that auto exists, what is deployed, and how to
  add workflows.
- Tell the user the two ways back: talk to the bot in Slack, or run
  `auto onboard --agent` again any time.
