# Chat assistant

An @mentionable Slack agent with its own identity: mention it to start a conversation, and it holds context across the whole thread. The simplest end-to-end workflow — and the best smoke test, because the output lands directly in front of the user.

```
.auto/
  environments/agent-runtime.yaml
  profiles/assistant.yaml
  tools/slack-chat.yaml
  sessions/assistant.yaml
```

## How it works

This example is the canonical demonstration of the **spawn vs. deliver attribution pattern** (validation enforces it — see `docs/sessions-and-triggers.md`):

- **Fresh mention → spawn**: `chat.message.mentioned` with `$.auto.attributions: { exists: false }` starts a new run.
- **Thread reply → deliver**: the same event plus `chat.message.subscribed`, with `$.auto.attributions: { exists: true }` and `$.auto.authored: false`, delivers into the existing run via `attributedRuns` — the agent keeps its conversational memory.
- The bridge between the two is the agent calling **`auto.chat.subscribe`** after its first reply, which attributes the thread to its run.
- The `identity:` block plus `auto sessions connect assistant` realizes a real workspace bot the user can @mention directly.

## Customize

- Replace the `slack` connection name; rename the identity to whatever persona fits the team.
- The profile here is a lightweight helper; make it an expert on anything by adding tools (a docs MCP server, the repo as a mount) and adjusting the instructions.
- Telegram works the same way — add a `telegram-chat` tool and mirror the triggers with `chat.message.direct` included (DMs are first-class there).

## Smoke test

Apply, run `auto sessions connect assistant` (guided browser flow to realize the bot), invite the bot to a channel, and @mention it. Confirm it replies in-thread; reply again and confirm it remembers the conversation.
