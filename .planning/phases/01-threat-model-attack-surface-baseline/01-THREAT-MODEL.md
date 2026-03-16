# Phase 1 Threat Model

## Scope

This document fixes the baseline security model for the Paperclip repo audit.

- Review target: repository code, local operator flows, bundled adapters/plugins, and release automation
- Review goal: identify where Paperclip could permit auth bypass, tenant boundary failure, arbitrary execution, filesystem escape, secret disclosure, unsafe default exposure, or release-chain trust failure
- Review style: repo-grounded threat modeling, not a live penetration test of an internet deployment
- Decision boundary for later phases:
  - `local_trusted` behavior can be an intentional sharp edge when it stays inside its documented trust assumptions
  - the same behavior becomes a real vulnerability when it violates documented mode boundaries, leaks across companies, or stays reachable in authenticated/public deployments

## Protected Assets

| Asset | Why it matters | Primary anchors |
|---|---|---|
| Board authority | Board principals can create companies, mutate config, approve joins, manage plugins, rotate secrets, and steer all agent work | `server/src/middleware/auth.ts`, `server/src/routes/authz.ts`, `server/src/routes/access.ts` |
| Company-scoped records | Companies, agents, issues, approvals, secrets, assets, and activity logs are the core tenant boundary | `doc/SPEC-implementation.md`, `packages/db/src/schema/*.ts`, `server/src/routes/*.ts` |
| Agent credentials | Agent API keys and local agent JWTs can authenticate automation and reach company-scoped APIs | `server/src/middleware/auth.ts`, `server/src/agent-auth-jwt.ts` |
| Secret material | Secret values, secret refs, master keys, and adapter env bindings can unlock external systems or Paperclip internals | `server/src/services/secrets.ts`, `server/src/routes/secrets.ts`, `doc/DATABASE.md` |
| Host execution surface | Heartbeats, adapters, plugin workers, runtime services, and worktree provisioning can spawn processes or mutate the local host | `server/src/services/heartbeat.ts`, `server/src/services/workspace-runtime.ts`, `server/src/services/plugin-worker-manager.ts` |
| Workspace and repository state | Project workspaces, git worktrees, provision commands, and runtime services can touch repos and developer machines | `server/src/services/workspace-runtime.ts`, `cli/src/commands/worktree.ts` |
| Plugin runtime boundary | Plugin packages, workers, UI bundles, webhooks, tools, and bridge endpoints can expand host capabilities | `server/src/services/plugin-loader.ts`, `server/src/routes/plugins.ts`, `server/src/routes/plugin-ui-static.ts` |
| Release credentials and artifacts | npm publish auth, GitHub release auth, tags, release notes, and generated package contents control supply-chain integrity | `.github/workflows/release.yml`, `scripts/release.sh`, `scripts/create-github-release.sh` |

## Principals

| Principal | Current implementation reality | Main authority / risk |
|---|---|---|
| Unauthenticated client | Requests begin as `req.actor.type === "none"` unless upgraded by local-trusted defaults, session lookup, or bearer auth | Attempts auth bypass, invite abuse, public surface discovery |
| Local board operator | In `local_trusted`, `actorMiddleware()` assigns implicit board authority with `userId: "local-board"` and instance-admin power | Full-control local board surface without login friction |
| Authenticated board user | In `authenticated`, Better Auth sessions resolve a real user, active company memberships, and instance-admin status | Full board access for allowed companies; higher blast radius for instance admins |
| Agent principal | Bearer API keys are hashed at rest; local agent JWTs are accepted when `PAPERCLIP_AGENT_JWT_SECRET` is configured | Company-scoped automation, heartbeat-triggered mutations, potential cross-company risk if checks fail |
| Plugin package / plugin worker | Installed plugins can contribute workers, jobs, webhooks, UI, and tools; workers run as child processes with host RPC access | Host capability expansion, execution surface, and data-access widening |
| Local CLI operator | CLI can onboard instances, repair config, create auth bootstrap invites, run backups, and create worktree-local clones | Local environment mutation, data export, and trust-boundary reconfiguration |
| Release actor / maintainer | Release scripts and GitHub workflows can rewrite package metadata, publish packages, push tags, and create releases | Supply-chain integrity and artifact trust |

### Attacker Profiles

Later phases should evaluate at least these attacker positions:

1. Unauthenticated remote or private-network client seeking board or agent access.
2. Authenticated board user attempting access beyond company membership or intended instance role.
3. Agent principal trying to reach another company, another run, or broader host state.
4. Plugin author or compromised plugin package seeking host privilege expansion.
5. Local operator mistake or malicious local user abusing trusted defaults, backups, or worktree commands.
6. Supply-chain actor who can influence package install, release automation, or published artifacts.

## Deployment Modes

Paperclip has one low-friction local mode and one authenticated mode with two exposure policies.

| Mode | Intended trust assumption | What later phases should treat as expected | What later phases should treat as suspicious |
|---|---|---|---|
| `local_trusted` | Single-operator, loopback-oriented workflow with no human login | Implicit board authority, local-only convenience defaults, low-friction startup | Any behavior that escapes local-only assumptions, crosses companies, leaks secrets, or remains reachable when the deployment is no longer strictly local |
| `authenticated` + `private` | Login required, private hostname policy, private-network exposure | Session-based board access plus allowlisted hostnames | Missing auth checks, weak hostname controls, join/bootstrap flows reachable without intended guardrails |
| `authenticated` + `public` | Login required, explicit public URL, internet-facing posture | Stricter deployment validation and explicit public base URL handling | Unsafe defaults, permissive session or origin handling, public unauthenticated mutation or data exposure |

### Code-Level Mode Controls

- `server/src/middleware/auth.ts`
  - grants implicit board authority in `local_trusted`
  - resolves Better Auth sessions in `authenticated`
  - accepts bearer agent API keys and local agent JWTs
- `server/src/config.ts`
  - defaults `deploymentMode` to `local_trusted`
  - forces `deploymentExposure` to `private` in `local_trusted`
  - defaults `companyDeletionEnabled` to `true` in `local_trusted`
  - keeps database backup enabled by default
- `server/src/auth/better-auth.ts`
  - derives trusted origins from configured base URL and allowed hostnames
  - falls back from `BETTER_AUTH_SECRET` to `PAPERCLIP_AGENT_JWT_SECRET` to `"paperclip-dev-secret"`
- `doc/DEPLOYMENT-MODES.md`
  - defines the intended separation between local-only convenience and authenticated/private/public posture

## Trust Boundaries

| Boundary | Expected guarantee | Key anchors |
|---|---|---|
| HTTP request -> actor resolution | Requests must become board, agent, or stay unauthenticated only through explicit mode-aware auth logic | `server/src/middleware/auth.ts`, `server/src/auth/better-auth.ts`, `server/src/agent-auth-jwt.ts` |
| Actor -> company-scoped authorization | Board users must stay within memberships unless instance admin; agents must stay inside their own company | `server/src/routes/authz.ts`, route/service call sites |
| Session/browser -> WebSocket subscription | Live-event websocket connections must enforce equivalent company access to the main API | `server/src/realtime/live-events-ws.ts`, `server/src/services/live-events.ts` |
| Local board migration -> authenticated board ownership | Authenticated deployments must safely transfer admin authority away from `local-board` | `server/src/board-claim.ts`, `server/src/routes/access.ts` |
| Board/API action -> durable state + activity log | Mutating actions should be attributable and company-scoped | `server/src/routes/*.ts`, `server/src/services/activity-log.ts` |
| Server orchestration -> child process / adapter runtime | Heartbeats and adapters must not execute attacker-controlled commands or leak excessive env/runtime context | `server/src/services/heartbeat.ts`, `packages/adapters/*/src/server/*.ts` |
| Server -> workspace/worktree/repository state | Workspaces and worktrees must stay inside intended repo and host boundaries | `server/src/services/workspace-runtime.ts`, `cli/src/commands/worktree.ts` |
| Server -> plugin package / worker / UI | Plugin install, activation, worker RPC, UI serving, and webhooks must not silently grant broader host or company access than declared | `server/src/services/plugin-loader.ts`, `server/src/services/plugin-worker-manager.ts`, `server/src/routes/plugins.ts`, `server/src/routes/plugin-ui-static.ts` |
| Server -> secret and storage providers | Secret material and stored objects must stay company-scoped and avoid accidental plaintext/log exposure | `server/src/services/secrets.ts`, `server/src/routes/secrets.ts`, `server/src/storage/service.ts`, `server/src/services/run-log-store.ts` |
| Local operator -> config, backups, releases | Operator tooling should make privilege changes and data export explicit, not implicit | `cli/src/index.ts`, `cli/src/commands/onboard.ts`, `cli/src/commands/db-backup.ts`, `.github/workflows/release.yml`, `scripts/release.sh` |

### Boundary Notes For Later Phases

- WebSocket auth is not a clone of the REST middleware. It has its own upgrade path and can accept bearer tokens from headers or query parameters.
- Plugin runtime hardening is split:
  - real enforcement exists in worker process isolation and controlled worker env setup
  - a VM sandbox helper exists in `server/src/services/plugin-runtime-sandbox.ts`, but Phase 1 found no current call site using it
- Local operator tooling is part of the threat model because Paperclip intentionally exposes strong local control surfaces like worktree creation, backups, and bootstrap invite generation.

## Assumptions

- The primary audit target is the repo snapshot, not a hardened production deployment.
- A Paperclip deployment is single-tenant at the infrastructure level but multi-company in data model and authorization logic.
- Company isolation is still security-relevant because one deployment can hold multiple companies and shared operator state.
- V1 intentionally allows low-friction local operation; later phases must separate documented local-trusted behavior from bugs that cross that boundary.
- Plugin packages, adapter configs, repo data, and invite flows are all attacker-controlled inputs in at least some scenarios.
- “Board access” is effectively full control of the deployment and all companies unless company membership or instance-role checks constrain it.

## Out of Scope

- Internet-facing pentest of a deployed Paperclip instance.
- Exhaustive third-party CVE triage disconnected from actual Paperclip code paths.
- Operating-system hardening outside what the repo directly provisions or documents.
- Non-security product design questions that do not affect trust boundaries or exploitability.

## Later Phase Questions

- Phase 2: Do REST and WebSocket paths enforce the same auth and company-scoping guarantees for board and agent principals?
- Phase 3: Which request, repo, plugin, or database-controlled inputs can reach child processes, workspace commands, or filesystem mutation?
- Phase 4: Does the plugin model preserve company and principal boundaries after install, activation, jobs, webhooks, tools, and UI bridge calls?
- Phase 5: Where can secrets, run logs, attachments, assets, and storage-provider artifacts leak sensitive company data?
- Phase 6: Which onboarding, doctor, worktree, CI, and release defaults can ship or preserve insecure states?
