# Phase 2 Research: Identity, Auth & Tenant Isolation

## Planning Question

Which authentication and authorization boundaries in Paperclip are most likely to allow board bypass, agent privilege escalation, or cross-company access, and how should the review split so those risks are checked without rediscovering the auth model?

## Phase Contract

Phase 2 is the first exploit-oriented review phase for the audit:

- `AUTH-01`: review whether board authentication and session handling include unsafe defaults, bypasses, or deployment-mode weaknesses
- `AUTH-02`: review whether agent authentication, company scoping, and permission checks prevent cross-company access and privilege escalation
- `AUTH-03`: review whether realtime and non-REST surfaces enforce the same access guarantees as the main API

Phase 1 already established the principal model, deployment modes, trust boundaries, severity rubric, and findings template. Phase 2 should now convert that baseline into concrete auth and tenant-isolation review artifacts.

## Repo-Grounded Facts That Shape The Plan

- Human auth is deployment-mode dependent:
  - `local_trusted` grants an implicit board actor with `userId: "local-board"` and instance-admin power
  - `authenticated` resolves board actors from Better Auth sessions
- Better Auth currently derives behavior from runtime config and environment:
  - trusted origins come from the public URL plus configured hostnames
  - the secret fallback chain is `BETTER_AUTH_SECRET` -> `PAPERCLIP_AGENT_JWT_SECRET` -> `"paperclip-dev-secret"`
  - email verification is not required for password auth
  - secure cookies are disabled when the public URL is `http://`
- Company access is enforced centrally by `assertCompanyAccess()` plus permission helpers in `accessService()`, but several routes have special-case behavior for instance admins, `local_implicit` board actors, or token-driven invite flows.
- Agent auth is dual-path:
  - bearer API keys are hashed at rest in `agent_api_keys`
  - local runtime JWTs are accepted through `verifyLocalAgentJwt()`
- The WebSocket live-events upgrade path performs its own auth and company checks rather than reusing Express middleware directly.
- Auxiliary auth-sensitive surfaces live outside ordinary CRUD patterns:
  - board claim bootstrap
  - invites and join requests
  - join-request API key claim
  - plugin route families and plugin UI/static serving

## Review Tracks The Plan Should Cover

### 1. Board Authentication And Session Posture

This track should answer:

- when a board actor exists without a login flow
- whether trusted-origin, cookie, signup, and hostname defaults are safe in `authenticated/private` and `authenticated/public`
- whether board bootstrap and recovery flows expand authority beyond intended assumptions

Primary evidence:

- `server/src/middleware/auth.ts`
- `server/src/auth/better-auth.ts`
- `server/src/middleware/private-hostname-guard.ts`
- `server/src/board-claim.ts`
- `server/src/routes/access.ts`
- `server/src/routes/authz.ts`
- `doc/DEPLOYMENT-MODES.md`

### 2. Agent Authentication, Permissions, And Company Scope

This track should answer:

- whether bearer keys and local JWTs grant equivalent or mismatched authority
- whether company membership and permission grants can be crossed or bypassed
- whether agent-facing routes consistently bind actions to the actor's company and effective permissions

Primary evidence:

- `server/src/middleware/auth.ts`
- `server/src/agent-auth-jwt.ts`
- `server/src/services/access.ts`
- `server/src/services/agent-permissions.ts`
- `server/src/routes/authz.ts`
- `server/src/routes/agents.ts`
- `server/src/routes/access.ts`
- `server/src/routes/companies.ts`
- `packages/db/src/schema/company_memberships.ts`
- `packages/db/src/schema/principal_permission_grants.ts`
- `packages/db/src/schema/agent_api_keys.ts`

### 3. Realtime And Auxiliary Authorization Parity

This track should answer:

- whether WebSocket upgrades enforce the same actor and company guarantees as HTTP routes
- whether token-driven or plugin-driven surfaces avoid standard authz helpers and drift from the main API's guarantees
- whether non-REST and auxiliary paths preserve tenant isolation and actor intent

Primary evidence:

- `server/src/realtime/live-events-ws.ts`
- `server/src/middleware/auth.ts`
- `server/src/routes/authz.ts`
- `server/src/routes/access.ts`
- `server/src/routes/plugins.ts`
- `server/src/routes/plugin-ui-static.ts`
- `server/src/middleware/private-hostname-guard.ts`

## Concrete Boundary Questions Worth Testing

These are the repo-backed questions the execution plans should resolve.

1. Does `local_trusted` leak into routes or modes that later assume authenticated board semantics?
2. Is the Better Auth secret and cookie/origin configuration safe by default for remotely reachable deployments?
3. Are there board bootstrap or invite paths that grant ownership, company access, or API credentials without the same checks as ordinary mutations?
4. Can agent JWTs, API keys, or route-specific fallback logic act across companies or with broader permissions than intended?
5. Do membership and permission helpers treat board users, instance admins, and agents consistently at route call sites?
6. Does the WebSocket upgrade path honor the same company and actor boundaries as REST endpoints?
7. Do plugin and auxiliary routes bypass central authz helpers or accept looser auth material than the main API?

## Existing Tests Worth Reusing As Evidence Anchors

- `server/src/__tests__/agent-auth-jwt.test.ts`
- `server/src/__tests__/private-hostname-guard.test.ts`
- `server/src/__tests__/invite-expiry.test.ts`
- `server/src/__tests__/invite-accept-gateway-defaults.test.ts`
- `server/src/__tests__/invite-accept-replay.test.ts`
- `server/src/__tests__/openclaw-invite-prompt-route.test.ts`
- `server/src/__tests__/board-mutation-guard.test.ts`
- `server/src/__tests__/companies-route-path-guard.test.ts`

These tests do not replace the review, but they identify where the repo already encodes intended auth behavior and where gaps are likely to be visible.

## Recommended Execution Outputs

Phase 2 should produce three artifacts:

- `02-BOARD-AUTH-REVIEW.md`
  - deployment-mode matrix
  - session and origin controls
  - board bootstrap and invite-related auth posture
  - findings or review leads
- `02-AGENT-AUTHZ-REVIEW.md`
  - agent credential paths
  - company-scope enforcement
  - membership and permission model review
  - findings or review leads
- `02-REALTIME-AUX-AUTHZ-REVIEW.md`
  - WebSocket authorization model
  - auxiliary and plugin route parity
  - mismatches against the HTTP auth model
  - findings or review leads

## Recommended Plan Split

Wave 1 can run in parallel:

- `02-01`: review board authentication, session resolution, deployment defaults, and bootstrap flows
- `02-02`: review agent keys, local JWTs, company memberships, grants, and route-level company scoping

Wave 2 should depend on both:

- `02-03`: compare realtime and auxiliary route authorization with the guarantees documented in the Wave 1 reviews

This split keeps the human and agent auth tracks independent first, then uses their outputs as the baseline for parity review.

## Verification Expectations For Phase Execution

When Phase 2 is executed, each artifact should be checked for:

- concrete file references to helpers, routes, schemas, and tests
- explicit distinction between `local_trusted` assumptions and remotely reachable authenticated risk
- coverage of both board and agent company-boundary rules
- direct comparison between HTTP auth guarantees and realtime or auxiliary-route behavior
- clear candidate findings, accepted-risk notes, or no-finding conclusions that can feed the shared findings log
