# TanStack AI Experiments — Repository Refactor and Collaborative Chat V1 Handoff

Status: repository-specific implementation brief. Source/configuration reviewed; no repository files were changed and no install, build, or runtime tests were executed while preparing this handoff. The integration and refactor remain work for the implementing agent.
Review date: September 11, 2026.
Repository: https://github.com/grmkris/tanstack-ai-expiriments
Reviewed main commit: `c58de3b1b04654b01997564a945e935f61e0d8c3`. Reinspect the current checkout and preserve newer work.

This document is the complete, authoritative implementation brief for this milestone. No previous conversations or specification files are required. Real-backend V1 uses TanStack Start in a Bun-workspaces TypeScript monorepo, an Effect 4 device-hosted API running on Bun, Drizzle over local bun:sqlite persistence, disk-backed Durable Streams on the owning host, server-owned queueing, StreamDB/TanStack DB synchronization, and multiple authenticated humans sharing one AI conversation. No hosted application database or managed stream service is required. Refactor the existing Better-T-Stack project in `tempalte/`; do not generate a second competing starter. This revision adopts Effect 4 for backend application services and lifecycle management; it does not replace TanStack AI, native sandbox recovery, Durable Streams, or StreamDB. Do not import earlier single-client queue or memory-only delivery defaults.

## Agent execution instructions

Implement this specification in the provided repository. The deliverable is a runnable application with verification evidence, not another proposal, a mock-only interface, or a list of future tasks.

1. Read repository instructions and inspect the existing code before editing. Preserve unrelated work. Complete the repository migration and Milestone 0 below, reusing the existing starter and consulting official scaffolding only to verify the required Start configuration, then implement the collaborative chat. TanStack Start, a monorepo, a device-hosted API, and local SQLite are required. Use Bun workspaces, Effect 4 application services, Drizzle with bun:sqlite, and a static TanStack Start SPA build. Prefer Effect HttpApi/HttpRouter with @effect/platform-bun for the API rather than adding Hono. Verify pinned runtime compatibility first. Do not silently downgrade to Effect 3, replace Bun, change SQLite, or swap AI frameworks; record any reproducible dependency blocker and the smallest viable adjustment. A narrow compatibility helper is preferable to silently replacing the application runtime.
2. Create a short implementation checklist mapped to the acceptance suite. Begin the integration proof, then continue through the implementation milestones. Do not stop after producing the checklist or an architecture document.
3. Verify current official documentation, installed types, and package source as necessary; pin compatible versions. Names in this document describe intended integration points, not proof that a particular installed release exports them. Complete a compiling integration proof before building extensive UI around an unverified API.
4. Follow the ownership boundaries and scope in this brief. Choose and document reasonable defaults for unspecified details; do not repeatedly request architectural decisions that can be resolved from these requirements. A discovered incompatibility must be documented with evidence and the smallest viable adjustment. Do not silently remove durability, real authorization, multi-user collaboration, or the required acceptance behavior to make a demo pass.
5. Build an early end-to-end slice with two separately authenticated users, one shared room, one real model/harness backend, and independently reconnectable readers. Then implement the shared queue, control semantics, persistence, recovery, and UX polish. Use deterministic fixtures alongside real execution, not as a substitute for it.
6. Supply local infrastructure configuration, migrations, a safe development seed, an `.env.example` without secrets, and reproducible startup/test commands. Keep seeded demo accounts development-only. Do not require a paid email-delivery service merely to test the invite flow locally.
7. Run type checking, relevant linting, the production build, and automated interaction/integration tests. Exercise concurrency and failures. Count backend execution starts as well as rendered messages. Fix failures and rerun affected tests. Do not claim that an unexecuted test passed.
8. Maintain a concise progress record with implemented work, test evidence, unresolved issues, and the next concrete step. At handoff, summarize the build/run commands, pinned versions, passing and failing tests, and any blocked external verification. Do not describe the application as complete while required acceptance tests remain failed, unsupported, or unverified.

### Access and verification boundaries

Repository write access, a usable development runtime, and credentials for at least one real model/harness are needed for full implementation and verification. Real Tailscale/mobile and provider-failure tests may additionally require deployment access or a physical device. Inspect the supplied environment and documentation first; do not guess secrets or fabricate integrations. When external access is genuinely missing, finish the work that can be verified locally and identify the exact blocked tests and configuration needed. Do not substitute a mock and label the blocked capability verified.

Use local development and isolated test resources by default. Do not deploy to production, delete unrelated data, or provision chargeable external infrastructure without appropriate authorization. Test agents must use disposable workspaces and bounded execution limits.

### Required delivery artifacts

- Implemented source, migrations, and local infrastructure configuration.
- Setup instructions, environment-variable documentation, and a two-user demo procedure.
- Automated tests mapped to the acceptance suite, including executable backend-start counters in concurrency tests.
- A verification report separating passed, failed, and blocked/unrun checks, with known integration limitations.

The product requirements below are the implementation target. The source references support library research; proposed application decisions are not upstream APIs.


## R. Repository-specific starting point and refactor plan

This section replaces any inherited assumption that the implementation starts from an empty repository. All product and reliability requirements in sections 1–12 remain in force. This is a refactor followed by feature implementation, not a requirement to preserve placeholder architecture.

### R.1 Evidence and precedence

At the reviewed commit, the repository contains the collaborative chat specification and a Better-T-Stack starter in the directory literally named `tempalte/`. The reviewed app is an authentication/UI starter, not an existing implementation of the required durable collaborative chat. [R1–R7]

The checked-in `COLLABORATIVE_CHAT_V1_SPEC.md` still names PostgreSQL as the authority. That requirement is stale and conflicts with the user's later explicit device-hosted SQLite direction. Replace its contents with this complete brief, or make it a short pointer to a single canonical `docs/spec.md`. Do not leave two apparently authoritative, contradictory requirements documents. A historical copy may be archived with a prominent superseded label. [R8]

The current checkout may have advanced beyond the reviewed commit. Inspect it first; adapt the plan to preserve newer implementation and unrelated work. Do not reset the repository to the audited commit. Read current local agent instructions before editing, and reconcile generic starter guidance with Effect v4's actual generator/service style. This handoff is not authorization to deploy, expose services publicly, delete data, or publish secrets.

### R.2 What to keep, what to change

| Existing path, relative to `tempalte/` | Reviewed state | Required treatment |
|---|---|---|
| `package.json` | Bun workspaces/catalogs, Bun 1.4.2 declaration, Turbo, Ultracite/Oxlint/Oxfmt | Keep the workspace/tooling approach; pin verified versions, replace floating `latest` tool entries, make commands work at repository root. |
| `apps/web/package.json`, `vite.config.ts` | React + TanStack Router + Vite; no Start plugin in the inspected config | Convert to actual TanStack Start SPA mode; preserve useful routes, forms, theme, and UI. Installing a package without converting the entry points is not completion. |
| `apps/server/src/index.ts` | Small Hono app with CORS, Better Auth forwarding, evlog integration, health-like root response | Convert the daemon composition root to Effect HttpApi/HttpRouter on Bun. Preserve auth behavior through a narrow request/response bridge. |
| `packages/db/src/index.ts` | `@libsql/client` + `drizzle-orm/libsql`, connection created at import time | Replace with scoped/injected local `bun:sqlite` + Drizzle; remove unused libSQL runtime dependencies after migration. |
| `packages/db/drizzle.config.ts` | Turso dialect and working-directory-relative env path | Use SQLite-compatible configuration, generated migrations, and one resolved absolute instance data path. |
| `packages/db/package.json` | Drizzle 0.45.2 range, Kit 0.31.10 range, `turso dev` helper | Do not blindly upgrade ORM major/RC versions. Verify selected APIs; remove Turso server as a default requirement. |
| `packages/auth/src/index.ts` | Better Auth + Drizzle adapter; creates its own database via `createDb()`; module-level auth instance | Keep Better Auth and schema; inject the managed database/config. No hidden extra application connection or per-request initialization. |
| `packages/ui` | Existing shared shadcn primitives/tokens | Preserve this package and its useful components. New chat feature components stay in the app unless genuinely shared. |
| `packages/env`, `packages/config` | Shared starter packages | Keep boundaries; adapt configuration for local hosting and root-level scripts. Do not rewrite every existing Zod schema merely to be Effect-pure. |
| `apps/web/src/lib/auth-client.ts` | Server URL handling includes deployment-specific/Vercel fallbacks | Use an explicit same-origin client configuration in the shipped device app; remove irrelevant cloud fallbacks. Test local and tailnet entry points. |
| `apps/web/vite.config.ts` | PWA auto-update enabled, including dev support | Retain PWA as a deliberate shell feature only; control updates and exclude authenticated/API/live stream data from service-worker caches. |

These observations come from source/config inspection. They are not claims that the original starter builds or that its authentication succeeds in any deployment. [R2–R7, R10–R12]

### R.3 Normalize layout without needless churn

Promote the workspace contents from `tempalte/` to repository root. Merge root and starter ignore rules instead of overwriting either. Preserve the Git repository, useful hidden files/skills, package names, and shared configs. Inspect symlinks and update any relative targets affected by the move. Do not copy a nested `.git`, generated dependencies, local databases, or credentials.

Keep `apps/server` as the daemon path; renaming it to `apps/api` provides no required behavior and is not a milestone. Keep `@template/*` package identities initially unless a coherent project rename is actually needed. Keep Turbo and `packages/ui`: the prior empty-repository advice not to introduce them is not a reason to remove existing working infrastructure.

Audit path-dependent references after moving: workspace globs, root scripts, tsconfig references, component aliases, Drizzle schema/migration paths, dotenv loading, build outputs, `.mcp.json`, and agent-skill links. Do not execute unreviewed setup hooks or read/print secret files just to discover configuration.

Create concise root instructions covering the selected architecture, required commands, browser/server import boundaries, and the prohibition on unverified completion claims. Generic Next.js-specific instructions should not guide this Start app. Preserve useful accessibility and code-quality rules; establish a narrow exception for Effect's supported generator APIs rather than disabling lint wholesale.

### R.4 Frontend conversion: real Start, same UI investment

Use the current official Start configuration as a reference. Convert the router/bootstrap/root document and generated route setup coherently; retain compatible file routes. Do not run both the standalone Router generator and Start's integration as competing owners. Pin a compatible Start/Router/plugin set. Preserve theme, toasts, auth forms, and shared components where useful. [R3, L2]

Ship Start SPA output served by the device daemon. Operational commands, auth, and live reads belong to `apps/server`. Do not introduce runtime Start server functions while shipping only static Start assets. Account for Start's shell prerender: browser globals must be guarded and private data must not be serialized into the shell.

Serve API and static assets before the shell fallback. A direct `/chat/<roomId>` navigation must load the app. An unknown `/api/*` path must return an API error, not a successful HTML document. Vite proxying in development must preserve the same public route layout as the installed build.

PWA rules are part of chat correctness: no cached session responses or room/run streams, no automatic reload that discards a draft, no mutation replay by a service worker unless explicitly designed around the same persisted idempotency contract. Use an update prompt/safe activation policy; removing PWA temporarily during conversion is acceptable, but do not claim offline execution or durable offline command delivery. Clear account-scoped private client state on logout/user change. [R10]

### R.5 Daemon and authentication refactor

Build one Effect application composition root for configuration, database, auth integration, HTTP, dispatcher, native run supervision, publisher, and reaper. Separate request lifetime from accepted work as required in section 0.11. Importing a schema/config module must not open SQLite or launch workers.

Preserve Better Auth rather than implementing passwords and sessions from scratch. Change its factory to accept a database and trusted runtime configuration supplied by the application. Mount its supported Fetch-style handler through a narrow HTTP adapter. Preserve multiple `Set-Cookie` headers, errors, body handling, and redirects. Do not implement a second session authority in Effect. Current upstream examples may use a newer adapter import path than the starter; use the path supported by the pinned release, not an automatic major-version migration. [R6, R13]

Existing auth config forces `SameSite=None` and `Secure=true`, and its browser URL resolver contains cloud fallbacks. Review these settings for the new same-origin deployment rather than copying them blindly. Prefer same-origin cookie semantics appropriate to each supported transport, secure cookies on HTTPS, explicit trusted origins, and localhost-only development exceptions. Test both localhost development and private HTTPS access. Do not infer a production hostname from an arbitrary untrusted Host header or allow all origins. [R6, R11]

Real room authorization is additional to login. The server entry point's user-identification logging middleware is not evidence of room access enforcement. Every command, history fetch, native run read, and room-state read must validate membership and role. Revalidate/revoke active readers according to the product contract.

Keep useful structured logging, but do not run two independent telemetry systems emitting the same events. The existing Hono-specific evlog wiring may be replaced by Effect logging or adapted at a small boundary; either choice must retain request/run/room correlation, error visibility, and redaction. Logging identities is not authorization. [R4]

### R.6 The linked Effect SQLite client: explicit decision, not assumed compatibility

The user supplied:
`https://effect.website/docs/v4/api/sql-sqlite-bun/SqliteClient`

It is a real Effect SQL driver over `bun:sqlite`. It manages its connection, provides the Effect SQL services, and supports Effect SQL transactions. Its documented API is not a Drizzle `Database`. The reviewed source creates the Bun handle internally; do not invent a public raw-handle accessor or pass the Effect service directly into `drizzle({ client })`. [R9]

**Default for this repository:** retain Drizzle query/schema/migration ownership and the existing Better Auth Drizzle integration; replace libSQL with Drizzle's documented Bun SQLite driver. Own the connection in an Effect Layer and expose application repositories as Effects. This is genuine Effect application architecture even though the ORM is adapted at its boundary. [R5, R6, E5]

**One bounded compatibility check is allowed before committing the driver decision:** inspect published, release-matched upstream exports for an official Effect-v4 + SQLite + Drizzle integration. Adopt it only if it demonstrably shares transaction/connection ownership, supports required SQL/results/migrations, and works with the chosen auth adapter. Prove commit, rollback, concurrent transaction isolation, no ambient-transaction escape in async callbacks, and auth compatibility under the actual runtime. A Postgres example is not proof of SQLite support; a v3 bridge is not a v4 dependency.

If no suitable bridge is verified, use the direct Drizzle/Bun default and record that decision. Do not spend the chat milestone building a custom ORM driver, casting away transaction types, importing a removed legacy integration, or maintaining two general-purpose query layers. The availability of native Effect SQL is not authorization to silently remove the user's Drizzle requirement. An all-Effect-SQL runtime with Drizzle only for schema/migrations would be a separate architecture choice, not a transparent upgrade.

Connection discipline is about explicit ownership and transaction correctness, not a claim that multiple SQLite connections are universally invalid. V1 should not accidentally open independent connections through module imports. If an upstream dependency genuinely needs a separate auth connection, document its limited ownership, lifecycle, contention policy, and lack of cross-connection atomic transactions; verify it rather than pretending it participates in the chat transaction.

The linked driver defaults to a five-second busy wait, and its docs warn that this blocks Bun's event loop. Its writable explicit transactions use `BEGIN IMMEDIATE`. Those defaults matter for Stop responsiveness; configure and test contention deliberately instead of expecting Effect interruption to preempt synchronous SQLite calls. Its unsupported streaming queries do not prevent chat streaming, which comes from AI/Durable Streams rather than a SQL cursor. [R9]

Choose one migration executor/history for an installation. Do not run both Drizzle and Effect migrators independently against the same schema. Installed application startup uses tested versioned migrations, not `db:push`; development push commands, if retained, must be labeled accordingly. Preserve existing user data, back it up, and test migration rollback/failure behavior. Destructive resets are limited to disposable test databases.

### R.7 First verified vertical slice and refactor gates

Do not refactor every supporting package before proving one working path. Record a baseline first, then keep changes reviewable:

1. **Baseline:** inspect git status and instructions; install from the existing lockfile in an appropriate environment; run existing build/type/lint commands; record actual failures and required env values without exposing secrets. Do not call the baseline verified unless it ran.
2. **Layout:** lift the starter, merge config paths, retain the existing Turbo/UI/auth structure, and verify root commands. Separate layout-only changes from behavior where practical.
3. **Local persistence/auth:** make data paths independent of CWD; migrate to scoped Drizzle/Bun SQLite; run a real signup/login/session/logout and restart test. Verify explicit resource acquisition/closure and no hidden second DB initialization.
4. **Start and Effect HTTP:** preserve auth while converting frontend entry points and backend routing. Serve the built app and `/api/auth/*` through one origin. Test deep links, API errors, cookie propagation, and no server code in browser assets.
5. **Durable collaboration proof:** add the required TanStack AI/StreamDB/Durable Streams integration; two separate authenticated readers see one persistent room and one deterministic streaming response. Close the initiating connection, reload/rejoin, and verify no duplicate output. Then prove one real backend; do not equate a fixture with real cancellation.
6. **Finish the product:** implement the full shared queue, exact-target stop/replace, response attempts, authorship, invitations, private drafts, fast composer, activity states, scroller behavior, recovery, and security tests from the following sections.

Do not stall on an unavailable model key: continue with deterministic transport/UI tests and mark only real-provider tests blocked. A mock-only build does not complete the milestone. Do not add a terminal, file explorer, editor, or multi-agent workbench.

### R.8 Repo-specific acceptance additions

In addition to the complete suite below, require:

- A clean root install/build/type/lint/test path; no second nested product or duplicate lockfile.
- No mandatory Turso CLI/server, libSQL remote URL, or PostgreSQL service for the reference installation.
- A release run from a different working directory opens the same configured instance database and serves deep chat links.
- Login and durable stream reads work through the same origin locally and in the configured private HTTPS deployment; actual physical-device tests are distinguished from browser emulation.
- API startup/shutdown and a frontend rebuild do not multiply DB clients, dispatchers, or reapers.
- SQL transaction failure rolls back message/receipt/queue/outbox together; a subsequent publication retry does not rerun the agent.
- Background SQLite contention has measured effects on SSE delivery and Stop latency; no unexamined five-second event-loop stall.
- Auth/session state is not built into the static shell or cached by the service worker; logout/account-switch cannot reveal another account's cached private state.
- Updating the PWA does not erase a draft, cancel shared work, or double-submit a command.
- Two real users and ten tabs remain independent readers of one assistant execution, not ten producers.

Deliver `docs/progress.md`, `docs/verification.md`, and a short `docs/decisions.md` (including the SQL-driver decision). Baseline failures and later fixes should be distinguishable. Avoid generating a forest of competing specification files.


## 0. Foundation refactor and device-hosted architecture

### 0.1 Fixed requirements and chosen defaults

This revision supersedes the earlier cloud-oriented deployment assumptions. Adapt the existing foundation first. The checked-in source is the starting point, not a constraint to preserve obsolete deployment decisions. Do not run a generator over the repository or create a second application.

User requirements and adopted stack direction: TanStack Start, a Bun-workspaces monorepo, Effect 4, Drizzle, SQLite, Bun, and an API running on a user's own computer/device. Retain all shared-chat, durability, attribution, and interaction requirements below.

Proposed implementation defaults:

| Concern | Default |
|---|---|
| Workspace/package manager | Bun workspaces; one committed lockfile |
| Frontend | TanStack Start + React + TypeScript + requested shadcn components |
| First shipping web build | Start SPA mode, served by the device daemon |
| API | Effect 4 HttpApi/HttpRouter with @effect/platform-bun, on a long-lived Bun process |
| Application services | Effect 4 Context services, Layers, typed errors, explicit scopes and supervised workers |
| Shared command schemas | Effect Schema, browser-safe; native TanStack contracts remain native |
| Application storage | Drizzle + bun:sqlite, file-backed |
| Delivery/sync storage | Local disk-backed Durable Streams; native run adapter plus StreamDB |
| Authentication | Retain Better Auth for local sessions; add safe owner setup, invitations and room membership; no mandatory hosted identity service |
| Tests | Compatible unit/integration runner plus Playwright with distinct authenticated contexts |

Bun workspaces, Bun SQLite, Drizzle's Bun driver, Effect's Bun platform and Effect HttpApi have official documentation or maintained upstream examples. This is not evidence that the entire selected AI/harness package combination already works together: verify native execution, cancellation, subprocess behavior, HTTP/SSE streaming, and persistence before extensive feature work. [L3, L5, L6, E3, E4]

Bun is the application runtime target. If a required dependency has a reproducible Bun incompatibility, document it and evaluate a narrow, explicit compatibility boundary before proposing a stack change. Hono plus one daemon-wide ManagedRuntime is an allowed fallback for a proven Effect HTTP integration blocker, not a second default router. Do not replace SQLite with a hosted database or delete durable collaboration to hide a compatibility failure. Do not build a universal runtime-abstraction layer merely to avoid making this choice.

### 0.2 Normalize the existing monorepo

Promote the existing `tempalte/` workspace to the repository root through reviewed moves. Keep `apps/web`, `apps/server`, and the existing shared packages. Consult the current official Start setup/basic example when converting the Router/Vite app, but do not regenerate the whole workspace. If an isolated temporary scaffold is useful as a comparison, do not ship it as another product. Pin the verified dependency set and retain one Bun lockfile. [L1, R1–R4]

Suggested structure:

```text
apps/
  web/                 # TanStack Start, React, chat UI, browser collections
  server/              # existing app converted to Effect/Bun daemon; HTTP, jobs, lifecycle
packages/
  ui/                  # existing shared shadcn primitives and design tokens
  auth/                # existing Better Auth; explicit injected database/config
  env/                 # existing browser/server configuration boundaries
  config/              # existing build/type/lint configuration
  contracts/           # browser-safe Effect Schema / HttpApi contracts; native AI types
  db/                  # Drizzle schema, migrations, Effect repositories/native stores
  chat/                # Effect services composing native TanStack integrations
scripts/               # local launcher, build assembly, diagnostics, smoke tests
docs/                  # authoritative brief, decisions, progress, verification
```

Preserve the existing `packages/ui` rather than moving its primitives back into the web app. Put chat-specific feature components in `apps/web`. Keep the existing Turbo task graph unless an actual defect requires a change; no task-runner migration is required. Do not create empty packages for future editors, terminals, cloud administration, or desktop wrappers. A monorepo does not mean a microservice architecture.

Import boundaries: web can import browser-safe contracts, not the SQLite driver, database connection, server secrets, or execution internals. `packages/chat` composes TanStack's native capabilities; it must not become another agent framework. Avoid barrels that accidentally re-export server code into the browser.

### 0.3 Separate code packages; one installed application

```text
Desktop browser / ten tabs / teammate's browser / mobile over Tailscale
                              |
                    One authenticated origin
                              |
                 Owning device: API daemon
                   |-- serves built Start UI
                   |-- authorization, commands, queue, execution
                   |-- app.sqlite
                   |-- authenticated proxy to local stream helper
                   |-- native harness lifecycle where configured
                              |
                   streams/ on the same host
```

One room belongs to one host instance. Other devices join that host; they do not each become writers to independent copies of the room database. Separate installations can own separate rooms. Multi-master replication, cross-host room federation, and offline merging of authoritative histories are outside V1.

Use TanStack Start SPA mode for the initial shipped UI. It produces an application shell suitable for static serving; using an external API is supported. Server functions/routes still need a Start server when used, so the device API must not accidentally depend on a Start server build that is not shipped. [L2]

Keep all operational endpoints in the daemon. Do not capture database contents, credentials, or user-specific data during shell prerendering. Serve asset paths and `/api/*` before the SPA fallback; a missing API route or asset must not return the HTML shell as a false success.

Development may run Vite and the API separately with a dev proxy. Production serves UI and API from the same origin. No mandatory public UI hosting, remote control plane, or separate SSR process. Keep authentication and execution contracts usable by a future native client without implementing one now.

### 0.4 SQLite is the host's authority, not browser storage

Persist identities, sessions, rooms, messages, attempts, command receipts, the accepted queue, native lifecycle stores, and a transactional publication outbox in host-local SQLite. Browser TanStack DB collections remain synchronized read models/optimistic views, not a second authoritative SQL database.

Use WAL with a local filesystem. WAL allows readers alongside a writer, but writes serialize and the database is not suitable for sharing through a network filesystem. Never place the live database in a shared network mount or attempt file-sync replication between clients. [L7]

Proposed initial connection settings, to be queried back and tested:

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 50; -- proposed starting value in milliseconds, not a universal optimum
PRAGMA synchronous = FULL;
```

`FULL` is the starting durability preference for accepted user commands; any relaxation needs a documented durability tradeoff. Start with a small synchronous busy timeout and a bounded, asynchronous contention retry only around a transaction known to have rolled back. Measure and adjust the proposed 50 ms on reference devices. Do not carry over a five-second synchronous wait just because a driver defaults to it. Consider an isolated DB worker only if profiling requires it. [L8, R9]

Keep transactions short: validate/claim/update/commit, then do network or model work outside the transaction. Use SQLite-compatible conditional updates and uniqueness constraints for admission and deduplication; `BEGIN IMMEDIATE` is an available mechanism when appropriate. Do not generate PostgreSQL-only row-lock/advisory-lock SQL. SQLite transaction behavior must be explicitly tested under racing submissions. [L9]

Retain the transactional outbox: committing SQLite does not atomically publish to a separate stream log. Restarting the publisher must converge through durable receipts/versioning without starting accepted AI work twice. Preserve native persistence contract fields rather than keeping only a simplified chat status.

Do not persist every token into both a SQLite message row and a room-state entity. Use native run delivery for deltas and persist canonical content at supported boundaries.

### 0.5 Local Durable Streams, without a mandatory cloud service

Use file-backed storage, never an accidental in-memory default. Upstream documents `@durable-streams/server` for development/testing and embedded Node use with `dataDir`; it recommends the Caddy implementation for production. Select the Caddy helper as the release target, with development alternatives explicitly labeled. [L10]

Keep stream data separate from SQLite in the instance data directory. Do not invent an unsupported SQLite backend for Durable Streams. A single launcher may supervise the daemon and stream helper: one installed product need not mean one OS process. Verify helper distribution, startup, shutdown, and persistence on each supported OS/architecture. Do not require the user to administer a cloud log service.

Bind the raw helper/admin endpoints to loopback or otherwise keep them private. Expose room-authorized reads through the daemon. A browser must not be able to bypass membership checks by addressing the helper directly.

Treat unavailable persistent storage as an explicit health/readiness failure; do not fall back silently to memory. Test restart/cursor recovery and the pinned stream server's crash semantics rather than assuming file-backed storage guarantees every form of power-failure atomicity.

### 0.6 Device lifecycle, data directory, and security

Use a configurable application data directory independent of the source checkout, current working directory, and installed executable. Example logical layout:

```text
APP_DATA_DIR/
  app.sqlite
  streams/
  logs/
  backups/
```

Resolve platform-appropriate defaults and use restrictive filesystem permissions. Never commit live data, credentials, generated session secrets, or invitation tokens. Credentials must stay server-side; use the selected secure credential storage mechanism and document its threat model.

Ship versioned migrations, migration locking, safe upgrade failure behavior, and backup/restore instructions. Provide a consistent backup implementation using supported SQLite snapshot/backup facilities or a safely quiesced procedure, not a blind copy of a live main database file. Test restoring to a separate directory. SQLite documents backup mechanisms and alternatives. [L11]

For an instance backup, also address the stream store and native runtime metadata. Coordinate a maintenance/quiescence point or document a recovery procedure that reissues projection epochs/cursors safely. Restoring SQLite to yesterday while blindly retaining newer streams can expose discarded state and is not an acceptable restore strategy.

Enforce one active daemon per data directory, reconcile interrupted state on startup, schedule reaping/finalization, and manage child-process lifetime explicitly. A frontend rebuild or closing all browsers must not cancel accepted work. Do not tie run cancellation to viewer disconnection.

Distinguish failure cases: browser disconnected; API restarted while a supported agent process/sandbox survived; host sleeping; machine rebooted; device unavailable or lost. Durable output cannot keep a powered-off computer executing. Do not silently relaunch a potentially side-effecting task after an ambiguous crash. Reconcile or present an interrupted/recovery-required state. The native sandbox journal and delivery log remain separate where that execution mode is used. [S9]

Bind to loopback by default. Remote access is explicit; Tailscale Serve is one supported private HTTPS path. [S14] Keep real application authentication even on a tailnet. Include Host/Origin validation, CSRF protection for cookie-authenticated mutations, and an explicit origin allowlist; do not treat all localhost traffic as trusted.

Production owner creation should require a local one-time bootstrap secret/flow. Development demo credentials must never be reachable in release mode. Do not expose an unclaimed administrator-creation endpoint to the network.

OS service installers, auto-update, code signing, native desktop shells, and mobile-hosted agent execution are not required to complete the initial chat milestone. Supply a reliable launcher and documented prerequisites first. Record actual tested OS/architecture/runtime support rather than claiming every device is supported.

### 0.7 Milestone 0 completion gate

Before broad feature work, verify:

1. The converted Start app builds inside the existing monorepo; shared contracts type-check, and server-only imports cannot reach the browser build.
2. One documented development command starts the UI, daemon, and persistent stream helper. One release command serves the built UI and API without the Vite development server.
3. A deep chat URL loads from the built shell; API errors remain API errors. No secrets or user history are baked into static assets.
4. A fresh SQLite migration and a migration rerun both behave correctly. Data survives daemon restart; the actual SQLite runtime version and relevant connection settings are recorded.
5. A second daemon cannot silently take ownership of the same data directory. Startup/shutdown do not accidentally spawn duplicate workers.
6. A deterministic native-event run is stored durably and read by two separately authenticated clients. Reload/rejoin adds no duplicate output. A removed/nonmember client cannot read the room.
7. Room-state updates project through Durable State into StreamDB, and a temporary publication outage catches up from the outbox.
8. Type checking, unit/smoke tests, and the production build pass from a clean checkout. Commit or otherwise record this verified scaffold separately, then proceed to the real-backend integration proof and remaining chat milestones.

Provide proposed root commands such as `dev`, `build`, `start`, `typecheck`, `test`, `test:e2e`, `db:migrate`, and `doctor`, with implementations and real documented behavior. No stub scripts reporting success. Mock scenarios prove UI/transport behavior only; they do not establish real harness cancellation or recovery.



### 0.8 Effect 4 adoption and version discipline

Use Effect as the backend application runtime for explicit dependencies, typed failures, scopes, worker supervision, safe retry schedules and observability. Use `Effect.gen`, named `Effect.fn`, `Context.Service` and `Layer` according to the selected v4 release. Do not copy v3 service, platform or fork APIs from memory. [E1–E4]

At the review date, Effect's official site advertises v4 through the `rc` distribution tag; the API pages inspected display `4.0.0-rc.114`. This identifies documentation inspected, not an assertion that this is the newest published registry release or that it has been tested here. Resolve a published compatible version at bootstrap and pin exact versions. Align Effect-maintained platform and test packages with that v4 release line. Do not leave a floating prerelease tag/range in the final manifest. Record the Bun, Drizzle ORM, Drizzle Kit, TanStack, stream package and actual SQLite engine versions too. [E1, E2, E3]

Effect HTTP/API functionality currently lives under `effect/unstable/http` and `effect/unstable/httpapi`; the migration documentation explains the stability distinction for these modules. Keep their integration localized. Do not treat a core v4 release as a blanket stability guarantee for every unstable module. Pin the ORM and its migration tool as a compatible pair rather than assuming matching distribution tags imply tested compatibility. [E2, E4, E5]

Before writing substantial Effect code, read the v4 upstream `LLMS.md`, migration guidance, and examples corresponding to the installed release. Provide a version-pinned read-only reference checkout or equivalent source access for the coding agent, excluded from compilation and bundling. Reference source is not an application dependency and must not be imported into application code. Add a concise project `AGENTS.md` with the chosen service, transaction, cancellation and streaming patterns. [E10]

### 0.9 One backend composition root; preserve native library ownership

The daemon has one application Layer graph and lifetime, not a fresh database and run supervisor per request. Proposed services below are application names, not upstream exports:

- `ChatCommands`: authentication-aware send, cancel queued input, stop, replace and regenerate operations.
- `ChatRepository`: synchronous Drizzle transactions exposed through typed Effect operations; canonical room, command, membership and queue state.
- `RunSupervisor`: process-local supervision of native TanStack drivers keyed by persisted run identity, using existing native ownership/recovery APIs.
- `RoomPublisher`: durable SQLite outbox consumption into the existing Durable State protocol.
- `ReaperScheduler`: schedules and reports the native reaper; it does not implement a competing recovery engine.

These can be a few cohesive modules inside existing packages; do not create a separate package/service for every method.

Effect owns the application's service wiring and supervised local work. TanStack AI owns native agent execution/events, streaming message semantics and supported harness lifecycle. Native TanStack stores and sandbox recovery remain authoritative for their own fields. Durable Streams owns delivery logs. StreamDB/TanStack DB owns the client materialized room view. Drizzle/SQLite owns accepted application state and atomic command admission.

Do not add Effect AI, Effect Atom as another shared room store, Effect Cluster, Workflow or EventLog as an alternative chat infrastructure in this milestone. Plain Effect fibers, Queue and PubSub are not restart-durable merely because they are Effects. A bounded in-process queue may wake a worker, but accepted pending requests and cancellation intent must remain in SQLite and be rediscovered after missed wakeups or restart. Do not confuse Effect's in-memory concurrency primitives with SQL transactions or native run fencing.

### 0.10 HTTP and browser boundary

Prefer Effect HttpApi for schema-defined command/status endpoints and HttpRouter for the required streaming/static routes, with Bun platform services. Keep standard authenticated HTTP commands and the existing native SSE/durable protocols. Do not wrap the entire stream protocol in Effect RPC or invent replacement event types. The maintained HttpApi example demonstrates schema-first endpoints, generated typed clients, middleware and platform integration. [E3, E4]

Prove an unbuffered bridge for native TanStack Web Responses/AsyncIterables and the authenticated stream proxy using the selected APIs. Preserve status codes, cursors, headers, cancellation semantics and incremental delivery. Do not serialize an SSE body as JSON, call `.text()` on a live stream, or terminate the producer when the reader is aborted.

Shared Effect Schema/HttpApi declarations may live in `packages/contracts`. They must not import database tables, server credentials or process integrations into the browser. Expose a narrow Promise/fetch-facing client boundary for React where useful; do not force components or TanStack's client processor into a second runtime/state machine. A generated Effect client is acceptable behind that boundary; there should be one command contract, not manually duplicated validation in several libraries.

Preserve native TanStack tool/interrupt schemas. Where Standard Schema interop is needed, verify the actual adapter against the installed versions, including transforms, errors and serialization. Do not assume all schema representations are interchangeable.

### 0.11 Scope and cancellation contract — mandatory

Structure lifetimes deliberately:

```text
Daemon application scope
  |-- SQLite and other shared resources
  |-- persistent-queue dispatcher
  |-- native run driver supervision (one entry per owned run)
  |-- outbox publisher
  |-- scheduled native reaper
  |-- HTTP request scopes
       |-- short authenticated command handling
       |-- independent viewer subscriptions
```

A normal child fiber is tied to its parent. Forking in an explicit scope ties it to that scope instead. Accepted runs and worker loops must belong to the daemon's managed scope, not to the submitting HTTP handler or an SSE viewer. Do not use untracked detached fibers as a substitute for lifecycle ownership. [E7]

A send command commits its authenticated message, idempotency receipt, pending-work entry and projection outbox change atomically, then acknowledges acceptance. The daemon dispatcher discovers and claims that durable work. It also scans on startup and periodically so losing a wakeup does not lose the command. It must not start an agent before acceptance is committed. Native run launch/recovery still needs its existing persisted identity and claim safeguards.

A viewer disconnect only releases that viewer's reader resources. Closing all viewers does not cancel accepted work. A Stop command authorizes and persists cancellation for an exact native run, invokes the supported backend cancellation path, and waits for or observes its confirmed terminal outcome. Do not interrupt the only native driver/pump before it has performed necessary cancellation and finalization. Bound cancellation attempts and report an unknown/still-stopping state when confirmation is missing; do not falsely release the room for replacement execution.

Effect Promise interoperability can propagate an AbortSignal, but the operation stops only if it honors that signal. An interrupted fiber is not proof that an external process or remote agent stopped. Preserve native TanStack cancellation for durable runs, including process/sandbox handling. Do not translate all interruptions into retryable provider failures. [E6, S5]

Graceful daemon shutdown is an explicit policy: stop admission, persist coordination state, drain or detach supported native drivers as documented, finish bounded cleanup, then close shared resources. Do not accidentally destroy a deliberately retained sandbox through a generic resource finalizer. A local process crash/reboot cannot preserve Effect fibers; startup reconciles durable records and native metadata, and must not blindly rerun ambiguous side effects.

### 0.12 Drizzle + bun:sqlite inside Effect

The default persistence route is ordinary `drizzle-orm/bun-sqlite`, not an unverified specialized Effect ORM driver. Drizzle officially supports Bun's synchronous SQLite driver and sync query methods. Effect also offers a separate `@effect/sql-sqlite-bun` driver, but that is an alternative database-access choice, not an additional mandatory layer under Drizzle. Do not mix independent transaction managers and assume they share a transaction. [E5, E8]

Own the database connection through a scoped application service. Expose repository methods as Effects, creating operations lazily and mapping expected driver failures into typed errors. Keep defects distinct from recoverable contention and validation conflicts. Drizzle remains the schema/query/migration layer, with one owner of transaction boundaries.

For synchronous transactions, execute and finish all SQL synchronously inside one callback. Do not return an Effect or a Promise from that callback and assume its later execution remains inside the transaction. Wrap the whole completed synchronous unit at the Effect boundary; use the v4 synchronous error-capture API verified in the installed package. Network calls, stream appends, agent execution and asynchronous waits happen only after commit. Verify rollback and outbox atomicity with failure-injection tests.

Bun SQLite is synchronous. Effect wrapping does not move a slow SQL query onto another thread, make it preemptible, or remove SQLite's single-writer constraint. Keep queries indexed/bounded and transactions short. Configure WAL, foreign keys and a bounded contention policy, and measure worst-case event-loop/Stop latency during writes. A long synchronous busy timeout can stall every request on that daemon. Add a dedicated database worker only if measurement justifies it, not as baseline infrastructure. [L5, L7–L9]

Schema integration and execution integration are different. Drizzle documents `drizzle-orm/effect-schema` for generating SQLite row schemas. It may be used after verifying compatibility with the chosen Effect v4 and Drizzle versions. It does not replace API authorization or make arbitrary database rows safe public command contracts. Server-assigned authorship, permissions, ownership claims and queue sequence fields must not become client-writable through a generated insert schema. Prefer explicit domain command schemas. [E9]

Retries must be narrowly classified. Safe examples include deduplicated outbox publication or a proven rolled-back SQLite transaction. Never put a blanket Effect retry around an entire agent turn, a partially performed mutation, or a replay consumer that might repeat external side effects. Idempotency keys and persisted receipts remain required.

### 0.13 Effect-specific integration and test gate

Add these checks to Milestone 0 and the main acceptance suite:

1. Exact v4 Effect/platform dependencies type-check together; current imports are used and no accidental v3 bridge dependency enters the application runtime.
2. One database/resource Layer is acquired for the daemon, reused by requests, and released on controlled shutdown; no per-request worker/reaper duplication.
3. A committed command is still dispatched after the submitting HTTP connection disappears before acknowledgment. Retrying it resolves the original receipt.
4. Aborting any viewer subscription leaves native production and other readers unaffected.
5. Stop reaches the selected real backend, not merely a cancelled fiber; Stop-and-replace does not overlap two native executions.
6. Injecting a transaction failure rolls back message, receipt, pending work and outbox together. An outbox publication failure does not relaunch the agent.
7. A missed in-process wakeup is recovered from SQLite; process-local fibers/queues are never the only evidence of accepted work.
8. Native stream framing/cursors pass unchanged through Effect HTTP, with no whole-response buffering and no duplicate reducer.
9. The API serves the built Start shell and authenticated streaming routes together; error responses and static fallbacks remain distinct.
10. Concurrent writes, long histories and stream traffic do not make local feedback or backend cancellation unacceptably unresponsive on the reference device.

Use version-compatible `@effect/vitest`/TestClock for pure Effect service, schedule and scope tests when helpful; test actual bun:sqlite code under Bun, and use Playwright for browser collaboration. Do not assume a Node-run Vitest process can import `bun:sqlite`. A Bun application does not require every test runner to execute inside Bun; document the runner boundary and never use a mocked driver as evidence of real SQLite transaction correctness.

These requirements specify the integration to implement. This handoff has not installed dependencies, compiled the Effect/Drizzle/TanStack combination, or executed the tests.


## 1. Product contract

Build one excellent chat experience. A conversation is a room with members and one logical assistant. A personal conversation is the same room model with one human member.

The reference demonstration must work with two separately authenticated people, ten desktop tabs, and a mobile browser accessing the application privately through Tailscale. Everyone sees the same accepted messages, shared queue, selected response attempts, and ongoing assistant output. Closing the initiating browser must not own or terminate the work.

Sending, queueing, cancellation, correction, regeneration, reconnecting, and reading streamed output must feel immediate and remain understandable under concurrent actions. No lost composer text, duplicate bubbles, accidental repeated executions, or hidden replacement of another person's messages.

In scope: authentication, room invitations/membership, attributed messages, one assistant per room, shared queue and controls, persistent history, response attempts, streaming, reconnect/catch-up, basic presence, and native durable-run wiring for the selected coding harness.

Out of scope: file explorer, editor, terminal, repository review, multi-agent orchestration, collaborative document editing, marketplace, attachments, voice, and general historical conversation trees. A harness can have a workspace internally without exposing a workbench.

## 2. Library boundaries and the architecture decision

Use these responsibilities, not a blanket installation of every overlapping transport:

| Concern | Selected role |
|---|---|
| Native AI execution and event types | TanStack AI and a supported model/harness adapter |
| React chat composition and stream processing | Native TanStack chat client/UI APIs; use the documented UI entry point and pinned types |
| Shared reactive records | TanStack DB collections through StreamDB |
| Structured synchronization | Durable State protocol through `@durable-streams/state/db` |
| Native per-run output durability | `@tanstack/ai-durable-stream` with the native response/replay lifecycle |
| Canonical server persistence | One host-local SQLite database via Drizzle/bun:sqlite, exposed through Effect services and implementing native persistence contracts |
| Backend application lifecycle | Effect 4 services/Layers and daemon-scoped supervision; no replacement AI/event protocol |
| Agent capture/recovery | Native sandbox durable-run, journal, takeover, and reaper machinery where supported |
| Presentation | Requested shadcn components, especially Message Scroller |

StreamDB is accessed through `createStreamDB` from `@durable-streams/state/db`; it builds TanStack DB collections over a Durable State stream. It provides a synchronization/read-model implementation rather than requiring our own browser CRUD-event reducer. [S1–S3]

There are TWO different AI integrations. `@durable-streams/tanstack-ai-transport` supplies a conversation-session connection and helpers, including prompt echo and history snapshots; its documented pattern explicitly supports multiple clients. Native `@tanstack/ai-durable-stream` implements TanStack's per-run durability interface used by sandbox recovery. They are not the same interface. [S4, S5]

Decision: start with native run durability plus StreamDB for room state. Do not also mirror all token events into a second session transport by default. The session transport is an alternative delivery architecture, not a requirement to layer onto the chosen path. It may be evaluated in the integration spike, but any change must preserve native recovery, one streaming reducer, one history authority, and the acceptance tests below.

## 3. Logical topology

```text
React chat UI, independently on every device
  |-- native TanStack stream processor: active response parts
  |-- StreamDB / TanStack DB: shared room records and controls
  |-- local UI: composer draft, scroll, selections, expansion
                 |
        Authenticated HTTP commands and stream reads
                 |
             Application backend
  |-- authorization and room admission
  |-- native TanStack execution / persistence / recovery
  |-- transactional room-state publisher
                 |
       +---------+----------------------+
       |                                |
 SQLite on local disk             Local Durable Streams
 canonical records                room/<id>/state
 native stores                    run/<runId>/events
 outbox / command receipts
       |
 Selected agent execution environment
 native capture journal when supported
```

The names above are proposed stream namespaces, not hard-coded upstream defaults.

Keep two logical streams because their jobs differ. The room-state stream continues across turns. A run event stream describes a particular native execution. Reuse the same Durable Streams service; do not add another event bus solely for room synchronization.

The room stream advertises accepted messages, member changes, queue changes, response-attempt selection, and run lifecycle. Consequently, an idle tab learns about the NEXT response as well as the current one. Native output remains native chunks; do not invent a universal token envelope or handwritten message-part assembler.

## 4. Authority, projection, and finalization

The database is authoritative for membership, accepted commands, message records, selected attempts, and run control. Native stores remain authoritative for their native lifecycle fields. StreamDB is the materialized client view for this design; its optimistic writes are not authorization or distributed execution locks.

Project committed, public room-state changes to the state stream. `createStreamDB` does not automatically watch arbitrary SQLite writes: the implementation must supply this publisher. Use a transactional outbox or an equivalently tested publication mechanism. Database success followed by a stream outage must not permanently hide an accepted command from other users.

Publish changes from ALL writers, including fresh execution, cancellation, recovery, and the reaper. Prefer integrating the outbox at the persistence/application-store transaction boundary instead of scattered route-level notifications. Retain a room revision and idempotent publication identity; retries must converge rather than repeatedly applying a user action.

Use native message parts for stored assistant content. For an in-progress response, the native processor's assembled message is a local rendering projection of that run's log. Overlay it onto the corresponding persisted message identity; do not render it as an unrelated second bubble. When final content is committed and observed, retire the live overlay without a visual jump or loss of partial output.

Do not update a SQLite message row or a StreamDB message entity for every token. Stream the deltas through native run delivery and persist/project content at the native supported persistence boundaries. Room-state events primarily describe accepted records and structural changes.

The client facade must coordinate history hydration, room revisions, and native run attachment. Choose one hydration owner; do not concurrently drive independent full-history replacement mechanisms. Native server-authoritative persistence is a useful starting point, but live room updates still require explicit integration. [S6]

Required race handling: loading history while a message arrives, a new run starting during attachment, terminal status arriving before the client drains the final output, and final persistence arriving after run completion. Never clear the partial response merely because a summary record is terminal. Snapshot/replay cutoffs must be consistent; blind "fetch then subscribe" is insufficient.

## 5. Identities and membership

Separate model role from real-world author. Two humans can both produce native `user` messages while retaining different authenticated authors.

Proposed application metadata, not an upstream API:

```ts
interface MessageAuthorship {
  messageId: string;
  roomId: string;
  authorId: string;
  authorKind: 'human' | 'assistant';
  roomSequence: number;
  revision: number;
  replyToMessageId?: string;
}
```

Use a stable room/thread mapping and native message/run identities. Keep immutable command IDs, response-attempt IDs, and author associations where native contracts do not represent them. A logical assistant turn may contain more than one native run; do not release the room's execution slot between tool/interrupt continuations just because one segment finished.

Suggested application records: rooms, memberships, message authorship/revisions, assistant attempts, queued commands, command receipts, and publication outbox. Reuse native messages/runs/interrupts storage rather than creating competing copies of native lifecycle state.

Membership roles: owner, participant, viewer. Owner manages membership and the shared agent configuration. Participants can post, ask the assistant, and stop shared work under the default room policy. Viewers only read. Participants can edit/cancel their own eligible messages; owners may moderate explicitly. Do not silently allow one participant to overwrite another's prompt.

Include genuine application sessions and an invitation flow in the template. A development seed creates two users and a shared room through the normal authorization paths. Disable development shortcuts in production. An invitation URL is a scoped, expiring invitation, not permanent stream credentials.

Derive author identity from the authenticated session, not a request body `authorId`. Protect every room snapshot, historical stream, live stream, command, and native run lookup. A run belongs to a room; possession of a runId is not permission.

Use room-scoped streams or equivalently server-authorized projections. A browser-side TanStack DB filter is not an access-control boundary. Never send a global stream with other rooms' private records and rely on filtering them out locally.

On membership removal, reject new commands and revoke future live access through an explicit bounded revocation mechanism. Stop/revalidate existing reads as necessary. Do not promise erasure of content already received by a removed participant.

## 6. What two humans plus one AI means

Default Send requests an assistant response. Provide a small secondary "Message room" mode to post human discussion without automatically invoking the assistant. Represent this as explicit server-side intent, not an AI guess based on punctuation or a mention string.

When a human asks while the assistant is idle, admit one assistant turn. When another asks while it is busy, accept the request into the shared FIFO queue. The UI says that it is queued and not yet included in the current reply. All members see the author and the same queue position.

Assign ordering on the server; client clocks do not determine canonical order. Preserve submission identity through uncertain acknowledgements. Retries reconcile an existing receipt; they must not create another assistant turn.

Before execution, freeze the exact context message IDs and revisions, requesting actor, selected profile revision, and relevant room state. The active run does not silently change its input when another person types. Exclude future queued questions until admitted. Define whether eligible room-only comments are included when the next turn starts, and record that choice.

Build model-visible speaker labels from authenticated author metadata at the server serialization boundary. Do not assume arbitrary message metadata survives every harness adapter. Test that the AI can distinguish participants. Display names and quoted content never grant tool permissions; all human content remains untrusted user-level content.

Shared configuration and tool access must be intentional. Joining a room must not silently expose an owner's unrelated private accounts, credentials, memory, or broader tools. V1 uses a limited shared execution profile and server-side tools only.

## 7. Shared interaction contract

### Send and queue

Capture text into recoverable local pending state immediately, render feedback, clear the composer, and preserve focus. The network acknowledgment and first model event are distinct statuses. Keep typing responsive throughout.

Accepted queue entries are server-owned. Do not also dispatch them through TanStack's client FIFO. The native queue's documented stop/error/reload semantics differ from a shared persistent queue. [S7]

Proposed initial limits: five outstanding AI requests per participant and twenty per room, configurable and enforced on the server. Rejected submissions remain recoverable in the originating client.

On normal completion, drain the next queued request through one server admission path. After a shared Stop or execution failure, preserve and pause remaining entries. Make the paused reason visible and require an authorized Resume. Closing a tab never pauses or deletes the server queue.

### Cancel queued input

A participant can cancel their own not-yet-started entry. The owner can moderate the queue. Undo restores an unsent local draft, not an automatic new request.

Use a conditional server transition so cancellation racing with dispatch yields one truthful result. If the request has started, show that fact and offer Stop; do not pretend it was never accepted.

### Stop

Target the exact native run/assistant attempt or pending submission. Do not implement Stop as "look up whichever run is active when the delayed request arrives".

React locally with `Stopping…`, retain output, and record cancellation intent through the supported backend path. Confirmation of intent is not confirmation of termination. Other views learn the same control status and actor attribution.

For durable harness runs, local `chat.stop()` is not remote cancellation. Use native cancellation intent plus the appropriate execution stop path. A terminal outcome must come from execution, not from the HTTP handler deciding success. [S5]

Do not let a late cancellation for run A stop replacement B. A response that finished before cancellation should retain its real completed state. On uncertain cancellation, retain the replacement/draft and show uncertainty rather than run both.

### Stop and send

Make this explicit. Atomically claim the replacement operation against the expected current attempt, preserve/pause existing queue entries, request cancellation, and wait for confirmed termination before launching the replacement. Two concurrent replacements must not both win.

### Regenerate / retry

Transport retry reuses command identity and attaches to existing work. Deliberate regeneration creates a new response attempt and does not duplicate the user message.

Support the latest eligible response only. Require an expected message/attempt revision; preserve prior attempts and publish the shared selected attempt. Allow users to inspect an older attempt locally without changing shared context selection.

Do not regenerate the same answer twice because two tabs clicked. The losing stale action should reconcile and explain which newer attempt is active.

For a coding harness, a rerun is new execution, not undo of tools or native session state. Keep its semantics honest even though V1 exposes only chat.

### Edit

Allow editing one's own latest eligible prompt, with revision preservation. Do not silently rewrite context already consumed by later turns or change another participant's message. V1 can reject historical edits that require branching, explaining why.

## 8. Two different concurrency boundaries

Room admission prevents two distinct human commands from starting overlapping assistant turns. Implement it transactionally on server-owned room/queue state. Native run recovery does not replace this admission rule: two different run IDs are still two jobs.

Native run ownership coordinates drivers attempting to drive the SAME execution. Reuse TanStack's lifecycle and ownership mechanisms. V1 has one active application daemon per data directory; enforce single-instance ownership and test sequential driver replacement after a crash. Supply the native lock/store contracts required by the chosen lifecycle, backed by the same local authority where necessary. A multi-host cluster and distributed database/lock infrastructure are not V1 requirements. `withLocks` exposes a capability; it does not automatically serialize an entire conversation. [S8]

Use one native RunStore shared between persistence and sandbox machinery. Preserve all required native durable fields rather than mapping only a simplified status enum. Private recovery fields belong on the server, not in a broadly published room projection.

No browser subscription, newly observed user message, or restored client state may itself start a new model invocation. The backend dispatches accepted intents. Use server-side tools for V1; spectators must never execute a tool once per tab.

## 9. Native durable-run requirements

For the coding-harness profile, select the external-log delivery tier. Here "external" means a delivery log separate from the execution journal, not a required cloud service: the Durable Streams service and its files run on the owning device by default. The native capture journal remains inside a surviving sandbox; the run log is the client-delivery source. A delivery log cannot capture output lost upstream of its writer. [S9]

Use the same native durability adapter identity/configuration and native run binding across producing, joining, and recovery paths. Configure the native RunStore and durability together. Build fresh execution, recovery, cancellation, and reaping from one server-side resolved profile definition so they cannot select different agents or credentials.

Schedule the native reaper/finalizer and document runtime, abandonment, and output-retention policies. Do not count visible browsers as the lifecycle authority. A run can finish with zero readers and still needs history finalization and cleanup. [S10]

Distinguish browser/network recovery, backend-driver recovery with a surviving sandbox, and loss of the sandbox itself. Do not promise the last case is recoverable through the delivery log. Direct model adapters also do not acquire the sandbox journal guarantees merely by using resumable delivery.

Known upstream qualifications at review time:

- The detailed takeover reference acknowledges a residual stale-append race because the append interface lacks an atomic compare-and-set. Use the documented locking/lease controls and test ownership loss; do not advertise unconditional exactly-once execution or airtight fencing. [S5]
- The simplified durable-runs explanation overstates completion-sentinel secrecy. The journal reference and inspected source derive the nonce from runId and acknowledge deliberate reproduction is possible. Treat it as accidental-output disambiguation, not a cryptographic authentication boundary against a malicious agent. Do not invent another sentinel implementation. [S11]

Pin package versions and verify these properties in the installed source during implementation. No integration or failure-recovery tests were run in preparing this plan.

## 10. UI quality and collaboration feedback

Keep one central conversation with a compact member header, activity line, queued-request area, and sticky composer. Do not add a workbench around it.

Use author identity, not native message role alone, for attribution and alignment. Show readable names/avatars for other humans and a distinct assistant identity. Group consecutive messages carefully without hiding who asked a question or stopped the run.

Show controls in context: "Replying to Alex", "Your follow-up is queued", "Stopped by Kristjan", "Queue paused", "Reconnecting", and "Another participant started a new response". Only derive model/tool activity from actual events; never fabricate reasoning or percentages.

Keep connection, execution, and submission status separate. A disconnected phone is not proof the assistant stopped. On return from mobile suspension, restore and catch up instead of promising continuous background rendering.

Keep drafts private to the current user/device or tab according to an explicit local-draft policy. Never sync partially typed content as a shared room message. Clear private caches on sign-out/account change and isolate cache keys by identity and room. Synchronize only a typing signal, not draft text.

Basic presence should track connection/session leases and aggregate by human identity. Ten tabs for one person must not display ten members, and closing one tab must not mark the person offline while others remain. Expire stale typing/presence; do not replay an old typing event as if it were current. Avoid durable high-frequency heartbeat writes into the permanent transcript.

Use shadcn Message Scroller as the scrolling owner. Follow streamed output only while the viewer is at the live edge; respect deliberate upward scrolling, show a jump-to-latest control, and preserve anchors during history load or expansion. [S12]

Keep composer input responsive during long code streams. Render received output without a decorative typewriter backlog. Memoize completed messages, isolate active-message rendering, retain partial output on stop/error, and sanitize untrusted rendering. Support Enter/Shift+Enter, IME composition, accessible labels/focus, reduced motion, and mobile keyboard layout.

Local Send/Stop feedback target: within 100 ms on the reference device, measured separately from remote acknowledgment and actual termination. This is a proposed acceptance target, not a claim about library benchmarks.

## 11. Deployment and operational boundaries

Reference deployment: one owning user device runs the long-lived API daemon, a local SQLite database, a persistent local Durable Streams helper, and the selected agent runtime. The daemon serves the compiled TanStack Start UI and authenticated API/stream routes on one origin. Other devices are readers and command clients of this host; they do not open or independently replicate its SQLite file. Do not require a cloud account, PostgreSQL, Redis, a multi-host cluster, or Docker for the base chat installation. An explicitly selected sandbox provider may have its own documented runtime prerequisites.

Require browser-facing HTTP/2 for the ten-tab deployment test. SSE over HTTP/1.x has a small browser/domain connection limit; test the negotiated protocol, proxy buffering, and actual behavior through the deployment origin. [S13]

Tailscale Serve may expose the app privately over HTTPS. [S14] Application authorization remains required. Keep durable-stream write/admin credentials server-side. Readers access authorized same-origin routes or equivalently scoped read capabilities. Derive upstream stream paths server-side from authorized IDs; do not accept arbitrary proxy destinations.

Bound queue size, message size, concurrent rooms per actor, run duration/cost, idle readers, and storage retention. Define deletion and retention separately from append-only delivery so old content is not accidentally retained forever. Snapshot/cursor handling must be implemented before claiming unbounded-history scalability; StreamDB's documented preload begins with stream materialization. [S2]

## 12. Acceptance suite

Use two genuinely separate authenticated browser contexts plus additional tabs. A mock username switch alone is not an authorization test.

| Scenario | Required outcome |
|---|---|
| Two users, ten tabs, one mobile | Same accepted history, authors, queue, and assistant attempt |
| Second user joins mid-response | Correct history followed by remaining live output, no duplicate text |
| Both users send at once | Stable server order and no overlapping assistant turns |
| Retransmit one timed-out command | Existing command is reconciled, not executed again |
| Idle tabs observe a later prompt | They discover the next turn without reloading |
| Participant cancels their queued item from mobile | Shared queue converges; someone else's input remains |
| Two viewers regenerate the same attempt | One transition wins; the other reconciles |
| Delayed Stop for A arrives after B starts | B is unaffected |
| Stop and replace during startup | No orphan process, duplicate run, or lost replacement |
| Initiating tab closes; all tabs close | Accepted work follows server policy and can finalize unseen |
| Mobile sleeps and returns | Catch-up succeeds and composer draft remains private |
| Driver restarts with sandbox surviving | Supported native recovery is exercised without launching another agent |
| Second local daemon starts / driver loses its claim | Enforced instance ownership and native ownership-loss behavior are tested; limitations documented |
| DB commit succeeds while state stream is unavailable | Outbox eventually publishes without duplicate commands |
| Final status precedes final message projection | Partial answer remains visible until reconciled |
| Member is removed | Future commands/reads fail and active reads are revoked per the documented bound |
| Unauthorized stream/run ID is supplied | No history, live output, or status is disclosed |
| Host becomes unavailable or sleeps | Clients report connection loss; no claim that powered-off local execution continues |
| Backup is restored into a separate data directory | SQLite, stream snapshots/cursors, and runtime reconciliation are consistent |
| Release runs without cloud application services | Built UI, local API, SQLite and persistent local streams start through documented launcher |
| Two people have identical display names | Stable author identities and permissions remain distinct |
| Tool event arrives in ten clients | One server tool execution, no spectator side effects |
| User scrolls up while another sends | Reading position remains stable |
| Two users have different private drafts | No draft leakage or cross-account cache recovery |

Count model/harness starts and tool side effects in tests, not just final UI bubbles. A visually deduplicated transcript can conceal duplicate paid execution.

## 13. Implementation order

0. Baseline and refactor: follow section R, adapt the existing starter, prove the exact Effect v4/Bun/Drizzle dependency set, retain auth/UI/tooling, and complete production static serving plus the Milestone 0/Effect smoke suites. Record the foundation refactor separately from chat feature work.
1. Integration proof: pin compatible packages; compile the native run stream, persistence, UI composition, and StreamDB room projection together. Demonstrate two authenticated readers and one real backend. Prove one history/stream ownership path before polish. Do not infer compatibility solely from similarly named packages.
2. Collaboration core: membership/invites, authorship, server command receipts, serial room admission, shared FIFO, and reliable publication. Seed the two-user demo through normal auth.
3. Interaction polish: fast composer, cancel/stop/replace/regenerate/edit semantics, activity feedback, attempts, scroller behavior, private drafts, and basic presence.
4. Durability and failure tests: reconnect, unseen completion, native reaper, driver restart where supported, cancellation races, outbox recovery, authorization/revocation, and real HTTP/2 ten-tab/mobile exercise.

Use deterministic native-event fixtures for UI/race tests and real-agent smoke tests for execution claims. Delivery of the template requires a verification report with pinned versions, passed tests, known limitations, and any unsupported selected-profile capability. Do not describe documentation inspection as runtime verification.

## 14. Sources and API verification references

These sources establish library behavior. The room model, policies, schema extensions, publication design, deployment defaults, and test criteria above are proposed application decisions.

- [S1] TanStack DB: https://tanstack.com/db/latest
- [S2] StreamDB: https://durablestreams.com/stream-db
- [S3] Durable State protocol and materialization: https://durablestreams.com/durable-state
- [S4] Durable Streams session integration: https://durablestreams.com/tanstack-ai
- [S5] Native takeover, cancellation, and ownership qualifications: https://tanstack.com/ai/latest/docs/sandbox/takeover
- [S6] Server-authoritative client persistence: https://tanstack.com/ai/latest/docs/persistence/client-persistence
- [S7] Native client queue: https://tanstack.com/ai/latest/docs/chat/queueing
- [S8] Lock capability and implementation contract: https://tanstack.com/ai/latest/docs/advanced/locks
- [S9] Durable run tiers: https://tanstack.com/ai/latest/docs/sandbox/durable-runs
- [S10] Reaping and retention: https://tanstack.com/ai/latest/docs/sandbox/reaping
- [S11] Run journal and nonce qualifications: https://tanstack.com/ai/latest/docs/sandbox/journal
- [S12] Message Scroller: https://ui.shadcn.com/docs/react/message-scroller
- [S13] SSE connection considerations: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- [S14] Tailscale Serve: https://tailscale.com/docs/features/tailscale-serve
- [S15] Native React UI composition: https://tanstack.com/ai/latest/docs/ui/react
- [S16] Native store reference: https://tanstack.com/ai/latest/docs/persistence/store-reference
- [S17] Native resumable streams: https://tanstack.com/ai/latest/docs/resumable-streams/overview
- [S18] Session transport README reviewed: durable-streams/durable-streams, packages/tanstack-ai-transport/README.md, blob 4c59c4f8dd08812138c627d798904ec8f09987e6.
- [S19] Journal source reviewed: TanStack/ai, packages/ai-sandbox/src/journal.ts, commit 44a73e0e8790f478d853bf3d843c44f1f501762e. The source explicitly qualifies the derived nonce's threat model.

### Additional source references verified for the device-hosted revision

New deployment references were reviewed on September 11, 2026. Existing AI/collaboration references above are retained from the prior handoff and must be checked against the installed versions in the integration proof.

- [L1] TanStack Start getting started / official scaffolding: https://tanstack.com/start/latest/docs/framework/react/getting-started
- [L2] TanStack Start SPA mode: https://tanstack.com/start/latest/docs/framework/react/guide/spa-mode
- [L3] Bun workspaces: https://bun.com/docs/pm/workspaces
- [L4] Hono on Bun: https://hono.dev/docs/getting-started/bun
- [L5] Bun SQLite driver: https://bun.com/docs/runtime/sqlite
- [L6] Drizzle Bun SQLite setup: https://orm.drizzle.team/docs/get-started/bun-sqlite-new
- [L7] SQLite WAL, concurrency and filesystem constraints: https://sqlite.org/wal.html
- [L8] SQLite connection pragmas: https://sqlite.org/pragma.html
- [L9] SQLite transaction semantics: https://sqlite.org/lang_transaction.html
- [L10] Durable Streams deployment and persistence: https://durablestreams.com/deployment
- [L11] SQLite backup mechanisms: https://sqlite.org/backup.html


### Effect 4 revision references

Reviewed September 11, 2026. These references establish individual documented building blocks; they are not proof of combined runtime compatibility. Prefer release-matched source when implementing.

- [E1] Effect 4 release-candidate announcement (updated August 12, 2026): https://effect.website/blog/releases/effect/40-rc
- [E2] Effect v3-to-v4 package/import/stability migration guidance (the linked guide retains a beta-status note; use current release metadata for release status): https://github.com/Effect-TS/effect/blob/main/MIGRATION.md
- [E3] Effect v4 Bun platform API: https://effect.website/docs/v4/api/platform-bun
- [E4] Maintained v4 HttpApi example, including typed clients and alternative Bun server: https://github.com/Effect-TS/effect/blob/main/ai-docs/src/51_http-server/10_basics.ts
- [E5] Drizzle Bun SQLite driver and synchronous APIs: https://orm.drizzle.team/docs/sqlite/connect-bun-sqlite
- [E6] Effect v4 Promise/AbortSignal semantics: https://effect.website/docs/v4/api/effect/Effect#tryPromise
- [E7] Effect v4 fork/explicit-scope semantics: https://effect.website/docs/v4/api/effect/Effect#forkIn
- [E8] Effect's separate Bun SQLite SQL driver: https://effect.website/docs/v4/api/sql-sqlite-bun
- [E9] Drizzle SQLite Effect Schema integration: https://orm.drizzle.team/docs/sqlite/effect-schema
- [E10] Effect maintainers' guidance on giving coding agents versioned source/pattern references: https://effect.website/blog/the-one-weird-git-trick-that-makes-coding-agents-more-effect-ive
- [E11] Effect v4 HttpApi reference: https://effect.website/docs/v4/api/effect/unstable/httpapi/HttpApi

### Repository and linked-client review references

These references were inspected for this repository-specific revision. Other library references above are inherited research pointers; revalidate them against installed versions. Proposed migration/policy/test decisions are authored requirements, not statements that upstream implements them automatically.

- [R1] Reviewed repository/main snapshot: https://github.com/grmkris/tanstack-ai-expiriments/tree/c58de3b1b04654b01997564a945e935f61e0d8c3
- [R2] Workspace manifest: https://github.com/grmkris/tanstack-ai-expiriments/blob/c58de3b1b04654b01997564a945e935f61e0d8c3/tempalte/package.json
- [R3] Web manifest: https://github.com/grmkris/tanstack-ai-expiriments/blob/c58de3b1b04654b01997564a945e935f61e0d8c3/tempalte/apps/web/package.json
- [R4] Server entry point: https://github.com/grmkris/tanstack-ai-expiriments/blob/c58de3b1b04654b01997564a945e935f61e0d8c3/tempalte/apps/server/src/index.ts
- [R5] Database initialization: https://github.com/grmkris/tanstack-ai-expiriments/blob/c58de3b1b04654b01997564a945e935f61e0d8c3/tempalte/packages/db/src/index.ts
- [R6] Auth initialization: https://github.com/grmkris/tanstack-ai-expiriments/blob/c58de3b1b04654b01997564a945e935f61e0d8c3/tempalte/packages/auth/src/index.ts
- [R7] Starter README: https://github.com/grmkris/tanstack-ai-expiriments/blob/c58de3b1b04654b01997564a945e935f61e0d8c3/tempalte/README.md
- [R8] Stale checked-in spec: https://github.com/grmkris/tanstack-ai-expiriments/blob/c58de3b1b04654b01997564a945e935f61e0d8c3/COLLABORATIVE_CHAT_V1_SPEC.md
- [R9] Linked Effect SQLite client: https://effect.website/docs/v4/api/sql-sqlite-bun/SqliteClient ; source linked by that page: https://github.com/Effect-TS/effect/blob/2600f62f4532026928454dcea8d1c48557b3f942/packages/sql/sqlite-bun/src/SqliteClient.ts
- [R10] Router/Vite/PWA config: https://github.com/grmkris/tanstack-ai-expiriments/blob/c58de3b1b04654b01997564a945e935f61e0d8c3/tempalte/apps/web/vite.config.ts
- [R11] Browser auth URL handling: https://github.com/grmkris/tanstack-ai-expiriments/blob/c58de3b1b04654b01997564a945e935f61e0d8c3/tempalte/apps/web/src/lib/auth-client.ts
- [R12] Drizzle configuration and dependencies: https://github.com/grmkris/tanstack-ai-expiriments/blob/c58de3b1b04654b01997564a945e935f61e0d8c3/tempalte/packages/db/drizzle.config.ts ; https://github.com/grmkris/tanstack-ai-expiriments/blob/c58de3b1b04654b01997564a945e935f61e0d8c3/tempalte/packages/db/package.json
- [R13] Better Auth Drizzle adapter documentation: https://better-auth.com/docs/adapters/drizzle
- [R14] Effect SQL client contract: https://effect.website/docs/v4/api/effect/unstable/sql/SqlClient

## Final handoff rule

Use this file alone as the authoritative brief for refactoring `grmkris/tanstack-ai-expiriments` into the Effect 4 / Bun / Drizzle / SQLite device-hosted collaborative chat. Replace or explicitly supersede the stale checked-in PostgreSQL spec. Reuse the existing starter, normalize its layout, convert the frontend/backend deliberately, verify the foundation, then implement the complete collaborative-chat acceptance suite. Do not generate a second starter or implement a workbench. This document is a source-reviewed plan, not a completed refactor or proof of runtime compatibility.
