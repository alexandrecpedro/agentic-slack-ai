---
status: accepted
---
# 0001 · Monorepo with three applications: BFF, chat and agents

The system is a monorepo with three separately deployed applications: Frontend/BFF (Next.js), chat API (NestJS), and agents (Python, with the Agent Service and the Worker as two processes of the same project). This way, the languages and lifecycle of each part remain separated without turning into microservices.

Discarded: single-process modular monolith with a worker; single monolith with background processing in the same process; capability-separated services with custom contracts.
Accepted cost: chat and agents run in different processes, connected via REST; three deployments to coordinate.
