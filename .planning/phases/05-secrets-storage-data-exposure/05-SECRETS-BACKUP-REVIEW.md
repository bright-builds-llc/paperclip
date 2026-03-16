# Phase 5: Secrets, Backup, And Seed Review

## Scope

This artifact reviews how Paperclip stores secret material, when secret refs become plaintext again, and which backup or seeding flows can duplicate secrets or the keys needed to decrypt them.

- Primary question: whether the repo's secret-handling model actually keeps sensitive values out of long-lived storage and copied operational artifacts
- Boundary lens: separate metadata-only secret handling from runtime resolution, database persistence, and local operator backup or worktree flows
- Principal focus: board users who configure agents, plugins that resolve secret refs, and local operators who create backups or seeded worktrees

## Secret Provider And Key Model

The secret provider surface is split cleanly between metadata and value material.

- `packages/db/src/schema/company_secrets.ts` stores only secret identity metadata such as `name`, `provider`, `externalRef`, and `latestVersion`
- `packages/db/src/schema/company_secret_versions.ts` stores per-version `material` plus `valueSha256`
- `server/src/routes/secrets.ts` exposes CRUD routes that return `company_secrets` rows rather than resolved secret values
- `server/src/services/secrets.ts` enforces same-company ownership before resolving a referenced secret version

The default provider is `local_encrypted`.

- `server/src/secrets/provider-registry.ts` registers `local_encrypted` plus stub external providers
- `server/src/config.ts` and `packages/shared/src/config-schema.ts` default the instance to `local_encrypted`
- `server/src/secrets/local-encrypted-provider.ts` encrypts values with AES-256-GCM, auto-creates a 32-byte master key if needed, and writes the key file with `0600` permissions on a best-effort basis
- the master key can come from `PAPERCLIP_SECRETS_MASTER_KEY`, `PAPERCLIP_SECRETS_MASTER_KEY_FILE`, or the default instance path

That gives Paperclip a real at-rest protection story for secret refs handled through the dedicated secret system.

## Secret Persistence, Redaction, And Runtime Resolution

The main weakness is not the dedicated secret tables. It is the compatibility path that still allows plaintext env values to live outside that system.

`packages/shared/src/validators/secret.ts` defines `EnvBinding` as a union that still accepts:

- raw legacy strings
- `{ type: "plain", value: string }`
- `{ type: "secret_ref", secretId, version }`

`server/src/services/secrets.ts` only rejects sensitive plaintext env keys when `strictMode` is enabled:

- `normalizeAdapterConfigForPersistence(...)` and `normalizeEnvConfig(...)` allow plain values when `opts.strictMode` is false
- `server/src/config.ts` and `packages/shared/src/config-schema.ts` default `secretsStrictMode` to `false`
- `server/src/routes/agents.ts` uses that default when creating, hiring, or patching agents, then persists the result into `agents.adapterConfig`
- `packages/db/src/schema/agents.ts` stores `adapterConfig` and `runtimeConfig` directly as JSONB

This means Paperclip's dedicated secret provider is real, but using it for agent env secrets is still partially optional by default.

Runtime resolution is more narrowly scoped than persistence.

- `secretService.resolveAdapterConfigForRuntime(...)` converts `secret_ref` bindings back into plaintext env values just before execution
- `server/src/services/plugin-secrets-handler.ts` only resolves plugin secret refs that are present in the plugin's config schema or config JSON, and it avoids logging resolved values directly

There are also meaningful positive controls on redacted views.

- `server/src/services/agents.ts` builds config revision snapshots with `sanitizeRecord(...)`, so secret-like adapter config keys are stored as redacted placeholders in `agent_config_revisions`
- `server/src/routes/agents.ts` returns config revisions through `redactConfigRevision(...)`, and rollback is blocked if the saved revision contains redacted markers
- secret CRUD routes return metadata rows rather than provider-resolved plaintext values

So the main at-rest risk is the still-supported plaintext path in live agent config, not the revision history or the secret CRUD surface itself.

## Backup, Seed, And Config Flows

The backup and worktree tooling copies whatever the database currently stores unless the caller opts into exclusions.

`packages/db/src/backup-lib.ts` is intentionally generic.

- it iterates every included table and emits SQL `INSERT` statements for every row
- the only filtering hooks are `excludeTables` and `nullifyColumns`
- `packages/db/src/backup.ts` calls `runDatabaseBackup(...)` without exclusions or nullifications

That means ordinary backups include:

- `agents.adapterConfig` JSON as stored in the database
- `company_secret_versions.material`
- `documents.latestBody`
- `document_revisions.body`
- `heartbeat_runs.resultJson`, `contextSnapshot`, and related persisted artifacts

Whether that is catastrophic depends on what the live database holds. For secret refs, the backup only contains encrypted provider material. For legacy or plain env bindings, the backup contains the plaintext value because that value lives directly in `agents.adapterConfig`.

Worktree seeding broadens the operational blast radius further, but still inside a local-operator workflow.

- `cli/src/commands/worktree.ts` calls `copySeededSecretsKey(...)` before the seed backup or restore flow
- `copySeededSecretsKey(...)` either writes `PAPERCLIP_SECRETS_MASTER_KEY` directly into the target worktree or copies the configured local-encrypted key file
- the seed flow then runs `runDatabaseBackup(...)` and restores that SQL into the target embedded Postgres instance
- `cli/src/commands/worktree-lib.ts` only excludes runtime-heavy tables in minimal mode; it does not exclude `company_secrets`, `company_secret_versions`, `agents`, `documents`, or `document_revisions`

So a seeded worktree can receive both:

- the encrypted secret material from the source database
- the key required to decrypt locally encrypted secret versions

That is a powerful duplication path, but it is currently only reachable through a trusted local operator CLI workflow.

The migration tooling reflects the same reality.

- `scripts/migrate-inline-env-secrets.ts` exists because inline plaintext env values are still a supported legacy state
- the migration helper can convert those values into `secret_ref` bindings, but nothing in the default config forces that migration to happen before an instance starts taking backups

## Findings Or Review Leads

- **Likely vulnerability / Medium:** Secret-at-rest protection for agent env values is opt-in rather than guaranteed. `EnvBinding` still accepts plaintext strings and `{ type: "plain" }`, `secretsStrictMode` defaults to false, and `server/src/routes/agents.ts` persists those accepted values directly into `agents.adapterConfig`. Because `packages/db/src/backup-lib.ts` dumps `agents` rows verbatim, any instance that still uses plaintext env bindings will copy external credentials into ordinary database backups and seeded worktree databases without encryption.
- **Accepted-risk sharp edge / Medium:** Worktree seeding intentionally copies the local-encrypted secrets master key and restores a database backup that still includes secret tables, documents, and other sensitive data. This materially broadens the blast radius of local-operator workflows, but the path stays inside documented local tooling rather than a remotely reachable route.
- **No finding:** The dedicated secret CRUD surface does not return resolved secret values. `company_secrets` stores metadata only, `company_secret_versions` stores provider material, and the default `local_encrypted` provider applies AES-256-GCM plus a locked-down local key file.
- **No finding:** Agent config revisions are persisted in a redacted form rather than as raw secret-bearing snapshots, and rollback refuses to reapply a revision that still contains redacted markers.

## Later Phase Questions

- Phase 6 should decide whether plaintext env bindings should remain a compatibility path in authenticated deployments, or whether strict mode should become the default outside explicitly local-trusted workflows.
- Phase 6 should decide whether scheduled DB backups need table exclusions, column nullification, encryption, or operator warnings when plaintext adapter config is still present.
- Phase 7 should rank whether the highest-value fix is defaulting to strict mode, migrating legacy inline secrets automatically, or changing backup and seed behavior first.
