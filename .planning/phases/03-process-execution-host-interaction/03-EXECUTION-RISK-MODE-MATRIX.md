# Phase 3: Execution Risk Mode Matrix

## Scope

This artifact applies the Phase 1 threat model and finding methodology to the Phase 3 execution review.

- Goal: separate remote-relevant vulnerabilities from trusted local-only or operator-controlled sharp edges
- Inputs: `03-PROCESS-EXECUTION-REVIEW.md`, `03-WORKSPACE-HOST-BOUNDARY-REVIEW.md`, `03-ENV-SESSION-LOG-REVIEW.md`, `01-THREAT-MODEL.md`, and `doc/DEPLOYMENT-MODES.md`
- Classification rule: do not label a powerful execution surface as a vulnerability unless a weaker principal, broader deployment mode, or broken boundary makes it reachable beyond its stated trust model

## Remote-Relevant Execution Risks

| Item | State | Severity | Principal | Deployment reach | Why it is a real risk |
|---|---|---|---|---|---|
| Same-company cross-agent heartbeat artifact reads | Confirmed vulnerability | High | Agent principal | `authenticated/private`, `authenticated/public`, and any mode with multiple agents | Same-company agents can read peer run details, logs, and events through routes guarded only by `assertCompanyAccess()`, exposing secrets, session metadata, and replayable local JWTs |
| Broad run-detail metadata exposure | Likely vulnerability | Medium | Agent principal | `authenticated/private`, `authenticated/public`, and any mode with multiple agents | `resultJson`, `contextSnapshot`, and session identifiers are exposed more broadly than necessary even when a secret is not directly logged |
| Token replay from leaked local JWTs | Needs runtime validation | Medium | Agent principal | `authenticated/private`, `authenticated/public`, and local trusted multi-agent use | The read path is confirmed, the 48-hour local JWT is real, but realistic replay ease depends on how often adapters echo `PAPERCLIP_API_KEY` or related secrets into readable artifacts |

The phase’s strongest confirmed issue is not arbitrary shell execution. It is lateral artifact exposure between same-company agents.

## Accepted Local-Only Sharp Edges

| Surface | State | Severity | Principal | Why it stays in accepted-risk territory for now |
|---|---|---|---|---|
| Board-authored local adapters and generic process adapter | Accepted-risk sharp edge | Informational | Authenticated board user or local board operator | Board power already includes creating agents and steering execution; reviewed code does not show a weaker principal reaching these sinks directly |
| `provisionCommand` and runtime-service shell commands | Accepted-risk sharp edge | Informational | Authenticated board user or local board operator | These are explicit operator shell hooks defined in workspace policy and remain inside the intended trusted-operator model |
| Absolute-path worktrees and service cwd resolution | Accepted-risk sharp edge | Informational | Authenticated board user or local board operator | The code deliberately accepts absolute paths and `~`, broadening the filesystem boundary without proving an unexpected escape |
| CLI worktree seeding of secrets, JWT secret, DB seed, and git hooks | Accepted-risk sharp edge | Medium | Local operator | The copied material materially expands blast radius, but the behavior matches the local-operator workflow rather than crossing into a weaker remote principal |

These items are still important because later plugin, data, or operations phases may show that a weaker actor can influence them indirectly.

## Boundary Classification

| Boundary | Reviewed paths | Strongest reachable principal today | Classification |
|---|---|---|---|
| Request or DB input -> local process execution | Heartbeat adapter execution, local adapters, generic process adapter | Board | Accepted-risk sharp edge |
| Workspace policy -> host shell mutation | `provisionCommand`, runtime services, worktree placement | Board | Accepted-risk sharp edge |
| CLI worktree creation -> copied secrets and hooks | `paperclipai worktree` setup flows | Local operator | Accepted-risk sharp edge |
| Child-process env and output -> readable company artifacts | Heartbeat run detail, events, and log routes | Agent | Confirmed vulnerability |
| Remote adapter config -> outbound HTTP or gateway execution | HTTP adapter, OpenClaw gateway | Board | Needs runtime validation |

Attacker-position summary:

- `local_trusted`: powerful operator behavior is expected, but it does not excuse peer-agent artifact exposure or shared-secret leakage once multiple principals exist.
- `authenticated/private`: the confirmed artifact-read finding remains relevant because same-company agents are first-class principals in this mode.
- `authenticated/public`: no new unauthenticated execution path was confirmed in Phase 3, but the same agent-to-agent artifact issue remains reachable once an attacker has agent credentials.

## Findings To Carry Forward

- **Phase 4:** Determine whether plugin packages or worker capability gaps can steer local adapters, workspace policy, or remote adapter destinations. This is the main path that could convert accepted operator sharp edges into true semi-trusted execution vulnerabilities.
- **Phase 5:** Re-review `resultJson`, `contextSnapshot`, session artifacts, and log content under the broader secrets and storage lens. Phase 3 confirmed the read surface; Phase 5 should decide how much sensitive material is actually retained and how redaction should change.
- **Phase 6:** Revisit operational defaults around `PAPERCLIP_AGENT_JWT_SECRET`, local JWT TTL, worktree secret seeding, and hook copying. These are not Phase 3 boundary breaks on their own, but they materially shape blast radius once another phase confirms exposure.
- **Phase 7:** Promote the same-company run-artifact exposure into the final findings catalog with explicit remediation: add agent-to-run ownership checks, tighten run-detail and log permissions, and apply stronger secret-aware redaction before artifact persistence and readback.

## Later Phase Questions

- Can any plugin-owned path upgrade from prompt control into adapter-type, command, or workspace-policy control?
- Should local agent JWTs be scoped more narrowly than company-wide agent identity when used as `PAPERCLIP_API_KEY`?
- Is there a legitimate product use case for peer-agent access to `resultJson`, `contextSnapshot`, and raw run logs, or should those become board-only artifacts?
