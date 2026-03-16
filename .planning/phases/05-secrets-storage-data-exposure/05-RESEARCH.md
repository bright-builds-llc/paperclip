# Phase 5 Research: Secrets, Storage & Data Exposure

## Planning Question

Which Paperclip paths can persist, copy, back up, or re-expose secrets and tenant data through secret providers, storage backends, inline documents, run artifacts, and operational logs, and how should the phase split so those exposure paths are reviewed without flattening all persistence into the same severity?

## Phase Contract

Phase 5 is the data-exposure phase of the audit:

- `DATA-01`: review whether secret storage, config defaults, backup flows, and redaction logic prevent sensitive data exposure
- `DATA-02`: review whether attachments, documents, assets, run logs, and storage providers expose tenant data or unsafe file content

Phase 3 already showed that run artifacts can expose same-company secrets and session data. Phase 4 showed that plugin-controlled paths can now reach more privileged runtime behavior. Phase 5 should determine what sensitive material is actually persisted, copied, and readable after those earlier boundaries have been crossed.

## Repo-Grounded Facts That Shape The Plan

- Secret persistence is split between metadata and version material:
  - `server/src/services/secrets.ts` stores secret metadata in `company_secrets` and encrypted or provider-specific material in `company_secret_versions`
  - `packages/shared/src/validators/secret.ts` still accepts legacy inline string env bindings, and strict mode only blocks new sensitive plain values when config is normalized for persistence
  - secret CRUD routes in `server/src/routes/secrets.ts` are board-only plus `assertCompanyAccess()`, and they return metadata rather than raw secret values
- The default local secret provider protects at rest but not against operational copying:
  - `server/src/secrets/local-encrypted-provider.ts` uses AES-256-GCM and auto-creates a 32-byte master key if none exists
  - the active key can come from `PAPERCLIP_SECRETS_MASTER_KEY`, a key file, or the default path under the Paperclip instance
  - `server/src/config.ts` and `packages/shared/src/config-schema.ts` default to `local_encrypted` with strict mode off unless the operator opts in
- Backup and worktree flows are part of the secret boundary:
  - automatic DB backups are enabled by default and `packages/db/src/backup.ts` passes no table exclusions or column nullification into `runDatabaseBackup(...)`
  - `packages/db/src/backup-lib.ts` can exclude tables or nullify columns, but the default full backup path includes whatever the database currently stores
  - `cli/src/commands/worktree.ts` copies the secrets master key into seeded worktrees and restores a DB backup into the new instance; the minimal seed mode excludes run and runtime tables but does not exclude secret tables, documents, or attachment metadata
- Runtime secret resolution reaches multiple powerful surfaces:
  - `secretService.resolveAdapterConfigForRuntime(...)` resolves secret refs into plain env values before heartbeat execution
  - Phase 4 already established that plugins can reach `ctx.secrets.resolve()` through `server/src/services/plugin-secrets-handler.ts` once capability and bridge boundaries are crossed
  - `scripts/migrate-inline-env-secrets.ts` exists because legacy inline secrets can still be present in persisted agent config
- Storage providers defend path boundaries, but they do not define read authorization by themselves:
  - `server/src/storage/service.ts` builds company-prefixed object keys and rejects cross-company or traversal-style keys
  - `server/src/storage/local-disk-provider.ts` resolves keys within the configured root, while `server/src/storage/s3-provider.ts` adds a provider prefix without changing the company-prefix model
  - direct asset image uploads and issue attachments stream bytes through memory, persist metadata in `assets`, and expose content again through `/api/assets/:assetId/content` or `/api/attachments/:attachmentId/content`
- Not all tenant content lives in object storage:
  - `documents.latestBody` and every row in `document_revisions.body` are stored inline in Postgres
  - issue document routes use `assertCompanyAccess()` and return full body or revision contents to any actor who passes that company check
  - this means DB backups and worktree seeding can copy full document text even when asset bytes stay in an external provider
- Persisted operational artifacts still have mixed redaction and access controls:
  - `server/src/routes/agents.ts` returns heartbeat run details with `redactCurrentUserValue(...)`, event payloads with `redactEventPayload(...)`, and run logs with no secret-aware redaction beyond current-user path hiding
  - `server/src/routes/activity.ts` exposes company and issue activity with `assertCompanyAccess()` and still contains the unauthenticated `/api/heartbeat-runs/:runId/issues` route found in Phase 2
  - agent config revision routes in `server/src/routes/agents.ts` use `redactEventPayload(...)`, which is materially stronger than the run-detail path
  - plugin logs are persisted in `plugin_logs`, pruned on retention, and exposed through board-only plugin log routes, but they can still contain plugin-supplied metadata after only generic sanitization

## Review Tracks The Plan Should Cover

### 1. Secrets, Key Management, And Backup Or Seed Flows

This track should answer:

- how secret values are created, rotated, stored, and resolved at runtime
- which config defaults or legacy compatibility paths still allow sensitive inline values to persist
- whether backups, worktree seeding, and key-copy behavior can duplicate secret material or the keys needed to decrypt it

Primary evidence:

- `server/src/services/secrets.ts`
- `server/src/routes/secrets.ts`
- `server/src/secrets/local-encrypted-provider.ts`
- `server/src/secrets/provider-registry.ts`
- `server/src/config.ts`
- `packages/shared/src/config-schema.ts`
- `packages/db/src/backup-lib.ts`
- `packages/db/src/backup.ts`
- `cli/src/commands/worktree-lib.ts`
- `cli/src/commands/worktree.ts`
- `scripts/migrate-inline-env-secrets.ts`

### 2. Assets, Attachments, Documents, And Storage Providers

This track should answer:

- how uploaded bytes and document bodies are partitioned between object storage and database rows
- whether storage providers, upload routes, and content-read routes preserve company boundaries and file-safety expectations
- whether document revisions, attachment metadata, or storage defaults broaden exposure through backups or broad same-company access

Primary evidence:

- `server/src/storage/service.ts`
- `server/src/storage/local-disk-provider.ts`
- `server/src/storage/s3-provider.ts`
- `server/src/storage/provider-registry.ts`
- `server/src/routes/assets.ts`
- `server/src/routes/issues.ts`
- `server/src/services/assets.ts`
- `server/src/services/documents.ts`
- `server/src/services/issues.ts`
- `packages/db/src/schema/assets.ts`
- `packages/db/src/schema/documents.ts`
- `packages/db/src/schema/document_revisions.ts`
- `packages/db/src/schema/issue_attachments.ts`
- `packages/db/src/schema/issue_documents.ts`

### 3. Run Logs And Other Persisted Operational Artifacts

This track should answer:

- which run details, logs, activity rows, config revisions, plugin logs, and session-linked artifacts remain readable after execution
- which surfaces get secret-aware redaction versus only username or path redaction
- how earlier plugin and same-company auth findings translate into concrete persisted-data exposure

Primary evidence:

- `.planning/phases/03-process-execution-host-interaction/03-ENV-SESSION-LOG-REVIEW.md`
- `.planning/phases/04-plugin-extension-boundary-review/04-VERIFICATION.md`
- `server/src/routes/agents.ts`
- `server/src/routes/activity.ts`
- `server/src/services/activity.ts`
- `server/src/services/activity-log.ts`
- `server/src/services/run-log-store.ts`
- `server/src/services/plugin-host-services.ts`
- `server/src/services/plugin-log-retention.ts`
- `server/src/redaction.ts`
- `server/src/log-redaction.ts`
- `packages/db/src/schema/heartbeat_runs.ts`
- `packages/db/src/schema/agent_task_sessions.ts`
- `packages/db/src/schema/agent_runtime_state.ts`
- `packages/db/src/schema/activity_log.ts`
- `packages/db/src/schema/plugin_logs.ts`

## Concrete Boundary Questions Worth Testing

These are the repo-backed questions the execution plans should resolve.

1. Do default backup flows serialize encrypted secret material, inline document bodies, config snapshots, and other sensitive database content without exclusion or nullification?
2. Does worktree seeding copy the master key and seeded database contents in a way that materially broadens secret exposure compared with a fresh instance?
3. How much real protection does strict secret mode provide when legacy inline strings are still accepted and existing persisted configs may already contain plaintext values?
4. Which secret and config read paths return metadata only, and which runtime or artifact paths can still surface resolved values indirectly?
5. Are company-prefixed object keys and traversal protections enough, or do broad same-company read routes still expose attachment or asset bytes too freely?
6. Which tenant content stays inline in Postgres rather than object storage, and what does that imply for backups, restores, and seeded worktrees?
7. Which persisted artifact routes apply `sanitizeRecord(...)` or `redactEventPayload(...)`, and which still rely only on `redactCurrentUserValue(...)` or no meaningful secret-aware redaction at all?
8. Given the Phase 4 plugin findings, what secrets or tenant data could a plugin-controlled path cause to be persisted and later read back through ordinary board or agent routes?

## Existing Tests Worth Reusing As Evidence Anchors

- `server/src/__tests__/redaction.test.ts`
- `server/src/__tests__/log-redaction.test.ts`
- `server/src/__tests__/storage-local-provider.test.ts`
- `server/src/__tests__/documents.test.ts`
- `server/src/__tests__/heartbeat-workspace-session.test.ts`
- `cli/src/__tests__/worktree.test.ts`

These tests do not prove the phase outcomes by themselves, but they mark where the repo already encodes intended redaction, storage-path, document, and worktree-seeding behavior.

## Recommended Execution Outputs

Phase 5 should produce three artifacts:

- `05-SECRETS-BACKUP-REVIEW.md`
  - secret provider and key model
  - persistence and runtime-resolution behavior
  - backup, seed, and config-copy flows
  - findings or review leads
- `05-ASSET-DOCUMENT-STORAGE-REVIEW.md`
  - storage provider and object-key model
  - asset and attachment upload or read surfaces
  - document and revision persistence
  - findings or review leads
- `05-PERSISTED-ARTIFACT-REVIEW.md`
  - heartbeat run and log surfaces
  - activity, config revision, and plugin log persistence
  - redaction coverage and read-surface gaps
  - findings or review leads

## Recommended Plan Split

Wave 1 can run in parallel:

- `05-01`: review secret providers, key defaults, runtime resolution, and backup or seed behavior
- `05-02`: review assets, attachments, documents, and storage-provider boundaries

Wave 2 should depend on both:

- `05-03`: review run logs and other persisted operational artifacts using the secret-handling and storage conclusions from Wave 1

This split keeps secret-handling and content-storage review independent first, then uses both results to judge whether persisted artifact exposure is a distinct vulnerability class or a downstream consequence of earlier storage and plugin trust decisions.

## Verification Expectations For Phase Execution

When Phase 5 is executed, each artifact should be checked for:

- concrete file references to secret services, provider modules, routes, storage backends, schemas, and backup or worktree tooling
- explicit distinction between encrypted-at-rest protections and operational copying or readback exposure
- direct treatment of DB backups and worktree seeding as part of the data-exposure boundary
- direct treatment of inline document bodies and revision history as database-resident tenant data
- direct treatment of run detail, run log, activity, config revision, and plugin log read surfaces as separate artifact classes
- clear candidate findings, accepted-risk notes, or no-finding conclusions that can feed the shared findings log
