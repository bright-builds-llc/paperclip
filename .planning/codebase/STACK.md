# Technology Stack

**Analysis Date:** 2026-03-15

## Languages

**Primary:**
- TypeScript 5.7.x drives the application, CLI, shared contracts, adapters, plugin SDK, and most tests. Representative paths: `server/src/index.ts`, `ui/src/App.tsx`, `cli/src/index.ts`, `packages/db/src/client.ts`, `packages/shared/src/api.ts`.

**Secondary:**
- JavaScript/ESM is used for repo tooling and release/dev scripts such as `scripts/dev-runner.mjs`, `scripts/generate-npm-package-json.mjs`, and GitHub workflow shell steps.
- Bash is used for operational helpers like `scripts/backup-db.sh`, `scripts/release.sh`, and smoke/provision scripts.
- Markdown is part of the product and contributor surface via `doc/`, `releases/`, `.agents/skills/`, and plugin/adapter docs.

## Runtime

**Environment:**
- Node.js 20+ is required at the repo level (`package.json` engines) and is the runtime for the server, CLI, build scripts, tests, and plugin tooling.
- Browser runtime for the operator UI is React 19 served either from Vite dev middleware or a built static bundle.
- PostgreSQL is the persistent store. Local default is embedded PostgreSQL via `embedded-postgres`; external Postgres is enabled through `DATABASE_URL`.

**Package Manager:**
- pnpm 9.15.4 (`packageManager` in `/package.json`).
- Lockfile: `pnpm-lock.yaml` is present and CI-owned per `doc/DEVELOPING.md`.

## Frameworks

**Core:**
- Express 5 powers the HTTP API in `server/src/app.ts`.
- React 19 powers the board UI in `ui/src/main.tsx` and `ui/src/App.tsx`.
- Drizzle ORM + `postgres` power data access and schema management in `packages/db/src/client.ts` and `packages/db/src/schema/*.ts`.
- Better Auth handles authenticated-mode user sessions in `server/src/auth/better-auth.ts`.

**Testing:**
- Vitest is the default test runner across the workspace (`vitest.config.ts`).
- Supertest is used for server route tests such as `server/src/__tests__/health.test.ts`.
- Playwright covers browser E2E flows in `tests/e2e/onboarding.spec.ts`.

**Build/Dev:**
- Vite 6 builds the UI and also runs dev middleware in the server process (`ui/vite.config.ts`, `server/src/app.ts`).
- TypeScript project references in `tsconfig.json` coordinate package builds.
- `tsx` runs the server, CLI, migrations, and other TypeScript scripts without a prebuild step.

## Key Dependencies

**Critical:**
- `express` for REST routing and middleware orchestration in `server/src/app.ts`.
- `drizzle-orm` and `postgres` for schema-first data access in `packages/db`.
- `embedded-postgres` for zero-config local installs in `server/src/index.ts`.
- `better-auth` for session-based authenticated deployments in `server/src/auth/better-auth.ts`.
- `@tanstack/react-query` for client-side data fetching/cache orchestration in `ui/src/main.tsx` and `ui/src/App.tsx`.
- `ws` for live event streaming and the OpenClaw gateway adapter in `server/src/realtime/live-events-ws.ts` and `packages/adapters/openclaw-gateway`.

**Infrastructure:**
- `zod` defines shared validators in `packages/shared/src/validators/*.ts`.
- `pino` and `pino-http` provide structured server logging.
- `@aws-sdk/client-s3` supports the optional S3 storage provider.
- `@paperclipai/plugin-sdk` and plugin packages add a first-party extension model beyond the core V1 control-plane paths.

## Configuration

**Environment:**
- Runtime configuration is loaded from repo-local `.paperclip/.env`, local `.env`, and process env in `server/src/config.ts`.
- Shared config shapes live in `packages/shared/src/config-schema.ts`.
- Important env groups include database (`DATABASE_URL`), deployment/auth (`PAPERCLIP_DEPLOYMENT_MODE`, `PAPERCLIP_PUBLIC_URL`, `BETTER_AUTH_SECRET`), storage (`PAPERCLIP_STORAGE_*`), secrets (`PAPERCLIP_SECRETS_*`), and scheduler flags.

**Build:**
- `tsconfig.json` and `tsconfig.base.json` define TypeScript references and shared compiler options.
- `ui/vite.config.ts`, `ui/vitest.config.ts`, and root `vitest.config.ts` define frontend and test behavior.
- `.github/workflows/pr-verify.yml`, `release.yml`, and `e2e.yml` encode CI verification and release automation.

## Platform Requirements

**Development:**
- Any OS that supports Node.js 20+ and pnpm 9+.
- Optional Docker for local Postgres or quickstart containers.
- Some adapter paths expect locally installed coding tools or API credentials, for example Codex, Claude, Cursor, Gemini, OpenCode, or OpenClaw-related services.

**Production:**
- Long-running Node process with writable filesystem access for logs, storage, backups, embedded DB data, plugin installs, and worktrees.
- External Postgres and S3-compatible storage are optional but supported.
- Git, npm access, and local process spawning matter for plugin installation and local adapter execution paths.

---

*Stack analysis: 2026-03-15*
*Update after major dependency or runtime changes*
