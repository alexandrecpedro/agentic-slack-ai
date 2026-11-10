---
status: accepted
---
# 0004 · Managed platform (PaaS) with managed database

The system runs on a managed platform with managed Postgres and Redis instances, instead of the single server suggested in the brief. HTTPS, deployment pipelines, and automatic rollbacks come out of the box, requiring less operational effort. The specific PaaS provider is still to be determined.

Discarded: a single 2 GB Virtual Private Server (VPS) running containers, as suggested in the brief; a larger single server.
Accepted cost: multiple services billed separately, with costs scaling per container; configuration management is tied to the platform provider.
