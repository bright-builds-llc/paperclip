# Testing Patterns

**Analysis Date:** 2026-03-15

## Test Framework

- Vitest is the primary automated test framework for the monorepo (`vitest.config.ts`).
- Root Vitest projects currently target `packages/db`, `packages/adapters/opencode-local`, `server`, `ui`, and `cli`.
- Supertest is used for HTTP-focused server tests.
- Playwright provides browser-level E2E coverage in `tests/e2e/`.

## Test File Organization

- Server tests live in `server/src/__tests__/`.
- CLI tests live in `cli/src/__tests__/`.
- UI tests are colocated near hooks/lib utilities such as `ui/src/hooks/useCompanyPageMemory.test.ts`.
- DB/package tests are sparse but present where behavior is localized, for example `packages/db/src/runtime-config.test.ts`.
- E2E tests live separately in `tests/e2e/*.spec.ts`.

## Test Structure

- Most tests use `describe` / `it` / `expect` with straightforward Arrange-Act-Assert flow even when comments are omitted.
- Route tests often spin up a tiny Express app and exercise it via Supertest, as in `server/src/__tests__/health.test.ts`.
- CLI/config tests commonly create temp directories, write config files, run commands, and assert side effects.
- E2E tests are more explicitly narrated with section comments and wait-based assertions around UI transitions.

## Mocking

- The codebase appears to prefer lightweight fakes, temp files, and real module execution over deep mocking.
- Supertest-based route tests avoid a full network stack while still exercising Express middleware.
- Environment manipulation is common in CLI and adapter tests via `beforeEach` / `afterEach`.
- No central mocking framework or elaborate factory library stands out from the current repo snapshot.

## Fixtures and Factories

- Temporary filesystem state is a recurring fixture strategy, especially in CLI tests like `cli/src/__tests__/doctor.test.ts`.
- UI fixtures exist for specialized flows, for example `ui/src/fixtures/runTranscriptFixtures.ts`.
- E2E onboarding generates unique names at runtime to avoid collisions.
- Server tests tend to rely on helper modules close to the tested subsystem rather than a global fixture directory.

## Coverage

- Coverage breadth is strongest around backend routes, adapters, CLI flows, onboarding logic, auth/worktree behavior, and runtime helpers.
- The server test suite is comparatively broad, with dedicated tests for adapters, auth, invites, worktree/runtime services, plugins, and route guards.
- Browser coverage is thin relative to total UI surface: the visible Playwright flow centers on onboarding.
- There is no visible coverage threshold or reporting configuration in the checked-in Vitest config.

## Test Types

- Unit tests: shared helpers, UI hooks/lib functions, adapter parsing, config/state utilities.
- Route/integration tests: Express routes exercised with Supertest.
- Runtime/integration tests: adapter execution, plugin worker management, workspace runtime logic.
- CLI integration tests: config loading, doctor/onboard/worktree command behavior.
- E2E tests: full browser startup and onboarding workflow.

## Common Patterns

- Reset global env state before/after tests when commands mutate `process.env`.
- Use explicit fixture creation instead of relying on developer machine state.
- Favor high-signal scenario tests over snapshot-heavy testing.
- Test names usually read as behavior statements rather than implementation details.
- Build/verification expectations in docs and CI are: `pnpm -r typecheck`, `pnpm test:run`, and `pnpm build`.

---

*Testing analysis: 2026-03-15*
*Update when frameworks, major suites, or coverage expectations change*
