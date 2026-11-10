
# Full Cycle Chat

Slack-style team chat where AI agents join as plugins. This glossary stores the product vocabulary; development process terms are in `AGENTS.md`, under the "Planning" section.

## Language

**Person**:
Someone with a workspace account created by the company, who logs into the chat using their own email and password.
*Avoid*: user, account, member

**Member**:
A Person while participating in a channel. Outside of a channel, they are simply a Person.
*Avoid*: participant

**Session**:
A Person's access to the chat in a browser, from the moment they log in until they log out, the Session expires due to inactivity, or it is revoked. This applies to all tabs in that browser.
*Avoid*: login (as a noun), token

**Task**:
Work that an agent delegates to a mentioned person, at the request of the person who mentioned them; it remains assigned to that person until they complete it, and it can be reopened. This is not a planning task (`F01-T02`), which is defined in `AGENTS.md`.
*Avoid*: to-do, pending item
