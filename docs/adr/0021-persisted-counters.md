---
status: accepted
---
# 0021 · Unread and mention counters persisted in transaction

Unread and mention counters for each person in each conversation are persisted counters, atomically incremented within the message transaction and reset upon reading, rather than calculated from the last read timestamp. The rationale is to be defined.

Discarded: counts calculated from the last read timestamp using a conversation and date index.
Accepted cost: a deleted or blocked message requires correcting the counters, and the counter may diverge from the last read timestamp; HLD section 7 needs to be updated to no longer state that unreads "derive from the last read"; other costs to be defined.
