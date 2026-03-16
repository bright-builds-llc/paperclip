# Codebase Structure

**Analysis Date:** 2026-03-15

## Directory Layout

- `server/` - Express API, orchestration services, plugin host, realtime, storage, and server startup.
- `ui/` - React/Vite board UI, contexts, API clients, hooks, pages, plugin bridge, and design guide.
- `cli/` - `paperclipai` CLI for onboarding, doctor, runtime operations, and scripted access to server APIs.
- `packages/db/` - Drizzle schema, migrations, DB client helpers, backup/migration utilities.
- `packages/shared/` - shared constants, API path constants, domain types, validators, and config schemas.
- `packages/adapters/*/` - adapter packages for Claude, Codex, Cursor, Gemini, OpenCode, Pi, and OpenClaw gateway.
- `packages/plugins/sdk/` - plugin authoring/runtime SDK.
- `packages/plugins/examples/` - reference plugins demonstrating host/plugin capabilities.
- `tests/e2e/` - Playwright end-to-end coverage.
- `doc/` - product, implementation, database, deployment, and operational documentation.
- `scripts/` - dev, release, backup, onboarding smoke, and packaging scripts.

## Directory Purposes

**Server Paths:**
- `server/src/routes/` aligns mostly one-to-one with API resource groups such as `issues.ts`, `agents.ts`, `companies.ts`, and `plugins.ts`.
- `server/src/services/` holds domain logic and orchestration helpers, including heavy runtime modules like `heartbeat.ts`.
- `server/src/middleware/` contains auth, logging, validation, hostname, and mutation guards.
- `server/src/realtime/`, `server/src/storage/`, and `server/src/auth/` isolate cross-cutting subsystems.

**UI Paths:**
- `ui/src/pages/` contains route targets.
- `ui/src/components/` contains reusable board components.
- `ui/src/context/` contains provider-based shared UI state such as selected company, dialogs, theme, panels, breadcrumbs, live updates.
- `ui/src/api/` mirrors server resource groups with fetch wrappers.
- `ui/src/plugins/` contains plugin UI bridge/slot mounting.

**Package Paths:**
- DB schema files are mostly one-file-per-table under `packages/db/src/schema/`.
- Shared contracts are split by domain under `packages/shared/src/types/` and `packages/shared/src/validators/`.
- Adapter packages follow a repeated shape with `src/index.ts`, `src/server/`, `src/ui/`, and `src/cli/`.

## Key File Locations

- Workspace root scripts and verification commands: `package.json`
- TypeScript project reference graph: `tsconfig.json`
- Server startup and app wiring: `server/src/index.ts`, `server/src/app.ts`
- Core authz/auth middleware: `server/src/middleware/auth.ts`, `server/src/routes/authz.ts`
- Heartbeat orchestration hotspot: `server/src/services/heartbeat.ts`
- Plugin host entry: `server/src/services/plugin-loader.ts`
- UI bootstrap and routing: `ui/src/main.tsx`, `ui/src/App.tsx`
- Company-scoped UI state: `ui/src/context/CompanyContext.tsx`
- Shared API path constants: `packages/shared/src/api.ts`
- Shared runtime/domain constants: `packages/shared/src/constants.ts`
- Schema export barrel: `packages/db/src/schema/index.ts`
- CLI command registration: `cli/src/index.ts`
- E2E entry point: `tests/e2e/onboarding.spec.ts`

## Naming Conventions

- Workspace packages use `@paperclipai/...` names.
- Server service factories are named `thingService(db)` and live in `server/src/services/thing.ts`.
- Route files usually use plural resource names matching API groups, for example `issues.ts`, `agents.ts`, `projects.ts`.
- React pages/components use PascalCase filenames such as `IssueDetail.tsx` and `CompanySettings.tsx`.
- DB schema files use snake_case names mirroring table concepts, for example `issue_read_states.ts`, `project_workspaces.ts`.
- Adapter packages expose a consistent split between root metadata and per-surface exports.

## Where to Add New Code

- New persisted domain objects: add schema in `packages/db/src/schema/`, export it from `packages/db/src/schema/index.ts`, then sync shared/server/ui contracts.
- New REST resource behavior: add or extend shared validators/types, update `server/src/routes/` and `server/src/services/`, then mirror in `ui/src/api/` and the relevant page/component.
- New board UI pages: `ui/src/pages/` plus route wiring in `ui/src/App.tsx`.
- New adapter support: add `packages/adapters/<name>/` following existing package structure, then register it where server/UI/CLI enumerate adapters.
- New plugin host features: update `packages/plugins/sdk`, plugin validators/types in `packages/shared`, and server plugin services together.
- New browser coverage: `tests/e2e/` for workflows, `ui/src/**/*.test.ts` for hooks/utilities, `server/src/__tests__/` for backend route/service behavior.

## Special Directories

- `.agents/skills/` holds repo-specific contributor skills and should be preserved.
- `.planning/` is where GSD planning/codebase artifacts live; it was absent before this run.
- `.paperclip/` is a generated repo-local runtime/config area for worktree-local instances and is gitignored.
- `releases/` stores release notes and changelog artifacts.
- `docs/` is distinct from `doc/` and appears to hold Mintlify-style website docs alongside the main product docs in `doc/`.

---

*Structure analysis: 2026-03-15*
*Update when directories, ownership boundaries, or extension surfaces change*
