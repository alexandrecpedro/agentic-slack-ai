---
status: accepted
---
# 0012 · Custom Python tool instead of MCP

The agent queries the business database using a custom Python tool within the Agent Service rather than an MCP server, as suggested in the brief. This keeps query auditing and limits under full control without introducing new components.

Discarded: Postgres MCP server used as a tool by the ADK.
Accepted cost: diverges from the MCP standard mentioned in the brief; every new data source requires custom code within the agent.
