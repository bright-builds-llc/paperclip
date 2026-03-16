# Phase 4 Research: Plugin & Extension Boundary Review

## Planning Question

Which plugin install, worker, capability, tool, job, webhook, and UI extension boundaries in Paperclip can let plugin-controlled code expand privilege, cross company or principal boundaries, or escape intended isolation, and how should the phase split so those questions are checked without flattening trusted-plugin assumptions into false positives?

## Phase Contract

Phase 4 is the plugin-boundary phase of the audit:

- `PLUG-01`: review whether plugin install and activation flows create untrusted code-execution or privilege-expansion risk
- `PLUG-02`: review whether plugin tools, jobs, webhooks, UI routes, and worker RPC paths preserve company and principal boundaries

Phase 3 already mapped the host-execution and artifact-exposure surfaces. Phase 4 should now determine how much of that power is reachable from plugin-controlled paths, and which plugin behaviors are intentionally trusted versus exploit-relevant.

## Repo-Grounded Facts That Shape The Plan

- The current plugin trust model is intentionally broad:
  - `doc/plugins/PLUGIN_AUTHORING_GUIDE.md` says to treat plugin workers and plugin UI as trusted code
  - `doc/plugins/PLUGIN_SPEC.md` says plugin UI runs as same-origin JavaScript inside the main app and is not sandboxed by manifest capabilities
- Install and activation are handled by `plugin-loader.ts` plus `plugin-lifecycle.ts`:
  - installs can come from npm packages or a local filesystem path
  - manifest validation checks route-path conflicts, entrypoints, and capability consistency
  - local installs persist `packagePath` so later worker resolution and dev flows can reuse the same filesystem path
  - upgrades treat new capabilities as an approval boundary and can transition a plugin to `upgrade_pending`
- Worker isolation is currently out-of-process, not sandboxed:
  - `plugin-worker-manager.ts` starts workers with `child_process.fork()`, IPC, stdio pipes, and optional `execArgv`
  - the repo has `plugin-runtime-sandbox.ts`, but the active plugin runtime path does not call it
  - local-path dev installs can load workers through a tsx loader in development
- Capability enforcement is split between install-time validation and runtime host RPC checks:
  - `plugin-capability-validator.ts` maps tools, jobs, webhooks, UI slots, and launchers to required capabilities
  - `packages/plugins/sdk/src/host-client-factory.ts` gates worker-to-host RPC methods by capability
  - `packages/plugins/sdk/src/protocol.ts` exposes a broad host surface including entities, issues, projects, agents, sessions, events, secrets, HTTP, and metrics
- Host services are powerful and only partly constrained by plugin scoping:
  - `plugin-host-services.ts` includes config, state, entities, events, outbound HTTP, secrets, activity, agent-session, and heartbeat wake-up operations
  - `ensurePluginAvailableForCompany()` is currently a no-op because plugins are instance-wide
  - many handlers still enforce company checks individually, but Phase 4 should verify whether the no-op availability check creates scope drift
  - outbound `http.fetch` includes SSRF defenses such as protocol allow-listing, DNS resolution, private-IP blocking, and pinned-IP execution
- Route-level plugin trust boundaries are mixed:
  - most `server/src/routes/plugins.ts` endpoints require `assertBoard(req)`
  - bridge data, action, and event routes only call `assertCompanyAccess()` when the request body actually carries a `companyId`
  - `POST /api/plugins/:pluginId/webhooks/:endpointKey` intentionally accepts unauthenticated inbound traffic and leaves signature verification to plugin code
  - plugin config writes strip `devUiUrl` unless the caller is an instance admin because it activates a dev proxy
- Plugin UI loading is same-origin and dynamic:
  - `server/src/routes/plugin-ui-static.ts` serves `/_plugins/:pluginId/ui/*` without an auth gate, limited to ready plugins with a UI entrypoint
  - in non-production, `devUiUrl` can proxy UI assets only to localhost targets
  - `ui/src/plugins/slots.tsx` fetches plugin source, rewrites bare imports to host-provided React and SDK shims, then imports the rewritten module through a blob URL
  - `ui/src/plugins/bridge.ts` and `ui/src/plugins/launchers.tsx` scope bridge calls using host context, but the UI code itself still runs as same-origin JavaScript
- Shared validators already encode some boundary rules:
  - `packages/shared/src/validators/plugin.ts` restricts `routePath`, blocks reserved host segments, requires `entrypoints.ui` when UI slots or launchers exist, and rejects duplicate tool, job, webhook, and slot identifiers
  - these validators reduce accidental conflicts, but they do not create a frontend sandbox or principal boundary by themselves

## Review Tracks The Plan Should Cover

### 1. Install, Activation, And Worker Lifecycle Boundaries

This track should answer:

- how npm installs, local-path installs, stored package paths, manifest validation, and route collision checks determine what code can run
- whether worker spawn, restart, upgrade, and dev-mode paths introduce extra privilege or untrusted execution assumptions
- whether the repo is relying on a sandbox that is not actually in the runtime path

Primary evidence:

- `doc/plugins/PLUGIN_SPEC.md`
- `doc/plugins/PLUGIN_AUTHORING_GUIDE.md`
- `server/src/services/plugin-loader.ts`
- `server/src/services/plugin-lifecycle.ts`
- `server/src/services/plugin-worker-manager.ts`
- `server/src/services/plugin-dev-watcher.ts`
- `server/src/services/plugin-runtime-sandbox.ts`
- `server/src/services/plugin-manifest-validator.ts`
- `server/src/services/plugin-config-validator.ts`

### 2. Capability Model, Host RPC, And Privilege Surfaces

This track should answer:

- whether declared capabilities match actual runtime enforcement
- which worker-to-host methods are ungated, broadly privileged, or only guarded by per-handler company checks
- whether secrets, outbound HTTP, agent sessions, or heartbeat wake-ups can cross intended company or principal boundaries

Primary evidence:

- `server/src/services/plugin-capability-validator.ts`
- `server/src/services/plugin-host-services.ts`
- `packages/plugins/sdk/src/host-client-factory.ts`
- `packages/plugins/sdk/src/protocol.ts`
- `server/src/services/plugin-secrets-handler.ts`
- `server/src/services/plugin-tool-dispatcher.ts`
- `server/src/routes/plugins.ts`

### 3. Tools, Jobs, Webhooks, And UI Isolation

This track should answer:

- whether tool execution and job scheduling preserve actor and company context correctly
- whether public webhook ingress has safe defaults or depends on plugin authors to build security correctly every time
- whether same-origin UI bundles, static serving, bridge routes, and launcher surfaces undermine the worker capability model

Primary evidence:

- `server/src/routes/plugins.ts`
- `server/src/routes/plugin-ui-static.ts`
- `server/src/services/plugin-tool-dispatcher.ts`
- `server/src/services/plugin-job-scheduler.ts`
- `server/src/services/plugin-job-coordinator.ts`
- `ui/src/api/plugins.ts`
- `ui/src/plugins/bridge.ts`
- `ui/src/plugins/slots.tsx`
- `ui/src/plugins/launchers.tsx`

## Concrete Boundary Questions Worth Testing

These are the repo-backed questions the execution plans should resolve.

1. Can npm or local-path install flows, stored `packagePath`, or dev tsx loading make plugin code execution broader than the operator-facing docs imply?
2. Is out-of-process `fork()` the only real runtime boundary, or does any active path actually use the VM sandbox helper?
3. Do install, enable, disable, restart, and upgrade flows preserve the intended approval boundary when capabilities change?
4. Does the capability model actually constrain the whole plugin system, or only worker-to-host RPC while same-origin UI can still call ordinary Paperclip HTTP APIs directly?
5. Can the broad host-service surface, including secrets, outbound HTTP, agent sessions, and heartbeat wake-ups, be reached across weaker company or plugin-availability checks than maintainers expect?
6. Do plugin tools and jobs execute with the correct company and actor scope, or can they inherit board-level or instance-wide power too easily?
7. Is public webhook ingress safe by default, or does delegating signature verification entirely to plugin code create exploitable defaults?
8. Do static UI serving, blob-based module loading, launchers, iframe rendering, or `devUiUrl` introduce trust-boundary drift beyond what the manifest capability model suggests?

## Existing Tests Worth Reusing As Evidence Anchors

- `server/src/__tests__/plugin-worker-manager.test.ts`
- `server/src/__tests__/plugin-dev-watcher.test.ts`
- `packages/plugins/examples/plugin-authoring-smoke-example/tests/plugin.spec.ts`

These tests do not prove the phase outcomes by themselves, but they mark where the repo already encodes expected worker, dev-watch, and example-plugin behavior.

## Recommended Execution Outputs

Phase 4 should produce three artifacts:

- `04-INSTALL-LIFECYCLE-REVIEW.md`
  - install source and validation model
  - worker process and lifecycle behavior
  - upgrade and dev-mode paths
  - findings or review leads
- `04-CAPABILITY-HOST-RPC-REVIEW.md`
  - declared capability model
  - worker-to-host RPC surfaces
  - privilege boundaries across secrets, HTTP, agent sessions, and company checks
  - findings or review leads
- `04-TOOLS-WEBHOOKS-UI-REVIEW.md`
  - tool execution and job scheduling surfaces
  - webhook ingress review
  - same-origin UI, bridge, and static-route isolation review
  - findings or review leads

## Recommended Plan Split

Wave 1 can run in parallel:

- `04-01`: review plugin install, activation, worker lifecycle, upgrade, and dev-mode boundaries
- `04-02`: review capability validation, host RPC surfaces, and plugin privilege boundaries

Wave 2 should depend on both:

- `04-03`: review tools, jobs, webhooks, and UI isolation using the lifecycle and privilege findings from Wave 1

This split keeps installation and host-privilege tracing independent first, then uses those results to judge whether tool, webhook, and UI paths are true vulnerabilities or expected consequences of the current trusted-plugin model.

## Verification Expectations For Phase Execution

When Phase 4 is executed, each artifact should be checked for:

- concrete file references to loader, lifecycle, worker, route, SDK, and UI runtime files
- explicit distinction between trusted-plugin assumptions and remotely relevant or operator-triggerable vulnerabilities
- direct treatment of same-origin UI as a separate trust boundary from worker capability gating
- direct treatment of public webhook ingress and localhost-only `devUiUrl` behavior
- coverage of both deployed npm installs and local-path development installs
- clear candidate findings, accepted-risk notes, or no-finding conclusions that can feed the shared findings log
