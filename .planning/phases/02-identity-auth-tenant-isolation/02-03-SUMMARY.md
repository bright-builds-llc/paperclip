---
phase: 02-identity-auth-tenant-isolation
plan: "03"
subsystem: security-docs
tags: [websocket, auxiliary-routes, parity, authz]
requires: [02-01, 02-02]
provides:
  - Realtime and auxiliary-route auth parity review
  - Confirmed findings for websocket hostname drift and unauthenticated run-to-issues metadata access
affects: [phase-02, phase-03, phase-07]
tech-stack:
  added: []
  patterns: [HTTP-vs-upgrade parity review, auxiliary-route auth analysis]
key-files:
  created:
    - .planning/phases/02-identity-auth-tenant-isolation/02-REALTIME-AUX-AUTHZ-REVIEW.md
  modified: []
key-decisions:
  - "Classify websocket hostname-check drift as a confirmed parity bug because the HTTP private-mode guard is explicit and the upgrade path bypasses it entirely."
  - "Classify the run-to-issues route as a confirmed vulnerability because it returns company-linked data without consulting `req.actor` at all."
patterns-established:
  - "Compare non-REST paths against the concrete HTTP auth model instead of assuming parity."
  - "Treat token-gated public routes and intentionally unauthenticated webhooks separately from true missing-auth bugs."
requirements-completed: [AUTH-03]
duration: 14min
completed: 2026-03-15
---

# Phase 2: Realtime And Auxiliary Authz Summary

**Compared websocket and auxiliary routes to the main HTTP auth model and found two confirmed parity bugs: private-mode websocket hostname bypass and an unauthenticated run-to-issues lookup route.**

## Performance

- **Duration:** 14 min
- **Started:** 2026-03-15T22:08:00Z
- **Completed:** 2026-03-15T22:22:00Z
- **Tasks:** 3
- **Files modified:** 1

## Accomplishments

- Compared websocket upgrade authorization to the HTTP actor and company-boundary model.
- Confirmed that `authenticated/private` hostname checks do not apply to live-events websocket upgrades.
- Confirmed that `GET /api/heartbeat-runs/:runId/issues` exposes issue metadata without any auth check.
- Separated intentionally public or token-gated auxiliary routes from true parity defects.

## Task Commits

This docs-only plan is included in the consolidated Phase 2 documentation commit.

## Files Created/Modified

- `.planning/phases/02-identity-auth-tenant-isolation/02-REALTIME-AUX-AUTHZ-REVIEW.md` - Websocket and auxiliary authorization parity review

## Decisions Made

- Treated websocket query-token support as a runtime-validation lead instead of a confirmed vulnerability because the code review alone does not show whether deployments leak those URLs.
- Kept plugin webhook auth out of the confirmed findings because signature verification is intentionally delegated to plugin logic.

## Deviations from Plan

- None

## Issues Encountered

- Auxiliary routes mix intentional public surfaces with internal helper routes, so each path had to be classified separately rather than by directory alone.

## User Setup Required

- None
