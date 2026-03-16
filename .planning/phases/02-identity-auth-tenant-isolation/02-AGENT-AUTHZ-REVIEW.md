# Phase 2 Agent Authz Review

## Scope

This artifact reviews agent authentication, company-scope enforcement, and permission handling for both agent actors and session-backed board users operating within company boundaries.

Primary files reviewed:

- `server/src/middleware/auth.ts`
- `server/src/agent-auth-jwt.ts`
- `server/src/routes/authz.ts`
- `server/src/services/access.ts`
- `server/src/services/agent-permissions.ts`
- `server/src/routes/agents.ts`
- `server/src/routes/access.ts`
- `server/src/routes/approvals.ts`
- `server/src/routes/costs.ts`
- `server/src/routes/activity.ts`
- `packages/db/src/schema/company_memberships.ts`
- `packages/db/src/schema/principal_permission_grants.ts`
- `packages/db/src/schema/agent_api_keys.ts`

## Agent Credentials

### API keys

- bearer API keys are looked up by SHA-256 hash in `agent_api_keys`
- revoked keys are excluded
- the resulting actor is bound to:
  - `agentId`
  - `companyId`
  - `keyId`
- the route layer then uses `assertCompanyAccess()` or route-specific checks to enforce company matching

### Local runtime JWTs

- if a bearer token does not match a stored API key, `actorMiddleware()` falls back to `verifyLocalAgentJwt()`
- JWT claims must include:
  - `sub`
  - `company_id`
  - `adapter_type`
  - `run_id`
  - time claims
- the middleware re-loads the agent record and rejects the token if:
  - the agent is missing
  - the DB company does not match `company_id`
  - the agent is `terminated` or `pending_approval`

Security reading:

- local JWTs are narrower than arbitrary bearer trust because the DB record is rechecked
- they are still accepted in all deployment modes when the bearer value is not an API key, so they remain part of the real auth surface

## Company Scope Enforcement

### Shared boundary helper

- `server/src/routes/authz.ts` is the core company-boundary helper:
  - unauthenticated actor -> `401`
  - agent actor -> companyId must match exactly
  - board actor -> company must be in `req.actor.companyIds` unless instance admin or `local_implicit`

### Membership and grant model

- `server/src/services/access.ts` requires an active `company_memberships` row before a permission grant counts
- board users use `canUser(companyId, userId, permissionKey)`
- agents use `hasPermission(companyId, "agent", agentId, permissionKey)`
- instance admins bypass per-company permission grants in `canUser()`

### Positive route patterns

The following patterns are internally consistent with the intended company model:

- `server/src/routes/access.ts`
  - invite creation, join approval, and permission management go through `assertCompanyPermission()`
- `server/src/routes/agents.ts`
  - config reads and hire flows use `assertCanCreateAgentsForCompany()`
  - agent updates use `assertCanUpdateAgent()`
- `server/src/routes/sidebar-badges.ts`
  - join-approval badge counts are gated by explicit permission checks

## Permission Model

Explicit permission keys currently in use:

- `agents:create`
- `users:invite`
- `users:manage_permissions`
- `tasks:assign`
- `tasks:assign_scope`
- `joins:approve`

Agent-side permission normalization is intentionally thin:

- `server/src/services/agent-permissions.ts` only derives `canCreateAgents` from role, defaulting true only for CEOs

Security reading:

- the coarse permission model is real and enforced in several important places
- because it is only partially adopted, route-level drift matters: any route that bypasses `assertCompanyAccess()` or the explicit permission helpers breaks the actual implementation contract

## Route Coverage

### Confirmed vulnerability: board-only mutation routes bypass company scope

The current implementation treats normal session-backed board users as company-scoped in `assertCompanyAccess()`, but several write routes skip that helper entirely.

Confirmed examples:

- `server/src/routes/approvals.ts`
  - `POST /api/approvals/:id/approve`
  - `POST /api/approvals/:id/reject`
  - `POST /api/approvals/:id/request-revision`
  - these load or mutate approvals by ID after `assertBoard(req)` only
- `server/src/routes/costs.ts`
  - `PATCH /api/companies/:companyId/budgets`
  - `PATCH /api/agents/:agentId/budgets`
  - these allow board mutations without first checking the caller's company access
- `server/src/routes/activity.ts`
  - `POST /api/companies/:companyId/activity`
  - this lets a board user create activity entries for an arbitrary company ID

Why this is a confirmed vulnerability:

- `server/src/routes/authz.ts` clearly defines company membership as the active boundary for non-admin board users
- adjacent routes in the same modules do use `assertCompanyAccess()`
- the affected endpoints mutate company-scoped state or workflow outcomes

Impact:

- an authenticated board user with access to company A can mutate approvals, budgets, or activity for company B if they know the target approval ID, agent ID, or company ID
- this is a tenant-isolation failure in `authenticated/private` and `authenticated/public`

Severity:

- High

Likely remediation:

- require `assertCompanyAccess(req, target.companyId)` before every board mutation on company-scoped resources
- consider a helper that resolves the entity first, then enforces company scope uniformly for ID-based routes

### Review lead: permission-model drift around direct agent management

Two routes appear weaker than the rest of the explicit permission model:

- `server/src/routes/agents.ts`
  - `POST /api/companies/:companyId/agents` uses only `assertCompanyAccess()` and does not reuse `assertCanCreateAgentsForCompany()`
- `server/src/routes/agents.ts`
  - `PATCH /api/agents/:id/permissions` restricts agent actors to CEO, but does not require `users:manage_permissions` or `agents:create` for board actors

Security reading:

- this is likely an authorization-model drift rather than a fully confirmed vulnerability
- the product docs still describe human permissions as coarse, while the codebase already enforces explicit board permissions in several related routes

Result state:

- Likely vulnerability / spec-ambiguous authz gap

## Findings Or Review Leads

| ID | Status | Severity | Summary |
|---|---|---|---|
| AUTH-F1 | Confirmed vulnerability | High | Board-only approval, budget, and activity mutations skip `assertCompanyAccess()` and let a scoped board user mutate another company |
| AUTH-F2 | Likely vulnerability | Medium | Direct agent creation and agent-permission updates are weaker than the explicit `agents:create` and membership-based permission model used elsewhere |

## Later Phase Questions

- Should the codebase continue toward company-scoped board permissions, or collapse back to the coarser “all board users can mutate all companies” model described in older docs?
- Are there additional ID-based board mutation routes outside Phase 2 scope with the same missing-company-check pattern?
- Should agent local JWTs gain stronger runtime binding, such as optional jti tracking or narrower audience separation, once execution-path review begins?
