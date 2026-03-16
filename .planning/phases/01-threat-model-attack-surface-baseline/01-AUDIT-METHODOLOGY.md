# Phase 1 Audit Methodology

This methodology fixes how later phases will classify findings, what evidence they must collect, and when a risky behavior is an accepted local-trusted sharp edge versus a real vulnerability.

## Severity Rubric

Use the highest applicable severity after considering exploit preconditions, blast radius, and deployment-mode reach.

| Severity | Use when | Typical Paperclip examples |
|---|---|---|
| Critical | Unauthenticated or low-friction attacker can reach deployment-wide control, cross-company data, arbitrary code execution, or secret exfiltration with limited preconditions | board auth bypass in authenticated/public mode, plugin/heartbeat path that becomes near-arbitrary host execution from attacker-controlled input |
| High | Authenticated or semi-trusted actor can cross company boundaries, escalate to broader authority, or expose sensitive data/material with meaningful but realistic preconditions | agent cross-company access, plugin bridge capability bypass, filesystem/worktree escape requiring authenticated setup |
| Medium | Exploit requires stronger foothold, specific deployment posture, or chained conditions, but impact is still meaningful | local operator or plugin author can access more than documented; secrets or logs leak in a constrained but real way |
| Low | Security-relevant weakness with limited exploitability, narrow blast radius, or mostly defense-in-depth impact | weak default warnings, incomplete logging/audit attribution, partial validation gaps that do not yet create direct compromise |
| Informational | No direct vulnerability established, but the repo contains a sharp edge, trust assumption, or review note that must remain visible | intentional `local_trusted` convenience behavior that is only safe inside its documented boundary |

### Severity Overrides

- Escalate one level if the issue crosses company boundaries.
- Escalate one level if the issue affects both `authenticated/private` and `authenticated/public`.
- De-escalate only when the only affected mode is clearly documented `local_trusted` behavior and the boundary does not escape that model.
- Never de-escalate a finding purely because the product is “single-tenant”; the repo still enforces multi-company boundaries.

## Exploitability Factors

Later phases should record these factors explicitly for every non-informational lead:

1. Attacker starting position
   - unauthenticated client
   - authenticated board user
   - agent principal
   - plugin/package author
   - local operator
   - release actor
2. Deployment mode reach
   - `local_trusted`
   - `authenticated/private`
   - `authenticated/public`
3. Entry point
   - REST
   - WebSocket
   - CLI/operator flow
   - heartbeat/adapter execution
   - workspace/worktree/runtime service
   - plugin worker/UI/webhook/tool path
   - storage/secret/release path
4. Boundary crossed
   - auth bypass
   - company isolation
   - host execution
   - filesystem containment
   - secret/data exposure
   - supply-chain integrity
5. Preconditions
   - required membership/role
   - required config/deployment mode
   - required plugin/package install or local filesystem access
   - required user interaction or board approval
6. Impact
   - control gained
   - data exposed
   - persistence gained
   - blast radius across companies, runs, or host state

## Evidence Standard

A finding is only “confirmed” when the review artifact captures all of the following:

- affected principal or attacker position
- exact entry point or trigger path
- trust boundary crossed
- affected deployment mode(s)
- concrete file anchors
- proof signal
  - direct code path
  - failing test
  - reproducible reasoning chain
  - runtime reproduction when required
- exploit preconditions
- impact statement in Paperclip terms
- remediation direction

Evidence is insufficient when it only says “this looks unsafe” without naming the concrete file path, deployment mode, or crossed boundary.

### Minimum File Anchoring Rule

Each finding must identify:

- one primary source file where the risky behavior lives
- one secondary file if authorization, config, or downstream impact depends on another layer

Examples:

- `server/src/middleware/auth.ts` + `server/src/routes/authz.ts`
- `server/src/services/heartbeat.ts` + `packages/adapters/codex-local/src/server/execute.ts`
- `server/src/routes/plugins.ts` + `server/src/services/plugin-loader.ts`

## Finding States

Use exactly one state per reviewed item.

| State | Meaning | When to use it |
|---|---|---|
| Confirmed vulnerability | Repo evidence shows a real boundary failure or unsafe operation with enough detail to act on | Default state for validated issues |
| Likely vulnerability | Strong code evidence exists, but a runtime or cross-file assumption still needs confirmation | Use sparingly and record what remains to prove |
| Accepted-risk sharp edge | Risky behavior is intentional, documented, and remains inside the declared trust model | Most common for clearly local-only convenience behavior |
| No finding | The reviewed path appears to enforce the expected guarantee | Record only when the path was a hotspot or requirement-critical |
| Needs runtime validation | Static review found a plausible issue but cannot settle it without running a targeted check | Use for future dynamic validation work, not as a final severity label |

## Accepted Risk Rules

A risky behavior can be logged as an accepted-risk sharp edge only if all of the following are true:

1. The behavior is consistent with `doc/DEPLOYMENT-MODES.md` or another explicit project contract.
2. The effect stays inside the documented trust boundary for that mode.
3. It does not cross companies, reveal secrets, or expose host mutation beyond the stated local operator assumption.
4. The same path is not reachable in a stronger deployment posture without equivalent guardrails.

Do not use accepted-risk language for:

- hardcoded or fallback secrets that survive into authenticated/public use
- cross-company authorization gaps
- unauthenticated invite/bootstrap abuse
- plugin, webhook, or heartbeat paths that let external or semi-trusted input reach host execution unexpectedly

## Logging Format

Later phases should append findings to `01-FINDINGS-LOG.md` using a stable schema.

Required fields:

- `ID`
- `Status`
- `Severity`
- `Principal`
- `Boundary`
- `Entry Point`
- `Deployment Mode`
- `Preconditions`
- `Affected Files`
- `Evidence`
- `Remediation`

### Triage Workflow

1. Start from a hotspot or requirement-specific review target.
2. Determine the attacker position and deployment mode.
3. Trace the boundary and file anchors.
4. Decide whether the behavior is:
   - confirmed vulnerability
   - likely vulnerability
   - accepted-risk sharp edge
   - no finding
   - needs runtime validation
5. Assign severity only after impact and exploitability are both written down.
6. Record remediation in maintainer language:
   - what to tighten
   - where
   - what regression to test

