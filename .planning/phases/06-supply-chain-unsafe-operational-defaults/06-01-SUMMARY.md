---
phase: 06-supply-chain-unsafe-operational-defaults
plan: "01"
subsystem: security-docs
tags: [onboarding, doctor, worktree, docker, release-defaults]
requires: []
provides:
  - Operational defaults review
  - Likely finding on HTTP public deployment posture
affects: [phase-06, phase-07]
tech-stack:
  added: []
  patterns: [default-path audit, operator-workflow review]
key-files:
  created:
    - .planning/phases/06-supply-chain-unsafe-operational-defaults/06-OPERATIONAL-DEFAULTS-REVIEW.md
  modified: []
key-decisions:
  - "Treat public-over-HTTP authenticated mode as the main operational default issue because the docs promise stricter doctor failures while the implementation only warns."
  - "Keep worktree cloning and quickstart secret defaults in accepted-risk territory because they remain operator-authored convenience paths, even though they amplify earlier Phase 5 risks."
patterns-established:
  - "Separate internet-facing deployment defaults from clearly local-only bootstrap convenience."
  - "Judge operator workflows by what run and doctor actually block, not by documentation wording alone."
requirements-completed: [SUPP-01]
duration: 18min
completed: 2026-03-16
---

# Phase 6: Operational Defaults Summary

**Confirmed that the release-train entrypoints are relatively strict, but the normal operator-facing setup path still permits internet-facing authenticated deployments over plain HTTP and preserves convenience-first secret defaults unless the operator hardens them manually.**

## Performance

- **Duration:** 18 min
- **Started:** 2026-03-16T18:20:00Z
- **Completed:** 2026-03-16T18:38:00Z
- **Tasks:** 3
- **Files modified:** 1

## Accomplishments

- Traced the full first-run and routine operator path across onboarding, `run`, `doctor`, worktree init, Docker smoke, and release-entrypoint scripts.
- Confirmed the main operational finding: authenticated public mode accepts `http://` URLs, `doctor` only warns, and `run` still starts the server despite the deployment-mode contract promising stricter doctor failures.
- Documented local positive controls such as loopback enforcement for `local_trusted`, automatic JWT and secrets-key creation, and branch- or clean-tree guardrails on release entrypoints.
- Classified worktree seeding and quickstart secret defaults as accepted-risk local sharp edges that still amplify the already-confirmed Phase 5 data-exposure issues.

## Task Commits

This plan is included in the consolidated Phase 6 completion commit.

## Files Created/Modified

- `.planning/phases/06-supply-chain-unsafe-operational-defaults/06-OPERATIONAL-DEFAULTS-REVIEW.md` - Operational-defaults and operator-workflow review

## Decisions Made

- Treated public-mode HTTP tolerance as the real shipped default problem because it can escape the local trust model and contradicts the documented doctor contract.
- Kept worktree and Docker smoke duplication behavior out of confirmed-vulnerability territory because those paths remain explicit local or smoke harness workflows.

## Deviations from Plan

- None

## Issues Encountered

- The same code paths mix genuine safeguards with convenience-first behavior, so the artifact had to separate "warn-only" enforcement from "hard-fail" enforcement before severity was defensible.

## User Setup Required

- None
