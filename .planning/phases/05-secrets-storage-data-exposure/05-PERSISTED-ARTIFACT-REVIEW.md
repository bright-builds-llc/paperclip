# Phase 5: Persisted Artifact Exposure Review

## Scope

This artifact reviews the run, activity, revision, and plugin-log records that remain after execution, and whether those artifacts are both properly redacted and scoped to the right readers.

- Primary question: whether persisted operational artifacts re-expose secrets, prompts, session context, or tenant data after the original execution path has finished
- Boundary lens: distinguish storage-path correctness from confidentiality of artifact read routes
- Principal focus: same-company agents, board users reading company-scoped history, and plugin-controlled paths that can cause new artifacts to be persisted

## Heartbeat Run And Log Surfaces

Heartbeat artifacts remain the broadest persisted-data exposure surface in the repo.

`packages/db/src/schema/heartbeat_runs.ts` stores:

- `resultJson`
- `sessionIdBefore`
- `sessionIdAfter`
- `contextSnapshot`
- log metadata such as `logRef`, `logBytes`, and `logSha256`
- excerpts and error fields

`server/src/services/run-log-store.ts` is careful about filesystem safety.

- `safeSegments(...)` strips dangerous path characters
- `resolveWithin(...)` rejects escapes outside the configured run-log base path

That remains a real `No finding` on path traversal.

The confidentiality problem is the read surface, not the storage path.

`server/src/routes/agents.ts` exposes:

- `GET /api/heartbeat-runs/:runId` returning the full run row through `redactCurrentUserValue(run)`
- `GET /api/heartbeat-runs/:runId/events` returning redacted event payloads
- `GET /api/heartbeat-runs/:runId/log` returning log chunks with no secret-aware redaction of the log content itself

All three routes use `assertCompanyAccess(req, run.companyId)`.

The same broad model appears again in issue-scoped history.

- `server/src/routes/activity.ts` exposes `GET /api/issues/:id/runs`
- `server/src/services/activity.ts` populates that response from `heartbeat_runs`
- the returned rows include `resultJson`
- the route again only requires `assertCompanyAccess(req, issue.companyId)`

So the Phase 3 run-artifact exposure finding is not limited to the direct heartbeat-run URLs. Persisted run data is discoverable again through issue history routes as well.

## Activity, Revision, And Plugin Artifact Surfaces

Not every artifact class is equally weak.

Activity rows are comparatively well-sanitized.

- `server/src/services/activity-log.ts` applies `sanitizeRecord(...)` to `details`
- it then applies `redactCurrentUserValue(...)` before persisting or publishing activity data
- `server/src/routes/activity.ts` company and issue activity endpoints are company-scoped

Config revision handling is also materially better than raw run artifacts.

- `server/src/services/agents.ts` builds config snapshots with `sanitizeRecord(...)`
- `packages/db/src/schema/agent_config_revisions.ts` stores the redacted snapshots in `beforeConfig` and `afterConfig`
- `server/src/routes/agents.ts` only exposes those revisions through `assertCanReadConfigurations(...)`, which is narrower than ordinary company membership
- rollback refuses revisions that still contain redacted placeholders, preventing secret placeholders from silently becoming live config

Plugin log handling sits in between.

- `server/src/services/plugin-host-services.ts` persists worker log and metric data into `plugin_logs`
- `sanitiseMeta(...)` strips reserved field names and applies a size limit, but it is not a secret-aware redactor
- `server/src/routes/plugins.ts` exposes plugin logs behind `assertBoard(req)`

Inside the trusted-plugin model, that is not a standalone vulnerability. But it does mean plugin-controlled code can persist sensitive values into `plugin_logs` if it chooses to log them, and those logs then become visible on the board operator surface.

There is also a still-open carry-forward route from Phase 2.

- `server/src/routes/activity.ts` keeps `GET /api/heartbeat-runs/:runId/issues`
- that route returns issue metadata for a run without any auth check

That is already a confirmed auth finding from Phase 2, but it remains part of the persisted-artifact map because it turns run identifiers into issue metadata without access control.

## Redaction Coverage And Gaps

Paperclip's artifact redaction is uneven rather than absent.

Stronger controls:

- `sanitizeRecord(...)` redacts secret-looking keys and JWT-shaped values in structured records
- config revisions and activity details use that structured redaction path
- event payload reads additionally pass through `redactEventPayload(...)`

Weaker controls:

- full heartbeat run reads only get `redactCurrentUserValue(...)`, which strips usernames and home-directory fragments but does not remove arbitrary secrets from `resultJson`, session ids, or context fields
- run-log reads return stored log content without secret-aware redaction
- issue run history returns `resultJson` again without the stronger structured-event redaction path
- plugin log metadata only receives size and reserved-key sanitation, not secret-aware filtering

The result is a layered artifact model:

- path safety and basic structured redaction exist
- but the most sensitive execution artifacts still remain readable too broadly once they have been persisted

## Findings Or Review Leads

- **Confirmed vulnerability / High:** Persisted heartbeat artifacts remain exposed to any same-company actor across multiple read surfaces. The direct heartbeat-run routes and the issue-history route both return secret-bearing artifact fields such as `resultJson`, `contextSnapshot`, session identifiers, or raw log content under ordinary `assertCompanyAccess(...)` checks, while only applying limited secret-aware redaction. This extends and reinforces the main Phase 3 same-company run-artifact exposure finding.
- **No finding:** `run-log-store.ts` continues to defend against straightforward path traversal or arbitrary file reads by constraining both log path segments and final resolved paths.
- **No finding:** Agent config revisions are materially better protected than run artifacts. They store sanitized snapshots, require elevated configuration-read access, and reject rollback of redacted revisions.
- **Accepted-risk sharp edge / Medium:** Plugin logs persist plugin-supplied metadata without secret-aware redaction. Because plugin code is already inside the trusted-plugin model and the log route is board-only, this is best treated as a hardening target and a Phase 4 carry-forward concern rather than a separate confirmed vulnerability.
- **Carry-forward finding:** `GET /api/heartbeat-runs/:runId/issues` remains unauthenticated and still exposes issue metadata derived from persisted run relationships. This was already confirmed in Phase 2 and remains unresolved.

## Later Phase Questions

- Phase 7 should decide whether the best remediation is route hardening, stronger secret-aware redaction on `resultJson` and run logs, or both.
- Phase 7 should connect the stored-artifact issue with the Phase 4 plugin findings, because plugin-controlled prompts, events, and logs can all widen what later lands in persisted artifacts.
- Phase 6 may also want to examine whether retention and operational export flows should treat run logs, plugin logs, and activity rows differently once secrets or HTML uploads have already crossed earlier boundaries.
