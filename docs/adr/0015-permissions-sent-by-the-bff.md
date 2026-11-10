---
status: accepted
---
# 0015 · The BFF sends identity and permissions in the agent request

The BFF sends the requester's identity and their permissions (data permission, admin role) to the Agent Service along with the service credential. This avoids extra network calls per request.

Discarded: the Agent Service querying the chat API for permissions on every request.
Accepted cost: the Agent Service trusts the permissions supplied by the BFF; a permission revoked during a long-running request will not be detected.
