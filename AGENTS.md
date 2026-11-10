# AGENTS.md

## Documents

Under `docs/` reside the documents that define the product, architecture, and implementation details. Below are the key project documents:

1. `docs/product-brief.md`: what and for whom. MVP scope, what is left out, plans, and metrics.
2. `docs/hld.md`: High-level design, the high-level architecture document. This is where architecture decisions, main flows, and the system overview live.
3. `docs/lld.md`: Low-level design, the low-level architecture document. This is where implementation details live.
4. `docs/design-system.md`: interface visual language. This is where color tokens, typography, spacing, radius and shadow, interaction states, components, responsive behavior, and the workflow with Figma live.
5. `docs/roadmap.md`: the features to be developed, dependencies between them, and the status of each.
6. `GLOSSARY.md`, at the root: the product domain vocabulary. Use these terms in documents and planning.

Each project has its own `AGENTS.md`, containing its commands, stack peculiarities, and code practices:

* `apps/chat-api/AGENTS.md`: NestJS API.
* `apps/web/AGENTS.md`: Next.js frontend.

**Where each piece of information lives**

* **What the system does** lives in `docs/`: scope, architecture, schema, contract, business rules, and configuration.
* **Never in both.** A piece of information has only one owner, and the other file points to it by document and section name, without repeating content.
* **HLD and LLD change via their skill.** When a task changes something recorded in `docs/hld.md` or `docs/lld.md`, load the `hld` or `lld` skill and use update mode before editing the file.
* **Information that should belong in memory files** such as AGENTS.md and rules must not be repeated in architecture documents. Create pointers from these files to the corresponding memory files.

## Docker Execution

Everything runs inside Docker Compose containers: dependency installation, development server, tests, lint, TypeScript compilation, and any other project command. **Never run `pnpm`, `node`, or `npx` directly on the host machine.**

The development environment uses `compose.yaml` at the root with four services:

| Service      | What it is                                                        | Host Address                     | Container-to-Container Address |
| ------------ | ----------------------------------------------------------------- | -------------------------------- | ------------------------------ |
| `web`      | Next.js Frontend                                                  | `http://localhost:3000`        | `http://web:3000`            |
| `chat-api` | NestJS API, REST and WebSocket                                    | `http://localhost:3001`        | `http://chat-api:3000`       |
| `postgres` | PostgreSQL, with `chat`, `agents`, and `business` databases | `localhost:5432`               | `postgres:5432`              |
| `s3`       | S3 Emulator (`seaweedfs`)                                       | `http://http://localhost:8333` | `http://s3:8333`             |

The `web` and `chat-api` containers install dependencies upon startup and then stay idle without launching the application. Commands run in them via `docker compose -f compose.yaml exec <service> <command>` and each project's commands are listed in its own `AGENTS.md`.

### Starting and Stopping

```bash
docker compose -f compose.yaml up -d --build --wait
docker compose -f compose.yaml exec web pnpm dev --hostname 0.0.0.0
docker compose -f compose.yaml exec chat-api pnpm start:dev
```

The `--wait` flag pauses the first command until `postgres` and `s3` are ready. Dependency installation continues inside `web` and `chat-api` after that, and on the first startup, it takes a few minutes: follow along with `docker compose -f compose.yaml logs -f <service>` before running the other two. Migrations do not run automatically on startup. The other two commands block the terminal, so run each in the background. The `--hostname 0.0.0.0` flag is required so that the web service accepts connections from outside the container.

**Stopping**

```bash
docker compose -f compose.yaml down
```

**Rare Procedures**

**Starting from scratch.** To erase data and recreate the database:

```bash
docker compose -f compose.yaml down -v
```

The `-v` flag deletes database and storage volumes: on the next startup, the script in `docker/postgres/init/` recreates databases and roles without tables. Then, start the environment as described in "Starting and Stopping".

**Installation failed.** The `web` or `chat-api` container stays up even when `pnpm install` fails, so the error only appears in the logs:

```Shell
docker compose -f compose.yaml logs chat-api
```

After fixing the issue, run the installation again inside the container with `docker compose -f compose.yaml exec chat-api pnpm install`.

### Inter-service Networking

When connecting between services (database, storage), whether in environment variables, configuration files, or code, **the host is always the Compose service name**, as shown in the "Container-to-Container Address" column of the table, and never `localhost` or `127.0.0.1`: inside a container, `localhost` refers to the container itself.

**Exception:** when a process talks to something running inside the same container, the address is `127.0.0.1`. This is the case for a service healthcheck checking its own container, and tests that spin up a temporary server inside the container where they run.

## Design and Figma

Figma is the project design tool, accessed via the `figma` MCP plugin server.

**Figma File:** [https://www.figma.com/design/plW17JDOoRHydUJIIi2u87/slack](https://www.figma.com/design/plW17JDOoRHydUJIIi2u87/slack)

**Whenever a task involves design or interface, read `docs/design-system.md` first**, whether for designing in Figma or implementing a screen or component in code. It defines tokens, components, and the workflow with Figma (synchronization, steps from Figma to code, assets, and visual validation), and its rules are mandatory.

## Planning

Feature planning and documentation live in local markdown files, which serve as the project's issue tracker. Skills that publish to a tracker write to these files, in the paths below and without triage labels.

The hierarchy is feature → task, and each feature has two documents: the PRD and the spec:

* **Feature:** a demonstrable delivery in the product, described solely by what a person can do, without technical details. This is not a Phase from `docs/product-brief.md`, which represents a product stage (MVP, operating as a product). The feature number is global, sequential, and does not change once assigned. In planning documents, "feature" always refers to this term; for the common meaning of the word, use another term (capability, behavior).
* **PRD:** the product requirements for a feature (problem, flows, stories, acceptance criteria, out of scope), created via the `/prd` skill. It points to `docs/product-brief.md`, which covers the entire product, rather than repeating metrics and non-functional requirements.
* **Spec:** how the feature will be built (implementation and test decisions). It follows and points to HLD, LLD, and ADRs.
* **Task:** a deliverable part of a feature, derived from the spec. Technical work happens inside tasks. Before writing code, each task gets a **plan**, created by the `/plan-task` skill, which is approved and committed on its own in the **plan commit**. After that, the plan is frozen as a record of what was approved: only the user can unfreeze it, and any user decision that turns out to be wrong is corrected in the spec, LLD, or a new task. Outside the feature folder, a task is identified as `F<NN>-T<NN>`, such as `F01-T02`.

**Folder Structure**

```
docs/features/<feature-num>-<title>/
  prd.md
  spec.md
  tasks/
    <NN>-<title>/       ← one folder per task
      task.md           ← the task
      plan.md           ← the task plan
      …                 ← task records, if necessary
```

Numbers have two digits, and titles are in unaccented kebab-case (`01-sign-in-and-out`). Tasks are numbered starting from `01` within each feature, in dependency order.

Aside from the task and plan, the task folder only receives records specific to that task, such as a manual test script, a measurement result, or visual validation screenshots. These are in Markdown or image formats, named in kebab-case (`manual-script.md`, `p95-measurement.md`). Code, test fixtures, and configuration never go into this folder: they belong in their respective project directories. Only `plan.md` is frozen in the plan commit; other records are written during implementation and included in the task PR.

**Status**

* The status of each feature (planned → PRD ready → spec ready → in progress → completed) lives in the roadmap, `docs/roadmap.md`, which also owns the feature list.
* Task status (todo → in progress → completed) lives in a Status field at the top of its `task.md`. It transitions to "in progress" at the plan commit, and to "completed" in the PR that delivers it. A feature transitions to "in progress" at the first task plan commit, and to "completed" in the PR of its final task.

## Git Conventions

* **Main branch:** `main`. Never commit directly to it.
* **Working branches:** `feature/*`, `bugfix/*`, `hotfix/*`, and `docs/*`, created from `main` and integrated via pull request.
* **Commits:** short, descriptive messages focused on the "why" of the change.
* **Merge:** merging is the user's decision. The agent only performs it when the user explicitly authorizes that PR or task. Without explicit authorization, the agent must not merge, and a denial of `gh pr merge` must not be bypassed through another path.

## Library Documentation Lookup

When you need to consult a library's documentation, use context7 to retrieve up-to-date and accurate information.

## Agent Skill Usage

Always load the skills required for the given task.
There are various convention and best-practice skills that can be loaded as needed by the task, in addition to those specific to each domain or technology used.

When a skill diverges from memory files or rules, what is stated in AGENTS.md or rules takes precedence.
