# Phase 7 Research: Findings Catalog & Remediation Roadmap

## Planning Question

How should the final phase turn the verified evidence from Phases 1 through 6 into a single maintainer-usable findings catalog and remediation roadmap without re-litigating earlier code review, double-counting extended findings, or losing the distinction between real vulnerabilities, accepted local sharp edges, and runtime-validation leads?

## Phase Contract

Phase 7 is the reporting and remediation-synthesis phase of the audit:

- `REPT-01`: maintainers can read a prioritized findings catalog with severity, exploit conditions, affected files, and recommended remediation steps
- `REPT-02`: maintainers can distinguish confirmed vulnerabilities from accepted-risk sharp edges and intentional `local_trusted` behavior

Unlike earlier phases, this phase is not discovering a new subsystem. Its job is to consolidate, de-duplicate, rank, and sequence the evidence already captured in Phases 2 through 6 while staying faithful to the rubric and evidence standards fixed in Phase 1.

## Repo-Grounded Facts That Shape The Plan

- Phase 1 already fixed the final reporting rules:
  - `01-AUDIT-METHODOLOGY.md` defines the severity rubric, exploitability factors, accepted-risk rules, and the exact allowed finding states
  - `01-FINDINGS-LOG.md` defines the stable per-finding fields the final catalog should preserve
- Phase 2 produced the audit's core auth and tenant-isolation findings:
  - confirmed high-severity cross-company board mutation routes that skip `assertCompanyAccess()`
  - confirmed medium-severity websocket private-hostname drift
  - confirmed medium-severity unauthenticated run-to-issues metadata access
  - a likely medium-severity authz-drift finding around direct agent creation and permission updates
  - a runtime-validation lead for weaker query-string websocket token handling
- Phase 3 and Phase 5 form one larger run-artifact exposure family:
  - Phase 3 confirmed that same-company agents can read peer heartbeat run details, events, and logs
  - Phase 5 extended that same issue into persisted issue-history, activity, and other artifact read surfaces
  - the final catalog should not present these as unrelated bugs; it should show one primary finding with multiple confirmed surfaces and one root remediation theme
- Phase 4 added distinct plugin-boundary issues:
  - confirmed instance-wide plugin management by non-instance-admin board users
  - confirmed cross-company worker actions through the plugin UI bridge
  - a likely unsafe-by-default public webhook ingress finding that depends on plugin authors implementing their own request verification
- Phase 5 added the clearest user-triggered content and secret-handling risks:
  - confirmed high-severity stored same-origin HTML execution through assets and attachments
  - likely medium-severity plaintext-compatible secret persistence and backup copying when strict mode remains off
  - accepted-risk local worktree duplication and plugin-log hardening concerns that should remain visible but not be promoted to the same class as the confirmed issues
- Phase 6 added deployment and release-surface integrity findings:
  - likely medium-severity public HTTP tolerance in authenticated public mode
  - likely medium-severity dependency-graph drift between PR verification and release verification
  - likely low-severity CLI publish-manifest drift from real adapter imports
  - accepted-risk and no-finding release-train notes that must remain visible so future hardening does not weaken the existing trusted-publishing guardrails
- The final phase must cluster related accepted-risk items as well:
  - Phase 3 worktree seeding and host-mutation sharp edges
  - Phase 5 worktree secret duplication and plugin-log persistence
  - Phase 6 worktree cloning defaults, quickstart convenience defaults, and release-time artifact copying
- The repo already contains the evidence needed for a strong final report:
  - every executed phase has a verification report
  - the strongest findings also have review artifacts with concrete routes, helpers, and file anchors
  - Phase 7 should cite those artifacts directly instead of reopening broad source-code discovery unless a file anchor is missing

## Review Tracks The Plan Should Cover

### 1. Findings Normalization And De-Duplication

This track should answer:

- which phase-local findings become distinct final catalog entries
- which later findings are extensions of the same earlier root problem
- which accepted-risk and no-finding items are important enough to preserve in the final report

Primary inputs:

- `01-AUDIT-METHODOLOGY.md`
- `01-FINDINGS-LOG.md`
- `02-VERIFICATION.md` through `06-VERIFICATION.md`
- the highest-signal review artifacts from Phases 2 through 6

### 2. Remediation Prioritization

This track should answer:

- which issues are most urgent by severity, exploitability, and blast radius
- which fixes collapse multiple findings at once because they share a root cause
- which items are immediate hotfixes versus structural hardening or deferred validation work

Primary inputs:

- the final findings catalog from Track 1
- the Phase 1 severity rubric
- the carry-forward questions and residual risks already recorded in phase verification reports

### 3. Maintainer Execution Roadmap

This track should answer:

- what order the maintainer should fix the findings in
- which target files, regression tests, and acceptance signals belong to each workstream
- which accepted-risk and runtime-validation items should remain deferred but visible

Primary inputs:

- the findings catalog
- the remediation-priority matrix
- the earlier phase verification reports and concrete review artifacts

## Concrete Questions Worth Resolving

1. Which findings should collapse into one final entry because they describe the same root boundary failure across more than one surface?
2. Which likely vulnerabilities already have enough static evidence to stay likely in the final report, and which should instead be grouped under broader hardening themes or deferred validation?
3. Which remediation themes have the highest leverage across multiple findings, such as:
   - consistent company-scope enforcement
   - secret and artifact redaction policy
   - active-content serving policy
   - plugin runtime privilege boundaries
   - deployment-mode enforcement
   - release and dependency integrity
4. Which fixes are short-path and realistic for immediate follow-up, and which require broader contract or product-model decisions?
5. Which accepted-risk or no-finding items must remain documented so maintainers do not accidentally remove useful guardrails while hardening nearby code?
6. Which regression tests or verification hooks should each prioritized fix add so the same class of issue does not recur?

## Existing Artifacts To Reuse

- `.planning/phases/01-threat-model-attack-surface-baseline/01-AUDIT-METHODOLOGY.md`
- `.planning/phases/01-threat-model-attack-surface-baseline/01-FINDINGS-LOG.md`
- `.planning/phases/02-identity-auth-tenant-isolation/02-VERIFICATION.md`
- `.planning/phases/02-identity-auth-tenant-isolation/02-AGENT-AUTHZ-REVIEW.md`
- `.planning/phases/02-identity-auth-tenant-isolation/02-REALTIME-AUX-AUTHZ-REVIEW.md`
- `.planning/phases/03-process-execution-host-interaction/03-VERIFICATION.md`
- `.planning/phases/03-process-execution-host-interaction/03-ENV-SESSION-LOG-REVIEW.md`
- `.planning/phases/03-process-execution-host-interaction/03-WORKSPACE-HOST-BOUNDARY-REVIEW.md`
- `.planning/phases/04-plugin-extension-boundary-review/04-VERIFICATION.md`
- `.planning/phases/04-plugin-extension-boundary-review/04-INSTALL-LIFECYCLE-REVIEW.md`
- `.planning/phases/04-plugin-extension-boundary-review/04-CAPABILITY-HOST-RPC-REVIEW.md`
- `.planning/phases/04-plugin-extension-boundary-review/04-TOOLS-WEBHOOKS-UI-REVIEW.md`
- `.planning/phases/05-secrets-storage-data-exposure/05-VERIFICATION.md`
- `.planning/phases/05-secrets-storage-data-exposure/05-SECRETS-BACKUP-REVIEW.md`
- `.planning/phases/05-secrets-storage-data-exposure/05-ASSET-DOCUMENT-STORAGE-REVIEW.md`
- `.planning/phases/05-secrets-storage-data-exposure/05-PERSISTED-ARTIFACT-REVIEW.md`
- `.planning/phases/06-supply-chain-unsafe-operational-defaults/06-VERIFICATION.md`
- `.planning/phases/06-supply-chain-unsafe-operational-defaults/06-OPERATIONAL-DEFAULTS-REVIEW.md`
- `.planning/phases/06-supply-chain-unsafe-operational-defaults/06-PACKAGE-BUILD-PUBLISH-REVIEW.md`

## Recommended Execution Outputs

Phase 7 should produce three artifacts:

- `07-FINDINGS-CATALOG.md`
  - final normalized finding inventory
  - explicit separation between confirmed, likely, accepted-risk, no-finding, and runtime-validation items
  - cross-phase clustering and provenance
- `07-REMEDIATION-PRIORITY-MATRIX.md`
  - ranked findings or workstreams
  - exploitability and blast-radius rationale
  - root-cause grouping and recommended regression guards
- `07-REMEDIATION-ROADMAP.md`
  - immediate versus near-term workstreams
  - deferred validation and accepted-risk backlog
  - execution order, target files, and follow-on verification expectations

## Recommended Plan Split

Wave 1:

- `07-01`: build the normalized findings catalog and de-duplicate cross-phase findings

Wave 2:

- `07-02`: build the remediation-priority matrix from the normalized catalog

Wave 3:

- `07-03`: turn the ranked matrix into the final maintainer roadmap and execution sequence

This keeps the phase sequential on purpose. Prioritization should depend on a stable final catalog, and the final roadmap should depend on both the catalog and the ranking work rather than re-triaging raw phase artifacts again.

## Verification Expectations For Phase Execution

When Phase 7 is executed, the artifact set should be checked for:

- direct reuse of the fixed Phase 1 finding states and evidence standard
- explicit de-duplication of multi-phase findings, especially heartbeat artifact exposure and local-worktree duplication sharp edges
- per-finding or per-workstream preservation of severity, exploit preconditions, deployment mode reach, affected files, and remediation direction
- clear separation between:
  - confirmed vulnerabilities to fix now
  - likely vulnerabilities that still deserve real engineering action
  - accepted-risk sharp edges to preserve as trust-model notes
  - no-finding controls that should not be regressed
  - runtime-validation items that belong in later dynamic testing
- a remediation matrix that explains not just ranking, but why certain fixes collapse multiple findings at once
- a final roadmap that lets maintainers begin remediation without reopening all earlier phase artifacts
