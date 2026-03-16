# Paperclip Security Audit

## What This Is

This project is a brownfield security audit of the existing Paperclip monorepo. It is focused on finding and cataloging exploitable vulnerabilities, unsafe operations, broken trust boundaries, and dangerous defaults across the server, UI, CLI, adapters, plugins, storage, and release tooling so maintainers can fix the highest-risk issues with confidence.

## Core Value

Produce a credible, exploit-focused inventory of Paperclip's real security risks so the most dangerous paths can be prioritized and remediated before they become incidents.

## Requirements

### Validated

- ✓ Paperclip already ships a multi-company AI control plane spanning server, UI, CLI, shared contracts, and a PostgreSQL-backed data model — existing
- ✓ Paperclip already supports both `local_trusted` and `authenticated` deployment modes with separate board and agent access paths — existing
- ✓ Paperclip already executes local agents, manages workspaces/worktrees, and hosts a plugin runtime with jobs, tools, UI slots, and webhooks — existing
- ✓ Paperclip already includes operational surfaces beyond the app core, including onboarding, invites, secrets/storage providers, realtime events, CI workflows, and release scripts — existing

### Active

- [ ] Produce a repo-wide threat model and attack-surface map covering HTTP, WebSocket, CLI, agent, plugin, storage, workspace, and release surfaces
- [ ] Audit authentication, authorization, and company-isolation behavior for exploitable bypasses, privilege escalation, and unsafe defaults
- [ ] Audit process execution, plugin loading, workspace/runtime services, and filesystem access for arbitrary code execution, sandbox escape, or privilege-boundary failures
- [ ] Audit secret handling, storage, redaction, config defaults, and data exposure paths for leakage or insecure persistence
- [ ] Catalog findings with severity, exploit preconditions, affected files, reasoning or proof, and recommended remediations
- [ ] Distinguish true vulnerabilities from operational sharp edges, expected `local_trusted` behavior, and intentional product tradeoffs

### Out of Scope

- Live penetration testing against an already deployed Paperclip instance — this project is repo/code audit first
- Exhaustive CVE triage for every dependency regardless of how the code actually uses it — dependency issues only matter here when the repo’s usage makes them relevant
- General feature development or product redesign — this project is about risk discovery, verification, and remediation planning

## Context

Paperclip is a TypeScript monorepo with `server/`, `ui/`, `cli/`, `packages/db/`, `packages/shared/`, multiple adapter packages, and a first-party plugin SDK/runtime. The repo already has a fresh `.planning/codebase/` map describing the current stack, architecture, testing patterns, and major concerns.

The highest-risk areas are not generic CRUD endpoints. They are the trust-boundary crossings: board vs agent auth, company scoping, `local_trusted` vs `authenticated` deployment behavior, local process spawning for agent adapters, plugin install/activation, workspace/worktree realization, secrets handling, release/CI automation, and any code path that converts repo or network input into filesystem, shell, or runtime behavior.

This audit is for maintainers/operators of Paperclip, not end users. “Done” is not “we read a lot of files.” Done is a file-grounded catalog of meaningful findings and unsafe operations that can drive remediation work without redoing discovery from scratch.

## Constraints

- **Brownfield**: The repo already has substantial existing behavior and extension surfaces — findings must respect actual implementation, not imagined architecture
- **Evidence**: Every finding must point to concrete files, trust boundaries, and exploit or failure reasoning — generic security advice is not enough
- **Scope**: Review code, configs, scripts, CI/release flows, and repo-documented runtime behavior — avoid speculating about external infrastructure not represented here
- **Prioritization**: Favor exploitability and blast radius over volume — the goal is useful risk ranking, not maximal issue count
- **Verification**: Where possible, claims should be backed by code path inspection, tests, or reproducible reasoning rather than intuition

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Audit the existing repo as a brownfield system | Paperclip already has significant runtime and extension complexity; security review must start from real code | — Pending |
| Keep scope repo-focused instead of external-infra-focused | The user asked for a deep audit of this repo specifically | — Pending |
| Skip a separate ecosystem research phase | This project is driven more by concrete code review than by generic domain research | — Pending |
| Prioritize auth, execution, plugin, secret, and release trust boundaries | Those are the most likely exploit-bearing surfaces in the current architecture | — Pending |

---
*Last updated: 2026-03-15 after initialization*
