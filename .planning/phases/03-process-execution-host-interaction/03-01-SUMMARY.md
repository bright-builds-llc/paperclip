---
phase: 03-process-execution-host-interaction
plan: "01"
subsystem: security-docs
tags: [execution, adapters, heartbeat, commands]
requires: []
provides:
  - Process-execution map from routes and persisted config into adapter execution
  - Comparison of local, process, HTTP, and gateway adapter risk
affects: [phase-03, phase-04, phase-05, phase-06]
tech-stack:
  added: []
  patterns: [execution-family comparison, file-grounded sink tracing]
key-files:
  created:
    - .planning/phases/03-process-execution-host-interaction/03-PROCESS-EXECUTION-REVIEW.md
  modified: []
key-decisions:
  - "Treat board-authored adapter execution as trusted-operator power unless a weaker principal can reach the same sink."
  - "Separate local child-process adapters from HTTP and gateway adapters so local-host execution is not overstated."
patterns-established:
  - "Trace execution sinks backward through heartbeat orchestration, secret resolution, and plugin wakeups."
  - "Evaluate adapter controls per family instead of assuming all adapters share the same guardrails."
requirements-completed: [EXEC-01]
duration: 18min
completed: 2026-03-15
---

# Phase 3: Process Execution Summary

**Mapped every reviewed execution sink back to its controlling inputs and concluded that the strongest local-host paths are board-controlled operator features, not immediate shell-injection bugs.**

## Performance

- **Duration:** 18 min
- **Started:** 2026-03-15T22:35:00Z
- **Completed:** 2026-03-15T22:53:00Z
- **Tasks:** 3
- **Files modified:** 1

## Accomplishments

- Documented how routes, persisted adapter config, secret resolution, issue or project workspace policy, and plugin wakeups converge in `heartbeat.ts`.
- Compared local CLI adapters, the generic process adapter, the HTTP adapter, and the OpenClaw gateway adapter in one artifact.
- Distinguished real local guardrails such as `shell: false`, absolute-directory checks, and command resolution from weaker trust assumptions in the generic process adapter.

## Task Commits

This plan is included in the consolidated Phase 3 completion commit.

## Files Created/Modified

- `.planning/phases/03-process-execution-host-interaction/03-PROCESS-EXECUTION-REVIEW.md` - Process-execution entry points, adapter families, and guardrail review

## Decisions Made

- Classified board-controlled execution power as an accepted operator boundary for now rather than a standalone vulnerability.
- Carried plugin-to-heartbeat prompt reach forward as a later plugin-boundary question instead of overstating it in Phase 3.

## Deviations from Plan

- None

## Issues Encountered

- The registry includes multiple adapter families with materially different trust assumptions, so the review had to stay comparative instead of summarizing them as a single execution surface.

## User Setup Required

- None
