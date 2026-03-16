# Codebase Concerns

**Analysis Date:** 2026-03-15

## Tech Debt

- `server/src/services/heartbeat.ts` is a major orchestration hotspot that mixes queueing, concurrency control, workspace realization, adapter execution, usage accounting, run logging, and session rotation. It is powerful but difficult to change safely.
- `server/src/index.ts` also carries a broad startup surface: config resolution, migrations, embedded Postgres bootstrapping, auth bootstrapping, storage, server start, and local-trusted setup.
- The current repo extends well beyond the original `server/ui/db/shared` V1 framing with plugins, worktree management, multiple adapters, invites, and richer auth. Docs acknowledge much of this, but the operational surface is now larger than a casual reader would expect.
- `docs/` and `doc/` both exist, which is workable but easy to confuse for new contributors.

## Known Bugs

- UI worktree-related issue property editing is intentionally disabled in multiple places via `TODO(issue-worktree-support)` comments:
  - `ui/src/adapters/runtime-json-fields.tsx`
  - `ui/src/components/ProjectProperties.tsx`
  - `ui/src/components/IssueProperties.tsx`
  - `ui/src/components/NewIssueDialog.tsx`
- That suggests the underlying capability exists in backend/runtime layers but is not yet ready for the board UX.

## Security Considerations

- `server/src/auth/better-auth.ts` falls back to a development secret (`paperclip-dev-secret`) when no auth secret env var is set. That is convenient for local use but dangerous if an authenticated deployment is exposed without explicit secret configuration.
- Agent execution paths deliberately pass substantial runtime context via environment variables and local process spawning. That is necessary for adapters, but it increases the importance of log redaction and careful env handling.
- Plugin installation and runtime activation rely on local filesystem access, `npm`, and worker process management. That is a meaningful trust boundary and should stay tightly controlled in non-local deployments.

## Performance Bottlenecks

- The server process is responsible for API handling, scheduler checks, websocket fanout, plugin lifecycle management, and heartbeat orchestration. There is no separate queue/worker tier in the current architecture.
- Local development can also run Vite middleware inside the server process, which is convenient but further centralizes load.
- Heavy plugin or heartbeat activity will contend for the same Node process and host filesystem resources.

## Fragile Areas

- Company-scoping rules are essential and cross-cutting. Any route/service/schema change that forgets company boundaries risks breaking a core product invariant.
- Heartbeat/workspace/session code is sensitive to local environment assumptions such as writable directories, git worktree availability, and external CLI/tool behavior.
- Plugin runtime services span validation, registry state, worker lifecycle, event bus, jobs, tools, and UI slot plumbing. Failures here are likely to be integration failures rather than isolated syntax errors.

## Scaling Limits

- Architecture is currently single-process for orchestration, with in-memory coordination for some live/runtime state.
- Websocket subscriptions and event buses are process-local.
- Plugin workers and local agent processes assume a host that can spawn and manage subprocesses; this does not trivially map to serverless or highly locked-down runtimes.

## Dependencies at Risk

- `embedded-postgres` is still a beta dependency, though it is central to the zero-config story.
- Adapter usefulness depends on third-party CLIs and API keys outside this repo’s control.
- Better Auth is integrated directly against Drizzle tables and deployment-mode logic; auth library changes could have wide blast radius.

## Missing Critical Features

- There is no visible repo-wide lint/format enforcement in package scripts or CI, so style drift and some classes of code-quality regressions depend on manual review.
- The repo has rich plugin and adapter surfaces, but the corresponding docs and tests still appear more mature for core onboarding/runtime paths than for every extension scenario.

## Test Coverage Gaps

- Browser E2E coverage is narrow relative to the number of board pages (`ui/src/pages/`), especially for plugins, approvals, costs, and multi-company flows.
- The server test suite is broad, but runtime interactions that depend on real external CLIs or real hosted providers still rely heavily on environment-sensitive tests or manual validation.
- Plugin examples and plugin-host interactions exist, but full-stack coverage of install -> activate -> UI slot -> webhook/job/tool behavior appears thinner than core route coverage.

---

*Concerns analysis: 2026-03-15*
*Update when major risks are addressed or new fragile areas emerge*
