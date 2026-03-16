# Phase 4 Plugin Install And Lifecycle Review

## Scope

This artifact traces how Paperclip installs, activates, upgrades, and manages plugins before any plugin worker code handles data, jobs, tools, or webhooks.

Primary files reviewed:

- `server/src/middleware/auth.ts`
- `server/src/routes/authz.ts`
- `server/src/routes/plugins.ts`
- `server/src/services/plugin-loader.ts`
- `server/src/services/plugin-lifecycle.ts`
- `server/src/services/plugin-worker-manager.ts`
- `server/src/services/plugin-dev-watcher.ts`
- `server/src/services/plugin-runtime-sandbox.ts`
- `server/src/services/plugin-manifest-validator.ts`
- `server/src/services/plugin-config-validator.ts`
- `doc/plugins/PLUGIN_SPEC.md`
- `doc/plugins/PLUGIN_AUTHORING_GUIDE.md`

## Install Source And Validation Model

### What the loader validates well

`server/src/services/plugin-loader.ts` applies meaningful install-time checks before a plugin is activated:

- npm installs use `execFile("npm", ["install", spec, "--prefix", targetInstallDir, "--save", "--ignore-scripts"])`
  - this avoids shell interpolation from package specs
  - it also blocks `preinstall` or `install` lifecycle hooks from running before manifest validation
- local-path installs are resolved to an absolute path and must exist on disk
- manifests must parse successfully, match a supported API version, satisfy capability consistency checks, and not request a newer host version than the running server
- page-slot `routePath` values are checked for duplicates and for conflicts with already-installed plugins
- local-path installs persist `packagePath` in the registry so later worker resolution can reuse the original filesystem path

### Where the trust boundary is still broad

The install surface is still intentionally powerful once the route is reachable:

- `POST /api/plugins/install` in `server/src/routes/plugins.ts` accepts npm packages or local filesystem paths
- `installPlugin()` persists the manifest immediately, then `lifecycle.load()` transitions the plugin to `ready` and starts the worker
- plugins are instance-wide rather than company-scoped, so install and lifecycle actions affect every company once the plugin is loaded

In other words, the loader protects against obvious package-shape, route-path, and install-hook problems, but it still treats plugin code itself as trusted once a caller is allowed to install it.

## Worker Process And Lifecycle Model

### Real runtime boundary

The active worker isolation model is an out-of-process Node child, not a VM sandbox:

- `server/src/services/plugin-worker-manager.ts` uses `child_process.fork()` with `stdio: ["pipe", "pipe", "pipe", "ipc"]`
- the worker receives a minimal environment rather than the full parent `process.env`
  - `PATH`
  - `NODE_PATH`
  - `PAPERCLIP_PLUGIN_ID`
  - `NODE_ENV`
  - `TZ`
  - optional `options.env`
- initialization happens over JSON-RPC after the process starts; if initialization fails, the host kills the worker and marks it crashed
- unexpected exits trigger exponential-backoff restarts up to the configured crash limit

This is a better boundary than running plugin code in-process, but it is still normal Node execution with host-selected entrypoints, IPC, stdout, and stderr.

### What is not actually in the runtime path

`server/src/services/plugin-runtime-sandbox.ts` exists and documents a VM-style sandbox helper, but the reviewed activation path never calls it. The real path is:

1. `routes/plugins.ts` or startup calls lifecycle or loader operations
2. `plugin-lifecycle.ts` transitions plugin state
3. `plugin-loader.ts` resolves the worker entrypoint and builds host handlers
4. `plugin-worker-manager.ts` forks the worker process

The repo therefore should not get security credit for sandbox behavior that is not active in the runtime path.

## Upgrade And Dev-Mode Paths

### Upgrade behavior

The intended approval boundary for new capabilities is visible, but the current implementation is stricter and slightly inconsistent:

- `plugin-loader.ts` detects added capabilities during `upgradePlugin()` and throws immediately
- `plugin-lifecycle.ts` still contains code that expects to compare old and new capabilities and move the plugin to `upgrade_pending`
- because the loader throws first, the lifecycle branch that transitions to `upgrade_pending` does not currently run for a capability-escalating upgrade

Security reading:

- this does not create a capability-bypass
- it does create implementation drift against the documented upgrade-pending workflow

### Development-only broadening

Local development broadens the trust boundary materially:

- repo-local or local-path plugins can run through the tsx loader via `workerOptions.execArgv = ["--import", DEV_TSX_LOADER_PATH]`
- `plugin-dev-watcher.ts` watches local-path plugin directories and restarts workers on file changes
- worker entrypoint resolution prefers persisted `packagePath` for local installs

These behaviors are useful for local authoring, but they confirm that dev-mode plugin execution trusts writable local paths and live source trees rather than a sealed published package artifact.

## Findings Or Review Leads

### Confirmed vulnerability: any board user can manage instance-wide plugins

- **State:** Confirmed vulnerability
- **Severity:** High
- **Principal:** Authenticated board user without instance-admin rights
- **Boundary:** Company-scoped board access -> instance-wide plugin code execution and control
- **Primary file:** `server/src/routes/plugins.ts`
- **Secondary files:** `server/src/middleware/auth.ts`, `server/src/routes/authz.ts`, `server/src/services/plugin-loader.ts`

Evidence chain:

- session-backed board actors in `server/src/middleware/auth.ts` carry `companyIds` plus `isInstanceAdmin`, proving non-admin board users are a real principal
- `assertBoard(req)` in `server/src/routes/authz.ts` checks only `req.actor.type === "board"`
- nearly every lifecycle and management route in `server/src/routes/plugins.ts` uses `assertBoard(req)` and nothing narrower:
  - install
  - delete
  - enable
  - disable
  - upgrade
  - config read and write
  - health, logs, jobs, manual job trigger, dashboard, UI contribution listing, and tool execution
- `plugin-loader.ts` can install from npm or a local filesystem path and `lifecycle.load()` immediately starts instance-wide worker code

Impact:

- a board user with access to company A can install or reconfigure plugin code that runs for companies B, C, and the rest of the instance
- the same user can disable or replace existing plugins, trigger global jobs, and alter instance-wide plugin configuration
- because plugin workers are real child processes, this is not just UI drift; it is a path to new host-executed code under a weaker-than-instance-admin principal

Likely remediation:

- require instance-admin authorization, or a dedicated explicit plugin-admin permission, for every instance-wide plugin lifecycle and config route
- split read-only plugin diagnostics from mutating lifecycle routes if non-admin board users need any visibility

### No finding: npm install path avoids shell injection and pre-validation install hooks

- `plugin-loader.ts` uses `execFile`, not `exec`
- `--ignore-scripts` prevents npm lifecycle hooks from executing before the host validates the manifest

This reduces supply-chain and command-construction risk at the install step itself, even though installing the plugin still authorizes later worker execution.

### No finding: worker startup does not leak the full parent environment by default

`plugin-worker-manager.ts` deliberately builds a minimal child environment instead of spreading `process.env`. That avoids leaking host secrets such as database URLs or unrelated API keys into plugin workers automatically.

### Accepted-risk sharp edge: local-path and tsx-backed development installs materially broaden trust

Local-path installs, live file watching, and tsx-backed entrypoints are only safe when the operator already trusts the local checkout. They are an important sharp edge, but they remain inside the documented local-authoring trust model rather than a new remotely reachable boundary break.

### Review lead: capability-escalation upgrades currently drift from the documented `upgrade_pending` path

`plugin-loader.ts` rejects new capabilities before `plugin-lifecycle.ts` can move the plugin into `upgrade_pending`. This is safer than silently accepting the upgrade, but it leaves the runtime behavior out of sync with the documented operator-approval workflow.

## Later Phase Questions

- Should instance-wide plugin management become instance-admin-only, or should the product grow an explicit plugin-admin permission model?
- Should deployed installs forbid local filesystem plugin paths entirely outside development mode?
- Should the runtime either remove the dormant VM sandbox helper or wire it into a real hardened execution path so maintainers are not misled by dead security code?
