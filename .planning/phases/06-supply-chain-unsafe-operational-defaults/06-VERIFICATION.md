# Phase 6 Verification

**Phase:** 6  
**Name:** Supply Chain & Unsafe Operational Defaults  
**Status:** passed  
**Verified at:** 2026-03-16T12:01:38Z

## Goal Check

Phase 6 goal: determine whether build, onboarding, and release operations can ship insecure states or erode trust boundaries.

Result: **Goal verified**

- onboarding, run, doctor, worktree, Docker, CI, and release trust assumptions are documented in `06-OPERATIONAL-DEFAULTS-REVIEW.md`
- package-management, lockfile policy, manifest generation, build, and publish-chain integrity assumptions are documented in `06-PACKAGE-BUILD-PUBLISH-REVIEW.md`
- a likely medium-severity finding now exists for authenticated public deployments over plain HTTP because doctor only warns and `paperclipai run` still starts
- a likely medium-severity finding now exists for PR and release verification using different dependency graphs because mutable resolution is allowed in PR CI while lockfile freezing is deferred
- a likely low-severity finding now exists for CLI publish-manifest generation drifting from the CLI's real adapter import graph

## Must-Have Verification

Score: **14/14**

| Check | Result | Evidence |
|---|---|---|
| `06-OPERATIONAL-DEFAULTS-REVIEW.md` exists | Pass | File present in phase directory |
| `06-PACKAGE-BUILD-PUBLISH-REVIEW.md` exists | Pass | File present in phase directory |
| Operational review required sections | Pass | Scope, onboarding/run/doctor defaults, worktree/Docker/local-operator flows, CI/release assumptions, findings or review leads, later phase questions |
| Operational review documents bootstrap and operator defaults | Pass | Covers `onboard`, `run`, `doctor`, server prompts, default secret mode, and deployment-mode checks |
| Operational review documents worktree and local-state cloning behavior | Pass | Covers secrets-key copying, database seeding, inherited backup settings, and preserved weak defaults in worktree creation |
| Operational review captures the public-HTTP warning-only issue | Pass | Connects `doc/DEPLOYMENT-MODES.md`, `cli/src/checks/deployment-auth-check.ts`, and `cli/src/commands/run.ts` |
| Operational review separates local sharp edges from shipped deployment defaults | Pass | Distinguishes local-operator worktree and quickstart behavior from remotely reachable public-mode startup behavior |
| Package review required sections | Pass | Scope, lockfile and package-manager model, public package inventory and manifest rules, build and publish artifact generation, findings or review leads, later phase questions |
| Package review documents lockfile and package-manager policy | Pass | Covers PR policy, PR verify, refresh-lockfile, release, and e2e workflow behavior |
| Package review captures PR vs release dependency-graph drift | Pass | Documents `--no-frozen-lockfile` in PR CI versus `--frozen-lockfile` in release and e2e after deferred lockfile refresh |
| Package review maps CLI manifest generation to actual adapter imports | Pass | Connects `scripts/generate-npm-package-json.mjs` to `cli/src/adapters/registry.ts` and identifies omitted adapter packages |
| Both plan summaries exist | Pass | `06-01-SUMMARY.md` and `06-02-SUMMARY.md` are present |
| Phase requirement is satisfied | Pass | `SUPP-01` maps to completed review artifacts and summaries |
| Findings remain evidence-based and clearly classified | Pass | Likely vulnerabilities, accepted-risk sharp edges, and no-finding controls are tied to concrete docs, workflows, commands, and source files |

## Repository Verification

- `pnpm -r typecheck` - passed
- `pnpm test:run` - passed
- `pnpm build` - passed

Non-blocking observations:

- test startup still emits an existing duplicate dependency-key warning for `@paperclipai/adapter-openclaw-gateway` in `server/package.json`
- UI build still emits large chunk warnings from Vite

None of these issues block Phase 6 completion.

## Residual Risks

- The public-HTTP finding depends on a deployment actually being exposed without TLS termination, but the current CLI and doctor flow do not hard-fail that configuration even in the documented public mode.
- The dependency-graph drift finding matters most when manifests change or registry resolution moves between PR verification and the later lockfile-refresh or release path.
- The publish-manifest drift issue is currently low severity because shared dependencies mask the omission, but the hard-coded package list remains brittle and can silently break runtime dependency preservation later.
