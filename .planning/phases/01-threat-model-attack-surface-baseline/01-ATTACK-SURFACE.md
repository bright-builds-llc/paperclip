# Phase 1 Attack Surface Inventory

This inventory maps the meaningful Paperclip entry points to concrete files so later phases can review exploitability instead of repeating discovery.

## REST API

The main API surface is assembled in `server/src/app.ts`. It mounts common middleware first, then route families for health, companies, agents, assets, projects, issues, goals, approvals, secrets, costs, activity, dashboard, sidebar badges, plugins, and access management.

| Surface | Principal(s) | Boundary crossed | Key anchors |
|---|---|---|---|
| `/api/auth/*` and `/api/auth/get-session` | Board users, browsers | Session resolution, trusted-origin handling | `server/src/app.ts`, `server/src/auth/better-auth.ts` |
| `/api/companies/*`, `/api/agents/*`, `/api/issues/*`, `/api/goals/*`, `/api/projects/*` | Board users, agents on selected routes | Core company-scoped CRUD and workflow mutation | `server/src/app.ts`, `server/src/routes/*.ts`, `server/src/routes/authz.ts` |
| `/api/secrets/*` | Board users | Secret creation, rotation, deletion, and provider selection | `server/src/routes/secrets.ts`, `server/src/services/secrets.ts` |
| `/api/assets/*` | Board users, agents depending on route | Company-scoped object storage, attachment access, object-key enforcement | `server/src/routes/assets.ts`, `server/src/storage/service.ts` |
| `/api/access/*` | Board users, invite links, OpenClaw joiners, bootstrap claimants | Invite issuance, join requests, board claim, membership/permission changes, public-ish onboarding routes | `server/src/routes/access.ts`, `server/src/board-claim.ts` |
| `/api/plugins/*` | Board users, webhook senders, plugin UI bridge callers | Plugin install/upgrade/config, health checks, logs, jobs, tools, webhooks, bridge calls | `server/src/routes/plugins.ts`, `server/src/services/plugin-loader.ts` |

### Route-Shaping Middleware

- `server/src/middleware/private-hostname-guard.ts`
  - gates authenticated/private deployments by hostname policy
- `server/src/middleware/auth.ts`
  - resolves local-trusted board auth, Better Auth sessions, agent API keys, and local agent JWTs
- `server/src/middleware/board-mutation-guard.ts`
  - blocks unsafe board mutations on protected surfaces
- `server/src/routes/authz.ts`
  - central company access helper used across route families

### Public Or Semi-Public REST Surfaces Worth Extra Attention

- invite acceptance and join-request submission in `server/src/routes/access.ts`
- board-claim routes in `server/src/routes/access.ts`
- plugin webhook ingestion in `server/src/routes/plugins.ts`
- skill/index skill-markdown routes in `server/src/routes/access.ts`

These routes are higher-value because they intentionally serve actors before the normal “signed-in board” steady state exists.

## WebSocket

Paperclip has a dedicated live-events WebSocket upgrade path at `/api/companies/:companyId/events/ws`.

| Surface | Principal(s) | Boundary crossed | Key anchors |
|---|---|---|---|
| Live events WebSocket | Local board, authenticated board user, agent bearer key | HTTP upgrade auth, company-scoped event subscription | `server/src/realtime/live-events-ws.ts`, `server/src/services/live-events.ts` |

Important details from the current implementation:

- bearer token may arrive through `Authorization` or the `token` query parameter
- in `local_trusted`, a browser without a token still becomes a board subscriber
- in `authenticated`, session headers are resolved through Better Auth for board subscriptions
- agent websocket access is limited by bearer key company match

This surface needs explicit parity review in Phase 2 because it does not reuse the main Express request middleware directly.

## CLI

The CLI is a security-relevant operator surface because it can change deployment mode, bootstrap auth, export data, and prepare isolated worktrees.

| Command family | What it touches | Key anchors |
|---|---|---|
| `paperclipai onboard` | Generates config, chooses deployment/auth/storage/secrets defaults, can bootstrap CEO invite flow in some modes | `cli/src/index.ts`, `cli/src/commands/onboard.ts` |
| `paperclipai doctor` | Validates and optionally repairs config, JWT, secrets, storage, DB, logging, and deployment/auth setup | `cli/src/index.ts`, `cli/src/commands/doctor.ts`, `cli/src/checks/*.ts` |
| `paperclipai run` | Auto-onboards, runs doctor, and starts the instance | `cli/src/index.ts`, `cli/src/commands/run.ts` |
| `paperclipai auth bootstrap-ceo` | Generates one-time bootstrap invite URLs for first instance admin | `cli/src/index.ts`, `cli/src/commands/auth-bootstrap-ceo.ts` |
| `paperclipai db:backup` | Creates one-off logical DB backups using configured or derived connection strings | `cli/src/index.ts`, `cli/src/commands/db-backup.ts` |
| `paperclipai allowed-hostname` | Expands authenticated/private hostname allowlist | `cli/src/index.ts`, `cli/src/commands/allowed-hostname.ts` |
| `paperclipai worktree *` | Creates repo-local Paperclip configs, isolated instances, DB clones, and git worktrees | `cli/src/index.ts`, `cli/src/commands/worktree.ts`, `cli/src/commands/worktree-lib.ts` |
| `paperclipai heartbeat run` | Invokes one agent heartbeat and streams live logs | `cli/src/index.ts`, `cli/src/commands/heartbeat-run.ts` |

## Adapters And Heartbeat Execution

Heartbeat execution is the highest-value host interaction surface in the current repo.

| Surface | Trigger input | Why it matters | Key anchors |
|---|---|---|---|
| Heartbeat orchestration | Agent config, runtime config, wake context, project/session workspace state | Central path for child-process execution, env injection, workspace choice, runtime services, and log persistence | `server/src/services/heartbeat.ts` |
| Process adapter | Server-side execution of configured commands | Classic command construction and env propagation boundary | `server/src/adapters/process/execute.ts` |
| HTTP adapter | Outbound callback-style execution | Remote execution boundary instead of local shelling out | `server/src/adapters/http/execute.ts` |
| Local coding adapters | Claude, Codex, Cursor, Gemini, OpenCode, PI local wrappers | Rich local execution surfaces with tool injection, env shaping, and session handling | `packages/adapters/*/src/server/*.ts` |

Notable execution-adjacent behaviors:

- `server/src/services/heartbeat.ts`
  - resolves workspaces and runtime services before execution
  - creates local agent JWTs for some adapter flows
  - redacts selected env keys before logging, but still handles large runtime context objects
  - persists run logs and excerpts
- `packages/adapters/codex-local/src/server/execute.ts`
  - prepares `CODEX_HOME`
  - injects Paperclip skills into Codex
  - builds agent/run/workspace env vars
- equivalent adapter packages exist for Claude, Cursor, Gemini, OpenCode, PI, and OpenClaw gateway

This bucket drives most of Phase 3.

## Workspace And Worktree

Workspace and worktree handling is its own attack surface because it can create directories, branches, provision commands, and runtime services on the host.

| Surface | Security-relevant behavior | Key anchors |
|---|---|---|
| Execution workspace realization | Decides whether a run uses project primary workspace, task session workspace, or agent home | `server/src/services/heartbeat.ts`, `server/src/services/execution-workspace-policy.ts` |
| Git worktree strategy | Creates/switches worktrees, derives branch names, can run repo-defined provision commands | `server/src/services/workspace-runtime.ts` |
| Runtime services | Starts local_process or adapter-managed services bound to a workspace or run | `server/src/services/workspace-runtime.ts` |
| CLI worktree init/make/env | Creates isolated instance configs, copies hooks, seeds databases, and chooses ports | `cli/src/commands/worktree.ts`, `cli/src/commands/worktree-lib.ts` |

Phase 3 should review:

- git command construction
- branch-name/path sanitization
- worktree path containment
- provision command trust assumptions
- runtime service env and lifecycle handling

## Plugins

Plugins add multiple separate surfaces, not one.

| Surface | What it exposes | Key anchors |
|---|---|---|
| Discovery and install | Scans local plugin dir and npm, installs via `npm install --ignore-scripts`, accepts local paths | `server/src/services/plugin-loader.ts` |
| Activation and worker startup | Resolves worker entrypoint, builds host handlers, starts child worker, syncs jobs, registers tools | `server/src/services/plugin-loader.ts`, `server/src/services/plugin-worker-manager.ts` |
| Worker RPC | JSON-RPC over stdio between host and worker | `server/src/services/plugin-worker-manager.ts`, `packages/plugins/sdk/src/worker-rpc-host.ts` |
| Tool execution | Board-triggered or agent-context plugin tool runs | `server/src/routes/plugins.ts`, `server/src/services/plugin-tool-dispatcher.ts` |
| UI bundles | Static serving from plugin UI directories under `/_plugins/:pluginId/ui/*` | `server/src/routes/plugin-ui-static.ts` |
| Bridge routes | `getData`, `performAction`, REST-friendly `data/:key`, `actions/:key`, and stream/SSE bridge | `server/src/routes/plugins.ts` |
| Webhook ingestion | Unauthenticated inbound deliveries when plugin declares receive capability | `server/src/routes/plugins.ts` |
| Example plugins | Bundled local-path example installs and example worker/UI code | `server/src/routes/plugins.ts`, `packages/plugins/examples/*` |

Important hardening notes from the repo snapshot:

- `plugin-loader` uses `execFile()` and `npm install --ignore-scripts` for npm installs
- plugin workers get a constrained env in `plugin-worker-manager` rather than inheriting full `process.env`
- `plugin-ui-static` defends against path traversal by resolving and validating requested paths
- a VM sandbox helper exists in `server/src/services/plugin-runtime-sandbox.ts`, but Phase 1 found no current runtime caller

This bucket drives most of Phase 4.

## Secrets And Storage

| Surface | Security concern | Key anchors |
|---|---|---|
| Secret CRUD and versioning | Secret creation, rotation, provider-backed material storage, inline env normalization | `server/src/routes/secrets.ts`, `server/src/services/secrets.ts` |
| Local encrypted secrets | Master-key file path, strict mode, local provider behavior | `doc/DATABASE.md`, `server/src/config.ts`, `server/src/secrets/local-encrypted-provider.ts` |
| Object storage | Company-prefixed object keys, namespace sanitization, retrieval/delete enforcement | `server/src/storage/service.ts`, `server/src/storage/local-disk-provider.ts`, `server/src/storage/s3-provider.ts` |
| Run logs | Per-run NDJSON logs stored under instance data | `server/src/services/run-log-store.ts`, `server/src/services/heartbeat.ts` |

These surfaces matter even in Phase 1 because they define where later phases should look for data exposure, secret leakage, and persistence risks.

## Release And Supply Chain

| Surface | Why it matters | Key anchors |
|---|---|---|
| GitHub Release workflow | Verifies only on `release/*` branches, installs from lockfile, runs release script, pushes stable tags, creates GitHub release | `.github/workflows/release.yml` |
| `scripts/release.sh` | Rewrites package versions, prepares publish artifacts, restores or cleans release state, checks npm publish auth | `scripts/release.sh`, `scripts/release-lib.sh` |
| `scripts/create-github-release.sh` | Publishes or updates GitHub releases from repo tags and release notes | `scripts/create-github-release.sh` |
| Refresh-lockfile / PR workflows | Influence what dependency graph is considered valid before merge/publish | `.github/workflows/refresh-lockfile.yml`, `.github/workflows/pr-verify.yml`, `.github/workflows/pr-policy.yml` |

The release chain is important because the repo explicitly treats lockfile ownership, npm publish auth, and generated publish artifacts as part of the trusted delivery path.

## Hotspots

These are the highest-value files to review first in later phases.

| File | Why it is a hotspot |
|---|---|
| `server/src/middleware/auth.ts` | Defines implicit board auth, session resolution, API key auth, and local JWT fallback |
| `server/src/auth/better-auth.ts` | Controls trusted origins, cookie security behavior, and secret fallback |
| `server/src/realtime/live-events-ws.ts` | Separate auth path for websocket upgrades; accepts query-token auth |
| `server/src/routes/access.ts` | Largest pre-auth and join/invite surface: bootstrap, claim, invite acceptance, OpenClaw join defaults, member/admin updates |
| `server/src/services/heartbeat.ts` | Main execution pipeline: workspace resolution, runtime services, env shaping, adapter dispatch, logs |
| `server/src/services/workspace-runtime.ts` | Git worktree creation, provision commands, runtime services, host filesystem mutation |
| `server/src/services/plugin-loader.ts` | Plugin install from npm/local path, activation, worker startup, job/tool registration |
| `server/src/services/plugin-worker-manager.ts` | Child worker process lifecycle and host/worker RPC boundary |
| `server/src/routes/plugins.ts` | Broadest plugin-facing HTTP surface, including unauthenticated webhook delivery and UI bridge routes |
| `cli/src/commands/worktree.ts` | High-impact local operator flow: worktrees, hook copying, DB seeding, port allocation |
| `cli/src/commands/db-backup.ts` | Direct data-export surface driven by config or env-derived connection strings |
| `.github/workflows/release.yml` | Release-chain trust root for publish verification and stable tag push |

