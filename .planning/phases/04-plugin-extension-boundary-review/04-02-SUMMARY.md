---
phase: 04-plugin-extension-boundary-review
plan: "02"
subsystem: security-docs
tags: [plugins, capabilities, rpc, company-scope, bridge]
requires: []
provides:
  - Capability and worker-to-host privilege review
  - Confirmed finding on cross-company bridge-driven worker access
affects: [phase-04, phase-05, phase-07]
tech-stack:
  added: []
  patterns: [capability-map review, bridge-to-host authority tracing]
key-files:
  created:
    - .planning/phases/04-plugin-extension-boundary-review/04-CAPABILITY-HOST-RPC-REVIEW.md
  modified: []
key-decisions:
  - "Treat worker capability gating and caller company scoping as separate boundaries, because the former is stronger than the latter in current code."
  - "Classify the bridge company's optional top-level `companyId` check as a real boundary failure once combined with the no-op company availability gate in host services."
patterns-established:
  - "Trace plugin privilege from manifest capabilities to real host handlers instead of assuming the capability list is the full authority story."
  - "Use the route-to-worker-to-host chain as the proof signal for plugin-driven tenant-boundary failures."
requirements-completed: [PLUG-01, PLUG-02]
duration: 20min
completed: 2026-03-16
---

# Phase 4: Capability And Host-RPC Summary

**Mapped the full worker-to-host privilege surface and identified the main Phase 4 tenant-isolation bug: plugin bridge calls can escape the current board user’s company scope once the worker starts making host-service calls.**

## Performance

- **Duration:** 20 min
- **Started:** 2026-03-16T15:32:00Z
- **Completed:** 2026-03-16T15:52:00Z
- **Tasks:** 3
- **Files modified:** 1

## Accomplishments

- Documented the runtime capability model from manifest validation through the SDK host-client factory.
- Compared gated worker RPC methods with intentionally ungated methods such as `config.get`, `entities.*`, and `log`.
- Confirmed that the UI bridge does not carry board caller company scope into worker host services, allowing cross-company worker actions when a plugin supplies its own company ID inside `params`.
- Captured the main positive controls in the host-service layer: SSRF protections for outbound HTTP and config-ref scoping for secret resolution.

## Task Commits

This plan is included in the consolidated Phase 4 completion commit.

## Files Created/Modified

- `.planning/phases/04-plugin-extension-boundary-review/04-CAPABILITY-HOST-RPC-REVIEW.md` - Capability model, worker RPC surface, and host-service boundary review

## Decisions Made

- Treated bridge company scoping as a separate boundary from worker capability enforcement so the real failure mode stayed visible.
- Kept instance-wide capability breadth visible as an accepted sharp edge rather than overclaiming it as a broken boundary by itself.

## Deviations from Plan

- None

## Issues Encountered

- The route layer, UI bridge layer, and host-service layer each enforce different pieces of scope, so proving the boundary failure required tracing all three together.

## User Setup Required

- None
