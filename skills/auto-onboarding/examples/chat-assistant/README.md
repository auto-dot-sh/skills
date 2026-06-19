# Chat assistant

An @mentionable Slack agent with its own identity: mention it to start a conversation, and it holds context across the whole thread. The simplest end-to-end workflow — and the best smoke test, because the output lands directly in front of the user.

```
.auto/
  agents/assistant.yaml
  fragments/environments/agent-runtime.yaml
```

## How it works

This example is the canonical demonstration of the **spawn vs. deliver attribution pattern** (validation enforces it — see `docs/agents-and-triggers.md`):

- **Fresh mention → spawn**: `chat.message.mentioned` with `$.auto.attributions: { exists: false }` starts a new run.
- **Thread reply → deliver**: the same event plus `chat.message.subscribed`, with `$.auto.attributions: { exists: true }` and `$.auto.authored: false`, delivers into the existing session via `attributedSessions` — the agent keeps its conversational memory.
- The bridge between the two is the agent calling **`auto.chat.subscribe`** after its first reply, which attributes the thread to its run.
- The inline `identity:` block and the connected Slack workspace realize an @mentionable agent handle the user can tag directly.

## Customize

- Replace the `slack` connection name; rename the identity to whatever persona fits the team.
- The `systemPrompt` here is a lightweight helper; make it an expert on anything by adding tools (a docs MCP server, the repo as a mount) and adjusting the instructions.
- Telegram works the same way — add a `telegram-chat` tool and mirror the triggers with `chat.message.direct` included (DMs are first-class there).

## Smoke test

After the PR merges and GitHub Sync applies the resources, invite the agent handle to a channel and @mention it. Confirm it replies in-thread; reply again and confirm it remembers the conversation.
