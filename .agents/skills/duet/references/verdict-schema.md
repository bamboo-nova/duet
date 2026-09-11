# Verdict conventions — shared by verification and review

Plan verification (Phase 3) and implementation review (Phase 5) use the same finding format and the same loop rules.

## Finding format

Have the counterpart output only JSON conforming to `verdict.schema.json`. Prose commentary is not accepted.

```json
{
  "target": "plan",
  "findings": [
    {
      "id": "F1",
      "severity": "high",
      "scope": "SG2",
      "title": "Rollback procedure undefined",
      "detail": "SG2 includes a DB migration, but the plan has no way to roll it back on failure.",
      "suggestion": "Add creation of a rollback procedure to SG2's done criteria."
    }
  ]
}
```

Severity criteria:

- high is a defect that must be fixed. Left alone, the plan or implementation does not hold.
- medium is a risk or an ambiguity. Either address it, or defer it with an explicit reason.
- low is an improvement suggestion. Record it and move on.

## Pass criteria and loop rules

- The pass condition is "zero findings with severity high". medium and low findings may remain.
- On failure, apply the findings and re-verify (or re-implement). The loop cap is 3.
- If it does not pass within 3 loops, format the remaining findings, present them to the user, and ask for a decision.
- Remaining medium and low findings must always be shown to the user at the approval gate. Never drop them silently.

## Recording applied findings

When applying findings to the plan, append the following to the "Verification history" section at the end of plan.md:

- the loop number and finding IDs
- what was done (applied / deferred) and the reason for any deferral

A resumed verification session reads this history to confirm the resolution status of its previous findings.
