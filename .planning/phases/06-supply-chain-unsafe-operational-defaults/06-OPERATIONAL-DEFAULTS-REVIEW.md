# Phase 6: Operational Defaults Review

## Scope

This artifact reviews the first-run and routine operator workflows that shape Paperclip's security posture before maintainers even reach the package publish chain.

- Primary question: whether onboarding, `run`, `doctor`, worktree, Docker smoke, CI, and release-entrypoint defaults can ship insecure deployment states or normalize risky trust assumptions
- Boundary lens: distinguish intentional local-operator convenience from defaults that can escape into authenticated or internet-facing deployments
- Principal focus: maintainers, local operators, and first-time operators following the documented setup paths

Primary files reviewed:

- `cli/src/commands/onboard.ts`
- `cli/src/prompts/server.ts`
- `cli/src/prompts/secrets.ts`
- `cli/src/commands/run.ts`
- `cli/src/commands/doctor.ts`
- `cli/src/checks/deployment-auth-check.ts`
- `cli/src/commands/worktree.ts`
- `cli/src/commands/worktree-lib.ts`
- `cli/src/config/env.ts`
- `cli/src/config/secrets-key.ts`
- `server/src/startup-banner.ts`
- `scripts/docker-onboard-smoke.sh`
- `.github/workflows/pr-verify.yml`
- `.github/workflows/release.yml`
- `doc/DEPLOYMENT-MODES.md`
- `doc/DEVELOPING.md`
- `doc/DOCKER.md`
- `doc/RELEASING.md`

## Onboarding, Run, And Doctor Defaults

Quickstart is still intentionally convenience-first.

`cli/src/commands/onboard.ts` defaults quickstart to:

- `local_trusted`
- private exposure
- embedded Postgres
- file logging
- `local_disk` storage
- `local_encrypted` secrets
- database backups enabled
- secret strict mode off

`cli/src/prompts/secrets.ts` makes that default explicit: `defaultSecretsConfig()` returns `strictMode: false`.

That means a stock first-run config preserves the Phase 5 secret-at-rest weakness as the normal starting posture. Secret refs are recommended, but not required, and backups stay on by default.

The guardrails are mixed rather than absent.

- `onboard.ts` auto-creates `PAPERCLIP_AGENT_JWT_SECRET`
- `cli/src/config/secrets-key.ts` auto-creates the local-encrypted master key file with mode `0o600`
- `cli/src/checks/deployment-auth-check.ts` hard-fails `local_trusted` if the configured host is not loopback
- the same check hard-fails authenticated public mode when `auth.publicBaseUrl` is missing or malformed

The main weakness is how public authenticated mode handles transport security.

`cli/src/prompts/server.ts` presents `authenticated + public` as the "Public internet" path and accepts either `http://` or `https://` URLs. `doc/DEPLOYMENT-MODES.md` says this mode has "stricter deployment checks and failures in doctor", but `deploymentAuthCheck(...)` only returns a warning, not a failure, when the explicit public URL uses plain HTTP.

`cli/src/commands/run.ts` blocks startup only when `summary.failed > 0`, so warning-only HTTP public deployments still start normally.

This is materially weaker than the documented policy for internet-facing mode.

## Worktree, Docker, And Local-Operator Flows

Worktree initialization intentionally clones a large amount of local state.

`cli/src/commands/worktree.ts` and `cli/src/commands/worktree-lib.ts` do all of the following by default:

- create repo-local `.paperclip/config.json` and `.paperclip/.env`
- auto-load the repo-local env in future CLI and server runs
- choose free app and embedded Postgres ports
- seed a new isolated instance from the current effective instance in `minimal` mode
- copy the local-encrypted secrets master key into the new worktree instance
- preserve the source deployment mode, exposure, backup settings, and `secrets.strictMode` when building the worktree config

The "minimal" seed mode drops heavy operational history tables, but it still preserves core app and auth state. Combined with the copied master key, this remains a powerful local duplication flow rather than a clean-room dev environment.

`cli/src/config/env.ts` also writes `PAPERCLIP_AGENT_JWT_SECRET` into a repo-local `.env` file when needed, which is convenient and consistent with the local trust model but reinforces that worktree state is a near-clone, not just a workspace helper.

The Docker smoke harness is similarly explicit about prioritizing operator convenience.

`scripts/docker-onboard-smoke.sh` defaults to:

- `authenticated/private`
- `HOST=0.0.0.0`
- a fixed admin email and password
- automatic board bootstrap when the instance is in authenticated mode

`doc/DOCKER.md` frames that script as a smoke harness, which keeps it inside an accepted test-only trust model. But it is still important because it normalizes real bootstrap flows and can be copied into ad hoc operator workflows.

## CI And Release Entrypoint Assumptions

The maintainer entrypoints themselves are relatively well guarded.

- `.github/workflows/release.yml` only runs publish from `release/*` branches and gates stable publish behind the `npm-release` environment with OIDC-capable permissions
- `scripts/release-start.sh`, `scripts/release-preflight.sh`, and `scripts/release.sh` require the expected `release/X.Y.Z` branch, clean worktree state, and unused tag/version space
- `doc/RELEASING.md` makes the same branch-driven release model explicit

That is materially stronger than the default operator flows.

The looser point is the everyday verification path.

`.github/workflows/pr-verify.yml` installs with `pnpm install --no-frozen-lockfile`, so ordinary PR validation is intentionally permissive about dependency graph drift. This becomes more important in the package review artifact, because it means the operator workflow and the release workflow are already evaluating different dependency assumptions before publishing.

## Findings Or Review Leads

- **Likely vulnerability / Medium:** Authenticated public deployments are only warning-gated when the configured public URL uses plain HTTP. `doc/DEPLOYMENT-MODES.md` says `authenticated + public` should have stricter doctor failures, but `cli/src/checks/deployment-auth-check.ts` returns `status: "warn"` rather than `fail` for non-HTTPS public URLs, and `cli/src/commands/run.ts` still starts the server because it only blocks on failures. Any operator who follows the public-hosting path without terminating TLS correctly can therefore launch an internet-facing board session flow over insecure transport.
- **Accepted-risk sharp edge / Medium:** Worktree initialization still defaults to a high-fidelity clone of local state, including copied secrets-key material, persisted auth or company data, and inherited weak secret defaults. This materially amplifies the Phase 5 secret and data-exposure risks, but it remains inside an explicitly local-operator workflow documented in `doc/DEVELOPING.md` rather than a remotely reachable boundary.
- **Accepted-risk sharp edge / Low:** Quickstart and `paperclipai run` intentionally prioritize successful local startup over hardened defaults. Out of the box they keep secret strict mode off and backups on, which preserves the Phase 5 plaintext-secret exposure path until an operator opts into stronger configuration.
- **No finding:** The CLI does meaningfully protect the `local_trusted` contract by failing non-loopback bindings in doctor and auto-generating local JWT and secrets-key material with locked-down file permissions.
- **No finding:** Release entrypoints are materially stricter than casual operator workflows. Branch matching, clean-worktree checks, release-train freeze checks, and trusted-publishing environment gates all exist in both scripts and workflow docs.

## Later Phase Questions

- Should `authenticated + public` over plain HTTP be upgraded from a warning to a hard failure in `doctor` and `run`?
- Should quickstart or worktree init surface a sharper warning when `secrets.strictMode` remains off, given the already-confirmed Phase 5 plaintext-secret exposure path?
- Should the worktree seed model eventually separate auth/bootstrap duplication from project or issue data seeding so local convenience does not automatically copy sensitive control-plane state?
