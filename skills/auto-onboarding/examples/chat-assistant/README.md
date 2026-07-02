# Chat assistant

An @mentionable Slack agent with its own identity: mention it to start a conversation, and it holds context across the whole thread. The simplest end-to-end workflow — and the best smoke test, because the output lands directly in front of the user.

```
.auto/
  agents/assistant.yaml
  fragments/environments/agent-runtime.yaml
```

## Create from the managed template

This archetype is published as the managed template `@auto/chat-assistant`. The default way to install it is a thin agent file in the user's repo — the template carries the prompts, triggers, tools, runtime, and identity with its avatar baked in, so no assets are copied:

```yaml
# .auto/agents/assistant.yaml
imports:
  - "@auto/chat-assistant@latest/agents/assistant.yaml"
variables:
  slackConnection: slack
```

Set the variables to the user's repo, connection names, and channel. Override any field by declaring it in the importing file (local fields win; triggers merge by their authoring `name:`), and use `remove: { triggers: [...], tools: [...] }` to drop inherited entries. The directory below is the source this template was derived from.

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
