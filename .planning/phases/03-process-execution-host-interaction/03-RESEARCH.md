# Phase 3 Research: Process Execution & Host Interaction

## Planning Question

Which Paperclip inputs can steer process execution, workspace or worktree mutation, runtime-service spawning, or execution artifacts, and how should the review split so the phase distinguishes real remotely relevant exploit paths from intentional local-operator sharp edges?

## Phase Contract

Phase 3 is the first host-execution phase of the audit:

- `EXEC-01`: review every path where request, database, plugin, or repo-controlled input reaches process execution or command construction
- `EXEC-02`: review workspace, worktree, filesystem, and runtime-service behavior for boundary escape or unexpected host mutation
- `EXEC-03`: review environment propagation, session handling, and logs for secret leakage or exposure of security-relevant runtime context

Phase 2 already established the company-boundary and authorization baseline. Phase 3 should now trace what authenticated actors, plugins, and local operators can make the server execute or persist on the host.

## Repo-Grounded Facts That Shape The Plan

- `heartbeat.ts` is the execution orchestrator:
  - it starts from persisted `agent.adapterConfig`
  - merges issue assignee adapter overrides and project or issue execution-workspace policy
  - resolves secret-backed env bindings through `secretService.resolveAdapterConfigForRuntime()`
  - realizes an execution workspace, optionally starts runtime services, injects a local agent JWT for supporting adapters, then calls `adapter.execute()`
- The execution surface is broader than one local CLI adapter:
  - local adapters (`claude_local`, `codex_local`, `cursor`, `gemini_local`, `opencode_local`, `pi_local`, `hermes_local`) eventually reach `runChildProcess()`
  - the generic `process` adapter accepts raw `command`, `args`, `cwd`, and `env`
  - the `http` and `openclaw_gateway` adapters are non-child-process execution paths but still carry config, context, and auth material into downstream execution systems
- Local adapter wrappers share the same core mechanics:
  - `ensureAbsoluteDirectory()` can create a missing absolute working directory
  - `ensureCommandResolvable()` validates the executable path
  - `runChildProcess()` uses `spawn(..., shell: false)` and can pipe prompts through stdin
  - adapter metadata logs redact env keys by sensitive-name pattern, but child stdout or stderr is preserved unless separately redacted
- Workspace realization supports host-mutating strategies:
  - `execution-workspace-policy.ts` allows `workspaceStrategy` and `workspaceRuntime` from project or issue data
  - `workspace-runtime.ts` can create git worktrees, choose absolute or repo-relative worktree parent directories, and run `provisionCommand` through `shell -c`
  - `teardownCommand` is parsed into policy objects but Phase 3 should confirm where, if anywhere, it is actually enforced
- Runtime services are another shell boundary:
  - `workspaceRuntime.services[]` can define `command`, `cwd`, `env`, `port`, `readiness`, `reuseScope`, and `stopPolicy`
  - local runtime services are started with `spawn(shell, ["-lc", command])`
  - rendered templates can inject workspace, issue, agent, env, and port values into both `cwd` and env
- Worktree CLI flows propagate host-sensitive state:
  - `worktree init` and `worktree:make` write repo-local `.paperclip/config.json` and `.paperclip/.env`
  - they can copy local-encrypted secrets master keys into a new instance
  - they persist the current `PAPERCLIP_AGENT_JWT_SECRET`
  - they copy shared git hooks into linked worktrees and optionally seed the new database from the source instance
- Execution artifacts are durable and exposed through ordinary routes:
  - `heartbeat_runs` stores `resultJson`, `sessionIdBefore`, `sessionIdAfter`, `logRef`, excerpts, and `contextSnapshot`
  - `agent_runtime_state` stores adapter state and session ids
  - `agent_task_sessions` stores arbitrary `sessionParamsJson`
  - `/api/heartbeat-runs/:runId`, `/events`, and `/log` are company-scoped but expose these artifacts to any actor who passes `assertCompanyAccess()`
- Log redaction is intentionally narrow:
  - run logs and route responses redact the current username and home-directory paths
  - adapter invocation metadata additionally redacts env keys that match sensitive patterns, plus resolved secret keys from `secretService`
  - Phase 3 should treat anything beyond that as potentially visible if a child process prints it
- Plugin-controlled execution input already reaches heartbeat:
  - `plugin-host-services.ts` can create plugin-owned task sessions and wake agents with prompt payloads
  - the VM helper in `plugin-runtime-sandbox.ts` exists but has no current call site, so Phase 3 should avoid over-crediting sandbox protection that is not active in the execution path

## Review Tracks The Plan Should Cover

### 1. Process Execution And Command Construction

This track should answer:

- which persisted config, route input, plugin input, or runtime context fields can influence `command`, `args`, `cwd`, stdin prompt, or downstream execution URL
- how local adapters differ from generic process and remote HTTP or gateway adapters
- where guardrails exist and where the code assumes trusted operator control

Primary evidence:

- `server/src/services/heartbeat.ts`
- `server/src/adapters/registry.ts`
- `server/src/adapters/process/execute.ts`
- `server/src/adapters/http/execute.ts`
- `packages/adapter-utils/src/server-utils.ts`
- `packages/adapters/*/src/server/execute.ts`
- `server/src/routes/agents.ts`
- `server/src/services/plugin-host-services.ts`

### 2. Workspace, Worktree, Filesystem, And Runtime-Service Boundaries

This track should answer:

- how execution workspaces are chosen and when worktree creation or workspace commands can reach outside the intended repo boundary
- whether runtime services or worktree provisioning can mutate host state in ways the product does not clearly contain
- how local operator tooling copies secrets, hooks, config, and workspace paths into new worktrees

Primary evidence:

- `server/src/services/execution-workspace-policy.ts`
- `server/src/services/workspace-runtime.ts`
- `cli/src/commands/worktree.ts`
- `cli/src/commands/worktree-lib.ts`
- `packages/shared/src/types/workspace-runtime.ts`

### 3. Env, Session, And Artifact Exposure

This track should answer:

- what env variables and auth material are injected into child processes or remote adapter requests
- what session identifiers or session state are persisted and when they are resumed, rotated, or exposed
- which logs, excerpts, result payloads, and context snapshots can reveal secrets or runtime-sensitive host details

Primary evidence:

- `server/src/services/heartbeat.ts`
- `server/src/services/secrets.ts`
- `server/src/services/run-log-store.ts`
- `server/src/log-redaction.ts`
- `server/src/routes/agents.ts`
- `packages/db/src/schema/heartbeat_runs.ts`
- `packages/db/src/schema/agent_runtime_state.ts`
- `packages/db/src/schema/agent_task_sessions.ts`

### 4. Local-Only Sharp Edges Versus Real Remote Risk

This track should answer:

- which execution paths are only reachable by a trusted local operator or plugin author and should be logged as sharp edges rather than vulnerabilities
- which paths remain reachable to authenticated board or agent actors in `authenticated/private` or `authenticated/public`
- how to classify broad host-control features like process adapters, worktree seeding, and runtime services without flattening all of them into the same severity

Primary evidence:

- `doc/DEPLOYMENT-MODES.md`
- `.planning/phases/01-threat-model-attack-surface-baseline/01-THREAT-MODEL.md`
- `.planning/phases/01-threat-model-attack-surface-baseline/01-AUDIT-METHODOLOGY.md`
- the three Phase 3 review artifacts produced by execution

## Concrete Boundary Questions Worth Testing

These are the repo-backed questions the execution plans should resolve.

1. Can route-controlled agent config, issue overrides, or plugin prompts reach `spawn`, shell commands, or remote execution adapters without a meaningful trust boundary in between?
2. Do local adapters rely on `shell: false` and absolute working-directory checks consistently, and where do they intentionally allow directory creation or prompt injection?
3. Can `workspaceStrategy.worktreeParentDir`, `branchTemplate`, or `provisionCommand` place worktrees or shell execution outside the intended repo boundary?
4. Do runtime services allow unexpected host mutation through shell `-lc`, templated env, absolute `cwd`, shared reuse, or weak stop policies?
5. Does worktree seeding copy secrets, JWT material, hooks, or project workspace paths into places that broaden the trust boundary beyond what maintainers expect?
6. Are session ids, `sessionParamsJson`, `contextSnapshot`, `resultJson`, run logs, or excerpts durable in ways that expose secrets or host-sensitive context to broader company actors?
7. Which of the above are real authenticated or public vulnerabilities, and which are expected consequences of documented local-operator or plugin-author trust?

## Existing Tests Worth Reusing As Evidence Anchors

- `server/src/__tests__/execution-workspace-policy.test.ts`
- `server/src/__tests__/workspace-runtime.test.ts`
- `server/src/__tests__/heartbeat-workspace-session.test.ts`
- `server/src/__tests__/codex-local-adapter-environment.test.ts`
- `server/src/__tests__/gemini-local-adapter-environment.test.ts`
- `server/src/__tests__/claude-local-adapter-environment.test.ts`
- `server/src/__tests__/cursor-local-adapter-environment.test.ts`
- `server/src/__tests__/opencode-local-adapter-environment.test.ts`
- `server/src/__tests__/log-redaction.test.ts`
- `cli/src/__tests__/worktree.test.ts`

These tests do not prove the phase outcomes by themselves, but they mark where the repo already encodes intended execution, workspace, and redaction behavior.

## Recommended Execution Outputs

Phase 3 should produce four artifacts:

- `03-PROCESS-EXECUTION-REVIEW.md`
  - input-to-execution map
  - adapter family comparison
  - command construction and execution guardrails
  - findings or review leads
- `03-WORKSPACE-HOST-BOUNDARY-REVIEW.md`
  - execution workspace realization
  - runtime service boundaries
  - worktree seeding and filesystem mutation review
  - findings or review leads
- `03-ENV-SESSION-LOG-REVIEW.md`
  - env propagation and secret handling
  - session artifact review
  - log and run-artifact exposure
  - findings or review leads
- `03-EXECUTION-RISK-MODE-MATRIX.md`
  - remote-relevant execution risks
  - accepted local-only sharp edges
  - classification and carry-forward list for later phases

## Recommended Plan Split

Wave 1 can run in parallel:

- `03-01`: trace heartbeat, adapter, and plugin-fed execution paths into command construction or downstream execution
- `03-02`: review workspaces, worktrees, runtime services, and CLI worktree seeding for boundary escape and host mutation
- `03-03`: review env propagation, session persistence, logs, and run-artifact exposure

Wave 2 should depend on all three:

- `03-04`: synthesize the results into a deployment-mode and attacker-position matrix that separates real authenticated or public risk from accepted local-only sharp edges

This split keeps the code-path tracing, filesystem review, and artifact-exposure review independent first, then uses their outputs to classify severity and trust-boundary meaning correctly.

## Verification Expectations For Phase Execution

When Phase 3 is executed, each artifact should be checked for:

- concrete file references to execution helpers, adapters, routes, schemas, and tests
- explicit distinction between local-operator or plugin-author trust and remotely relevant authenticated risk
- coverage of both local child-process execution and non-local adapter execution paths
- direct treatment of worktree provisioning, runtime services, and worktree seeding as host-mutation boundaries
- direct treatment of env injection, session persistence, logs, and durable artifacts as exposure boundaries
- clear candidate findings, accepted-risk notes, or no-finding conclusions that can feed the shared findings log
