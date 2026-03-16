# External Integrations

**Analysis Date:** 2026-03-15

## APIs & External Services

- LLM/provider configuration supports OpenAI and Anthropic-oriented flows through onboarding and adapter execution paths. Relevant code: `packages/shared/src/config-schema.ts`, `cli/src/commands/onboard.ts`, `packages/adapters/*/src/server/execute.ts`.
- Local agent adapters integrate with external CLIs and runtimes rather than hosting models directly. Examples: `packages/adapters/codex-local`, `packages/adapters/claude-local`, `packages/adapters/cursor-local`, `packages/adapters/gemini-local`, `packages/adapters/opencode-local`, `packages/adapters/pi-local`.
- The OpenClaw gateway adapter adds another external runtime integration surface in `packages/adapters/openclaw-gateway`.
- Plugin installation/discovery depends on the local filesystem plus npm package resolution in `server/src/services/plugin-loader.ts`.
- Release automation depends on GitHub and npm publishing scripts in `scripts/release.sh`, `scripts/create-github-release.sh`, and `.github/workflows/release.yml`.

## Data Storage

- Primary state lives in PostgreSQL through Drizzle tables exported from `packages/db/src/schema/index.ts`.
- Embedded PostgreSQL is the default local integration when `DATABASE_URL` is unset (`server/src/index.ts`, `doc/DATABASE.md`).
- File storage is abstracted behind storage providers with `local_disk` and `s3` support (`server/src/storage/index.ts`, `packages/shared/src/config-schema.ts`).
- Secret storage uses provider abstraction with `local_encrypted` as the default and optional external-secret-provider naming reserved in shared config constants.
- Database backup support is configured in the shared config schema and exercised through `pnpm db:backup` / `cli/src/commands/db-backup.ts`.

## Authentication & Identity

- Human auth in authenticated mode uses Better Auth sessions backed by Drizzle tables (`server/src/auth/better-auth.ts`, `packages/db/src/schema/auth.ts`).
- Board access is implicit in `local_trusted` mode via actor middleware (`server/src/middleware/auth.ts`).
- Agent auth uses bearer API keys stored as hashes in `agent_api_keys` or local JWTs in `server/src/middleware/auth.ts` and `server/src/agent-auth-jwt.ts`.
- Company membership and permission grants are persisted in `company_memberships` and `principal_permission_grants`, with access logic in `server/src/services/access.ts`.
- Invite/bootstrap flows extend identity management through `packages/db/src/schema/invites.ts`, `packages/db/src/schema/join_requests.ts`, and related CLI/server routes.

## Monitoring & Observability

- Structured application logging uses `pino`/`pino-http` in `server/src/middleware/logger.ts` and `server/src/app.ts`.
- Activity logging for domain mutations is centralized through `server/src/services/activity-log.ts` and persisted in `packages/db/src/schema/activity_log.ts`.
- Live operational updates are streamed over WebSockets from `server/src/realtime/live-events-ws.ts`.
- Heartbeat run logs and summaries are stored and summarized via `server/src/services/run-log-store.ts` and `server/src/services/heartbeat-run-summary.ts`.
- There is no evidence of external SaaS monitoring, tracing, or metrics export in the current repo snapshot.

## CI/CD & Deployment

- PR verification runs `pnpm -r typecheck`, `pnpm test:run`, and `pnpm build` in `.github/workflows/pr-verify.yml`.
- Browser E2E coverage is wired in `.github/workflows/e2e.yml`.
- `pr-policy.yml` enforces lockfile policy and PR hygiene.
- `refresh-lockfile.yml` auto-refreshes `pnpm-lock.yaml` outside normal PRs.
- `release.yml` and shell scripts in `scripts/` handle packaging, npm publish, and GitHub release creation.

## Environment Configuration

- Central runtime config resolution lives in `server/src/config.ts`; shared shapes live in `packages/shared/src/config-schema.ts`.
- Common env namespaces:
  - `PAPERCLIP_*` for deployment, storage, secrets, backups, scheduling, and runtime workspace info.
  - `DATABASE_URL` for external Postgres.
  - `BETTER_AUTH_*` / `PAPERCLIP_PUBLIC_URL` for authenticated deployments.
  - provider-specific keys such as `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, or adapter-specific API keys.
- Worktree-local repo instances add `.paperclip/config.json` and `.paperclip/.env` as described in `doc/DEVELOPING.md`.

## Webhooks & Callbacks

- Plugin webhooks are a first-class integration surface: `POST /api/plugins/:pluginId/webhooks/:endpointKey`, with schema/types in `packages/shared/src/types/plugin.ts` and runtime handling in server plugin services.
- Heartbeat invocations distinguish `scheduler`, `manual`, and callback-style triggers through shared constants and `server/src/services/heartbeat.ts`.
- Live board updates use WebSocket subscriptions at `/api/companies/:companyId/events/ws` in `server/src/realtime/live-events-ws.ts`.
- Adapter wakeups and runtime callbacks are represented in wake request and heartbeat tables such as `packages/db/src/schema/agent_wakeup_requests.ts` and `packages/db/src/schema/heartbeat_runs.ts`.

---

*Integration analysis: 2026-03-15*
*Update when external dependencies or deployment surfaces change*
