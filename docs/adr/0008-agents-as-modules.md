---
status: accepted
---
# 0008 · Agents as modules within the Agent Service, registered internally

Each agent is a module within the Agent Service (an ADK agent), and the registry of which agents exist and are connected resides directly within the Agent Service; administrators connect and disconnect agents through this registry. This way, adding a new agent requires no changes to the chat or the BFF (goal b).

Discarded: one service per agent, registered by endpoint address; agent registry maintained within the chat API, with each agent acting as a bot user.
Accepted cost: a new agent requires a new release of the Agent Service; each agent also requires a bot user in the chat to write messages, splitting the registration across two locations.
