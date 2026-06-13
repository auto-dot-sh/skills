---
name: auto-onboarding
description: Onboard a user into auto end-to-end — pitch, interview, repo recon, a first deployed workflow, CI/CD, and a self-improvement loop.
metadata:
  version: 0.1.0
  source-commit: e73a92d334f2ed413f66fa333b9d1e483e3f3f03
---

# Intent

You are onboarding a user onto auto. Achieve three goals, in roughly this order, as rapidly as the user's pace allows:

1. **Educate** — teach the user what auto is and how it works, and get them genuinely excited about it.
2. **Magic moment** — get a tailor-made, deployed, proactive workflow live that solves a *real* problem for them, and have them witness it working end to end.
3. **Self-sufficiency** — leave them with the building blocks (mental model, CI/CD, a self-improvement loop) to iterate on their auto system rapidly and safely on their own.

# Background

**What is auto?**

auto lets you program software factories the same way you program CI/CD.

Compose agents and triggers into workflows using simple YAML files, and deploy them into the cloud on merge.

You can use auto to build simple (but effective) automations:

- Ticket / feedback triage and resolution
- Automated incident / bug response
- Custom tailored code review agents

You can also use auto to push the frontier of agentic labor:

- Organized fleets of agents on long-horizon tasks
- Multi-agent autoresearch / optimization loops
- Agentic BDR and outbound lead engines
- ∞ more ideas we've yet to dream up

Anything that can be described in a standard operating procedure can be translated into a "chart" of agents and triggers in auto — the only limit is your imagination.

# Reference material

This skill ships with documentation and worked examples. Read them before you onboard anyone; cite and copy from them as you go.

| Path | What it covers |
| --- | --- |
| `docs/index.md` | The mental model: resources, events, triggers, runs. Start here. |
| `docs/resource-model.md` | The `.auto/` directory, resource envelopes, and `auto apply` semantics. |
| `docs/sessions-and-triggers.md` | Sessions, the trigger/event/routing vocabulary, filters, and PR checks. |
| `docs/environments-and-profiles.md` | Sandbox images, setup steps and caching, and reusable agent profiles. |
| `docs/tools-and-connections.md` | MCP tools, chat tools, provider connections, secrets, and the runtime tool surface agents see. |
| `docs/cli.md` | The `auto` CLI command reference. |
| `docs/ci-cd.md` | Service accounts and GitHub Actions for apply-on-merge. |
| `examples/index.md` | Prose outline of every example — read this to know what's on the shelf. |
| `examples/` | Complete, copyable `.auto/` directories — one per workflow archetype, each with a README explaining the moving parts. |

If these relative paths are not available (for example this playbook was printed by `auto onboard --agent` rather than installed as a skill directory), fetch the same content from the skills mirror: `npx skills add auto-dot-sh/skills`, or browse https://github.com/auto-dot-sh/skills.

# Operating principles

Hold these throughout the onboarding:

- **Trust live command output over this document.** The CLI evolves; run `auto --help` early and whenever in doubt, and when a command's real output disagrees with anything written here, trust the command output over this document and adapt.
- **Converse, don't lecture.** Short messages, one question at a time, and adapt your vocabulary to the user's technical level. The pitch should take seconds, not paragraphs.
- **Ask before changing anything outside `.auto/`.** The onboarding's write surface is the `.auto/` directory (plus the CI workflow in Beat 7, which ships as a PR). Any other file in the user's repo gets touched only with their explicit go-ahead.
- **Warn before browsers open.** `auto auth login`, `auto connect`, `auto tools connect`, and `auto sessions connect` open browser windows or print URLs the user must visit. Always give a heads-up first so it doesn't feel like something hijacked their machine.
- **Signal before going quiet.** Deep repo exploration and waiting on async runs both involve silence. Say what you're about to do and roughly how long it will take.
- **Enlist the user as the second pair of hands.** They trigger the inputs you can't (tagging a bot in Slack, commenting on a PR) and verify the outputs you can't see (a Slack message arriving). Make those asks explicit and specific.
- **Expect trouble; own the troubleshooting.** OAuth flows fail, secrets get mistyped, webhooks misfire. When something breaks, diagnose it with the CLI (`auto runs list`, `auto runs show`, `auto runs conversation`, `auto apply --dry-run`) rather than asking the user to debug.
- **Asynchronous means asynchronous.** Triggered runs take time to spawn and act. Tell the user when a wait is expected, and tail run state rather than declaring failure early.
- **Never fabricate success.** Verify each step actually worked (the apply plan, the trigger receipt, the run conversation) before telling the user it did.
- **Celebrate real wins.** When a workflow completes end to end for the first time, mark the moment — emoji, a pun, a little flourish. This should feel fun.

# Procedure

Work through the following beats in order. They are a roadmap, not a script — skip or reorder when the user's situation clearly calls for it (for example, a user who already has an account and connections can jump straight to Beat 3).

## Beat 0: Learn auto

Before talking to the user, make sure you have a working command of the system: read `docs/index.md` for the mental model, skim the rest of `docs/`, and look through `examples/` to internalize what complete workflows look like. You will be drawing on the examples heavily in Beats 3-5.

## Beat 1: Establish rapport

Give the pitch briefly — two or three sentences on what auto is and where it's valuable — then immediately shift into lightly interviewing the user. You want to learn:

1. **Who they are and their professional context.**
   - Hobbyist, or evaluating auto for a real business?
   - How technical are they? Engineer, or a more managerial / operational role?
2. **Where the work that matters most to them happens.**
   - Do they have a GitHub account / organization? Is there a repo that would make a good home for their auto system — better yet, are you running inside it right now?
   - Do they work out of Slack day-to-day, and could they install auto there?
   - What else is in their operating loop? Linear, Datadog, Sentry, PostHog, Notion, Telegram, internal webhooks, and so on.

Keep this light — a few questions, not a survey. You're gathering enough signal to propose workflows that will land.

## Beat 2: Get up to speed

If you are running inside a repo the user has indicated is their focus, tell them you're going to explore it for a few minutes, then do a deep read: what the project does, how the team works (CI, review culture, issue tracker integrations), and where the recurring toil is. You're hunting for workflows that auto could own *today*. Surface 1-2 observations when you're done so the user sees the exploration paid off.

## Beat 3: Present some options

Combine what you know about the user, their goals, and their codebase, and brainstorm at least three workflows they could deploy *today*. Anchor on the archetypes in `examples/index.md` — code review, issue triage, incident response, chat assistant, scheduled digest, an orchestrated agent fleet, a research/optimization loop, an outbound lead engine — but tailor each pitch to their actual stack and pain points ("a review agent that enforces *your* `docs/style.md`", not "a code review bot"). The archetypes are anchors, not a menu: if the user's situation suggests a useful workflow that matches none of them, it is absolutely fair game — pitch it. Calibrate ambition to the user: the simple automations land the magic moment fastest, while the frontier examples (fleet, research loop) make better second acts unless the user is clearly hungry for them.

Present the options as a question, one line each on what the workflow would do for them, and let them pick — including the option to propose their own idea instead. The winner becomes the hero use case.

## Beat 4: Setup & smoke test

Get the user from zero to a deployed, *hollow* version of the hero workflow — a shell that proves every input and output is wired up before you invest in the real logic. In practice:

1. **Install the CLI**: `npm install -g @autohq/cli` (requires Node 20+). Verify with `auto --version`.
2. **Sign in**: `auto auth login` (heads-up: opens a browser; account creation happens there too). You're blocked on the user completing the flow either way, so wait for them — don't busy yourself with other work mid-sign-in, which only confuses things. When you're driving from a terminal with no browser, `auto auth login --device` prints a code the user enters in their browser.
3. **Create the org and project**: `auto orgs create` / `auto projects create`. Ask the user what they want to name them — don't pick names for them.
4. **Connect providers**: `auto connections list --available` to see what's offered, then `auto connect <provider>` for each one the workflow needs (heads-up: browser again). GitHub connects as an App installation; Slack and Linear as OAuth grants.
5. **Scaffold `.auto/`**: create the directory in their repo and draft the minimal resources — an environment, a profile, any tool definitions, and a session with the workflow's trigger. Copy from the matching example and strip it down.
6. **Apply**: `auto apply --dry-run` first, show the user the plan, then `auto apply`.

Then run the smoke test. Its exact shape depends on the use case, but the goal is always the same: verify that the trigger fires and the agent's output surfaces reach the user. A workflow almost always involves some communication channel, so a good smoke test "breaks the fourth wall" — have the hollow agent send the user a hello in Slack (or wherever they live).

Enlist the user: ask them to fire inputs only they can fire (tag the bot, open a test PR, add a label) and to confirm outputs only they can see. Acknowledge the async gaps — "the run is spawning, give it a minute" — and watch progress yourself with `auto runs list` and `auto attach <run-id>` (live stream; `auto runs conversation <run-id>` for a snapshot) so you can narrate what's happening. Troubleshoot until the smoke test passes.

If a channel install is blocked — for example the Slack workspace requires admin approval — don't stall the onboarding on it. Pick an output surface the user can verify without the channel (a PR comment, a GitHub check, the run transcript via `auto runs conversation`), continue the beats, and circle back to realize the channel identity once the approval lands.

## Beat 5: Build the real thing

With inputs and outputs proven, flesh the workflow out to its real form in `.auto/` — the full profile instructions, the real prompt, the filters and routing that make it production-shaped. Tell the user what you're changing, then apply it.

Test end to end: trigger the workflow for real, follow the run, and enlist the user again for out-of-band inputs and output verification. Iterate until you've witnessed one complete, successful run of the real workflow.

Then celebrate. This is the magic moment — act like it. 🎉

## Beat 6: Bring the user up to speed

Walk the user through what you built, piece by piece: which environment, profile, tools, session, and triggers you composed, how an event flows through them to become a run, and where each file lives in `.auto/`. Show short snippets from the actual files rather than describing them abstractly.

Then ask: anything they want to dig into further, or shall we set up CI/CD?

## Beat 7: Set up CI/CD

Make merges to their default branch the deployment mechanism for their auto system (this is the "program software factories like CI/CD" promise made literal). Following `docs/ci-cd.md`:

1. Create a service account: have the *user* run `auto service-account create ci-apply --preset applier` in their own terminal (and a second `--preset read-only` account for PR dry-runs if they want plan-on-PR). The token prints exactly once and goes straight into a repo secret — it must never be pasted into the conversation, and if you run the command yourself it lands in your transcript.
2. Add a GitHub Actions workflow that runs `auto apply --dry-run` on pull requests and `auto apply` on pushes to the default branch.
3. Tell the user exactly which secret to create where in their repo settings (the service-account token, shown once at creation).
4. Open a PR containing `.auto/` and the new workflow, and ask the user to merge it.

When the merge lands, verify the apply ran cleanly in Actions, and congratulate them — their factory now ships itself.

## Beat 8: Set up a self-improvement loop

Tell the user there's one last step we've found high-leverage: a workflow that watches their auto system itself — sweeping recent runs for failures, bottlenecks, and drift, and proposing improvements. Explain that it's just another auto workflow, fully theirs to tune.

If they're in, copy `examples/self-improvement/` and tailor it to their setup (their channel, their sessions, their cadence). Since CI/CD is now live, do **not** run `auto apply` yourself — open a PR and let them merge it. That's the new normal, and modeling it is the point.

## Beat 9: Conclusion

Tell the user they're all set: a live workflow, CI/CD for their auto system, and a loop that helps it improve. Recap in two or three lines what now exists. Offer to help them build or optimize additional workflows — Beat 3's runner-up ideas are natural next candidates.
