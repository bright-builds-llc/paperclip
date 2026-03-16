---
phase: 05-secrets-storage-data-exposure
plan: "03"
subsystem: security-docs
tags: [artifacts, logs, runs, activity, revisions]
requires: [05-01, 05-02]
provides:
  - Persisted artifact exposure review
  - Consolidated artifact-read picture across run, activity, and plugin surfaces
affects: [phase-05, phase-07]
tech-stack:
  added: []
  patterns: [artifact redaction review, carry-forward exposure mapping]
key-files:
  created:
    - .planning/phases/05-secrets-storage-data-exposure/05-PERSISTED-ARTIFACT-REVIEW.md
  modified: []
key-decisions:
  - "Treat the persisted-artifact story as a carry-forward amplification of Phase 3 rather than pretending the issue starts at one route."
  - "Give explicit credit to config revision sanitization so the artifact distinguishes stronger and weaker persistence classes."
patterns-established:
  - "Map every artifact class to both its redaction layer and its read guard before assigning findings."
  - "Use carry-forward language when a later phase proves the same vulnerability spans more surfaces than first documented."
requirements-completed: [DATA-01, DATA-02]
duration: 15min
completed: 2026-03-16
---

# Phase 5: Persisted Artifact Summary

**Showed that persisted artifact exposure is broader than the original Phase 3 run endpoints: the same-company run-artifact problem reappears in issue-history reads, while other artifact classes such as config revisions are materially better protected.**

## Performance

- **Duration:** 15 min
- **Started:** 2026-03-16T17:57:00Z
- **Completed:** 2026-03-16T18:12:00Z
- **Tasks:** 3
- **Files modified:** 1

## Accomplishments

- Traced heartbeat run rows, event reads, log reads, and issue-scoped run history back to the underlying persisted fields.
- Confirmed that the Phase 3 same-company artifact exposure finding still applies and is reachable through additional persisted read surfaces such as `/api/issues/:id/runs`.
- Documented stronger controls on activity details and agent config revisions, including structured redaction and narrower configuration-read gates.
- Classified plugin log persistence as a sharp edge rather than a new core vulnerability because it remains inside the trusted-plugin and board-operator model.

## Task Commits

This plan is included in the consolidated Phase 5 completion commit.

## Files Created/Modified

- `.planning/phases/05-secrets-storage-data-exposure/05-PERSISTED-ARTIFACT-REVIEW.md` - Persisted run, activity, revision, and plugin-log exposure review

## Decisions Made

- Extended the earlier run-artifact finding rather than duplicating it as unrelated route bugs.
- Treated config revisions as a positive control because they store sanitized snapshots and reject redacted rollback states.

## Deviations from Plan

- None

## Issues Encountered

- The artifact review crossed several prior phases because the strongest persisted-data problems only become clear when route auth, redaction helpers, and execution artifacts are considered together.

## User Setup Required

- None
