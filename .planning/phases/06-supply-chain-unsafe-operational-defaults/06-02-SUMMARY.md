---
phase: 06-supply-chain-unsafe-operational-defaults
plan: "02"
subsystem: security-docs
tags: [lockfile, release, npm, packaging, publish]
requires:
  - 06-01
provides:
  - Package/build/publish integrity review
  - Likely finding on mutable PR dependency verification
affects: [phase-06, phase-07]
tech-stack:
  added: []
  patterns: [release-pipeline audit, publish-artifact review]
key-files:
  created:
    - .planning/phases/06-supply-chain-unsafe-operational-defaults/06-PACKAGE-BUILD-PUBLISH-REVIEW.md
  modified: []
key-decisions:
  - "Treat the PR-versus-release dependency-graph split as the main supply-chain integrity issue because it affects what code gets validated versus what code gets frozen and released."
  - "Record the hard-coded CLI manifest generator mismatch as a lower-severity likely issue because the generator is already out of sync with imports even though current shared dependencies partly mask the break."
patterns-established:
  - "Separate strong release-train authorization controls from weaker package-shape and dependency-resolution assumptions."
  - "Judge publish integrity by the artifact-generation scripts and package `files` rules, not just by the presence of Changesets and OIDC."
requirements-completed: [SUPP-01]
duration: 16min
completed: 2026-03-16
---

# Phase 6: Package, Build, And Publish Summary

**Confirmed that the release train itself has strong branch and publishing guardrails, but the package integrity model still has two important weak spots: PRs validate against a mutable dependency graph, and CLI publish dependency generation is hand-maintained and already out of sync with the adapter imports the CLI actually uses.**

## Performance

- **Duration:** 16 min
- **Started:** 2026-03-16T18:39:00Z
- **Completed:** 2026-03-16T18:55:00Z
- **Tasks:** 3
- **Files modified:** 1

## Accomplishments

- Traced the full package-management and release path across root manifest policy, bot-owned lockfile refresh, PR verify, release workflows, CLI publish-manifest rewriting, and release-time artifact copying.
- Identified the main supply-chain integrity issue: PR verification installs with `--no-frozen-lockfile` while lockfile refresh is deferred to a later bot PR, so reviewed code and released code can be validated against different dependency graphs.
- Documented a second lower-severity release-integrity problem: `generate-npm-package-json.mjs` is already out of sync with the CLI adapter import graph and therefore depends on manual list maintenance for completeness.
- Recorded the stronger existing controls too: trusted publishing, clean-worktree checks, release-branch discipline, and lockfile-refresh protection with `--ignore-scripts`.

## Task Commits

This plan is included in the consolidated Phase 6 completion commit.

## Files Created/Modified

- `.planning/phases/06-supply-chain-unsafe-operational-defaults/06-PACKAGE-BUILD-PUBLISH-REVIEW.md` - Package/build/publish integrity review

## Decisions Made

- Centered the artifact on what gets validated versus what gets frozen and published, because that is the main security boundary in the current supply chain.
- Treated release-time `skills` and `ui-dist` copying as a bounded maintainer sharp edge rather than the main vulnerability class, since branch and cleanup controls materially constrain it.

## Deviations from Plan

- None

## Issues Encountered

- The publish path mixes strong branch/auth controls with brittle manifest generation, so the artifact had to separate "who may publish" from "what exactly gets published."

## User Setup Required

- None
