# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-03-15)

**Core value:** Produce a credible, exploit-focused inventory of Paperclip's real security risks so the most dangerous paths can be prioritized and remediated before they become incidents.
**Current focus:** Phase 7 — Findings Catalog & Remediation Roadmap

## Current Position

Phase: 7 of 7 (Findings Catalog & Remediation Roadmap)
Plan: 3 of 3 in current phase
Status: Phase 7 planned and locally verified; ready to execute
Last activity: 2026-03-16 — Created Phase 7 research and executable plan docs
Progress: [█████████░] 86%

## Performance Metrics

**Velocity:**
- Total plans completed: 18
- Average duration: 15 min
- Total execution time: 4.7 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| 1. Threat Model & Attack Surface Baseline | 3 | 27 min | 9 min |
| 2. Identity, Auth & Tenant Isolation | 3 | 42 min | 14 min |
| 3. Process Execution & Host Interaction | 4 | 65 min | 16 min |
| 4. Plugin & Extension Boundary Review | 3 | 59 min | 20 min |
| 5. Secrets, Storage & Data Exposure | 3 | 52 min | 17 min |
| 6. Supply Chain & Unsafe Operational Defaults | 2 | 34 min | 17 min |

**Recent Trend:**
- Last 5 plans: 20 min, 17 min, 15 min, 18 min, 16 min
- Trend: Stable in the mid-teens even as the work shifted from repo-internal trust boundaries into operator and release-surface analysis

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Phase 0: Treat this as a brownfield repo-security audit, not a feature-build project
- Phase 0: Skip generic ecosystem research and prioritize concrete repo evidence
- Phase 0: Focus first on auth, execution, plugin, data, and release trust boundaries
- Phase 1: Treat deployment mode as part of the threat model, not background setup detail
- Phase 1: Keep CLI/operator and release automation in the attack-surface inventory alongside server routes
- Phase 1: Use explicit finding states to separate confirmed vulnerabilities, likely vulnerabilities, accepted-risk sharp edges, and runtime-validation leads
- Phase 2: Use the implemented company-membership model, not broad product language alone, as the tenant-isolation contract for route review
- Phase 2: Treat missing `assertCompanyAccess()` checks on company-scoped board mutations as confirmed vulnerabilities
- Phase 2: Treat websocket and auxiliary-route auth as separate trust boundaries, not automatic extensions of the HTTP path

### Pending Todos

- Confirmed Phase 2 findings remain open for cross-company board mutations, websocket private-hostname drift, and unauthenticated run-to-issues metadata access
- Confirmed Phase 3 and Phase 5 review now show same-company heartbeat artifact exposure also reaches persisted issue-history and activity read surfaces
- Confirmed Phase 4 findings remain open for instance-wide plugin management by non-instance-admin board users and cross-company worker actions through the plugin UI bridge
- Confirmed Phase 5 finding remains open for stored same-origin HTML execution through asset and attachment content routes
- Likely Phase 5 finding remains open for plaintext-compatible secret persistence and backup copying when strict mode remains off
- Likely Phase 6 finding now exists for authenticated public deployments starting over plain HTTP with only a warning gate
- Likely Phase 6 finding now exists for PR verification and release verification using different dependency graphs
- Likely low-severity Phase 6 finding now exists for CLI publish-manifest drift from the imported adapter set
- Phase 7 now has three executable plans covering the final findings catalog, remediation-priority matrix, and maintainer remediation roadmap

### Blockers/Concerns

- Default attachment and asset policy still allows user-controlled HTML to be served inline from the Paperclip origin
- Secret-at-rest posture still depends on operator discipline unless strict mode is enabled and plaintext-compatible bindings are removed
- Artifact redaction remains inconsistent across heartbeat, activity, and plugin log surfaces
- Authenticated public deployments still do not hard-fail missing HTTPS in the documented public mode
- Reviewed and released dependency graphs can diverge under the current lockfile policy
- Publish artifact generation still depends on hard-coded workspace lists and release-time file copying

## Session Continuity

Last session: 2026-03-16 07:06 CDT
Stopped at: Phase 7 planned and ready to execute
Resume file: None
