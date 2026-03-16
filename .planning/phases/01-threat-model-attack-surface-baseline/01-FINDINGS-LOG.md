# Phase 1 Findings Log Template

Use this log for all later audit phases. Append rows instead of inventing new ad hoc formats.

## Logging Format

| ID | Status | Severity | Principal | Boundary | Entry Point | Deployment Mode | Preconditions | Affected Files | Evidence | Remediation |
|---|---|---|---|---|---|---|---|---|---|---|
| 02-EXAMPLE | Example only | High | Authenticated board user | Company isolation | `GET /api/companies/:id/...` | `authenticated/private`, `authenticated/public` | Valid session, membership edge case | `server/src/middleware/auth.ts`; `server/src/routes/authz.ts` | Describe the exact code path or proof signal | Describe the smallest robust fix |

Delete the example row when the first real finding is logged.

## Field Guidance

- `Status`
  - `Confirmed vulnerability`
  - `Likely vulnerability`
  - `Accepted-risk sharp edge`
  - `No finding`
  - `Needs runtime validation`
- `Severity`
  - `Critical`
  - `High`
  - `Medium`
  - `Low`
  - `Informational`
- `Preconditions`
  - say what the attacker must already control or satisfy
- `Affected Files`
  - include at least one primary file and one secondary file when the risk spans layers
- `Evidence`
  - summarize the proof signal, not just a suspicion
- `Remediation`
  - state the likely change direction in maintainer language

## Per-Finding Checklist

Before adding a row, confirm the finding includes:

- Severity chosen from the fixed rubric
- Deployment Mode explicitly named
- Preconditions explicitly named
- Affected Files explicitly named
- Boundary explicitly named
- Remediation direction explicitly named

