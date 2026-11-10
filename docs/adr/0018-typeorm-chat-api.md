---
status: accepted
---
# 0018 · TypeORM in the chat API

The chat API accesses the chat database via TypeORM, and its migrations are the owner of that database schema. The rationale is to be defined.

Discarded: Prisma; Drizzle.
Accepted cost: query typing is weak, and generated migrations need to be manually reviewed; other costs to be defined.
