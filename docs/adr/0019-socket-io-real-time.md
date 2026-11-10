---
status: accepted
---
# 0019 · Socket.IO for real-time communication

The chat API WebSocket (ADR 0005) uses Socket.IO, implemented in the NestJS gateway and the Frontend. The rationale is to be defined.

Discarded:`ws` with NestJS `WsAdapter`.
Accepted cost: Socket.IO proprietary protocol, and the Frontend depends on `socket.io-client`; Redis-based recovery (HLD 6.2) remains handled by the project; other costs to be defined.
