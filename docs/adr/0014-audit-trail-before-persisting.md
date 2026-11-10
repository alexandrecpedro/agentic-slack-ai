---
status: accepted
---
# 0014 · Audit trail for every verified number prior to persistence

Before saving a response or generating a PDF, the code verifies that every number in the text appears in the query result; if any number does not appear, the response fails over to "could not retrieve". This way, fabricated numbers are blocked in real-time, aiming for zero unverified numbers delivered.

Discarded: numbers populated by code from model-authored placeholders; reliance solely on evaluation datasets.
Accepted cost: a derived number, such as a percentage change, must be returned ready-made by the query; otherwise, the response is blocked.
