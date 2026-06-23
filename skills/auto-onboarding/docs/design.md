# Design

## Agent Avatar Catalog

Every onboarding-authored agent must have an inline `identity` with
`displayName`, `username`, `avatar.asset`, and `description`. Pick the closest
role fit from this catalog, copy the PNG into the target repository under
`.auto/assets/`, and reference it as `.auto/assets/<name>.png`.

The public catalog is also published at
`https://supple-echo-r9y2.here.now/avatars/<file>`, for example
`https://supple-echo-r9y2.here.now/avatars/pr-reviewer.png`.

| File | Best fit |
| --- | --- |
| `architect.png` | System design, architecture, broad implementation planning |
| `auditor.png` | Compliance checks, policy review, accounting-style verification |
| `cartographer.png` | Research coordination, mapping unknown systems, discovery loops |
| `chatterbox.png` | Chat assistants, conversational helpers, lightweight Q&A |
| `chief-of-staff-engineers.png` | Engineering orchestration, agent fleets, multi-task coordination |
| `dispatcher.png` | Routing, queueing, smoke tests, inter-agent messaging |
| `handoff.png` | Taking ownership of delegated work or PR follow-up |
| `herald.png` | Announcements, release notes, broadcasts, comms summaries |
| `inspector.png` | Focused investigation, debugging, diagnostics |
| `introspector.png` | Session review, system self-improvement, behavior analysis |
| `janitor.png` | Cleanup, maintenance, hygiene sweeps, stale-resource pruning |
| `mason.png` | Building reusable foundations, infra, platform construction |
| `navigator.png` | Onboarding, guided setup, workflow selection |
| `patch.png` | Coding fixes, issue implementation, PR repair |
| `pr-reviewer.png` | Pull request review, code quality gates |
| `scout.png` | Lead research, external research, prospecting |
| `scribe.png` | Documentation, notes, meeting summaries, reports |
| `self-improvement.png` | Self-improvement loops, session retrospectives, agent improvement campaigns |
| `sentinel.png` | Incident response, monitoring, alerts |
| `ship-digest.png` | Shipping reports, daily digests, merge summaries |
| `staff-engineer.png` | Senior implementation agents, scoped PR ownership |
| `triage.png` | Issue triage, prioritization, intake classification |
| `tuner.png` | Benchmarking, optimization, experiments |
| `watchdog.png` | Continuous watching, guardrails, recurring safety checks |
