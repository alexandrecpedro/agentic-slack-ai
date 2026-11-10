---
status: accepted
---
# 0017 · Three independent pnpm projects, without workspaces

`web`,`chat-api`, and `chat-events` are independent pnpm projects, each with its own lockfile and without a workspace; `chat-events` is included as a `file:` dependency in both `web` and `chat-api`. The rationale is to be defined.

Discarded: pnpm workspaces; pnpm workspaces with Turborepo; npm workspaces.
Accepted cost: any change in `chat-events` requires reinstalling in both projects; there is no build cache or task orchestration across projects; other costs to be defined.
