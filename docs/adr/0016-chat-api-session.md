---
status: accepted
---
# 0016 · The chat API owns the session

The chat API validates the password and creates the session in the chat database; the identifier is stored in an httpOnly cookie valid for both the BFF and the WebSocket. This way, there is a single source of truth, revocation is simply deleting the session, and the WebSocket validates the exact same session without needing a browser token.

Discarded: the BFF owning authentication and issuing a short-lived token to the chat API.
Accepted cost: every request and every connection query the session in the chat database.
