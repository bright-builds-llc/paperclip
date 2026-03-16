---
phase: 02-identity-auth-tenant-isolation
plan: "02"
subsystem: security-docs
tags: [agent-auth, tenant-isolation, permissions, company-scope]
requires: []
provides:
  - Agent credential and company-scope review
  - Confirmed company-boundary bypass findings for board-scoped mutations
affects: [phase-02, phase-03, phase-07]
tech-stack:
  added: []
  patterns: [credential-path review, tenant-boundary analysis]
key-files:
  created:
    - .planning/phases/02-identity-auth-tenant-isolation/02-AGENT-AUTHZ-REVIEW.md
  modified: []
key-decisions:
  - "Classify missing `assertCompanyAccess()` checks on company-scoped board mutations as confirmed vulnerabilities because the same codebase otherwise enforces board company membership."
  - "Treat direct-agent-create permission drift as likely rather than confirmed because the human permission model is still partially spec-ambiguous."
patterns-established:
  - "Use `assertCompanyAccess()` consistency as the main proof signal for board and agent tenant isolation."
  - "Separate confirmed cross-company bypasses from permission-model drift."
requirements-completed: [AUTH-02]
duration: 16min
completed: 2026-03-15
---

# Phase 2: Agent Authz Summary

**Mapped the real credential and permission model for agents and surfaced the first confirmed tenant-isolation bug class: company-scoped board mutations that skip `assertCompanyAccess()`.**

## Performance

- **Duration:** 16 min
- **Started:** 2026-03-15T21:52:00Z
- **Completed:** 2026-03-15T22:08:00Z
- **Tasks:** 3
- **Files modified:** 1

## Accomplishments

- Documented the two real agent credential paths: hashed API keys and local runtime JWTs.
- Mapped the membership and permission-grant model used by `accessService()` and route helpers.
- Confirmed that approval, budget, and activity mutations bypass company-scope checks for scoped board users.
- Flagged weaker permission-model routes around direct agent creation and permission editing as likely gaps, not overclaimed vulnerabilities.

## Task Commits

This docs-only plan is included in the consolidated Phase 2 documentation commit.

## Files Created/Modified

- `.planning/phases/02-identity-auth-tenant-isolation/02-AGENT-AUTHZ-REVIEW.md` - Agent credential, permission, and company-scope review

## Decisions Made

- Used the implemented company-membership model, not the broadest product language, as the tenant-isolation contract for this phase.
- Kept findings focused on routes where the missing guard is directly visible in current code.

## Deviations from Plan

- None

## Issues Encountered

- Human permission behavior is only partially rolled out in the server, so some route drift could not be classified beyond “likely” without broader product decisions.

## User Setup Required

- None
