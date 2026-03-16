---
phase: 05-secrets-storage-data-exposure
plan: "01"
subsystem: security-docs
tags: [secrets, backups, worktrees, config-defaults]
requires: []
provides:
  - Secret provider and backup review
  - Likely finding on plaintext-compatible secret persistence
affects: [phase-05, phase-06, phase-07]
tech-stack:
  added: []
  patterns: [secret-lifecycle tracing, backup-boundary review]
key-files:
  created:
    - .planning/phases/05-secrets-storage-data-exposure/05-SECRETS-BACKUP-REVIEW.md
  modified: []
key-decisions:
  - "Treat the dedicated secret provider system as a positive control while still calling out the default plaintext compatibility path as the real at-rest weakness."
  - "Classify worktree seeding as a trusted local-operator sharp edge rather than a remotely relevant vulnerability."
patterns-established:
  - "Separate secret metadata protection from the wider problem of plaintext-compatible config and backup copying."
  - "Judge backup risk by what the database actually stores, not by the existence of a secret-ref system alone."
requirements-completed: [DATA-01]
duration: 20min
completed: 2026-03-16
---

# Phase 5: Secrets And Backup Summary

**Confirmed that the dedicated secret system is reasonably well-contained, but the default config still leaves secret-at-rest protection partially optional because plaintext env bindings can be persisted and then copied into ordinary backups.**

## Performance

- **Duration:** 20 min
- **Started:** 2026-03-16T17:20:00Z
- **Completed:** 2026-03-16T17:40:00Z
- **Tasks:** 3
- **Files modified:** 1

## Accomplishments

- Traced the full secret model across CRUD routes, database tables, provider selection, master-key handling, and runtime resolution.
- Documented the key positive controls: metadata-only secret reads, AES-256-GCM local encryption, config revision sanitization, and plugin secret-ref scoping.
- Identified the main weakness as the still-supported plaintext env path: strict mode defaults off, plaintext env bindings remain valid, and ordinary DB backups dump those rows verbatim.
- Documented seeded worktree behavior as a powerful but local-operator-only duplication path because the workflow copies the secrets key and restores a full database backup.

## Task Commits

This plan is included in the consolidated Phase 5 completion commit.

## Files Created/Modified

- `.planning/phases/05-secrets-storage-data-exposure/05-SECRETS-BACKUP-REVIEW.md` - Secret provider, backup, and worktree-seeding exposure review

## Decisions Made

- Classified plaintext-compatible agent config persistence as the main actionable Phase 5 secret-storage problem instead of the dedicated `company_secrets` tables.
- Kept worktree seeding in accepted-risk territory because the full-copy behavior is explicit local tooling rather than a remotely reachable surface.

## Deviations from Plan

- None

## Issues Encountered

- The secret story spans clean provider-backed storage and weaker compatibility paths in ordinary agent config, so the artifact had to separate those layers before severity was defensible.

## User Setup Required

- None
