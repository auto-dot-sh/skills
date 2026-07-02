# Design

## Agent Avatar Catalog

Every onboarding-authored agent must have an inline `identity` with
`displayName`, `username`, `avatar`, and `description`. Agents created from an
`@auto/<name>` managed template inherit theirs. For bespoke agents, pick the
closest role fit from this catalog and declare the avatar by path plus content
hash — the platform stores every catalog image, so a declared catalog `sha256`
needs **no image file in the user's repo** and nothing gets copied around:

```yaml
identity:
  displayName: Scout
  username: scout
  avatar:
    asset: .auto/assets/scout.png
    sha256: 37e366f18de50b2c9d98f1603954821f56f5de32dbe6b5d4ceb9968b2c6a7e3d
```

For a custom image instead of a catalog one, commit the PNG in the user's repo
under `.auto/assets/` and reference just its `asset` path; the apply uploads
the bytes.

To preview the images, the catalog is also published at
`https://supple-echo-r9y2.here.now/avatars/<file>`, for example
`https://supple-echo-r9y2.here.now/avatars/pr-reviewer.png`.

| File | Best fit | `sha256` |
| --- | --- | --- |
| `architect.png` | System design, architecture, broad implementation planning | `bd15f0e58e87c551105e4ed114f6f6fc5b1763d3e66e1e4d120e6dfc395be638` |
| `auditor.png` | Compliance checks, policy review, accounting-style verification | `c39b4aa60612ad0e6629f5fe5c1b1e0344afc4b140cd12d32cd5a9e1592a15d3` |
| `cartographer.png` | Research coordination, mapping unknown systems, discovery loops | `0622761d36ad5f0387f27ca2430ccd4caea63ed824a8b56db4127b7ef5e773a8` |
| `chatterbox.png` | Chat assistants, conversational helpers, lightweight Q&A | `2a24461a9e8726ccfcccfc44b91d5a213f1254254ccf54a25c0c3a1cb5dcffea` |
| `chief-of-staff-engineers.png` | Engineering orchestration, agent fleets, multi-task coordination | `b08efda811c7fd04b18961730d7410b103668514c4b2610c952d1e7b6e21725b` |
| `dispatcher.png` | Routing, queueing, smoke tests, inter-agent messaging | `84b53bb873894f185fdb7d1d12f4bfbd319c1f31392ddc728566104bcb399748` |
| `handoff.png` | Taking ownership of delegated work or PR follow-up | `60b4c94286a571d738edf59b6b5c9a90c6c9fec3f179adb14e75649d4118839a` |
| `herald.png` | Announcements, release notes, broadcasts, comms summaries | `50395a53249a96be9ca1eb160bde02c8ceb681bf08b589c8472ed493efceeb64` |
| `inspector.png` | Focused investigation, debugging, diagnostics | `40c01b275a5f7c7f2aa96e2cf34d5dc328810660b3c1238cbee2df6afdf45a0f` |
| `introspector.png` | Session review, system self-improvement, behavior analysis | `23cf88f32083a5d5879be598338c5e3710c5f0053fb3351170953dcfb0351bfe` |
| `janitor.png` | Cleanup, maintenance, hygiene sweeps, stale-resource pruning | `128d478cc788cbcf04f30f3a4966bbb27c07ad11451785a9ccafb5480e57f657` |
| `mason.png` | Building reusable foundations, infra, platform construction | `079b2f484443aabdc239d33bc79648f885660342d557c424001e4bc56a23160d` |
| `navigator.png` | Onboarding, guided setup, workflow selection | `7532e0e44d3d98ae2e9afe325a67abfd6d0e5c055a3872dfd74225cb9da08ad4` |
| `patch.png` | Coding fixes, issue implementation, PR repair | `56c69edfd17415184b852c94a808ea6fd8afebc885deb1f1963ddf6420baa70f` |
| `pr-reviewer.png` | Pull request review, code quality gates | `8b901940476d9f4b43d944ce6e6f0166c2a57eb33e03464275f2f2599e27a254` |
| `scout.png` | Lead research, external research, prospecting | `37e366f18de50b2c9d98f1603954821f56f5de32dbe6b5d4ceb9968b2c6a7e3d` |
| `scribe.png` | Documentation, notes, meeting summaries, reports | `b535c024ab5e03f4edd419660e14501c51a41ebff02cbdcd11d72340f85d32f8` |
| `self-improvement.png` | Self-improvement loops, session retrospectives, agent improvement campaigns | `5f8e96bb0919d0fc689e1593b70a2b0c2c28913c210c76b7e2d3d5f22a94b1dd` |
| `sentinel.png` | Incident response, monitoring, alerts | `8b8c15db5c65b19fcd81a856cc6b4c56cb64a2b6b473eedcf7159ee0e07f55ec` |
| `ship-digest.png` | Shipping reports, daily digests, merge summaries | `67492c7a80d2f247cc78166298667a467f4afc393847ec10f993a5845a5f3c73` |
| `staff-engineer.png` | Senior implementation agents, scoped PR ownership | `061da0b6fb1154a8687fd4991258121decd20ffa637aea67a79874411870fd1a` |
| `triage.png` | Issue triage, prioritization, intake classification | `d52ca728efaa37a7d72996f63100f6f24c0fb1a3732752e868adc0cb44be9535` |
| `tuner.png` | Benchmarking, optimization, experiments | `f22e7775ec99bb0b96aacbb30991aa1b9e9eda32c84489eea2e09e4be13605a3` |
| `watchdog.png` | Continuous watching, guardrails, recurring safety checks | `faf7e577111128810a8f580142857028d54f7267121b7f3c25b62b655b5664f8` |
