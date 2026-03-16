# Requirements: Paperclip Security Audit

**Defined:** 2026-03-15
**Core Value:** Produce a credible, exploit-focused inventory of Paperclip's real security risks so the most dangerous paths can be prioritized and remediated before they become incidents.

## v1 Requirements

### Threat Model

- [ ] **THRT-01**: Maintainer can read a documented threat model covering board users, agents, plugins, local operators, deployment modes, and trust boundaries
- [ ] **THRT-02**: Maintainer can see all meaningful entry points and attack surfaces mapped to concrete files, routes, commands, and runtime flows

### Authentication & Access Control

- [ ] **AUTH-01**: Maintainer can verify whether board authentication and session handling include unsafe defaults, bypasses, or deployment-mode weaknesses
- [ ] **AUTH-02**: Maintainer can verify whether agent authentication, company scoping, and permission checks prevent cross-company access and privilege escalation
- [ ] **AUTH-03**: Maintainer can verify whether realtime and non-REST surfaces enforce the same access guarantees as the main API

### Execution & Runtime Safety

- [ ] **EXEC-01**: Maintainer can see every path where request, database, plugin, or repo-controlled input reaches process execution or command construction
- [ ] **EXEC-02**: Maintainer can see unsafe workspace, worktree, filesystem, or runtime-service behavior that could escape intended boundaries or mutate host state unexpectedly
- [ ] **EXEC-03**: Maintainer can see whether environment propagation, session handling, and logs can leak secrets or security-relevant context into child processes or artifacts

### Plugins & Extension Surfaces

- [ ] **PLUG-01**: Maintainer can verify whether plugin install and activation flows create untrusted code-execution or privilege-expansion risk
- [ ] **PLUG-02**: Maintainer can verify whether plugin tools, jobs, webhooks, UI routes, and worker RPC paths preserve company and principal boundaries

### Secrets, Data Exposure & Storage

- [ ] **DATA-01**: Maintainer can verify whether secret storage, config defaults, backup flows, and redaction logic prevent sensitive data exposure
- [ ] **DATA-02**: Maintainer can verify whether attachments, documents, assets, run logs, and storage providers expose tenant data or unsafe file content

### Supply Chain & Operational Safety

- [ ] **SUPP-01**: Maintainer can verify whether onboarding, CI, release, worktree, and package-management flows can ship insecure defaults or weaken trust boundaries

### Reporting

- [ ] **REPT-01**: Maintainer can read a prioritized findings catalog with severity, exploit conditions, affected files, and recommended remediation steps
- [ ] **REPT-02**: Maintainer can distinguish confirmed vulnerabilities from accepted-risk sharp edges and intentional `local_trusted` behavior

## v2 Requirements

### Dynamic Validation

- **DYNM-01**: Maintainer can run targeted exploit reproductions or proof-of-concept checks for the highest-severity findings
- **DYNM-02**: Maintainer can run runtime fuzzing or hostile-input harnesses against the riskiest boundaries

### Dependency Intelligence

- **DEPS-01**: Maintainer can correlate third-party dependency advisories with actual Paperclip code paths and runtime exposure
- **DEPS-02**: Maintainer can generate an SBOM-oriented review artifact for release hardening

## Out of Scope

| Feature | Reason |
|---------|--------|
| Live internet-facing pentest of a deployed Paperclip environment | This project is a repo/code audit, not an infrastructure engagement |
| Exhaustive CVE triage for every transitive dependency | Too noisy without tying advisories to real repo usage and exploitability |
| Product feature development unrelated to risk reduction | This project exists to find and plan fixes for unsafe behavior |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| THRT-01 | Phase 1 | Pending |
| THRT-02 | Phase 1 | Pending |
| AUTH-01 | Phase 2 | Pending |
| AUTH-02 | Phase 2 | Pending |
| AUTH-03 | Phase 2 | Pending |
| EXEC-01 | Phase 3 | Pending |
| EXEC-02 | Phase 3 | Pending |
| EXEC-03 | Phase 3 | Pending |
| PLUG-01 | Phase 4 | Pending |
| PLUG-02 | Phase 4 | Pending |
| DATA-01 | Phase 5 | Pending |
| DATA-02 | Phase 5 | Pending |
| SUPP-01 | Phase 6 | Pending |
| REPT-01 | Phase 7 | Pending |
| REPT-02 | Phase 7 | Pending |

**Coverage:**
- v1 requirements: 15 total
- Mapped to phases: 15
- Unmapped: 0 ✓

---
*Requirements defined: 2026-03-15*
*Last updated: 2026-03-15 after roadmap creation*
