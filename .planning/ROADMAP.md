# Roadmap: Paperclip Security Audit

## Overview

This roadmap turns a broad repo-security review into a sequence of focused audit passes. It starts by fixing the threat model and attack surface, then works trust boundary by trust boundary across auth, execution, plugins, data handling, and operational delivery surfaces before ending in a ranked findings catalog and remediation plan.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Threat Model & Attack Surface Baseline** - Define attacker model, trust boundaries, and the concrete review surface
- [x] **Phase 2: Identity, Auth & Tenant Isolation** - Review board, agent, company-scoping, and realtime access controls
- [x] **Phase 3: Process Execution & Host Interaction** - Review shell/process spawning, workspaces, worktrees, runtime services, and env propagation
- [x] **Phase 4: Plugin & Extension Boundary Review** - Review plugin install, activation, RPC, tools, jobs, webhooks, and UI extension isolation
- [ ] **Phase 5: Secrets, Storage & Data Exposure** - Review sensitive data handling, storage providers, logs, and persistence surfaces
- [ ] **Phase 6: Supply Chain & Unsafe Operational Defaults** - Review onboarding, CI, release, package, and operational trust assumptions
- [ ] **Phase 7: Findings Catalog & Remediation Roadmap** - Consolidate validated findings, accepted-risk notes, and ranked follow-up work

## Phase Details

### Phase 1: Threat Model & Attack Surface Baseline
**Goal**: Create the audit baseline, attacker model, severity rubric, and file-grounded attack-surface inventory
**Depends on**: Nothing (first phase)
**Requirements**: THRT-01, THRT-02
**Success Criteria** (what must be TRUE):
  1. Maintainer can read documented attacker profiles, principals, trust boundaries, and deployment assumptions for Paperclip
  2. Maintainer can see entry points across REST, WebSocket, CLI, adapters, plugins, scripts, and release flows mapped to concrete files
  3. Severity, exploitability, and evidence standards are fixed before deeper review begins
**Plans**: 3 plans

Plans:
- [x] 01-01: Inventory principals, deployment modes, and trust boundaries
- [x] 01-02: Map entry points and high-risk subsystems
- [x] 01-03: Define severity rubric and findings evidence format

### Phase 2: Identity, Auth & Tenant Isolation
**Goal**: Determine whether authentication and authorization boundaries can be bypassed or crossed
**Depends on**: Phase 1
**Requirements**: AUTH-01, AUTH-02, AUTH-03
**Success Criteria** (what must be TRUE):
  1. Maintainer can see whether board session handling and deployment-mode defaults expose unsafe authenticated/public behavior
  2. Maintainer can see whether agent auth, company scoping, and permissions allow cross-company access or privilege escalation
  3. Maintainer can see whether websocket and other non-REST surfaces enforce access guarantees consistent with the main API
**Plans**: 3 plans

Plans:
- [x] 02-01: Review board authentication, session resolution, and deployment defaults
- [x] 02-02: Review agent keys, JWTs, membership checks, and company scoping
- [x] 02-03: Review websocket/realtime and auxiliary route authorization parity

### Phase 3: Process Execution & Host Interaction
**Goal**: Determine whether runtime execution paths can lead to arbitrary code execution, host mutation, or secret leakage
**Depends on**: Phase 2
**Requirements**: EXEC-01, EXEC-02, EXEC-03
**Success Criteria** (what must be TRUE):
  1. Maintainer can see every child-process, adapter, and command-construction path with its controlling inputs and safety controls
  2. Maintainer can see any workspace, worktree, filesystem, or runtime-service path that can escape intended boundaries or mutate host state unexpectedly
  3. Maintainer can see whether env propagation, session metadata, and logs can leak secrets or sensitive runtime context
**Plans**: 4 plans

Plans:
- [x] 03-01: Trace heartbeat and adapter process-execution paths
- [x] 03-02: Review workspaces, worktrees, and runtime-service filesystem boundaries
- [x] 03-03: Review env propagation, session artifacts, and log exposure
- [x] 03-04: Separate authenticated/public risk from expected local-only sharp edges

### Phase 4: Plugin & Extension Boundary Review
**Goal**: Determine whether plugin surfaces expand privilege or cross tenant/principal boundaries unsafely
**Depends on**: Phase 3
**Requirements**: PLUG-01, PLUG-02
**Success Criteria** (what must be TRUE):
  1. Maintainer can see whether plugin install and activation create untrusted code-execution or privilege-expansion risk
  2. Maintainer can see whether plugin RPC, tools, jobs, webhooks, and UI routes preserve company and principal isolation
  3. Maintainer can see whether capability gating matches what the runtime actually exposes
**Plans**: 3 plans

Plans:
- [x] 04-01: Review plugin install, activation, and worker lifecycle boundaries
- [x] 04-02: Review capability gating, host RPC handlers, and privilege surfaces
- [x] 04-03: Review plugin tools, jobs, webhooks, and UI routing isolation

### Phase 5: Secrets, Storage & Data Exposure
**Goal**: Determine whether Paperclip can expose secrets or tenant data through persistence and artifact handling
**Depends on**: Phase 4
**Requirements**: DATA-01, DATA-02
**Success Criteria** (what must be TRUE):
  1. Maintainer can see whether secret storage, key defaults, backups, and redaction logic prevent sensitive-data exposure
  2. Maintainer can see whether assets, attachments, documents, storage providers, and run logs can leak tenant data or unsafe file content
  3. Maintainer can see where sensitive data is persisted, copied, or surfaced back to operators and agents
**Plans**: 3 plans

Plans:
- [ ] 05-01: Review secrets providers, config defaults, redaction, and backup behavior
- [ ] 05-02: Review assets, attachments, documents, and storage-provider boundaries
- [ ] 05-03: Review run logs and other persisted operational artifacts

### Phase 6: Supply Chain & Unsafe Operational Defaults
**Goal**: Determine whether build, onboarding, and release operations can ship insecure states or erode trust boundaries
**Depends on**: Phase 5
**Requirements**: SUPP-01
**Success Criteria** (what must be TRUE):
  1. Maintainer can see whether onboarding, doctor, worktree, CI, and release flows introduce unsafe defaults or bypasses
  2. Maintainer can see whether packaging and package-management assumptions create supply-chain or integrity risk
**Plans**: 2 plans

Plans:
- [ ] 06-01: Review onboarding, doctor, worktree, CI, and release trust assumptions
- [ ] 06-02: Review package-management, build, and publish-chain unsafe defaults

### Phase 7: Findings Catalog & Remediation Roadmap
**Goal**: Turn the audit evidence into a maintainer-usable findings catalog and prioritized fix plan
**Depends on**: Phase 6
**Requirements**: REPT-01, REPT-02
**Success Criteria** (what must be TRUE):
  1. Maintainer can read a ranked findings catalog with severity, exploit preconditions, affected files, and concrete remediation guidance
  2. Maintainer can distinguish confirmed vulnerabilities from accepted-risk sharp edges and intentional `local_trusted` behavior
  3. Maintainer can start remediation work without repeating discovery or re-triaging the audit evidence
**Plans**: 3 plans

Plans:
- [ ] 07-01: Consolidate validated findings and accepted-risk notes
- [ ] 07-02: Rank remediations by severity, exploitability, and blast radius
- [ ] 07-03: Publish final findings catalog and follow-on fix roadmap

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6 → 7

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Threat Model & Attack Surface Baseline | 3/3 | Complete | 2026-03-15 |
| 2. Identity, Auth & Tenant Isolation | 3/3 | Complete | 2026-03-15 |
| 3. Process Execution & Host Interaction | 4/4 | Complete | 2026-03-15 |
| 4. Plugin & Extension Boundary Review | 3/3 | Complete | 2026-03-16 |
| 5. Secrets, Storage & Data Exposure | 0/3 | Not started | - |
| 6. Supply Chain & Unsafe Operational Defaults | 0/2 | Not started | - |
| 7. Findings Catalog & Remediation Roadmap | 0/3 | Not started | - |
