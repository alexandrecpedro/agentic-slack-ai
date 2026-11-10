---
status: accepted
---
# 0010 · The agent writes to the conversation via the chat API as a bot

The Agent Service and Worker save responses, tasks, and deliverables through the chat API as bots with dedicated credentials, and they read the messages required for summaries through it. This way, the same path handles both synchronous and asynchronous requests, and the BFF does not have to wait.

Discarded: the BFF waiting for the agent's response and saving it via the chat API.
Accepted cost: the Agent Service and Worker require credentials in the chat API, which must now accept writes from bot users with limited permissions.
