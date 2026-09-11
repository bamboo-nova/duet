# Requirements interview — procedure and template

Purpose: when the goal is ambiguous, Codex invents one by inference. Fix the requirements in a document before entering decomposition and verification.

## Procedure

Ask questions in rounds. Batch only the questions whose premises are already settled, wait for the answers, then move to the next round. Defer any question that depends on an earlier answer to a later round.

Attach a recommended answer to every question, so the user can decide instantly.

Finish when every section of the template below is filled and no unsettled premise remains. Never write "TBD" for an unfilled section; have the user decide it.

## Question angles

1. Goal. What counts as success? Make the working artifact, deliverables, and audience concrete.
2. Non-goals. State explicitly what is out of scope this time. Draw the outer boundary first.
3. Constraints. Tech stack, consistency with existing code, deadlines, allowed dependencies.
4. Acceptance criteria. How completion is confirmed: tests, manual verification steps, performance conditions.
5. Environment. Target repository, runtime environment, permissions. Investigate facts yourself instead of asking the user.

## requirements.md template

```markdown
# Requirements: <project name>

- Created: <YYYY-MM-DD>
- Status: approved | draft

## Goal

<one sentence, followed by a list of concrete deliverables>

## Non-goals

- <what will not be done>

## Constraints

- <technical / deadline / dependency constraints>

## Acceptance criteria

- [ ] <a verifiable condition; no vague wording>

## Premises / environment

- <facts confirmed by investigation>
```

## Approval gate 1

Present requirements.md to the user and get explicit approval before moving to Phase 2. If change requests come back, apply them and present again.
