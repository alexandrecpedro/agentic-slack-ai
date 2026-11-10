---
status: accepted
---
# 0007 · Managed Redis for queues, reconnection events, and counters

A managed Redis instance stores the long-running request queue, recent user events for WebSocket reconnection, and agent rate-limit and cost counters. It is designed for queuing and serves all three use cases.

Discarded: queuing directly in Postgres.
Accepted cost: an additional paid service; Redis becomes a single point of failure: if it goes down, agents, queues, and reconnection mechanisms all stop.
