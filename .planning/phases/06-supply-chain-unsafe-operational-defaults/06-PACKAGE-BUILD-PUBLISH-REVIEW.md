# Phase 6: Package, Build, And Publish Review

## Scope

This artifact reviews how the Paperclip monorepo becomes releasable npm packages and whether dependency-resolution policy, manifest rules, and release-time artifact mutation preserve package integrity.

- Primary question: whether lockfile policy, workspace manifests, build scripts, and release publish steps can make reviewed code differ from released code or ship incomplete artifacts
- Boundary lens: distinguish strong release-train controls from weaker package-shape and dependency-resolution assumptions
- Principal focus: maintainers shipping releases and end users installing public packages from npm

Primary files reviewed:

- `package.json`
- `pnpm-workspace.yaml`
- `.npmrc`
- `.changeset/config.json`
- `.github/workflows/pr-policy.yml`
- `.github/workflows/pr-verify.yml`
- `.github/workflows/refresh-lockfile.yml`
- `.github/workflows/release.yml`
- `scripts/build-npm.sh`
- `scripts/generate-npm-package-json.mjs`
- `scripts/prepare-server-ui-dist.sh`
- `scripts/release.sh`
- `scripts/release-lib.sh`
- `cli/package.json`
- `cli/src/adapters/registry.ts`
- `server/package.json`
- `packages/plugins/sdk/package.json`
- `doc/PUBLISHING.md`
- `doc/RELEASING.md`

## Lockfile And Package-Manager Model

The workspace package manager contract is explicit.

- root `package.json` pins `packageManager` to `pnpm@9.15.4`
- `pnpm-workspace.yaml` defines a broad multi-package workspace across `packages/*`, adapters, plugin examples, `server`, `ui`, and `cli`
- `.npmrc` sets `auto-install-peers=true`
- `.changeset/config.json` versions `paperclipai` and `@paperclipai/*` packages as a fixed release unit while ignoring the private `@paperclipai/ui`

The lockfile policy is more unusual.

- `.github/workflows/pr-policy.yml` forbids normal PRs from committing `pnpm-lock.yaml`
- when manifests change, the same workflow only validates that `pnpm install --lockfile-only --ignore-scripts --no-frozen-lockfile` can resolve successfully
- `.github/workflows/refresh-lockfile.yml` later regenerates `pnpm-lock.yaml` on `master` and opens or updates a bot PR
- `.github/workflows/pr-verify.yml` installs with `pnpm install --no-frozen-lockfile`
- release and e2e workflows install with `--frozen-lockfile`

That means Paperclip intentionally has two dependency graphs in play:

- the mutable graph used for PR verification
- the checked-in lockfile graph used for release and other frozen-lockfile workflows

This is not just stylistic drift. It means merge-time CI can pass against a dependency resolution that is not yet represented in the eventual released lockfile.

## Public Package Inventory And Manifest Rules

Package publication is broad but explicit.

- `doc/PUBLISHING.md` and `doc/RELEASING.md` treat `paperclipai` plus public workspace packages as one release unit
- public packages generally publish only `dist`, but some also ship runtime extras such as `skills` or `ui-dist`
- `server/package.json` publishes `dist`, `ui-dist`, and `skills`
- adapter packages such as `@paperclipai/adapter-cursor-local` and `@paperclipai/adapter-gemini-local` publish `dist` plus `skills`

The repo already shows manifest hygiene drift.

`server/package.json` contains `@paperclipai/adapter-openclaw-gateway` twice in its dependency object. Node-style JSON parsing makes that effectively "last key wins", so it is not catastrophic by itself, but it proves the publish-facing manifest surface is not being kept perfectly clean.

## Build And Publish Artifact Generation

The release chain has strong controls around *when* publish can happen, but weaker assumptions about *what* gets published.

### CLI packaging

`scripts/build-npm.sh` does three publish-shaping steps:

- bundles the CLI with esbuild
- rewrites `cli/package.json` into a publishable manifest through `scripts/generate-npm-package-json.mjs`
- copies the repo `README.md` into `cli/README.md`

`generate-npm-package-json.mjs` depends on a hard-coded `workspacePaths` list to collect the external dependencies that must remain in the published CLI manifest.

That list includes:

- `cli`
- `packages/db`
- `packages/shared`
- `packages/adapter-utils`
- `packages/adapters/claude-local`
- `packages/adapters/codex-local`
- `packages/adapters/opencode-local`
- `packages/adapters/openclaw-gateway`

But `cli/src/adapters/registry.ts` also imports:

- `@paperclipai/adapter-cursor-local/cli`
- `@paperclipai/adapter-gemini-local/cli`
- `@paperclipai/adapter-pi-local/cli`

So the manifest generator is already out of sync with the runtime import graph.

Today that mismatch is partly masked because the omitted packages only depend on `@paperclipai/adapter-utils` and `picocolors`, both of which are already pulled in through other included packages. But the completeness guarantee now depends on manual list maintenance rather than being derived from the actual bundle inputs.

### Server and adapter publish payloads

`scripts/release.sh` mutates the working tree to build release payloads:

- runs `pnpm build`
- runs `scripts/prepare-server-ui-dist.sh` to copy `ui/dist` into `server/ui-dist`
- copies the repo `skills/` directory into `server`, `packages/adapters/claude-local`, and `packages/adapters/codex-local`
- later restores those artifacts via `restore_publish_artifacts()`

This is bounded by clean-worktree enforcement and cleanup, but it means publish contents are created by imperative copy steps rather than by immutable package build outputs alone.

The strong side of the model is that release cleanup is explicit and stable-release publishing is gated by branch discipline, tag freezes, npm version checks, and trusted publishing. The weak side is that package shape is still partly maintained through ad hoc file copying and hard-coded manifest rewriting.

## Findings Or Review Leads

- **Likely vulnerability / Medium:** PR verification and release verification operate on different dependency graphs. Normal PR CI installs with `--no-frozen-lockfile` while manual lockfile edits are blocked and lockfile refresh is deferred to a later bot PR. Release and e2e paths then switch back to `--frozen-lockfile`. This means reviewed and merged code can be validated against a dependency resolution that is not the one eventually frozen and released, creating a real supply-chain integrity gap whenever manifests change or upstream package resolution shifts.
- **Likely vulnerability / Low:** The CLI publish-manifest generator is already out of sync with the CLI's adapter import graph. `scripts/generate-npm-package-json.mjs` hard-codes bundled workspace packages and omits `adapter-cursor-local`, `adapter-gemini-local`, and `adapter-pi-local` even though `cli/src/adapters/registry.ts` imports them. Current impact is partially masked by shared external dependencies, but the release path now relies on manual list maintenance rather than actual bundle inputs to decide which runtime dependencies are preserved.
- **Accepted-risk sharp edge / Low:** Release-time artifact copying still mutates the working tree by inserting `ui-dist` and `skills` into publishable packages. Clean-worktree enforcement and explicit cleanup keep this inside the maintainer release trust model, but it remains a brittle publish-shape mechanism that can drift if package `files` rules or intended public skill contents change.
- **No finding:** The release train itself has meaningful integrity guardrails. `release-start`, `release-preflight`, `release.sh`, and `.github/workflows/release.yml` all require the expected branch, clean worktree state, unused version/tag space, and trusted publishing through the `npm-release` environment.
- **No finding:** Lockfile refresh intentionally uses `--ignore-scripts` and fails if files other than `pnpm-lock.yaml` change, which is a real control against accidental install-script side effects during bot-managed lockfile updates.

## Later Phase Questions

- Should PR verification switch to a frozen-lockfile model once the bot-refreshed lockfile is available, or should lockfile generation move earlier so the reviewed graph and the released graph match?
- Should CLI dependency generation be derived from esbuild metadata or package import analysis instead of a hand-maintained workspace list?
- Should publish artifacts such as `skills` and `ui-dist` be produced by package-local build steps only, so release scripts stop copying runtime files into package directories directly?
