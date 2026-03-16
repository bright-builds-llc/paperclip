# Phase 2 Board Auth Review

## Scope

This artifact reviews how board authority is established and constrained in `local_trusted`, `authenticated/private`, and `authenticated/public`, with focus on:

- actor resolution and session establishment
- origin and hostname controls for board mutations
- bootstrap and recovery flows that can expand board authority
- whether any confirmed weakness exists in the board-auth path itself, separate from downstream company-scope bugs

Primary files reviewed:

- `server/src/middleware/auth.ts`
- `server/src/auth/better-auth.ts`
- `server/src/index.ts`
- `server/src/app.ts`
- `server/src/middleware/board-mutation-guard.ts`
- `server/src/middleware/private-hostname-guard.ts`
- `server/src/board-claim.ts`
- `server/src/routes/access.ts`
- `server/src/routes/authz.ts`

## Deployment Mode Matrix

| Mode | How board authority is established | Controls that matter | Security reading |
|---|---|---|---|
| `local_trusted` | `actorMiddleware()` synthesizes `{ type: "board", userId: "local-board", isInstanceAdmin: true, source: "local_implicit" }` for every request | loopback host binding and local deployment assumption | intentional sharp edge; acceptable only while the documented localhost-only trust assumption holds |
| `authenticated/private` | Better Auth session resolves a DB user, then `actorMiddleware()` loads active `company_memberships` plus `instance_user_roles` | `privateHostnameGuard`, `boardMutationGuard`, trusted origins, configured auth secret | normal private-network board mode; remote risk is dominated by route-level authz consistency rather than session establishment |
| `authenticated/public` | Same session flow as private mode | explicit public URL, non-empty auth secret, trusted origins, browser cookie security | internet-facing posture is stricter; startup rejects missing auth secrets and missing explicit public URL, but cookie security still depends on HTTPS deployment |

## Session And Origin Controls

### Actor resolution

- `server/src/middleware/auth.ts` establishes the board actor in two ways:
  - implicit `local_trusted` board identity
  - Better Auth session resolution in `authenticated` mode, with active company memberships loaded into `req.actor.companyIds`
- `server/src/routes/authz.ts` treats instance admins and `local_implicit` board actors as company-scope bypasses, while normal session-backed board users are limited to `companyIds`

### Auth secret and startup posture

- `server/src/auth/better-auth.ts` contains a fallback chain ending in `"paperclip-dev-secret"`, but `server/src/index.ts` refuses to start authenticated mode unless `BETTER_AUTH_SECRET` or `PAPERCLIP_AGENT_JWT_SECRET` is set
- Result: the hardcoded fallback is code smell, but not a confirmed production vulnerability through the normal startup path reviewed here

### Origin and hostname controls

- `server/src/middleware/board-mutation-guard.ts` blocks non-safe board mutations unless `Origin` or `Referer` matches the instance host or known dev origins
- `server/src/middleware/private-hostname-guard.ts` blocks unexpected hostnames in `authenticated/private`
- `server/src/app.ts` mounts both controls before the API router, so normal HTTP board mutations inherit them

### Better Auth defaults

- `requireEmailVerification` is disabled
- signup disablement is config-driven (`authDisableSignUp`)
- secure cookies are disabled when the configured public URL is `http://`
- trusted origins are derived from the explicit public URL plus allowed hostnames

These settings are acceptable for local or private workflows, but `authenticated/public` relies on operators using HTTPS and sane signup settings rather than the code enforcing them unconditionally.

## Board Bootstrap And Invite Flows

### Board claim

- `server/src/board-claim.ts` only creates a claim challenge when authenticated mode is active and `local-board` is the sole instance admin
- challenge material is high entropy:
  - token: 24 random bytes hex
  - code: 12 random bytes hex
- `POST /api/board-claim/:token/claim` in `server/src/routes/access.ts` requires a signed-in board session before promoting the caller to instance admin and owner across all companies

Security reading:

- this is intentionally powerful bootstrap behavior, but it is tightly gated and not a confirmed auth bypass from code review alone
- the flow exists specifically to migrate from long-running local-trusted installs into authenticated mode without lockout

### Invite and company-access administration

- invite creation, join approval, and user company-access management are not public board powers in the current implementation
- `server/src/routes/access.ts` routes these through `assertCompanyPermission()` or instance-admin checks
- this is narrower than the coarse “board can do everything” language in the long-horizon docs, and it means downstream routes that skip `assertCompanyAccess()` are meaningful tenant-isolation bugs in the current codebase, not merely theoretical RBAC drift

## Findings Or Review Leads

### No confirmed board session-establishment bypass found

The core board session path is internally consistent:

- authenticated mode requires an auth secret at startup
- session-backed board actors load active memberships and instance-admin state
- board mutations inherit origin checks through `boardMutationGuard`
- authenticated/private HTTP traffic inherits hostname filtering through `privateHostnameGuard`

### Review lead: HTTPS posture is deployment-enforced only partially

- `server/src/auth/better-auth.ts` disables secure cookies when the public URL is `http://`
- `cli/src/checks/deployment-auth-check.ts` warns for `authenticated/public` with non-HTTPS URLs, but does not hard-fail

Security reading:

- this is not a confirmed bypass by itself
- it remains a sharp edge for public deployments if operators ignore the warning and run over plain HTTP

### Review lead: downstream route consistency matters more than the board auth path

The board-auth path itself is not where the strongest confirmed weaknesses landed. The confirmed Phase 2 vulnerabilities are downstream company-scope gaps:

- board-only approval resolution routes that skip company checks
- budget and activity mutation routes that skip company checks
- realtime and auxiliary routes that do not inherit the same guards as normal HTTP routes

Those are documented in the Phase 2 agent and parity artifacts.

## Later Phase Questions

- Should authenticated/public reject `http://` public URLs outright instead of warning?
- Should board auth defaults require explicit sign-up disablement or email verification for public deployments?
- Are there any non-HTTP board entry points outside Express middleware that bypass `boardMutationGuard` or `privateHostnameGuard`?
