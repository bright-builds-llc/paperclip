---
phase: 04-plugin-extension-boundary-review
plan: "01"
subsystem: security-docs
tags: [plugins, install, lifecycle, workers, management-routes]
requires: []
provides:
  - Plugin install and activation review
  - Confirmed finding on instance-wide plugin management by non-admin board users
affects: [phase-04, phase-06, phase-07]
tech-stack:
  added: []
  patterns: [install-path tracing, lifecycle-boundary review]
key-files:
  created:
    - .planning/phases/04-plugin-extension-boundary-review/04-INSTALL-LIFECYCLE-REVIEW.md
  modified: []
key-decisions:
  - "Treat instance-wide plugin lifecycle control as a real authorization boundary because normal board sessions remain company-scoped elsewhere in the codebase."
  - "Give credit for npm install hardening and minimal worker env handling, but not for the dormant sandbox helper that is not in the runtime path."
patterns-established:
  - "Separate install-time package validation from the later trust boundary created by starting worker code."
  - "Judge plugin lifecycle routes against the implemented board company-scope model, not the broadest product-language notion of board power."
requirements-completed: [PLUG-01]
duration: 22min
completed: 2026-03-16
---

# Phase 4: Install And Lifecycle Summary

**Reviewed the full plugin install and activation path and concluded that the main boundary break is not npm hook execution but the fact that any board user can manage instance-wide plugin code and lifecycle.**

## Performance

- **Duration:** 22 min
- **Started:** 2026-03-16T15:10:00Z
- **Completed:** 2026-03-16T15:32:00Z
- **Tasks:** 3
- **Files modified:** 1

## Accomplishments

- Mapped npm installs, local-path installs, manifest validation, route-path collision checks, and activation through the loader and lifecycle manager.
- Documented the real worker runtime model: `fork()` plus IPC and crash recovery, not an active VM sandbox.
- Confirmed that global plugin lifecycle routes are only `assertBoard`-gated even though the rest of the codebase treats non-admin board users as company-scoped.
- Captured the stricter-than-documented capability-upgrade behavior and the extra trust introduced by local-path and tsx-backed dev flows.

## Task Commits

This plan is included in the consolidated Phase 4 completion commit.

## Files Created/Modified

- `.planning/phases/04-plugin-extension-boundary-review/04-INSTALL-LIFECYCLE-REVIEW.md` - Plugin install source, worker lifecycle, and management-boundary review

## Decisions Made

- Classified instance-wide plugin lifecycle control by any board user as a confirmed vulnerability instead of treating it as generic board power.
- Kept local-path and dev-loader behavior in accepted-risk territory because it stays inside the documented local-authoring model.

## Deviations from Plan

- None

## Issues Encountered

- The lifecycle code and the loader disagree slightly about how capability-escalating upgrades should reach `upgrade_pending`, so the artifact had to call out implementation drift separately from security impact.

## User Setup Required

- None
