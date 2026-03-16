# Phase 3: Process Execution Review

## Scope

This artifact traces how Paperclip turns persisted config, issue or project state, plugin prompts, and secret-backed settings into execution for heartbeat runs.

- Primary question: which inputs can steer local child processes or downstream adapter execution
- Deployment lens: treat board-authored execution as trusted-operator power unless the same path is reachable by weaker principals in `authenticated/private` or `authenticated/public`
- Principal focus: board users, same-company agents, plugin-authored session prompts, and local operators

## Execution Entry Points

`server/src/routes/agents.ts` starts the main heartbeat paths and resolves the target agent before handing execution to `server/src/services/heartbeat.ts`. Inside heartbeat execution, the server builds one merged runtime view before calling any adapter:

1. Persisted agent `adapterConfig` is loaded from the database.
2. Issue assignment overrides and project-level execution workspace policy can add or replace runtime fields before the adapter runs.
3. `server/src/services/secrets.ts` resolves secret-backed adapter config and env bindings in `resolveAdapterConfigForRuntime()`.
4. `realizeExecutionWorkspace()` applies project or issue workspace strategy and may create or reuse a worktree-backed execution directory.
5. Plugin-owned session wakeups in `server/src/services/plugin-host-services.ts` can inject `payload: { prompt: params.prompt }` and plugin-scoped task keys into the same heartbeat path.
6. `heartbeat.ts` persists `sessionIdBefore`, `contextSnapshot`, log metadata, and later `resultJson` and `sessionIdAfter` around the adapter invocation.

The important Phase 3 conclusion is that the execution sink is centralized in `adapter.execute(...)`, but the inputs are not. They come from route-selected agents, DB-backed adapter config, issue or project workspace policy, plugin wakeup prompts, and secret resolution.

## Adapter Execution Families

| Family | Primary files | Local host execution | Controlling inputs | Main controls |
|---|---|---|---|---|
| Local CLI adapters | `packages/adapters/claude-local/src/server/execute.ts`, `packages/adapters/codex-local/src/server/execute.ts`, `packages/adapters/cursor-local/src/server/execute.ts`, `packages/adapters/gemini-local/src/server/execute.ts`, `packages/adapters/opencode-local/src/server/execute.ts`, `packages/adapters/pi-local/src/server/execute.ts` | Yes | command, cwd, env, prompt, optional instructions file, runtime service metadata | `ensureAbsoluteDirectory()`, `ensureCommandResolvable()`, shared `runChildProcess()` with `shell: false` |
| Generic process adapter | `server/src/adapters/process/execute.ts` | Yes | raw `command`, `args`, `cwd`, `env`, timeouts | shared `runChildProcess()` with `shell: false`, but no absolute-directory or command-resolvability precheck |
| Generic HTTP adapter | `server/src/adapters/http/execute.ts` | No direct local spawn | arbitrary `url`, `method`, `headers`, `payloadTemplate`, full execution `context` | only fetch timeout handling; no allowlist or destination restriction |
| OpenClaw gateway adapter | `packages/adapters/openclaw-gateway/src/server/execute.ts` | No direct local spawn in Paperclip process | gateway URL, auth setup, ws payloads, remote result payloads | remote gateway protocol; risk shifts to remote service trust and returned artifacts |

Two distinctions matter:

- Paperclip has both local-host execution and remote-adapter execution families, so “adapter risk” is not one thing.
- The local CLI adapters have real guardrails that the generic process adapter does not fully mirror.

## Command Construction And Guardrails

### Local CLI adapters

The local adapters consistently:

- create or validate an absolute `cwd` with `ensureAbsoluteDirectory(..., { createIfMissing: true })`
- check that the configured executable is resolvable with `ensureCommandResolvable()`
- run the final command through shared `runChildProcess()` which calls `spawn(..., { shell: false })`
- pass the user or plugin prompt through stdin instead of interpolating it into a shell string
- optionally inject instruction-file contents into stdin or a temporary file path, which broadens what repo-controlled files can influence the run but does not itself become shell injection

They also inject runtime context into env, including `PAPERCLIP_API_KEY`, `PAPERCLIP_API_URL`, workspace metadata, and in some adapters `PAPERCLIP_RUNTIME_SERVICES_JSON`. That is an exposure concern, but it is not a command-construction bypass by itself.

### Generic process adapter

`server/src/adapters/process/execute.ts` is the least constrained local execution family:

- it accepts raw `command`, `args`, `cwd`, and `env`
- it logs redacted env metadata, then calls `runChildProcess()`
- it still benefits from `shell: false`
- it does not call `ensureAbsoluteDirectory()` or `ensureCommandResolvable()`

That means there is still no shell interpolation step, but Paperclip trusts the configured `cwd` and the operator-provided executable path more directly here than in the local CLI adapters.

### HTTP and gateway adapters

The HTTP adapter and OpenClaw gateway adapter do not execute local shell commands inside the Paperclip server. Their main risks are:

- outbound request control to arbitrary remote destinations
- sending broad execution context downstream
- trusting remote result payloads that are later persisted as run artifacts

These paths matter for data exposure and plugin reach, but they are not local host-execution bugs by default.

## Findings Or Review Leads

- **Accepted-risk sharp edge / Informational:** Board-authored adapter config is effectively operator-level execution authority. This is consistent with the current product model because board users can create agents, change adapter types, and steer workspace policy. It only becomes a higher-severity finding if a weaker principal can reach the same sink.
- **Accepted-risk sharp edge / Informational:** The generic `process` adapter is the least constrained local execution path because it trusts raw `command`, `cwd`, and env more directly than the curated local adapters. Static review does not show shell injection because the shared runner uses `shell: false`, but this remains a broad operator power surface.
- **Needs runtime validation:** The HTTP adapter can send full execution context to arbitrary configured URLs. Phase 3 treats this as an expected remote-execution boundary for a board-controlled adapter, but later phases should confirm whether plugins or weaker actors can steer those destinations indirectly.
- **Needs runtime validation:** Plugin session prompts in `server/src/services/plugin-host-services.ts` flow into heartbeat execution as prompt payloads. That is expected for plugin-controlled agents, but Phase 4 needs to confirm whether plugin capability gating keeps untrusted plugin code from reaching local-host adapters unexpectedly.
- **No finding:** The shared process runner in `packages/adapter-utils/src/server-utils.ts` uses `spawn()` with `shell: false`, so prompt text and most user-controlled strings do not become shell-interpreted command fragments in the reviewed local execution families.

## Later Phase Questions

- Phase 4: can plugin install or capability gaps let untrusted plugin code choose local-host adapter types, commands, or gateway destinations?
- Phase 5: how often do adapter `resultJson` payloads, stdout, or stderr contain tokens or prompts that later become readable artifacts?
- Phase 6: are there release or onboarding defaults that make the powerful `process` adapter or remote HTTP execution available more broadly than intended?
