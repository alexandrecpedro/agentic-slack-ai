# 01: Chat API Base

**Status:** completed

**What to build:** the chat API transitions from a Nest scaffolding skeleton to establish the foundational infrastructure required by all features:

* validated configuration upon startup;
* chat database access with migrations;
* standardized error format;
* end-to-end testing foundation backed by a real database.

No product behavior yet: verification ensures the API starts up successfully, rejects invalid configuration, and returns errors matching the contract format. References:

* F01 spec, "Foundation built by F01";
* LLD, "Cross-cutting implementation concepts" ("Configuration" and "Errors"), "Data", and "Development Environment".

**Blocked by:** None (can start immediately)

* [x] **AC1** · API configuration is read and validated once upon startup using Zod. Without a mandatory variable like `DATABASE_URL`, the process fails during startup, reporting the variable name in the error message. No other part of the code reads the environment directly.
* [x] **AC2** · The API connects to the chat database via TypeORM, featuring versioned migrations in the repository along with commands to generate, run, and revert migrations. The naming strategy converts identifiers to `snake_case`, and entity IDs use application-generated UUID v7.
* [x] **AC3** · The base domain error, error code types, and global filter are registered in the root module, adhering to the API error rules. Input validation is similarly registered at the root module level.
* [x] **AC4** · A non-existent route responds with a 404 in `application/problem+json` including the corresponding `code`. An unexpected error responds with a 500 status without internal messages or stack traces, while logging the error.
* [x] **AC5** · End-to-end tests run against the `chat_test` database. It is provisioned by the Postgres initialization script or, on pre-existing volumes, by `globalSetup`, which also executes migrations.
* [x] **AC6** · The test foundation provides `createTestApp()` and `uniqueEmail()`, and an end-to-end test validates AC4 using this setup.
* [x] **AC7** · The API `AGENTS.md` documents the migration commands and test database notes.
* [x] **AC8** · `lint`, `tsc`, and all three test suites pass successfully inside the container.
