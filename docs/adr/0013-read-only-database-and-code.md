---
status: accepted
---
# 0013 · Read-only enforcement at both database and code levels

The agent queries the business database using a database user with read-only privileges and no access to other databases, and the code pre-validates that the query is read-only. The database provides a hard guarantee even against manipulated queries (goal d), while the code provides early rejection.

Discarded: database-only guarantee; code-only guarantee.
Accepted cost: two layers of rules to maintain in sync; database permissions must be configured carefully since the databases share the same instance.
