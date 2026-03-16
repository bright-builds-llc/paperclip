# Phase 2 Realtime And Auxiliary Authz Review

## Scope

This artifact compares realtime and auxiliary routes with the main HTTP auth model documented in the board and agent Phase 2 reviews.

Primary files reviewed:

- `server/src/app.ts`
- `server/src/index.ts`
- `server/src/realtime/live-events-ws.ts`
- `server/src/middleware/private-hostname-guard.ts`
- `server/src/middleware/auth.ts`
- `server/src/routes/authz.ts`
- `server/src/routes/activity.ts`
- `server/src/services/activity.ts`
- `server/src/routes/access.ts`
- `server/src/routes/plugins.ts`
- `server/src/routes/plugin-ui-static.ts`

## WebSocket Upgrade Path

### What matches the HTTP model

- board websocket access in authenticated mode resolves a real session and checks:
  - instance-admin role, or
  - active membership in the requested company
- agent websocket access accepts only stored API keys and requires `key.companyId === requested companyId`
- this is narrower than HTTP for agents because local JWTs are not accepted on the websocket path

### What differs from the HTTP model

- websocket authorization runs in `server/src/realtime/live-events-ws.ts` under raw `server.on("upgrade")`
- the code accepts bearer material from:
  - `Authorization: Bearer ...`
  - `?token=...` query parameter
- `local_trusted` board connections are accepted with no token, matching the local-trusted HTTP sharp edge

## Auxiliary Route Families

### Intentionally public or token-gated surfaces

The following auxiliary surfaces intentionally do not use normal board or agent auth:

- invite inspection and onboarding routes in `server/src/routes/access.ts`
- board-claim challenge inspection in `server/src/routes/access.ts`
- join-request API key claim in `server/src/routes/access.ts` guarded by a claim secret
- plugin webhook ingestion in `server/src/routes/plugins.ts`, where request authentication is expected to happen inside the plugin worker logic
- plugin UI static bundle serving in `server/src/routes/plugin-ui-static.ts`

These are meaningful attack surface, but they are not by themselves parity bugs with the company-auth model.

### Confirmed auxiliary mismatch

- `GET /api/heartbeat-runs/:runId/issues` in `server/src/routes/activity.ts` does not call `assertCompanyAccess()` or any other auth helper before returning issue metadata for the run
- the backing service resolves the run, uses its company ID internally, and returns issue summaries without checking the caller at all

## Parity Gaps

### Confirmed vulnerability: websocket upgrades bypass the private-hostname guard

In `authenticated/private`, HTTP requests are filtered by `privateHostnameGuard` in `server/src/app.ts` before auth and routing. The websocket path does not inherit that guard:

- `server/src/app.ts` mounts `privateHostnameGuard(...)` as Express middleware only
- `server/src/index.ts` separately calls `setupLiveEventsWebSocketServer(server, ...)`
- `server/src/realtime/live-events-ws.ts` performs auth and company checks, but never validates the request hostname against `allowedHostnames`

Impact:

- a client with a valid board session or agent API key can connect to live events through hostnames that the private HTTP surface would reject
- this weakens the boundary promised by `authenticated/private`

Severity:

- Medium

Likely remediation:

- apply the same hostname validation in the websocket upgrade path before `authorizeUpgrade()`
- keep one shared allow-set implementation so HTTP and websocket cannot drift

### Confirmed vulnerability: `GET /api/heartbeat-runs/:runId/issues` is unauthenticated

`server/src/routes/activity.ts` exposes:

- `GET /api/issues/:id/activity` with `assertCompanyAccess()`
- `GET /api/issues/:id/runs` with `assertCompanyAccess()`
- `GET /api/heartbeat-runs/:runId/issues` with no auth check

Impact:

- any caller who knows or can obtain a run ID can retrieve issue identifiers, titles, statuses, and priorities linked to that run
- the leak crosses company boundaries because the route does not consult `req.actor` at all

Severity:

- Medium

Likely remediation:

- resolve the run first, then apply `assertCompanyAccess(req, run.companyId)` before returning issue data

### Review lead: query-string websocket tokens are weaker operationally than header auth

The websocket path accepts `?token=` as a fallback bearer source. That is useful for some clients, but weaker operationally because URLs tend to leak into:

- browser history
- reverse-proxy logs
- access logs and debugging output

Result state:

- Needs runtime validation

The code review confirms the behavior but not whether real deployments log these URLs today.

## Findings Or Review Leads

| ID | Status | Severity | Summary |
|---|---|---|---|
| AUTH-F3 | Confirmed vulnerability | Medium | Live-events websocket upgrades bypass the private-hostname guard enforced on normal HTTP requests |
| AUTH-F4 | Confirmed vulnerability | Medium | `GET /api/heartbeat-runs/:runId/issues` returns company-linked issue metadata with no auth check |
| AUTH-F5 | Needs runtime validation | Low | Websocket query-string token fallback is operationally weaker than header-based bearer handling |

## Later Phase Questions

- Should websocket board subscriptions also validate `Origin` explicitly, not just session or membership, to match the intent of `boardMutationGuard` and private deployment boundaries?
- Do any plugin or invite helper routes perform network access or host interaction that should be pulled into Phase 3 immediately?
- Should public auxiliary routes like invite resolution testing and plugin webhooks get their own hardening pass once execution and data-exposure phases begin?
