---
status: accepted
---
# 0009 · The BFF triggers the agent

The BFF saves the message via the chat API, detects a mention of a connected agent or a response to a pending question, and calls the Agent Service without waiting for a response. This way, the chat API remains completely agnostic of agents.

Discarded: the chat API triggering the agent upon saving the message.
Accepted cost: follow-ups without mentions and quick-reply buttons rely on the BFF; only the path through the BFF triggers agents.
