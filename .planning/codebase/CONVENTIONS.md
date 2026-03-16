# Coding Conventions

**Analysis Date:** 2026-03-15

## Naming Patterns

- TypeScript/React code generally uses `camelCase` for variables/functions and `PascalCase` for React components and types.
- Domain enum values and constants are centralized in uppercase exports like `ISSUE_STATUSES`, `AGENT_ADAPTER_TYPES`, and `PERMISSION_KEYS` in `packages/shared/src/constants.ts`.
- Service modules follow `thingService(db)` naming and return a plain object of operations, for example `issueService(db)` and `accessService(db)`.
- API client modules in the UI mirror resource groups: `issuesApi`, `agentsApi`, `companiesApi`, and so on.
- File names usually reflect the domain or UI surface directly; schema files stay close to table names.

## Code Style

- The repo is ESM-first TypeScript, including explicit `.js` import suffixes in server/CLI/package TS sources.
- Modules are generally narrow by domain, but a few orchestration files are intentionally large because they centralize runtime behavior (`server/src/index.ts`, `server/src/services/heartbeat.ts`).
- UI code uses the `@/` alias for `ui/src`.
- Shared contracts are expected to stay synchronized across `packages/db`, `packages/shared`, `server`, and `ui`; this is a repo-level engineering rule, not just an incidental pattern.
- There is no repo-wide ESLint or Prettier setup visible in package scripts or CI. Consistency is currently maintained through TypeScript, tests, and review.

## Import Organization

- External libraries are typically imported first, followed by workspace packages (`@paperclipai/...`), then relative imports.
- UI imports frequently mix aliased `@/...` component/lib paths with relative imports for nearby files.
- Server modules prefer relative imports between server-local layers and workspace-package imports for shared/db concerns.

## Error Handling

- Request validation happens at route boundaries with shared Zod schemas.
- Server domain code uses explicit HTTP/domain helpers such as `conflict`, `notFound`, `forbidden`, and `unauthorized`.
- Authz is usually enforced before calling business logic through helpers like `assertCompanyAccess()`.
- Startup code in `server/src/index.ts` is defensive around migrations, embedded DB bootstrapping, and local trusted board initialization.
- CLI execution lets errors bubble to a single top-level handler in `cli/src/index.ts`.

## Logging

- Server logging uses `pino` and `pino-http`.
- Mutating actions often write both operational logs and persisted activity-log records.
- Heartbeat and adapter logs are treated as product data, with storage/summarization/redaction helpers in `server/src/services/run-log-store.ts`, `server/src/services/heartbeat-run-summary.ts`, and `server/src/redaction.ts`.
- Adapters and runtime helpers propagate `PAPERCLIP_*` context to child processes rather than relying on hidden globals.

## Comments

- Inline comments are present but used sparingly; they usually explain intent or edge cases rather than narrating trivial code.
- Product/test flows sometimes include longer section comments for readability, especially in `tests/e2e/onboarding.spec.ts`.
- Some TODO comments capture intentionally deferred UI surfaces, notably issue worktree support in `ui/src/components/IssueProperties.tsx` and related files.

## Function Design

- Services favor small helper functions around a shared closure over `db`.
- Route modules often define small auth/validation helpers inside the route factory when they are route-specific.
- Shared validators/types keep domain parsing close to the edge instead of scattering ad hoc object checks.
- UI components frequently compose hooks + query clients + presentational pieces instead of using a large global state store.

## Module Design

- The main architectural rule is contract-first workspace packages: `packages/shared` and `packages/db` are foundational layers.
- Adapters are designed as self-contained packages with parallel server/UI/CLI surfaces.
- Plugin authoring is documented and SDK-driven, with unusually rich inline docs in `packages/plugins/sdk/src/index.ts`.
- Company scoping is a design convention visible across routes, services, schema, and UI selection context.

---

*Convention analysis: 2026-03-15*
*Update when style tooling or domain-level coding rules change*
