# Environments and fragments

## Environments

An environment defines the cloud sandbox runs execute in: base image, image build steps, cached setup commands, resource limits, and env vars. Environments are authored inline on agents, usually through a fragment import when more than one agent uses the same runtime.

```yaml
# .auto/fragments/environments/agent-runtime.yaml
harness: claude-code
environment:
  name: agent-runtime
  image:
    kind: preset
    name: node24                  # preset base image (Node 24 on Debian slim)
  resources:                      # optional
    cpuCount: 2
    memoryMB: 8192
  env:                            # optional; literal values or { $secret: name }
    CI: "true"
  steps:                          # optional Dockerfile-style image build steps
    - RUN apt-get update && apt-get install -y --no-install-recommends jq && rm -rf /var/lib/apt/lists/*
    - RUN npm install -g tsx
  setup:                          # optional named setup commands run at sandbox start
    - name: install-dependencies
      commands:
        - npm install
      cache:                      # optional per-step cache
        files:                    # relative paths whose content keys the cache
          - package-lock.json
        paths:                    # absolute paths under /home/user to restore
          - /home/user/.npm
  setupCache:
    ttl: 24h                      # cache lifetime: e.g. 30m, 24h, 7d
```

Guidance:

- **`steps` vs `setup`**: `steps` bake into the image (system packages, global CLIs — slow-changing, shared across runs). `setup` runs against the mounted workspace at sandbox start (dependency installs — repo-specific, cacheable by lockfile).
- Start minimal. The `node24` preset plus a dependency-install setup step covers most JS/TS repos; add `steps` only when the agent needs system binaries.
- Mirror local development: if the team needs `psql` or `redis-cli` to debug, the agent probably does too.

Agents then import the environment fragment:

```yaml
name: pr-review
imports:
  - ../fragments/environments/agent-runtime.yaml
systemPrompt: |
  You are the code review agent for acme/widgets.
```

## Shared Instructions

An agent's `systemPrompt` is its durable instruction set. Put reusable runtime
configuration in fragment imports; put the agent's own standing orders directly on
the agent that owns them.

```yaml
name: reviewer
imports:
  - ../fragments/environments/agent-runtime.yaml
systemPrompt: |                   # ≤100k chars; the agent's standing orders
  You are the code review agent for acme/widgets.
  ...
```

What belongs in `systemPrompt` (vs an agent's `initialPrompt`):

- **Persona and scope** — who the agent is, what it owns, what it must never do ("do not push commits, approve, or merge").
- **House rules** — repo conventions to read first, test/typecheck expectations, communication norms (which channel, threading protocol, link formats).
- **Tool protocols** — how to use the granted tools correctly (e.g. "pass the channel name directly; do not search to resolve it", Slack mrkdwn link syntax, the GitHub attribution marker).
- **Failure posture** — what to do when blocked: stop and explain rather than invent.

The agent's `initialPrompt` then carries only the per-run task and the event context. Several agents can import the same environment fragment while keeping different prompts, tools, and triggers.

Writing instructions that hold up in production:

- Be explicit about output discipline: exactly one PR comment, exactly one top-level Slack message per artifact, details in threads. Agents repeat what you tolerate.
- Encode idempotency: "look for an existing thread for this PR before creating one."
- Bound side effects: enumerate what the agent must never create or mutate (labels, statuses, users) when metadata hygiene matters.
- Require the GitHub attribution marker on anything the agent posts to GitHub (see `sessions-and-triggers.md`).
