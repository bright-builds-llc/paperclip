# Phase 1 Research: Threat Model & Attack Surface Baseline

## Planning Question

What does a maintainer need documented before deeper security review starts so later phases can distinguish real vulnerabilities from intentional local-only sharp edges?

## Phase Contract

Phase 1 is the baseline-setting phase for the audit:

- `THRT-01`: document attacker profiles, principals, deployment modes, and trust boundaries
- `THRT-02`: map meaningful entry points and attack surfaces to concrete files, routes, commands, and runtime flows

The roadmap also requires a severity rubric and evidence standard before later review phases begin.

## Repo-Grounded Facts That Shape The Plan

- Paperclip is a control plane with a multi-company data model and company-scoped invariants, even though V1 is single-tenant at the deployment level.
- Human auth is mode-dependent:
  - `local_trusted` gives implicit board access with no login flow
  - `authenticated` uses Better Auth sessions and supports `private` and `public` exposure policies
- Agent auth uses bearer API keys hashed at rest, plus local agent JWTs for some runtime flows.
- The server orchestrates external execution paths instead of only storing data:
  - heartbeat runs
  - adapter execution
  - workspace/worktree realization
  - plugin worker startup
- Local defaults are intentionally powerful:
  - embedded Postgres
  - local disk storage
  - local encrypted secrets
  - local plugin directory
  - worktree-local instance support
- The repo already contains plugin, adapter, CLI, and release surfaces that materially expand attack surface beyond ordinary CRUD routes.

## Principals To Model

These are the principals and attacker positions the phase should document explicitly.

| Principal / Actor | Why it matters | Core evidence |
|---|---|---|
| Board operator in `local_trusted` mode | Has implicit board authority without login; baseline for intentional sharp edges | `server/src/middleware/auth.ts`, `doc/DEPLOYMENT-MODES.md` |
| Board user in `authenticated` mode | Session-based human principal with company membership and instance-admin differences | `server/src/middleware/auth.ts`, `server/src/auth/better-auth.ts`, `server/src/routes/authz.ts` |
| Agent principal | Bearer API keys and local JWTs can mutate company-scoped resources and run heartbeats | `server/src/middleware/auth.ts`, `server/src/agent-auth-jwt.ts`, `server/src/routes/authz.ts` |
| Plugin author / plugin worker | Plugin install, activation, RPC, jobs, tools, UI, and worker processes can expand host privilege | `server/src/services/plugin-loader.ts`, `server/src/routes/plugins.ts`, `packages/plugins/sdk/src/worker-rpc-host.ts` |
| Local operator / CLI user | CLI commands can bootstrap auth, configure instances, run backups, and initialize worktrees | `cli/src/index.ts`, `cli/src/commands/*` |
| Network client / browser / WebSocket subscriber | REST and WebSocket surfaces differ in transport and auth behavior | `server/src/app.ts`, `server/src/realtime/live-events-ws.ts` |
| Repo contributor / release actor | CI, release, package publishing, and example plugins influence supply-chain trust | `.github/workflows/*.yml`, `scripts/release.sh`, workspace packages |

## Deployment And Exposure Assumptions

The threat model needs a mode-aware section, otherwise later findings will mix intended local-trusted behavior with remotely exploitable issues.

| Mode | Security meaning | Phase 1 implication |
|---|---|---|
| `local_trusted` | Loopback-oriented, no login, optimized for single-operator local use | Record sharp edges separately from vulnerabilities unless they cross stated local assumptions |
| `authenticated` + `private` | Login required, private-hostname trust policy, lower-friction URL handling | Review auth/session and hostname controls as network-reachable surfaces |
| `authenticated` + `public` | Login required, explicit public URL, stricter deployment checks | Treat insecure defaults and auth mistakes as internet-facing exposure candidates |

Important concrete defaults to anchor in the threat model:

- `actorMiddleware()` grants implicit board authority in `local_trusted`
- Better Auth falls back to `BETTER_AUTH_SECRET`, then `PAPERCLIP_AGENT_JWT_SECRET`, then `"paperclip-dev-secret"`
- `companyDeletionEnabled` defaults to `true` in `local_trusted`
- database backups are enabled by default
- local storage, secrets, plugins, and workspaces resolve under instance or home paths

## Trust Boundaries To Capture

The phase should define these boundaries before any finding triage:

1. HTTP request -> actor resolution
2. Actor principal -> company-scoped authorization
3. Browser/session context -> WebSocket upgrade authorization
4. Board/API action -> DB mutation and activity log
5. Server orchestration -> child process / adapter execution
6. Server orchestration -> workspace, worktree, filesystem, and runtime services
7. Server host -> plugin package install, activation, worker RPC, jobs, tools, and UI assets
8. Server -> secrets provider and storage provider
9. Local CLI / onboarding / worktree helper -> config, env files, backups, instance data
10. Repo / CI / release actor -> built artifacts, published packages, and release branch automation

## Attack Surface Buckets

Phase 1 should inventory surfaces by bucket so later phases can review them without rediscovery.

| Surface bucket | Why it is in scope | File clusters to anchor |
|---|---|---|
| REST API assembly and middleware | Entry point for most board and agent operations | `server/src/app.ts`, `server/src/middleware/*.ts`, `server/src/routes/*.ts` |
| Human auth and session resolution | Controls authenticated board access and trusted origins | `server/src/auth/better-auth.ts`, `server/src/middleware/auth.ts`, `server/src/routes/access.ts` |
| Company-scoped authz helpers | Core tenant-isolation boundary | `server/src/routes/authz.ts`, service and route call sites |
| WebSocket live events | Separate upgrade path with token/query/session behavior | `server/src/realtime/live-events-ws.ts`, `server/src/services/live-events.ts` |
| Heartbeat and adapter execution | Highest-value execution boundary; can reach host processes, env, logs, and workspaces | `server/src/services/heartbeat.ts`, `server/src/adapters/*`, `packages/adapters/*/src/server/*.ts` |
| Workspace, worktree, runtime services | Determines filesystem and repo boundary enforcement | `server/src/services/workspace-runtime.ts`, `server/src/services/execution-workspace-policy.ts`, `cli/src/commands/worktree*.ts` |
| Plugin install and runtime | Introduces npm/local-path loading, worker processes, RPC, jobs, tools, webhooks, and UI routes | `server/src/services/plugin-loader.ts`, `server/src/services/plugin-*.ts`, `server/src/routes/plugins.ts`, `server/src/routes/plugin-ui-static.ts`, `packages/plugins/sdk/src/*` |
| Secrets and storage | Sensitive data persistence, redaction, and provider behavior | `server/src/routes/secrets.ts`, `server/src/services/secrets.ts`, `server/src/storage/*`, `packages/db/src/schema/company_secrets.ts` |
| CLI and operator flows | Setup, doctor, backup, auth bootstrap, and run commands can alter trust assumptions | `cli/src/index.ts`, `cli/src/commands/onboard.ts`, `cli/src/commands/doctor.ts`, `cli/src/commands/auth-bootstrap-ceo.ts`, `cli/src/commands/db-backup.ts`, `cli/src/commands/run.ts` |
| CI, release, and package publishing | Supply-chain and release-integrity surface | `.github/workflows/release.yml`, `.github/workflows/pr-verify.yml`, package manifests, release scripts |

## Concrete Hotspots Already Worth Flagging

These are not final findings yet. They are the evidence-backed hotspots that justify the Phase 1 plan split.

1. `server/src/middleware/auth.ts`
   - implicit board identity in `local_trusted`
   - session lookup in `authenticated`
   - fallback from API keys to local agent JWTs
2. `server/src/auth/better-auth.ts`
   - trusted-origin derivation and secret fallback behavior
3. `server/src/realtime/live-events-ws.ts`
   - token from header or query string
   - session-based upgrade path for board clients
4. `server/src/services/heartbeat.ts`
   - child-process execution, env propagation, session state, workspace resolution, and log persistence
5. `server/src/services/plugin-loader.ts`
   - npm install or local-path loading
   - manifest and capability validation
   - worker startup and host-handler wiring
6. `server/src/routes/plugins.ts`
   - plugin lifecycle control plane, bundled example exposure, tool and bridge routes
7. `server/src/config.ts`
   - deployment/auth/storage/secrets defaults with env and file overrides
8. `cli/src/index.ts` plus onboarding/worktree/auth bootstrap commands
   - first-run, repair, invite, backup, and worktree setup flows
9. `.github/workflows/release.yml`
   - release-branch-only publish path and package verification assumptions

## Recommended Execution Outputs

Phase 1 should create these artifacts:

- `01-THREAT-MODEL.md`
  - principals
  - assets
  - deployment modes
  - trust boundaries
  - explicit assumptions and out-of-scope notes
- `01-ATTACK-SURFACE.md`
  - file-grounded inventory of REST, WebSocket, CLI, adapter, plugin, storage, and release surfaces
  - high-risk subsystem markers
- `01-AUDIT-METHODOLOGY.md`
  - severity rubric
  - exploitability and impact scoring guidance
  - accepted-risk vs confirmed-vulnerability rules
- `01-FINDINGS-LOG.md`
  - reusable evidence template for later phases

## Recommended Plan Split

Wave 1 can run in parallel:

- `01-01`: threat model, principals, deployment modes, and trust boundaries
- `01-02`: file-grounded attack-surface inventory

Wave 2 should depend on both:

- `01-03`: audit methodology, severity rubric, evidence rules, and findings log template

This keeps the baseline docs independent where possible and makes the rubric depend on the actual surface inventory rather than guesses.

## Verification Expectations For Phase Execution

When Phase 1 is executed, each artifact should be checked for:

- concrete file references rather than generic prose
- explicit distinction between `local_trusted` sharp edges and authenticated/public risk
- coverage of REST, WebSocket, CLI, execution, plugin, data, and release surfaces
- reusable severity and evidence guidance that later phases can apply consistently
