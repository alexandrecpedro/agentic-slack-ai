---
status: accepted
---
# 0020 · Business data message emitted person-to-person over WebSocket

A message marked as "business data" is sent over the WebSocket in a per-person event: users with data permission receive the content, and other members receive the blocked version. This way, the message appears complete immediately for those with permission.

Discarded: the same blocked version for everyone, with the content fetched via REST by those with permission; hiding the message from those without permission (violating HLD 6.4 and the brief).
Accepted cost: business data content is stored in Redis reconnection events for authorized users; these messages are emitted person-to-person rather than once per conversation.
