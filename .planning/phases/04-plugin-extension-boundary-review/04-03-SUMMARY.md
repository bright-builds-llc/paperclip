---
phase: 04-plugin-extension-boundary-review
plan: "03"
subsystem: security-docs
tags: [plugins, tools, jobs, webhooks, ui]
requires: ["04-01", "04-02"]
provides:
  - Tool, job, webhook, and UI isolation review
  - Likely-finding classification for unsafe-by-default webhook ingress
affects: [phase-04, phase-05, phase-07]
tech-stack:
  added: []
  patterns: [public-ingress review, same-origin UI boundary analysis]
key-files:
  created:
    - .planning/phases/04-plugin-extension-boundary-review/04-TOOLS-WEBHOOKS-UI-REVIEW.md
  modified: []
key-decisions:
  - "Keep tool execution separate from the more general bridge path because the tool route preserves company scope more explicitly."
  - "Classify webhook ingress as likely rather than confirmed because the host is unsafe-by-default, but concrete exploitability still depends on plugin handler behavior."
patterns-established:
  - "Review public webhook ingress and same-origin UI as separate trust boundaries, not as extensions of capability-gated worker RPC."
  - "Keep documented trusted-plugin UI assumptions visible even when they are not an immediate hidden vulnerability."
requirements-completed: [PLUG-02]
duration: 17min
completed: 2026-03-16
---

# Phase 4: Tools, Webhooks, And UI Summary

**Finished the plugin surface review by separating the relatively well-scoped tool path from the riskier webhook and same-origin UI paths, and classified webhook ingress as the main unsafe-by-default design lead.**

## Performance

- **Duration:** 17 min
- **Started:** 2026-03-16T15:52:00Z
- **Completed:** 2026-03-16T16:09:00Z
- **Tasks:** 3
- **Files modified:** 1

## Accomplishments

- Documented the board-scoped tool route, namespaced tool registry, and worker-running checks.
- Reviewed scheduler, job-run, and manual-trigger behavior under the current instance-wide plugin model.
- Classified public webhook delivery as an unsafe-by-default ingress path because the host delegates all signature verification to plugin code while still exposing privileged worker authority.
- Documented the real UI runtime: same-origin JavaScript, shared host React runtime, blob-based module loading, and localhost-only dev proxy protections.

## Task Commits

This plan is included in the consolidated Phase 4 completion commit.

## Files Created/Modified

- `.planning/phases/04-plugin-extension-boundary-review/04-TOOLS-WEBHOOKS-UI-REVIEW.md` - Tool, job, webhook, static-bundle, and UI-boundary review

## Decisions Made

- Kept the tool HTTP path in no-finding territory because it carries an explicit company-scoped `runContext`.
- Treated same-origin UI as an accepted sharp edge because the repo documents it clearly, even though it sharply limits the meaning of manifest capabilities.

## Deviations from Plan

- None

## Issues Encountered

- The same-origin UI model is both documented and risky, so the write-up had to keep that assumption visible without double-counting it as a hidden bug after the bridge-scope finding was already confirmed in 04-02.

## User Setup Required

- None
