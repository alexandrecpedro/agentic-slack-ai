# Spec · F01 · Login and Session

2026-11-09 · Luiz Carlos · Status: ready · PRD: [prd.md](prd.md)

How feature F01 will be built. What the user can do, acceptance criteria, and out-of-scope
items are listed in the PRD. Session and login rules, values, and cross-cutting conventions
reside in the [LLD](../../lld.md), which this spec references rather than repeating.

## Problem

Without identifying who uses the chat, nothing else in the product functions: there is no message author,
no private channel participation, and no data permissions. Currently, there is no way to log in (PRD,
section 1, "Executive Summary").

## Solution

The user logs in with the email and password of the account created for them by the company,
stays logged in across all browser tabs while using the chat at least once every 30 days, and logs
out via the avatar menu. The installation operator creates accounts, resets passwords, and
revokes sessions via a chat API command. The complete flow is in PRD, section 2, "Flow".

## User Stories

User stories 1 to 7 from the PRD (section 2, "User Stories") apply as written, along with their
acceptance criteria. This spec adds stories for the installation operator, which the PRD leaves
out of the product:

8. As an operator, I want to create a user account with an email, name, and password so that
   the user can log into the chat.
9. As an operator, I want to reset a user's password so they can log back in when they forget
   it.
10. As an operator, I want to revoke all sessions for a user to remove access from a lost or
    shared browser.
11. As an operator, I want to delete expired sessions to prevent the database from accumulating
    dead rows.
12. As an operator, I want to type the password without it remaining in the terminal history to
    avoid leaving credentials exposed on the machine.

## Implementation Decisions

### Foundation Built by F01

F01 is the first feature, and the codebase is still the skeleton of both frameworks. It establishes
the infrastructure that subsequent features will reuse:

- **Chat API:** configuration module with startup validation (LLD, "Cross-cutting
  implementation concepts", section "Configuration"); TypeORM with migrations
  ([ADR 0018](../../adr/0018-typeorm-chat-api.md)) and UUID v7 IDs
  ([ADR 0022](../../adr/0022-uuid-v7.md)); the base domain error, global error filter, and
  authentication guard described in the API error rules (`.claude/rules/nest-api-errors.md`).
- **Web:** configuration module with identical validation; the complete set of design system
  tokens in light and dark themes with the Lato font. The light theme is the default; the dark
  theme applies only with `data-theme="dark"` on the root element, which F09 links to theme
  switching. Design system primitives and libraries are introduced as the screens in this spec
  consume them (`docs/design-system.md`, "UI stack").
- **API Tests:** a separate test database, utilizing the test foundation described in "Testing
  Decisions".
- **Web Tests:** Playwright running in its own compose service (LLD, "Development Environment"),
  using the foundation described in "Testing Decisions". The service and command are added to
  the root `AGENTS.md` and to the web during the task that sets them up.

### People (Chat API)

Table `people`, `planned`:

| Column                     | Type         | Rule                                           |
| -------------------------- | ------------ | ---------------------------------------------- |
| `id`                       | uuid         | UUID v7, generated in application code         |
| `email`                    | text, unique | Stored normalized: trimmed and lowercase       |
| `name`                     | text         | Name displayed in the avatar menu and initials |
| `password_hash`            | text         | Argon2id (LLD, section "Security")             |
| `created_at`, `updated_at` | timestamptz  | LLD convention, section "Data"                 |

`@user`, position, phone, presence status, administrator role and data permissions are deferred to the
features that utilize them (F05, F09, F10 and F13).

### Sessions (Chat API)

Table `sessions`, `planned`:

| Column                     | Type         | Rule                                                            |
| -------------------------- | ------------ | --------------------------------------------------------------- |
| `id`                       | uuid         | UUID v7                                                         |
| `person_id`                | uuid         | Reference to `people`; deleting a Person deletes their Sessions |
| `token_hash`               | text, unique | SHA-256 of the token; the raw token is never stored nor logged  |
| `expires_at`               | timestamptz  | Expiration time, renewed upon use                               |
| `last_used_at`             | timestamptz  | Last usage timestamp that renewed the validity                  |
| `created_at`, `updated_at` | timestamptz  | LLD convention                                                  |

Rules for creation, verification, renewal, expiration, revocation, and sign-out are located in
the LLD, section "States and Rules", subsection "Session and Login".

**Rotas**, all `planned` and using the API error rule body format:

| Route                            | Payload                 | Success                                                      | Errors                                                                                                                          |
| ------------------------------- | ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| `POST /v1/sessions`           | `email`, `password` | 201 with `token` and `person` (`id`, `name`, `email`) | `validation_failed` (400), `invalid_credentials` (401), `too_many_login_attempts` (429, with `Retry-After` in seconds) |
| `GET /v1/sessions/current`    | Session cookie       | 200 with `person`                                           | `unauthenticated` (401)                                                                                                      |
| `DELETE /v1/sessions/current` | Session cookie       | 204                                                          | `unauthenticated` (401)                                                                                                      |

- Input validation for `POST /v1/sessions` accepts a valid email format of up to 254 characters
  and a non-empty password of up to 256 characters to prevent excessively large payloads from
  burdening Argon2id.
- `unauthenticated` covers missing cookies, unknown tokens, expired Sessions, and revoked Sessions
  without distinguishing between them.
- The authentication guard reads the session from the cookie—the exact credential passed along by
  the BFF (LLD, section "Interfaces"). F03 reuses this exact verification during WebSocket handshakes.

### Login attempt rate limiting (Chat API)

Table `login_attempts`, `planned`: one row per normalized email, containing `id`, `email` (unique),
`failed_count`, `first_failed_at`, `blocked_until`, `created_at` and `updated_at`. The rules are
detailed on LLD, section "States and rules", subsection "Session and login"; tracking attempts
strictly by email is established in [ADR 0023](../../adr/0023-login-attempt-limits-by-email.md).

### Account management command (Chat API)

A script built into the chat API, executed within the container, which boots the Nest context
without HTTP and invokes the People and Sessions services directly. There is no CLI library
framework.

| Operation                | Arguments                         | Effect                                                         |
| ------------------------ | --------------------------------- | -------------------------------------------------------------- |
| `create`                 | Email, name; password by terminal | Crates the Person. Existing emails are rejected with a message |
| `reset-password`         | Email; new password by terminal   | Updates the hash and revokes all Sessions for the Person       |
| `revoke-sessions`        | Email                             | Deletes all Session for the Person                             |
| `purge-expired-sessions` | —                                 | Deletes expired Sessions                                       |

The password is read from the terminal with echoing disabled and is never accepted via arguments
or environment variables. The command is added to the API `AGENTS.md`, section "Commands",
during the task that implements it.

### BFF and screens (web)

- **Routes:** `/signin` is the sign-in screen, and `/` is the Home screen. Authenticated screens
  reside under a shared layout. The return URL parameter is `?for=`, with rules specified in the LLD,
  section "Client", "Session on the Client".
- **Browser Calls:** signing in and signing out are Server Actions (LLD, "Code structure"). Each
  invokes a web HTTP service communicating with the chat API; neither components nor Server Actions
  invoke `fetch` directly (`apps/web/AGENTS.md`, "External APIs").
- **Cookie:** the BFF sets the cookie containing the token returned by `POST /v1/sessions` and
  clears it upon sign-out or upon receiving a 401 response. Attributes and lifetimes match the
  LLD, section "Security".
- **Session Verification:**  the Next.js proxy checks solely for the presence of the cookie, whereas
  the authenticated screen layout and the sign-in screen (`/signin`) perform full verification.
  Behavior for each response type is defined in the LLD, section "Client", "Session on the Client".
- **Sign-In screen:** replicated the Figma Login section (19:1276). States include:

  - Default: frame 19:1306, with initial focus on Email.
  - Validation error: frame 19:1336. Validation runs on both client and Server Action
    using PRD messages (Story 2), shifting focus to the first invalid field.
  - Submitting: frame 19:1369. The button enters a `loading` state, blocking duplicate
    submissions.
  - Incorrect credentials: frame 19:1399, previously named "Server Error", to be renamed. The
    message appears below Password, the password field is cleared, focus moves to Password, and
    email remains populated.
  - Too many attempts: new frame. Message: "Too many sign-in attempts. Please wait a few minutes
    and try again."
  - Submission failure: new frame for network or server failures. Message is "Unable to sign in
    right now. Check your connection and try again."

  Nos dois estados novos, a mensagem fica no formulário, entre a Senha e o botão, com
  `role="alert"`. O e-mail e a senha continuam preenchidos, e o foco não muda. Os campos com erro
  levam `aria-invalid` e `aria-describedby` apontando para a mensagem.
- **Figma update, prior to Sign-In screen code:** rename the frame 19:1399 to "Login / Incorrect
  credentials" and create "Login / Too manu attempts" and "Login / Submission failure" following
  the same pattern using local components and variables (`docs/design-system.md`, "From code to
  Figma").
- **Home na F01:** the actual shell utilizing only existing features rather than a placeholder screen.

  - Rail: "FC", active "Home" (Start) and intials avatar. Excludes Activity (F08), theme control
    (F09) and presence indicator (F09).
  - Sidebar: header "Full Cycle" only, omitting the new message icon and the channels, direct
    messages, and agents sections (F02, F04, F05, F10).
  - Main: "Home / Empty" state (19:1645).

  The shell expands with each subsequent feature.
- **Avatar menu (UserMenu):** triggered via the Rail avatar using Radix DropdownMenu, supporting
  mouse and keyboard navigation. In F01, it features a header displaying the initials avatar and
  name (without presence status) and a single item, "Sign out of Full Cycle" (Shell, 25:25749).
  Signing out triggers the sign-out Server Action, which closes the Session on API
  (`DELETE /v1/sessions/current`), clears the cookie and navigates to `/signin`. If the API call
  fails, the cookie is cleared regardless and the Person navigates to `/signin`.
- **Waiting and verification error::** while the authenticated layout verifies the session, "Doors /
  Loading app" (19:1770) is displayed. If verification fails due to network or server, "Doors /
  Session Error" (19:1793) appears full-screen without the shell; clicking "Try again" re-renders
  the route. "Doors / Waiting Shell" (19:1799) is excluded from F01.
- **Browser history back-navigation after sign-out::** authenticated screens are excluded from
  browser caching. The mechanism is described in the LLD, section "Client", "Session on the Client".

### Real-time

F01 does not introduce WebSockets. The aspect of PRD stories 5 and 6 concerning "refused
real-time connection" becomes an acceptance criterion for F03: the handshake utilizes the
session verification defined in this spec and rejects connections with expired or revoked
sessions (LLD, section "Client", "Session on the Client"). In F01, the "next action"
corresponds to navigation or a BFF call receiving a 401.

## Testing Decisions

**What makes a good test here:** test proves external behavior - specifically HTTP responses,
database rows, or redirections - rather than internal service structures. The structure of each
test follows `.claude/rules/nest-api-testing.md`.

**Chat API, end-to-end tests with a real database.** They are the primary seam: all F01 rules
reside within the API and are validated through it.

- Signing in with valid email and password: responds 201, returns token and stores session hash
  instead of raw token.
- E-mail is accepted case-insensitively and with trailing/leading spaces.
- Input validation, including maximum lengths: responds 400 with `validation_failed`, and the
  attempt does not count toward rate limits.
- Non-existent email and incorrect password: both respond 401 with identical `invalid_credentials`
  and identical body structures.
- Rate limiting: the 5th failure within the sliding window blocks access; during the block, even
  correct passwords receive 429 with `Retry-After`; the attempts during the block do not extend it;
  post-block, correct passwords succeed; successful logins prior to the limit reset the count; rate
  limits also apply to unregistered emails; simultaneous failures count as two.
- Verification: responds 200 with a Person; renews validity when past the `session renewal interval`
  and does not renew prior to it.
- Expired, revoked session, and unknown tokens: all respond 401 with `unauthenticated`, and expired
  sessions are deleted.
- Sign-out: responds 204, and subsequent requests with the same session respond 401; other sessions
  for the same Person remain valid.

Time-sensitive tests (sliding windows, blocks, expirations and renewals) manipulate row timestamps
in the database during the Arrange phase, via test foundation shortcuts, avoiding fake clocks and
`overrideProvider`.

**Account Command:** integration test covering all four operations against test database. Terminal
password input is abstracted behind a minor interface, allowing tests to subsitute predefined input.

**API Test Foundation, created in F01:**

- `createTestApp()`;
- `uniqueEmail()`;
- `authRoutes` shortcuts (creating a Person directly via service and sign-in);
- Shortcuts for aging Session and attempt counters;
- The `chat_test` database, provisioned by the Postgres initialization script or, on pre-existing
  volumes, by `globalSetup`, which also runs migrations.

**Web, end-to-end tests via Playwright.** This is the sole web seam. Tests run exclusively in Chromium
headless mode, alongside `playwright-cli` sessions, against the running development environment (web,
chat API, and `chat` database), validating behavior via the UI. Coverage includes:

- Successful sign-in;
- Validation errors, incorrect credentials, and rate-limiting blocks, matching texts and focus behaviors
outlined in "BFF and Screens";
- `?for=`, including malicious values like `//other-site`, which default to Home;
- Authenticated users opening `/signin` being redirected to Home;
- A second tab in the same browser automatically recognizing authentication;
- Reopening the browser within a new context retaining saved cookies;
- Signing out redirects to "Sign In", and secondary tabs redirect to "Sign In" upon their next action;
- Revoked Session redirects to "Sign In" uppon next action;
- Back-navigation via history after sign-out revealing no chat content;
- Full keyboard-only navigation flow;
- Accessibility auditing via `@axe-core/playwright` on the Sign-In and Home screens, verifying zero
  violations.

**How web tests are written:** following the `playwright-cli` skill workflow (plan, generate and fix).
The agent explores the screen from a seed test, writes the test plan, generates one test file per
scenario containing recorded actions, and fixes broken tests through the same channel. The F01 test
plan resides alongside the tests in the `web` project, references this spec, and governs scenario
steps, while this spec remains authoritative over scope. Skill commands execute within the `e2e`
container, never via local `npx`.

**Web Test Foundation, created in F01:**

- The `person` fixture provisions a Person directly in the database with a unique e-mail and a fixed
  Argon2id hash for a known test password, returning email and credentials; storage and exposure
  details reside in the LLD, "Development Environment";
- The `loggedInPage` fixture signs in through the sign-in form interface;
- Seed test open `/signin` with a pre-configured Person. Login scenarios utilize `person`, while
  authenticated scenarios utilize `loggedInPage`;
- Revocation and time-dependent tests manipulate database rows in the Arrange phase, mirroring the
  API approach.

**Manual Verification, documented in the task directory:**

- **"Session Error":** the agent stops the chat API process, opens an authenticated screen, verifies
  the message and button, restarts the API, and verifies that clicking "Try again" renders the
  requested screen. Snapshots and captures are recorded.
- **Visual Validation:** captures at 1440 × 900 in light and dark themes (toggling `data-theme`),
  compared against Figma frame screenshots via `get_screenshot` (`docs/design-system.md`,
  "Visual Validation").

**Manual Script, executed by a human and logged in the same directory:** verifying the Lighthouse
Accessibility score of 100 on the Sign-In screen via Chrome DevTools.

Vitest is omitted in F01. It will be introduced when pure functions with high combinatorial paths
require lightweight validation.

**Prior Exemples:** no behavioral tests currently exist in the codebase. F01 tests establish the
pattern for future features.

## Out of Scope

- Everything listed in the PRD under "Out of Scope".
- WebSockets and connection refusal for invalid sessions, deferred to F03 (section "Real-Time").
- Presence, profile, notifications, and theme switching within the avatar menu and Rail (F05, F08, and F09).
- Sidebar items and Activity.
- Periodic session cleanup routines, IP-based rate limiting, and Redis.
- Playwright testing across browsers other than Chromium, and isolated instances of the web and API
  specifically for testing.

## Notes

- F01 resolved several open questions from the LLD: session identifier format and storage,
  cookie attributes, SameSite and CSRF protection, how the BFF transmits sessions to the chat API,
  rate-limiting keying mechanisms, configuration validation libraries, error body formats, and expired
  session management. The record resides in the LLD sections cited above.
- The risk of a user locking another user's account for 15 minutes was accepted in
  [ADR 0023](../../adr/0023-login-attempt-limits-by-email.md).
- Referenced Figma nodes originate from the audit conducted for this spec (slack file, Screens page)
  and identifiers may change if frames are recreated; names remain authoritative.
