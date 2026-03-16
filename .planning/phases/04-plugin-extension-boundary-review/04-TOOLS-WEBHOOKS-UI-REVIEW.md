# Phase 4 Tools, Webhooks, And UI Review

## Scope

This artifact reviews the plugin surfaces that remain after installation and worker startup: tools, scheduled jobs, webhook ingress, UI bundles, and the board-facing bridge/runtime.

Primary files reviewed:

- `server/src/routes/plugins.ts`
- `server/src/routes/plugin-ui-static.ts`
- `server/src/services/plugin-tool-dispatcher.ts`
- `server/src/services/plugin-tool-registry.ts`
- `server/src/services/plugin-job-scheduler.ts`
- `server/src/services/plugin-job-coordinator.ts`
- `server/src/services/plugin-job-store.ts`
- `ui/src/api/plugins.ts`
- `ui/src/plugins/bridge.ts`
- `ui/src/plugins/slots.tsx`
- `ui/src/plugins/launchers.tsx`
- `packages/plugins/sdk/src/define-plugin.ts`
- `doc/plugins/PLUGIN_SPEC.md`
- `doc/plugins/PLUGIN_AUTHORING_GUIDE.md`

## Tool And Job Surfaces

### Tools

The tool execution surface is narrower than the general bridge surface:

- `GET /api/plugins/tools` and `POST /api/plugins/tools/execute` are board-only
- `POST /api/plugins/tools/execute` requires `runContext.agentId`, `runId`, `companyId`, and `projectId`
- the route enforces `assertCompanyAccess(req, runContext.companyId)`
- `plugin-tool-registry.ts` namespaces tools by plugin key and refuses execution if the worker is not running

Security reading:

- tool execution is still powerful because it runs plugin code during agent flows
- the reviewed HTTP path does preserve a concrete company binding through `runContext.companyId`

### Jobs

The job surface is more instance-wide:

- job declarations are synced from manifests into `plugin_jobs`
- the scheduler dispatches `runJob` by `pluginId` and `jobKey` with no company context
- manual trigger routes are board-only and scoped only to the plugin's job ID, not to any company

This matches the current instance-wide plugin model more than a company-local automation model. It is operationally broad, but the main broken boundary in this phase is still the management-route scope and the bridge scope, not the scheduler itself.

## Webhook And Ingress Surfaces

`POST /api/plugins/:pluginId/webhooks/:endpointKey` is intentionally public:

- no board auth is required
- the host only checks that:
  - webhook ingestion is enabled
  - the plugin exists
  - the plugin is `ready`
  - the plugin declares `webhooks.receive`
  - the `endpointKey` is declared in the manifest
- the host records the delivery, then forwards `rawBody`, `parsedBody`, headers, and a request ID to the worker's `handleWebhook` method

The security contract is pushed into plugin code:

- `packages/plugins/sdk/src/define-plugin.ts` says the plugin is responsible for signature verification
- the route provides no mandatory host-managed secret, signature helper, or per-company scoping

Because plugin workers can call broad host services after a webhook delivery, this route is not a harmless ingestion endpoint. It is an unauthenticated entry path into plugin-granted authority.

## UI Bundle Bridge And Route Isolation

### Static bundle serving

`server/src/routes/plugin-ui-static.ts` has real filesystem protections:

- only ready plugins with `entrypoints.ui` serve bundles
- requested paths are resolved under the plugin UI directory
- `realpathSync()` containment checks prevent symlink traversal
- `devUiUrl` proxying is disabled in production and restricted to localhost targets in development

Those are meaningful controls.

### Runtime isolation reality

The frontend runtime is still intentionally unsandboxed:

- `ui/src/plugins/slots.tsx` fetches plugin UI source from `/_plugins/:pluginId/ui/...`
- it rewrites bare imports for React, React DOM, and the plugin SDK UI hooks to host-provided shims
- it then imports the rewritten source via a blob URL and mounts exports into the host React tree
- `ui/src/plugins/bridge.ts` provides authenticated bridge hooks, but the loaded plugin code is still same-origin JavaScript with access to ordinary browser APIs such as `fetch`

`doc/plugins/PLUGIN_SPEC.md` and `doc/plugins/PLUGIN_AUTHORING_GUIDE.md` both match this runtime:

- plugin UI is same-origin JavaScript inside the main app
- plugin UI is trusted code
- manifest capabilities do not sandbox the UI layer

This means capability approval only describes worker-side host RPC, not the full power of installed frontend code.

## Findings Or Review Leads

### Likely vulnerability: webhook ingress is unsafe by default for privileged plugins

- **State:** Likely vulnerability
- **Severity:** Medium
- **Principal:** Unauthenticated external caller plus any installed webhook-capable plugin
- **Boundary:** Public ingress -> plugin-granted host authority
- **Primary file:** `server/src/routes/plugins.ts`
- **Secondary files:** `packages/plugins/sdk/src/define-plugin.ts`, `server/src/services/plugin-host-services.ts`

Evidence chain:

- the webhook route is public by design
- the host enforces only plugin existence, readiness, declared capability, and endpoint presence
- the SDK contract says signature verification is the plugin's responsibility
- once inside the worker, the plugin can call host services with the authority granted by its manifest capabilities and the instance-wide company model

Why this stays at "likely" instead of "confirmed":

- the host framework is clearly unsafe-by-default for webhook-capable plugins
- the concrete exploit still depends on a plugin whose webhook handler accepts attacker-controlled input without its own verification or policy checks

Likely remediation:

- add host-managed webhook auth primitives such as shared-secret validation helpers or mandatory verifier configuration
- make unauthenticated webhook execution an explicit opt-in with sharper warnings and tests

### Accepted-risk sharp edge: same-origin plugin UI bypasses the manifest capability story

- **State:** Accepted-risk sharp edge
- **Severity:** Informational

This is explicitly documented and directly visible in `slots.tsx`, `bridge.ts`, and the plugin docs:

- plugin UI runs in the host origin
- plugin UI shares host React/runtime state
- plugin UI can call ordinary HTTP routes outside worker capability gates

It is not a hidden boundary break, but it is a crucial trust assumption that operators need to keep in view when approving UI-capable plugins.

### No finding: the tool execution HTTP path preserves company scope explicitly

The reviewed `POST /api/plugins/tools/execute` path requires a company-bound `runContext` and enforces `assertCompanyAccess()` on that company before dispatching to the worker. That is materially stronger than the general-purpose UI bridge path.

### No finding: static UI serving and `devUiUrl` include real path and proxy protections

The static route does not rely only on string prefix checks:

- it verifies readiness and UI declaration
- it enforces realpath containment
- it keeps dev proxying localhost-only

Those protections do not create a sandbox, but they do close the obvious filesystem-traversal and generic dev-proxy SSRF classes.

## Later Phase Questions

- Should webhook-capable plugins be forced to declare a host-managed verification mechanism instead of relying entirely on custom worker code?
- Should UI-capable plugins surface a stronger operator warning that frontend code exceeds worker capability gates by design?
- Should public static bundle serving remain unauthenticated once the repo grows richer plugin dashboards and more sensitive extension logic?
