# Phase 6 Research: Supply Chain & Unsafe Operational Defaults

## Planning Question

Which onboarding, doctor, worktree, CI, release, package-management, and publish-artifact paths can ship Paperclip in insecure states or widen trust boundaries through convenience defaults, dependency drift, or release-time artifact mutation, and how should the phase split so operator-default review stays distinct from package and publish-chain review?

## Phase Contract

Phase 6 is the supply-chain and operational-defaults phase of the audit:

- `SUPP-01`: review whether onboarding, CI, release, worktree, and package-management flows can ship insecure defaults or weaken trust boundaries

Earlier phases already found auth, plugin, and data-surface weaknesses. Phase 6 should determine whether Paperclip's maintainer and operator workflows preserve those boundaries or make them easier to ship, clone, or miss during setup and release.

## Repo-Grounded Facts That Shape The Plan

- Quickstart and `--yes` onboarding still prefer convenience-first local defaults:
  - `cli/src/commands/onboard.ts` defaults quickstart to `local_trusted/private`, embedded Postgres, file logging, `local_disk` storage, `local_encrypted` secrets, database backups enabled, and secret strict mode off
  - the same flow auto-creates `PAPERCLIP_AGENT_JWT_SECRET`, auto-creates the local secrets key file, can auto-run the server, and prints the resulting mode without forcing a hardening review
  - environment-aware defaults can silently reshape that quickstart profile through `PAPERCLIP_PUBLIC_URL`, deployment mode, storage, backup, and secrets env vars
- `paperclipai run` and `paperclipai doctor` are operational trust boundaries, not just convenience wrappers:
  - `cli/src/commands/run.ts` auto-onboards if config is missing, runs `doctor` with repair enabled by default, then starts the server
  - `cli/src/commands/doctor.ts` can auto-repair local prerequisites and `cli/src/checks/deployment-auth-check.ts` enforces only a narrow set of deployment constraints, such as loopback binding for `local_trusted` and explicit auth URLs for authenticated public mode
  - the startup banner in `server/src/startup-banner.ts` surfaces mode, backup, and JWT state, but it does not itself tighten unsafe defaults
- Worktree tooling intentionally clones powerful local state:
  - `cli/src/commands/worktree.ts` and `cli/src/commands/worktree-lib.ts` create repo-local config and env files, mirror git hooks, seed a logical DB snapshot into the new instance, and copy the local-encrypted secrets master key
  - the default `minimal` seed mode drops some runtime history tables but preserves companies, auth state, issues, documents, secret tables, and other core data
  - inherited config keeps the source deployment mode, exposure, backup settings, and secret strict-mode setting unless the operator changes them later
- Docker and onboarding smoke flows deliberately exercise internet-adjacent behavior:
  - `scripts/docker-onboard-smoke.sh` defaults to `authenticated/private`, binds `HOST=0.0.0.0`, auto-bootstraps a board user, and uses fixed smoke credentials unless overridden
  - `doc/DOCKER.md` and `doc/DEVELOPING.md` document these flows as normal validation paths, so Phase 6 needs to classify which behaviors are explicit smoke-only shortcuts versus defaults that could be copied into real deployments
- CI policy is split between dependency-resolution flexibility and release immutability:
  - `.github/workflows/pr-policy.yml` blocks manual lockfile edits in normal PRs and regenerates the lockfile with `pnpm install --lockfile-only --ignore-scripts --no-frozen-lockfile` when manifests change
  - `.github/workflows/refresh-lockfile.yml` makes GitHub Actions the owner of `pnpm-lock.yaml`, again using `--ignore-scripts`
  - `.github/workflows/pr-verify.yml` installs with `pnpm install --no-frozen-lockfile`, while release and e2e workflows install with `--frozen-lockfile`
- The release train has strong branch and tag guardrails, but it mutates publish artifacts in-place:
  - `scripts/release-start.sh`, `scripts/release-preflight.sh`, `scripts/release.sh`, and `.github/workflows/release.yml` require a `release/X.Y.Z` branch, clean worktree, version-freeze checks, and either npm auth or GitHub Actions trusted publishing through the `npm-release` environment
  - `scripts/release.sh` temporarily rewrites workspace version state, copies `skills/` into `server/`, `packages/adapters/claude-local`, and `packages/adapters/codex-local`, builds `server/ui-dist`, and later restores those artifacts during cleanup
  - `doc/RELEASING.md` and `doc/PUBLISHING.md` treat these mutations as the maintainer-approved path, so Phase 6 needs to test whether they can leak maintainer-only content or create publish drift
- Package and publish-chain integrity currently relies on some hard-coded assumptions:
  - `scripts/build-npm.sh` rewrites `cli/package.json` into a publishable manifest via `scripts/generate-npm-package-json.mjs`, copies the repo README into `cli/README.md`, and leaves restoration to later cleanup
  - `generate-npm-package-json.mjs` hard-codes the workspace packages whose dependencies are folded into the published CLI and only externalizes `@paperclipai/server`
  - `cli/src/adapters/registry.ts` imports CLI helpers from `@paperclipai/adapter-cursor-local`, `@paperclipai/adapter-gemini-local`, and `@paperclipai/adapter-pi-local`, but those packages are not included in the hard-coded bundle-source list, making publish-manifest completeness a concrete review question
  - `server/package.json` still contains a duplicate dependency key for `@paperclipai/adapter-openclaw-gateway`, which is already surfacing as a warning during repo verification and signals manifest hygiene drift inside the publish surface
- Workspace packaging policy is explicit but broad:
  - root `package.json` and `pnpm-workspace.yaml` publish multiple public packages together under pnpm 9.15.4, with `.npmrc` setting `auto-install-peers=true`
  - `.changeset/config.json` versions `paperclipai` and all `@paperclipai/*` packages as one fixed release unit while ignoring the private `@paperclipai/ui` package
  - many public packages publish `dist` plus supplemental runtime files such as `skills` or `ui-dist`, so the release artifact boundary is wider than just compiled JS

## Review Tracks The Plan Should Cover

### 1. Operator And Bootstrap Defaults

This track should answer:

- what quickstart, `run`, `doctor`, worktree, and Docker smoke defaults actually do before an operator makes any hardening decisions
- which defaults are explicitly local-only or smoke-only sharp edges and which can ship into broader environments
- whether CI and release entrypoints reinforce safe operational posture or merely assume maintainers already know the right mode and branch discipline

Primary evidence:

- `cli/src/commands/onboard.ts`
- `cli/src/commands/run.ts`
- `cli/src/commands/doctor.ts`
- `cli/src/checks/deployment-auth-check.ts`
- `cli/src/commands/worktree.ts`
- `cli/src/commands/worktree-lib.ts`
- `server/src/startup-banner.ts`
- `scripts/docker-onboard-smoke.sh`
- `.github/workflows/pr-verify.yml`
- `.github/workflows/release.yml`
- `doc/DEVELOPING.md`
- `doc/DOCKER.md`
- `doc/RELEASING.md`

### 2. Package Management, Build, And Publish Integrity

This track should answer:

- whether lockfile, package-manager, and workspace versioning policy produce deterministic dependency and publish behavior
- whether public package manifests, prepack steps, and CLI manifest rewriting produce complete and reproducible npm artifacts
- whether release-time artifact copying and cleanup can leak maintainer-only files or ship incomplete dependencies

Primary evidence:

- `package.json`
- `pnpm-workspace.yaml`
- `.npmrc`
- `.changeset/config.json`
- `.github/workflows/pr-policy.yml`
- `.github/workflows/refresh-lockfile.yml`
- `scripts/build-npm.sh`
- `scripts/generate-npm-package-json.mjs`
- `scripts/prepare-server-ui-dist.sh`
- `scripts/release.sh`
- `scripts/release-lib.sh`
- `cli/package.json`
- `server/package.json`
- `cli/src/adapters/registry.ts`
- `packages/*/package.json`
- `packages/adapters/*/package.json`
- `packages/plugins/sdk/package.json`
- `doc/PUBLISHING.md`

## Concrete Boundary Questions Worth Testing

These are the repo-backed questions the execution plans should resolve.

1. Do quickstart onboarding and `paperclipai run` make it too easy to land in convenience-first states such as `local_trusted`, secret strict mode off, or backup-enabled copies of sensitive data without surfacing the risk clearly enough?
2. Does `doctor` only validate syntax and reachability, or does it meaningfully catch insecure deployment posture before the server starts?
3. Does worktree initialization intentionally duplicate too much trusted state, including auth, documents, secret tables, or decryptable material, for a default operator workflow?
4. Are Docker smoke defaults and documented one-liners clearly bounded as test harnesses, or do they normalize exposure patterns and credentials that could leak into real deployments?
5. Does PR verification test the same dependency graph that maintainers eventually release, given the split between `--no-frozen-lockfile` and bot-managed lockfile refreshes?
6. Does the CLI publish path generate a complete manifest for every adapter package the CLI actually imports, or can hard-coded bundle-package lists drift from runtime reality?
7. Do prepack and release-time artifact-copy steps risk shipping maintainer-only `skills`, stale UI bundles, or otherwise inconsistent publish payloads?
8. How much of the release chain's security depends on maintainer discipline versus hard enforcement in scripts, workflows, and environment protections?

## Existing Tests Worth Reusing As Evidence Anchors

- `cli/src/__tests__/doctor.test.ts`
- `cli/src/__tests__/worktree.test.ts`
- `paperclipai/src/__tests__/allowed-hostname.test.ts`
- `server/src/__tests__/forbidden-tokens.test.ts`
- `server/src/__tests__/paperclip-skill-utils.test.ts`

These tests do not prove Phase 6 outcomes by themselves, but they mark where the repo already encodes intended onboarding, worktree, token-screening, and skill-packaging behavior.

## Recommended Execution Outputs

Phase 6 should produce two artifacts:

- `06-OPERATIONAL-DEFAULTS-REVIEW.md`
  - onboarding, `run`, and `doctor` bootstrap model
  - worktree, Docker, and local-operator duplication flows
  - CI and release entrypoint assumptions
  - findings or review leads
- `06-PACKAGE-BUILD-PUBLISH-REVIEW.md`
  - lockfile and package-manager model
  - public package inventory and manifest rules
  - CLI and server publish-artifact generation
  - findings or review leads

## Recommended Plan Split

Wave 1:

- `06-01`: review onboarding, `run`, `doctor`, worktree, Docker, CI, and release-entrypoint trust assumptions

Wave 2:

- `06-02`: review package-management, build, and publish-chain unsafe defaults after the first artifact establishes which operational defaults and release entrypoints actually matter

This keeps the first plan focused on how Paperclip is bootstrapped and operated, then uses that context to judge whether the package and publish chain faithfully ship the intended runtime surface or introduce their own integrity and leakage problems.

## Verification Expectations For Phase Execution

When Phase 6 is executed, each artifact should be checked for:

- concrete file references to onboarding, doctor, run, worktree, Docker, workflow, release, package, and build scripts
- explicit distinction between accepted local-operator or smoke-test sharp edges and defaults that can ship into normal deployments or releases
- direct treatment of lockfile ownership, frozen versus non-frozen installs, and workspace versioning as part of the supply-chain model
- direct treatment of CLI manifest rewriting, server prepack UI bundling, and release-time skill copying as publish-artifact boundaries
- clear candidate findings, accepted-risk notes, or no-finding conclusions that can feed the final findings catalog
