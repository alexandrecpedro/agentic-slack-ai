---
status: accepted
---
# 0011 · Order identifier restricts bot actions

When saving a message containing a mention, the chat API emits an order identifier; the bot must present this identifier with every action, and the API only accepts actions tied to a real order on behalf of the user who placed it. This way, the bot can only act within the scope of existing orders.

Discarded: accepting the "on behalf of" identity supplied directly by the bot, only verifying that the users participate in the conversation.
Accepted cost: an additional mechanism to manage, including an expiration time for the identifier.
