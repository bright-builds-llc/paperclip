# Phase 3: Env, Session, And Log Review

## Scope

This artifact reviews what security-relevant context is injected into execution, what runtime and session artifacts persist after a run, and who can read those artifacts.

- Primary question: whether secrets, tokens, session ids, prompts, or other runtime context can leak into child processes or post-run artifacts
- Boundary lens: connect env propagation and persistence to the actual route read surface, not just to storage schemas
- Principal focus: board users, same-company agents, plugin-created task sessions, and any actor that can read heartbeat runs

## Environment Propagation

`server/src/services/secrets.ts` resolves secret-backed adapter config before execution, and `heartbeat.ts` passes the resolved values into adapter execution. The local CLI adapters then inject several Paperclip-specific env values, including combinations of:

- `PAPERCLIP_API_KEY`
- `PAPERCLIP_API_URL`
- `PAPERCLIP_RUNTIME_SERVICES_JSON`
- workspace metadata and run identifiers

For local adapters, `PAPERCLIP_API_KEY` is a locally minted JWT from `createLocalAgentJwt(...)` when `PAPERCLIP_AGENT_JWT_SECRET` is configured. The token is intentionally usable as agent auth against the API. It is therefore sensitive even though it is ephemeral.

Paperclip does some redaction:

- adapter metadata logging uses env redaction helpers before emitting on-meta details
- event payload redaction can strip obvious secret-like keys and JWT-looking values
- current-user path redaction removes usernames and home-directory fragments from text or values

But those controls are uneven. Child stdout, stderr, and many returned `resultJson` payloads are not guaranteed to pass through full secret redaction before persistence.

## Session Artifacts

`server/src/services/heartbeat.ts` persists and reuses multiple session-related fields:

- `sessionIdBefore`
- `sessionIdAfter`
- `contextSnapshot`
- task-session `sessionParamsJson`
- session handoff fields such as `paperclipSessionHandoffMarkdown`

`packages/db/src/schema/agent_task_sessions.ts` and `packages/db/src/schema/agent_runtime_state.ts` keep durable session and runtime state that can include workspace paths, task keys, repo context, and adapter session identifiers. Plugin session flows in `server/src/services/plugin-host-services.ts` create plugin-scoped task sessions and then wake heartbeat runs using stored session context plus `payload: { prompt: params.prompt }`.

Local agent JWTs matter here because:

- they are signed with the deployment-wide `PAPERCLIP_AGENT_JWT_SECRET`
- their default TTL is 48 hours
- `server/src/middleware/auth.ts` accepts the token as agent auth and falls back to the JWT claim `run_id` when the caller does not supply `x-paperclip-run-id`

If such a token appears in logs or a result artifact, a same-company actor can replay it as agent auth for that window.

## Run Logs And Execution Metadata

The storage layer in `server/src/services/run-log-store.ts` is well-defended against path traversal:

- `safeSegments()` normalizes path components
- `resolveWithin()` rejects escapes outside the configured log base path

That is a real `No finding`.

The more serious problem is the read surface. In `server/src/routes/agents.ts`:

- run detail reads only require `assertCompanyAccess(req, run.companyId)` and then return `redactCurrentUserValue(run)`
- run event reads require the same company check and additionally apply `redactEventPayload(event.payload)`
- run log reads require only `assertCompanyAccess(req, run.companyId)` and then return log content with current-user path redaction

`assertCompanyAccess()` in `server/src/routes/authz.ts` permits any same-company agent. It only blocks cross-company agent access. That means one agent can read another agent's heartbeat runs, logs, session ids, `contextSnapshot`, and `resultJson` within the same company.

Because local adapters inject `PAPERCLIP_API_KEY` and other runtime context into child processes, and because current-user redaction does not remove arbitrary secrets, the company-wide run-artifact surface is broad enough to enable lateral token or secret exposure.

## Findings Or Review Leads

- **Confirmed vulnerability / High:** Same-company agents can read other agents' heartbeat run details, event streams, and raw logs because the relevant routes use `assertCompanyAccess()` without agent-to-run ownership checks. Combined with limited redaction, this enables lateral exposure of `PAPERCLIP_API_KEY` JWTs, resolved secrets, prompts, `contextSnapshot`, and other run artifacts.
- **Likely vulnerability / Medium:** The run detail path returns broad fields such as `sessionIdBefore`, `sessionIdAfter`, `resultJson`, and `contextSnapshot` with only current-user path redaction. Even when a token is not exposed, this is more session and workspace metadata than a peer agent likely needs.
- **No finding:** `run-log-store.ts` protects file-backed logs against straightforward path traversal by constraining both segment values and final resolved paths.
- **Needs runtime validation:** The exact exploitability of token replay depends on whether specific adapters or prompts echo `PAPERCLIP_API_KEY` or other secrets into stdout, stderr, or `resultJson` often enough in realistic runs. The code path is real; the remaining question is how easy it is to weaponize in practice.

## Later Phase Questions

- Phase 4: can plugins subscribe to or proxy the same run-artifact routes, widening the lateral exposure beyond agent principals?
- Phase 5: should `resultJson`, `contextSnapshot`, and run-log reads receive the same secret-aware redaction treatment as event payloads, or stronger permission gating?
- Phase 6: should local JWT TTL, scope, or replay rules be tightened so leaked `PAPERCLIP_API_KEY` values have less post-run value?
