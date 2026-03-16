# Phase 3: Workspace And Host Boundary Review

## Scope

This artifact reviews how Paperclip realizes execution workspaces, creates git worktrees, starts runtime services, and seeds CLI-created worktrees.

- Primary question: where host state can be mutated outside the main project workspace
- Boundary lens: distinguish repo-contained worktree behavior from features that intentionally permit absolute paths, shell commands, shared services, or copied secrets
- Principal focus: board-authored workspace policy and local CLI operator flows

## Execution Workspace Realization

`server/src/services/execution-workspace-policy.ts` parses project and issue workspace policy into two main strategies:

- `project_primary`
- `git_worktree`

For `git_worktree`, `server/src/services/workspace-runtime.ts`:

- derives the branch name from templates and run context
- resolves `worktreeParentDir`, allowing absolute paths and `~` via `resolveConfiguredPath()`
- creates the parent directory if needed
- runs `git worktree add` into that derived location

This keeps the default workflow repo-adjacent, but not repo-confined. A board-authored policy can move the worktree parent outside the primary repository tree entirely.

Workspace policy can also specify `provisionCommand`. When present, `workspace-runtime.ts` executes it through `spawn(shell, ["-c", input.command])` inside the realized workspace. That is deliberate shell execution, not a sanitized wrapper.

`teardownCommand` is parsed from workspace policy, but this Phase 3 review did not find a corresponding runtime call site that executes it. That looks like an implementation gap or unfinished cleanup feature rather than a direct exploit.

## Runtime Services And Host Mutation

`workspaceRuntime.services[]` is a second host-mutation surface layered on top of the execution workspace. In `server/src/services/workspace-runtime.ts`, Paperclip:

- renders templated env and cwd values for each service
- resolves the service cwd with `resolveConfiguredPath()`, again permitting absolute paths
- allocates ports and readiness checks
- starts services with `spawn(shell, ["-lc", command])`
- persists service records and may reuse them across runs depending on lifecycle and `reuseScope`

The key security meaning:

- runtime services are true shell commands, not argument-vector spawns
- `reuseScope` can preserve state across runs at `project_workspace` or other shared scopes
- stop policies such as `idle_timeout` and `on_run_finish` affect cleanup, but shared services can outlive a single run materially

This is not accidental host escape. It is an operator feature that assumes workspace policy is trusted. Later phases should treat shared-service persistence as part of the artifact and secret surface, not just as execution convenience.

## Worktree Seeding And Filesystem Boundaries

The CLI worktree flow in `cli/src/commands/worktree.ts` widens the boundary further by copying or writing operational state into derived worktrees:

- it can copy or point to the secrets master key with `PAPERCLIP_SECRETS_MASTER_KEY` or `PAPERCLIP_SECRETS_MASTER_KEY_FILE`
- it persists `PAPERCLIP_AGENT_JWT_SECRET` into the worktree env when present
- it seeds the worktree database
- it mirrors shared git hooks into the linked worktree git directory

The tests in `cli/src/__tests__/worktree.test.ts` confirm these behaviors are intentional.

This is powerful and operationally convenient, but it means a worktree is not just a code checkout. It can become a fully bootable clone carrying secrets material, shared JWT signing state, database seed data, and hook-based execution behavior.

## Findings Or Review Leads

- **Accepted-risk sharp edge / Informational:** `provisionCommand` and runtime-service commands are board-controlled shell hooks. They intentionally mutate the host and workspace and therefore sit outside any sandbox guarantee.
- **Accepted-risk sharp edge / Informational:** Execution workspaces may live outside the primary repo tree because `resolveConfiguredPath()` accepts absolute paths and `~`. That is consistent with the current trusted-operator model, but it materially broadens the filesystem boundary.
- **Accepted-risk sharp edge / Medium:** CLI worktree seeding copies secrets master key material, shared agent JWT signing secrets, database seed state, and git hooks into derived worktrees. This is likely acceptable for a single trusted operator, but it increases the blast radius of any later worktree compromise or accidental sharing.
- **Needs runtime validation:** `teardownCommand` appears in parsed policy but not in the realized runtime path reviewed here. The likely impact is stale worktrees or service residue rather than direct compromise, but it should be confirmed in implementation or tests.
- **No finding:** The tested git worktree path in `server/src/__tests__/workspace-runtime.test.ts` shows the expected repo-scoped creation and reuse behavior when the operator does not override parent directories or shell hooks.

## Later Phase Questions

- Phase 4: can plugins author or mutate workspace-runtime policy, or do they only inherit operator-defined workspaces?
- Phase 5: which runtime-service env vars or worktree-seeded files later become readable through logs, UI, or stored artifacts?
- Phase 6: should operational tooling warn more aggressively before copying `PAPERCLIP_SECRETS_MASTER_KEY`, `PAPERCLIP_AGENT_JWT_SECRET`, or git hooks into new worktrees?
