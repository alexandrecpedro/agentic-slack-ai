---
status: accepted
---
# 0002 · Next.js as BFF, with direct WebSocket connection to the chat API

The browser makes all REST calls to Next.js, which acts as a BFF for both the chat API and the Agent Service; only the WebSocket connection goes directly to the chat API. This way, the browser has a single REST entry point, while real-time communication bypasses the BFF.

Discarded: browser calling the chat API and Agent Service directly; browser making REST calls directly to the chat API.
Accepted cost: an extra network hop for each REST call; session authentication must be shared between the BFF and the chat API, requiring subdomains of the same domain.
