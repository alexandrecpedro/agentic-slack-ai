# F01-T01 Plan: Chat API Base

Task: [task.md](task.md) · Spec: [spec.md](../../spec.md) · PRD: [prd.md](../../prd.md)

## Criteria

### AC1

**Mark:** integration test

**Scenario:** Given the API environment without `DATABASE_URL`, when the API process starts, then it terminates with a non-zero exit code before opening the port, and the error message includes the name `DATABASE_URL`.

**Planned test:** API e2e (process level, without HTTP), `test/startup.e2e-spec.ts`, "the API fails to start without DATABASE_URL and reports the variable name". The test spawns `src/main.ts` in a child process using the container environment minus `DATABASE_URL`. The constraint "no other code reads the environment directly" is enforced by linter rules (see "Decisions").

### AC2

**Mark:** human review

**Scenario:** Given the task diff, when the reviewer opens `src/database/` and `package.json`, they find TypeORM connected to the chat database via validated configuration, `SnakeNamingStrategy` used in both application and CLI options, `synchronize` disabled, the base entity generating ids via `crypto.randomUUIDv7()` before insertion, and the scripts `migration:generate`, `migration:run`, and `migration:revert`. Inside the container, `pnpm migration:run` completes successfully against the `chat` database and creates the `migrations` table. Connection establishment is proven by every e2e test running against the database.

### AC3

**Mark:** human review

**Scenario:** Given the task diff, when the reviewer opens `AppModule`, they find the global filter registered under `APP_FILTER` and the `ValidationPipe` under `APP_PIPE`, with none of these in `main.ts`; and in `src/common/errors/`, they find `DomainError`, the `ErrorCode` type, and the filter mapping `code` to status.

### AC4

**Mark:** HTTP test

**Scenario 1:** Given the API is running, when a request is made to `GET /v1/non-existent-route`, a 404 is returned with `Content-Type: application/problem+json` and a body containing `status` 404 and `code` `not_found`.

**Scenario 2:** Given a route failing with an unexpected error whose message contains an internal detail, when called, it returns 500 with `code` `internal_error`, the body does not expose internal messages or stack traces, and the error is logged at the `error` level.

**Planned test:** API e2e, `test/errors.e2e-spec.ts`, "responds 404 in problem+json with the not_found code on a non-existent route" and "responds 500 without internal message or stack trace and logs the error". The failing route is a controller declared strictly in the test file and passed to `createTestApp()`. Logging is verified using a spy on `Logger.prototype.error`, which is the idiomatic side-effect test (`.claude/rules/nest-api-testing.md`).

### AC5

**Mark:** integration test

**Scenario:** Given the e2e suite initialized, when the test application queries its connected database, the database is `chat_test` and the `migrations` table exists.

**Planned test:** API e2e, `test/database.e2e-spec.ts`, "the test application uses the chat_test database with executed migrations", via the `DataSource` of the application instance created with `createTestApp()`. The pre-existing volume path executes on every `globalSetup` run (creating the database only if missing); reviewers verify this once by dropping `chat_test` and running `pnpm test:e2e`.

### AC6

**Mark:** human review

**Scenario:** Given the task branch, when the reviewer opens `test/support/`, they find `createTestApp()` and `uniqueEmail()`, and `test/errors.e2e-spec.ts` starts the application solely through `createTestApp()` and closes it with `app.close()`.

### AC7

**Mark:** documentation

**Scenario:** Given `apps/chat-api/AGENTS.md`, when the reviewer reads the "Commands" section, they find the three migration commands with the name argument; and in the "Testing" section, the note that the e2e suite runs against `chat_test`, created by the initialization script or, on a pre-existing volume, by `globalSetup` which also executes migrations, pointing to LLD "Development Environment".

### AC8

**Mark:** human review

**Scenario:** Given the task branch, when the reviewer runs `pnpm lint`, `pnpm exec tsc --noEmit`, `pnpm test`, `pnpm test:int`, and `pnpm test:e2e` inside the `chat-api` container, all complete without errors.

## Contract

### GET / · removed

Nest scaffolding route ("Hello World!"), with no consumers. Removed alongside `AppController` and `AppService`.

No new routes, events, or tables. Error response bodies adhere to the LLD convention ("Cross-cutting implementation concepts", section "Errors") and apply to all routes; the `migrations` table belongs to TypeORM itself.

## Contract Verification

* `GET /` removed: no tests. The route has no consumers, and responses to any non-existent route are already covered by AC4.

## Decisions

* **Configuration:** `@nestjs/config` with `ignoreEnvFile: true` and `validate` pointing to the Zod schema in `src/config/env.schema.ts`. Validation errors list every invalid variable by name. Variables are supplied by Docker Compose and, in production, by the PaaS; `.env.example` documents the list. Rationale: single source of truth for both container and PaaS environments.
* **Variables for this task:** `DATABASE_URL` (mandatory) and `PORT` (default 3000), previously read directly by `main.ts`. `SESSION_TTL_DAYS` and `WEB_ORIGIN` will be introduced in the tasks that use them.
* **Environment access:** only `src/config/` touches `process.env`. The TypeORM CLI, running outside Nest, calls `loadEnv()` from `src/config/` using the same schema. An ESLint `no-restricted-properties` rule blocks `process.env` in `src/**` outside of `src/config/**`. Rationale: turns policy into machine-enforced rules rather than relying solely on review.
* **Unified TypeORM options:** `src/database/database.options.ts` consolidates options (connection URL, `SnakeNamingStrategy` from `typeorm-naming-strategies`, `synchronize: false`), shared by `TypeOrmModule.forRootAsync` and the CLI `data-source.ts`. The application uses `autoLoadEntities`; the CLI discovers entities via glob patterns. The application does not run migrations on startup, as specified in the root `AGENTS.md`.
* **Migrations path:** `apps/chat-api/migrations/`, per LLD "Code Structure", outside `src/` and excluded from `tsconfig.build.json`, executed via CLI with ts-node. Production migration execution is deferred to the deployment task.
* **Id generation:** Abstract base entity in `src/database/` with a UUID `id` (`@PrimaryColumn`), generated via `crypto.randomUUIDv7()` in a `@BeforeInsert` hook when empty, alongside `createdAt`/`timestamptz` and `updatedAt`/`timestamptz`. Container Node 26 natively supports `randomUUIDv7`, eliminating the `uuid` package. Direct insertions bypassing `.save()` (`insert`, QueryBuilder) generate ids using the same function.
* **Error body structure:** `type` `about:blank`, `title` corresponding to the HTTP status phrase, `status`, `detail`, and `code`, omitting `instance` (LLD "Cross-cutting implementation concepts", section "Errors"). `DomainError` carries the `code` and message, which populates `detail`.
* **Error codes for this task:** `validation_failed` (400), `not_found` (404), `payload_too_large` (413), `request_rejected` (other 4xx exceptions from Nest HTTP exceptions or body parsers, preserving status), and `internal_error` (500). ValidationPipe 400 errors and malformed JSON errors map to `validation_failed` since clients treat both as invalid input. For non-domain errors, `detail` uses a fixed string per code in Portuguese rather than exception messages (which contain framework paths and framework internals). 5xx HTTP exceptions are treated as unexpected errors.
* **`validation_failed` without field lists:** the web layer validates on the client and Server Actions using PRD messages (spec "BFF and screens"), so no client currently utilizes field-level error arrays. They will be introduced when required.
* **Unexpected error logging:** Nest `Logger` at the `error` level passing the exception. Structured logging libraries will be finalized in the LLD "Logs" section.
* **`/v1` prefix:** `configureApp(app)` in `src/app.setup.ts` applies `setGlobalPrefix('v1')` and is shared by `main.ts` and `createTestApp()`, ensuring consistency with the test application. Filters and pipes are registered in `AppModule` as required by the API `AGENTS.md`.
* **Test database:** Compose passes `TEST_DATABASE_URL` (`chat` on `chat_test`) and `TEST_DATABASE_ADMIN_URL` (local `postgres` superuser) to `chat-api`. `globalSetup` connects as superuser and creates `chat_test` if missing, setting owner to `chat` and revoking `CONNECT` from `PUBLIC` matching initialization scripts; it then executes `pnpm migration:run` in a child process pointing `DATABASE_URL` to `chat_test`. Jest e2e `setupFiles` swaps `DATABASE_URL` for `TEST_DATABASE_URL` before importing `AppModule`. Rationale: the `chat` role lacks `CREATEDB`, and only the superuser can create the database on existing volumes; credentials mirror existing local compose configurations. Child-process migration execution avoids depending on Jest to load TypeScript migration files.
* **`createTestApp(options?: { controllers?: Type[] })`:** builds `AppModule` combined with extra controllers without `overrideProvider`, applies `configureApp`, and invokes `init()`. Extra controllers support base-level testing such as the failing route required by AC4.
* **`uniqueEmail(prefix = 'pessoa')`:** `<prefix>-<uuid>@teste.local`, lowercased.
* **Input validation:** `class-validator` and `class-transformer` inside Nest's `ValidationPipe` with `whitelist` and `transform` enabled. Rationale: idiomatic Nest approach where `@nestjs/swagger` reads identical decorators to generate the OpenAPI snapshot planned in the LLD. Rejected: Zod with `nestjs-zod`, which inserts an extra abstraction layer between Zod and Swagger. Architectural selection documented in LLD "Code Structure".

## Files and Interfaces

* `apps/chat-api/package.json`: dependencies `@nestjs/config`, `zod`, `@nestjs/typeorm`, `typeorm`, `pg`, `typeorm-naming-strategies`, `class-validator`, `class-transformer`; dev dependency `@types/pg`. Scripts `migration:generate` (accepts `migrations/<Name>`), `migration:run`, `migration:revert`; `test` script updated with `--passWithNoTests`.
* `apps/chat-api/src/config/env.schema.ts`: `envSchema` (Zod), `type Env`, `parseEnv(raw): Env` throwing with invalid variable names, and `loadEnv(): Env` reading `process.env`.
* `apps/chat-api/src/config/config.module.ts`: `AppConfigModule`, global, configured with `ConfigModule.forRoot({ isGlobal, ignoreEnvFile, validate: parseEnv })`.
* `apps/chat-api/src/database/database.options.ts`: `buildDataSourceOptions(databaseUrl)`.
* `apps/chat-api/src/database/database.module.ts`: `DatabaseModule`, utilizing `TypeOrmModule.forRootAsync` reading `DATABASE_URL` from `ConfigService`.
* `apps/chat-api/src/database/data-source.ts`: CLI `DataSource` utilizing `loadEnv()`.
* `apps/chat-api/src/database/base.entity.ts`: abstract base entity with `id`, `createdAt`, `updatedAt`, and the id `@BeforeInsert` hook.
* `apps/chat-api/migrations/.gitkeep`: migrations directory, empty in this task.
* `apps/chat-api/src/common/errors/domain.error.ts`: `abstract class DomainError extends Error` with `abstract readonly code: ErrorCode`.
* `apps/chat-api/src/common/errors/error-code.ts`: `type ErrorCode` and `ERROR_STATUS: Record<ErrorCode, number>`.
* `apps/chat-api/src/common/errors/problem-details.filter.ts`: `ProblemDetailsFilter` (`@Catch()`) translating `DomainError`, `HttpException`, and body parser errors, and logging unexpected failures.
* `apps/chat-api/src/app.module.ts`: imports `AppConfigModule` and `DatabaseModule`; registers `APP_FILTER` and `APP_PIPE`; removes `AppController` and `AppService`.
* `apps/chat-api/src/app.setup.ts`: `configureApp(app: INestApplication)`.
* `apps/chat-api/src/main.ts`: invokes `configureApp` and reads port via `ConfigService`.
* `apps/chat-api/src/app.controller.ts`, `app.service.ts`, `app.controller.spec.ts`: removed.
* `apps/chat-api/test/app.e2e-spec.ts`: removed.
* `apps/chat-api/test/jest-e2e.json`: configures `globalSetup` and `setupFiles`.
* `apps/chat-api/test/support/global-setup.ts`: provisions `chat_test` if missing and runs migrations.
* `apps/chat-api/test/support/test-env.ts`: maps `DATABASE_URL` ← `TEST_DATABASE_URL`.
* `apps/chat-api/test/support/create-test-app.ts`: `createTestApp()`.
* `apps/chat-api/test/support/unique-email.ts`: `uniqueEmail()`.
* `apps/chat-api/test/startup.e2e-spec.ts`, `test/errors.e2e-spec.ts`, `test/database.e2e-spec.ts`: criterion tests.
* `apps/chat-api/tsconfig.build.json`: excludes `migrations`.
* `apps/chat-api/eslint.config.mjs`: `no-restricted-properties` restriction on `process.env` in `src/**` outside `src/config/**`.
* `apps/chat-api/.env.example`: `DATABASE_URL` and `PORT` with placeholder values.
* `apps/chat-api/AGENTS.md`: migration commands and test database notes (AC7).
* `docker/postgres/init/01-create-databases.sh`: creates `chat_test`, owned by `chat`, applying identical `REVOKE`/`GRANT CONNECT` permissions as other databases.
* `compose.yaml`: supplies `TEST_DATABASE_URL` and `TEST_DATABASE_ADMIN_URL` to `chat-api`.

## Existing Tests That Change

* `src/app.controller.spec.ts` and `test/app.e2e-spec.ts` are removed alongside the scaffolding route they tested. The unit test suite has no tests in this task, hence `pnpm test` includes `--passWithNoTests`, matching `test:int`.

## Data Changes

None.

## Updated Documents

* `docs/lld.md`, "Code Structure" section: added row "Chat API input validation", detailing `class-validator` and `class-transformer` usage and rationale.
* `docs/lld.md`, "Cross-cutting implementation concepts", "Errors" section: first item updated to specify response body fields (`type` `about:blank`, `title`, `status`, `detail`, `code`, without `instance`).
* `docs/lld.md`, "Values and Configuration", "Environment Variables" section: added row for Chat API `PORT`, "HTTP port where the API listens".
* `docs/lld.md`, "Development Environment" section, "Chat API test database" item: `globalSetup` connects as local superuser to create the database.
* `docs/roadmap.md`: F01 status updated to "in progress".
