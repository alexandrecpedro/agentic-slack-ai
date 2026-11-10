---
status: accepted
---
# 0006 · Dedicated worker for long-running requests

Long-running requests, such as generating PDF reports, are placed in a queue and processed by a dedicated Worker separate from the Agent Service. This ensures that heavy operations do not block or delay other requests (goal c).

Discarded: background execution within the Agent Service.
Accepted cost: an additional paid container and an extra component (the queue).
