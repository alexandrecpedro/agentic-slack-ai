---
status: accepted
---
# 0023 · Login attempt limits tracked by email only

The login attempt limit counts failures by normalized email, including for emails without an account, and blocks that email even with the correct password, as requested by the F01 PRD. Thus, brute-force protection does not depend on the source IP, which behind the PaaS proxy cannot yet be reliably read (HLD, section 12), and the response does not reveal whether the email has an account.

Discarded: counting by email and IP pair; counting email and IP separately; storing the count in Redis.
Accepted cost: anyone who knows someone else's email can lock them out for a `login lockout` period by repeatedly entering wrong passwords (F01 PRD, "Product Risks").
