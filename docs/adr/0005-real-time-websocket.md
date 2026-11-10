---
status: accepted
---
# 0005 · WebSocket for real-time communication

Messages, "typing" indicators, presence, and notifications reach the browser via WebSocket, opened directly with the chat API. It is a bidirectional channel, naturally suited for "typing" status and presence.

Discarded: SSE for server-to-client events combined with REST for client-to-server requests; polling.
Accepted cost: stateful connection; reconnection handling and missed event recovery fall under the project's responsibility (using Redis events).
