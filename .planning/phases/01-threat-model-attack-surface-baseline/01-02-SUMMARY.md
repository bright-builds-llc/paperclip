---
phase: 01-threat-model-attack-surface-baseline
plan: "02"
subsystem: security-docs
tags: [attack-surface, routes, plugins, cli, runtime]
requires: []
provides:
  - File-grounded attack-surface inventory
  - Prioritized hotspot list for later exploit review
affects: [phase-02, phase-03, phase-04, phase-05, phase-06]
tech-stack:
  added: []
  patterns: [entry-point inventory, hotspot-first review mapping]
key-files:
  created:
    - .planning/phases/01-threat-model-attack-surface-baseline/01-ATTACK-SURFACE.md
  modified: []
key-decisions:
  - "Treat CLI, worktree, plugin, and release operations as first-class attack surface, not implementation details."
  - "Inventory public, semi-public, and operator-only routes separately so later phases can prioritize review."
patterns-established:
  - "Every attack-surface bucket maps to specific routes, commands, services, packages, or workflows."
  - "Later phases should start from hotspot files rather than route families alone."
requirements-completed: [THRT-02]
duration: 10min
completed: 2026-03-15
---

# Phase 1: Attack Surface Summary

**Mapped Paperclip’s meaningful entry points across REST, WebSocket, CLI, heartbeat/adapters, workspaces, plugins, secrets/storage, and release operations into one audit-ready inventory.**

## Performance

- **Duration:** 10 min
- **Started:** 2026-03-16T02:22:25Z
- **Completed:** 2026-03-16T02:32:00Z
- **Tasks:** 3
- **Files modified:** 1

## Accomplishments

- Cataloged the server/network surfaces that matter for auth, membership, invite, and plugin review.
- Elevated non-HTTP surfaces such as worktree creation, heartbeat execution, plugin workers, and release automation into the formal audit scope.
- Produced a hotspot list that concentrates later phases on the most security-relevant files first.

## Task Commits

This docs-only plan was executed as one consolidated implementation commit because all three tasks built a single artifact.

1. **Consolidated plan implementation** - `c8d5b4d` (feat)

## Files Created/Modified

- `.planning/phases/01-threat-model-attack-surface-baseline/01-ATTACK-SURFACE.md` - Canonical attack-surface inventory and hotspot map

## Decisions Made

- Included operator tooling and release automation in the attack surface because they directly alter trust boundaries and deployment state.
- Kept unauthenticated and pre-auth flows visible instead of burying them inside general route coverage.

## Deviations from Plan

- Consolidated the three plan tasks into one implementation commit because the work produced a single tightly coupled Markdown artifact. No scope was added or removed.

## Issues Encountered

- None

## User Setup Required

- None
