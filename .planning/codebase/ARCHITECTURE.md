# Architecture

**Analysis Date:** 2026-03-15

## Pattern Overview

**Overall:** Full-stack TypeScript monorepo for a multi-company AI control plane, with one Node server hosting the API, background orchestration, plugin lifecycle, and either a built UI or Vite dev middleware.

**Key Characteristics:**
- Shared contracts are centralized in workspace packages instead of duplicated between server, UI, and CLI.
- Backend request handling follows route -> validation/auth -> service -> Drizzle data access.
- Agent execution is adapter-driven and intentionally externalized: Paperclip orchestrates local processes, HTTP gateways, and plugin workers rather than embedding one runtime.
- Company scoping is a core architectural invariant, reinforced in authz helpers and service usage.
- Extension points now include adapter packages, CLI surfaces, and a plugin worker/runtime system in addition to the original V1 server/ui/db/shared shape.

## Layers

**Shared Contract Layer:**
- Purpose: Define stable types, constants, API paths, and validation schemas.
- Contains: `packages/shared/src/types/*.ts`, `packages/shared/src/validators/*.ts`, `packages/shared/src/constants.ts`, `packages/shared/src/api.ts`.
- Depends on: TypeScript + Zod only.
- Used by: server routes/services, UI API clients/pages, CLI commands, plugin SDK.

**Persistence Layer:**
- Purpose: Model and access durable state.
- Contains: `packages/db/src/schema/*.ts`, `packages/db/src/client.ts`, migration helpers, backup helpers.
- Depends on: Drizzle ORM, `postgres`, migration files.
- Used by: server services, CLI commands, auth adapter, runtime scripts.

**Application Server Layer:**
- Purpose: HTTP API, auth resolution, orchestration, storage, realtime, and plugin host lifecycle.
- Contains: `server/src/app.ts`, `server/src/routes/*.ts`, `server/src/services/*.ts`, `server/src/realtime/*.ts`, `server/src/storage/*.ts`.
- Depends on: shared contracts, db package, adapters, plugin SDK, Express middleware.
- Used by: UI, CLI, agent runtimes, plugins.

**Operator UI Layer:**
- Purpose: Human board interface for companies, agents, issues, approvals, costs, plugins, and onboarding.
- Contains: `ui/src/pages/*.tsx`, `ui/src/components/*`, `ui/src/context/*`, `ui/src/api/*`.
- Depends on: shared types, server API, React Query, custom router, plugin UI bridge.
- Used by: board operators in browser.

**Execution Surface Layer:**
- Purpose: Start and talk to agents or plugins.
- Contains: `packages/adapters/*`, `server/src/services/heartbeat.ts`, `server/src/services/plugin-*.ts`, `packages/plugins/sdk`.
- Depends on: OS processes, filesystem, WebSockets, provider credentials, workspace/runtime metadata.
- Used by: heartbeat scheduler/manual invocations, plugin pages, CLI commands.

**CLI Layer:**
- Purpose: Bootstrap, operate, and script a Paperclip instance.
- Contains: `cli/src/index.ts`, `cli/src/commands/*`, `cli/src/client/*`, `cli/src/config/*`.
- Depends on: server/db/shared packages and local filesystem state.
- Used by: local operators, tests, release/dev scripts.

## Data Flow

**Board HTTP Request:**

1. Browser app starts from `ui/src/main.tsx` and mounts providers plus React Query.
2. UI page or component calls a typed API helper from `ui/src/api/*.ts`.
3. Request hits Express in `server/src/app.ts`.
4. Middleware resolves actor context, host restrictions, and validation.
5. Route handler in `server/src/routes/*.ts` asserts company access and calls a service factory such as `issueService(db)`.
6. Service reads/writes Drizzle tables from `@paperclipai/db`.
7. Mutations log activity and may publish live events.
8. UI invalidates/refetches queries or receives websocket updates.

**Agent Heartbeat Run:**

1. A wakeup is scheduled or requested in `server/src/services/heartbeat.ts`.
2. Heartbeat service loads the agent, task/workspace context, and secrets.
3. Adapter resolution picks the correct local or gateway adapter from `server/src/adapters/index.ts`.
4. Runtime metadata and `PAPERCLIP_*` env vars are assembled, including workspace and auth data.
5. The external process or gateway runs and streams logs/events back.
6. Usage, results, run logs, cost events, and issue/session state are persisted.
7. Live events update the board UI and activity feed.

**Plugin Lifecycle:**

1. Plugin discovery/install runs through `server/src/services/plugin-loader.ts`.
2. Manifest validation, capability checks, and registry persistence occur before activation.
3. Worker processes are managed by plugin worker services and host RPC handlers.
4. Jobs, tools, UI slots, streams, and webhooks are registered into the host runtime.
5. Plugin state and logs are persisted in dedicated plugin tables and surfaced through UI/API routes.

**State Management:**
- Durable state is almost entirely database-backed.
- Short-lived coordination uses in-memory maps/event buses for locks, websocket subscribers, and plugin worker handles.
- UI state is split between React context (company/theme/panels/dialogs) and React Query cache.

## Key Abstractions

**Service Factory:**
- Purpose: Encapsulate domain logic behind a `...Service(db)` constructor.
- Examples: `companyService(db)`, `issueService(db)`, `heartbeatService(db)`, `accessService(db)`.
- Pattern: Stateless module factory over a shared DB handle.

**Actor Context:**
- Purpose: Normalize board, agent, or unauthenticated callers before business logic runs.
- Examples: `req.actor` in `server/src/middleware/auth.ts`, company checks in `server/src/routes/authz.ts`.
- Pattern: Request-scoped auth/authz object with company-aware enforcement.

**Adapter:**
- Purpose: Abstract how an agent heartbeat is executed.
- Examples: `packages/adapters/codex-local`, `packages/adapters/claude-local`, `packages/adapters/openclaw-gateway`.
- Pattern: Consistent package surface with root metadata plus `server`, `ui`, and `cli` entry points.

**Plugin Manifest + Worker:**
- Purpose: Extend the host with tools, jobs, UI slots, webhooks, and RPC-based worker logic.
- Examples: `packages/plugins/sdk/src/define-plugin.ts`, `server/src/services/plugin-loader.ts`.
- Pattern: Manifest-driven process isolation with host capability gating.

## Entry Points

**Server Startup:**
- Location: `server/src/index.ts`
- Triggers: `pnpm dev`, `pnpm dev:once`, built server start, CLI `paperclipai run`.
- Responsibilities: load config, bootstrap DB/auth/storage, start HTTP server, migrations, websocket server, schedulers.

**HTTP App Assembly:**
- Location: `server/src/app.ts`
- Triggers: server startup.
- Responsibilities: mount middleware, REST routes, plugin services, static UI or Vite middleware.

**UI Startup:**
- Location: `ui/src/main.tsx`
- Triggers: browser navigation.
- Responsibilities: initialize providers, query client, service worker, and plugin bridge.

**CLI Startup:**
- Location: `cli/src/index.ts`
- Triggers: `pnpm paperclipai ...`
- Responsibilities: parse commands, load env/config, dispatch operational tasks.

## Error Handling

**Strategy:** Validate early at boundaries, throw typed HTTP/domain errors from services/routes, and let central middleware or command-level handlers convert them to responses or process failures.

**Patterns:**
- Request validation uses shared Zod schemas via route middleware.
- Authz failures use explicit helpers such as `forbidden()`, `unauthorized()`, and `assertCompanyAccess()`.
- Server startup handles migration drift and configuration issues explicitly in `server/src/index.ts`.
- CLI commands typically bubble errors to the top-level `program.parseAsync().catch(...)`.

## Cross-Cutting Concerns

**Logging:**
- Request logging and structured runtime logging use Pino.
- Domain mutations also produce activity-log records for board visibility.

**Validation:**
- `packages/shared` is the contract source of truth for request bodies and enum domains.
- Route handlers generally validate before touching services.

**Authentication & Authorization:**
- Actor middleware resolves board vs agent identity.
- Company access is enforced in route helpers and service usage patterns.

**Realtime & Auditability:**
- Live event publishing and websocket subscriptions cut across heartbeats, activity, and plugin updates.
- Mutation flows commonly pair state changes with persisted activity records.

**Filesystem-Coupled Runtime:**
- Embedded DB data, storage, logs, workspaces, plugins, and some adapter session state all rely on writable local paths.

---

*Architecture analysis: 2026-03-15*
*Update when major patterns, layers, or runtime boundaries change*
