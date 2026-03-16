# Phase 4 Capability And Host-RPC Review

## Scope

This artifact reviews how plugin capabilities are declared, enforced, and translated into real worker-to-host power once a plugin worker is running.

Primary files reviewed:

- `server/src/services/plugin-capability-validator.ts`
- `packages/plugins/sdk/src/host-client-factory.ts`
- `packages/plugins/sdk/src/protocol.ts`
- `packages/plugins/sdk/src/types.ts`
- `server/src/services/plugin-host-services.ts`
- `server/src/services/plugin-secrets-handler.ts`
- `server/src/routes/plugins.ts`
- `doc/plugins/PLUGIN_SPEC.md`

## Capability Model

The capability model is real for worker-to-host RPC, but only for that boundary.

Install-time and runtime controls line up in several useful ways:

- `packages/shared/src/validators/plugin.ts` and `server/src/services/plugin-capability-validator.ts` require declared features such as tools, jobs, webhooks, UI slots, and launchers to have matching capabilities
- `packages/plugins/sdk/src/host-client-factory.ts` maps each worker-to-host method to a capability in `METHOD_CAPABILITY_MAP`
- capability failures become `CAPABILITY_DENIED` JSON-RPC errors rather than silent no-ops

The model is also intentionally broad in a few places:

- `config.get` is always allowed
- `entities.upsert` and `entities.list` are always allowed because they are plugin-scoped
- `log` is always allowed

The important limitation is structural:

- the capability model gates worker-to-host RPC calls only
- it does not constrain same-origin plugin UI code calling ordinary Paperclip HTTP routes directly

That documented limitation becomes important again in the UI section of Phase 4.

## Worker-To-Host RPC Surfaces

`packages/plugins/sdk/src/protocol.ts` and `host-client-factory.ts` expose a broad platform surface to plugin workers:

| Surface | Examples | Main guard |
|---|---|---|
| Config and plugin state | `config.get`, `state.get/set/delete` | capability gates except `config.get` |
| Plugin-owned entities | `entities.upsert`, `entities.list` | plugin scoping, no extra capability |
| Event bus | `events.emit`, `events.subscribe` | event capabilities |
| Network and secrets | `http.fetch`, `secrets.resolve` | `http.outbound`, `secrets.read-ref` |
| Activity and metrics | `activity.log`, `metrics.write`, `log` | capability gates except `log` |
| Company and project data | `companies.*`, `projects.*` | read capabilities, plus host-side company checks where applicable |
| Issue and comment APIs | `issues.*`, `issues.listComments`, `issues.createComment` | capability gates plus host-side company checks |
| Agent control | `agents.*`, `agents.sessions.*` | capability gates plus host-side company checks |
| Goal APIs | `goals.*` | capability gates plus host-side company checks |

Two positive controls stand out:

- `server/src/services/plugin-host-services.ts` implements outbound HTTP through `validateAndResolveFetchUrl()` and `executePinnedHttpRequest()`, including protocol allow-listing, DNS resolution, private-IP blocking, and pinned-IP execution
- `server/src/services/plugin-secrets-handler.ts` only resolves secret UUIDs that appear in the plugin's own config, and rate-limits repeated lookups

## Host Privilege Surfaces And Company Boundaries

The weakest boundary in the host-service layer is not capability gating. It is company scoping.

### Instance-wide plugin assumption

`server/src/services/plugin-host-services.ts` explicitly says:

- plugins are instance-wide in the current runtime
- `ensurePluginAvailableForCompany()` is therefore a no-op

That means the host does not have a plugin-availability boundary per company. Company protection must come entirely from:

- route-layer checks before a worker call starts, or
- per-handler record checks such as `requireInCompany(...)`

### What handlers actually trust

Many handlers take a caller-supplied `companyId` and then:

- call `ensurePluginAvailableForCompany(companyId)` which currently does nothing
- use `requireInCompany(...)` or `inCompany(...)` only to compare fetched records against that same supplied company ID

This works only if the supplied `companyId` is itself trustworthy. That assumption is true for some paths and false for others.

### Sensitive host operations

Several RPC handlers do meaningful work under plugin identity rather than human actor identity:

- `agents.invoke()` wakes an agent with `requestedByActorType: "system"` and `requestedByActorId: pluginId`
- `agentSessions.sendMessage()` wakes an agent run and subscribes to live events for the run
- issue, comment, and goal mutations use the supplied company-scoped identifiers, not the original board actor

Those are legitimate plugin powers only if the path into the worker preserves the correct company boundary first.

## Findings Or Review Leads

### Confirmed vulnerability: the UI bridge does not preserve board caller company scope

- **State:** Confirmed vulnerability
- **Severity:** High
- **Principal:** Company-scoped board user, or malicious plugin UI running under that board session
- **Boundary:** Company-scoped board access -> cross-company plugin worker authority
- **Primary file:** `server/src/routes/plugins.ts`
- **Secondary files:** `server/src/services/plugin-host-services.ts`, `packages/plugins/sdk/src/types.ts`, `server/src/routes/authz.ts`

Evidence chain:

- the bridge routes in `server/src/routes/plugins.ts` are board-only
- those routes call `assertCompanyAccess(req, body.companyId)` only when a top-level `companyId` field is present in the request body
- the worker-side `ctx.data.register()` and `ctx.actions.register()` handlers receive arbitrary `params: Record<string, unknown>` from the request body
- plugin UI code can therefore omit the top-level `companyId` while still sending `params.companyId = "<other-company>"`
- once the worker uses that value in host calls such as `ctx.issues.list({ companyId })`, `ctx.projects.get(projectId, companyId)`, or `ctx.agents.invoke(agentId, companyId, ...)`, the host layer accepts it because:
  - `ensurePluginAvailableForCompany()` is a no-op
  - per-handler checks only validate that fetched records belong to the supplied company ID, not that the original board caller was allowed to name that company

Impact:

- a non-instance-admin board user who can access company A can drive plugin worker reads or mutations against company B through the plugin bridge
- because plugin UI runs same-origin and can make arbitrary authenticated requests, the plugin author does not need the bridge helper hooks to exploit this path
- the crossed boundary is the same company-membership boundary Phase 2 already established for ordinary board routes

Likely remediation:

- propagate the caller's allowed company scope into bridge calls and enforce it inside `plugin-host-services.ts`
- reject bridge requests that omit company scope for company-sensitive handlers, or make company scope an explicit part of the host-to-worker bridge contract rather than an optional convention inside `params`

### No finding: outbound plugin HTTP is materially hardened against basic SSRF

`http.fetch` does more than plain `fetch()`:

- protocol allow-listing
- DNS resolution before connect
- private-IP blocking
- pinned-IP execution to reduce DNS rebinding risk

This does not make outbound access harmless, but the reviewed code does provide a real SSRF boundary.

### No finding: secret resolution is constrained to the plugin's declared config references

`plugin-secrets-handler.ts` only resolves UUID-shaped refs that appear in the plugin's own config, then fetches the latest version through the configured provider. This prevents a worker from enumerating arbitrary company secret IDs outside its configured references.

### Accepted-risk sharp edge: capabilities are instance-wide once granted

`companies.list`, `companies.get`, and other broad capabilities remain instance-wide by design. That is not a broken boundary by itself, but it means capability approval is effectively global for the whole instance rather than company-local.

## Later Phase Questions

- Should bridge requests carry an explicit actor and allowed-company envelope into worker handlers instead of relying on plugin-defined `params`?
- Should instance-wide capabilities like `companies.read` and `agents.invoke` be split into narrower company-scoped or operator-approved variants?
- Should plugin availability eventually become company-scoped so `ensurePluginAvailableForCompany()` can enforce a real boundary instead of a placeholder?
