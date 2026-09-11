---
name: duet
description: A development pipeline that pairs Claude Code (planning & code review) with Codex (plan verification & implementation) in fixed roles. Drives a requirements interview, a plan-verification loop, ADR export, and subgoal-by-subgoal implementation. Use for development tasks that should go all the way from design to implementation. 設計から実装まで通したい開発タスク(要件定義→計画→検証→実装)にも使う。
---

# duet — a Claude Code × Codex collaborative development pipeline

## Roles (fixed; they never change based on which CLI launched the skill)

- Claude Code does planning, subgoal decomposition, and review of implemented code.
- Codex does plan verification (hole-finding) and implementation.
- The model running this skill (the launcher) does orchestration. The Phase 1 interview is also done by the launcher.

Neither role is tied to a specific model. Model selection is left entirely to each CLI's config default.

How each role is executed depends on the launcher.

- Codex's role always runs in a `codex exec` child process, regardless of the launcher. It never runs in the current session.
- Claude Code's role runs in the current session when the launcher is Claude Code. When the launcher is Codex, delegate it to `claude -p`.
- See `references/counterpart-cli.md` for the exact invocation commands.

## Global prohibitions

- Never perform git operations (add / commit / branch / worktree). The user does them. State this prohibition explicitly in every implementation prompt sent to Codex.
- Never specify a model when invoking the counterpart CLI. Leave it to each CLI's config default.

## State management

Keep all state in `.duet/` inside the target repository. Create the directory if it does not exist.

```
.duet/
├── state.json            # phase, loop counters, session IDs
├── requirements.md       # Phase 1 output
├── plan.md               # Phase 2–3 output (includes verification history)
├── prompts/              # copies of every prompt sent to the counterpart CLI
├── verification-<n>.json # verdict of verification loop n
├── review-<SG>-<n>.json  # verdict of review n for subgoal SG
└── codex-events-*.jsonl  # codex exec event logs (used to capture session IDs)
```

Format of state.json:

```json
{
  "phase": "interview | plan | verify | adr | implement | done",
  "verifyLoop": 0,
  "verifySessionId": null,
  "adrPath": null,
  "subgoals": [
    {
      "id": "SG1",
      "title": "subgoal name",
      "status": "pending | implementing | review | done",
      "implSessionId": null,
      "reviewLoop": 0
    }
  ]
}
```

At skill startup, always check `.duet/state.json`. If it exists, resume from the recorded `phase` and report the progress to the user in one line. If it does not exist, create it and start from Phase 1. Update state.json every time a phase advances.

## Phase 1: Requirements interview (launcher)

Follow the procedure in `references/interview-template.md` to pin down requirements through rounds of questions. Fill in every section of the template and write the result to `.duet/requirements.md`.

Approval gate 1: present requirements.md to the user and get explicit approval. Do not proceed until approved.

## Phase 2: Planning and subgoal decomposition (Claude Code)

Take requirements.md as input and produce `.duet/plan.md` in the following format.

```markdown
# Implementation plan: <project name>

## Approach

<a few lines describing the overall approach>

## Subgoals

### SG1: <name>

- Goal: <what this subgoal achieves>
- Changes: <files / modules>
- Done criteria: <verifiable conditions>
- Depends on: <preceding subgoal IDs, or "none">

## Verification history

<appended during Phase 3>
```

Subgoals are the unit of the implementation loop. Cut each one at a granularity that is self-contained as "a single implementation instruction to Codex".

## Phase 3: Plan-verification loop (Codex)

Follow the conventions in `references/verdict-schema.md`.

1. Write the verification prompt to `.duet/prompts/verify-<n>.md`. It must contain:
   - an instruction to read requirements.md and plan.md
   - an instruction to verify each subgoal individually, then verify overall consistency (gaps between subgoals, dependency contradictions, dropped requirements)
   - an instruction to output only the verdict JSON
2. Run the verification with `codex exec` on the first loop and `codex exec resume` afterwards (see counterpart-cli.md). After the first run, take the session ID from the event log and store it in `verifySessionId`.
3. Read the verdict and apply the findings to plan.md (Claude Code's role). Record what was applied and what was deferred in the "Verification history" section.
4. Judge. If there are zero high findings, the plan has converged. Otherwise increment `verifyLoop` and go back to step 1. The cap is 3 loops.

Approval gate 2: on convergence or reaching 3 loops, present plan.md and the remaining findings (medium / low, plus any unresolved high) to the user and get approval.

## Phase 4: ADR export

Write the approved plan as an ADR at the root of the target repository. Name the file `adr-<serial>-<slug>.md` (determine the next serial from existing adr-*.md files). Cover context, decision, considered alternatives with rejection reasons, and consequences. Record the path in `adrPath` in state.json.

## Phase 5: Implementation loop (per subgoal)

Process subgoals one at a time in dependency order.

1. Write the implementation prompt to `.duet/prompts/impl-<SG>.md`. It must contain the relevant subgoal from plan.md, the path to requirements.md, the git-operation prohibition, and the done criteria.
2. Have Codex implement it with `codex exec` (workspace-write + network allowed). Store the session ID in `implSessionId`.
3. Claude Code reviews the result. Produce `review-<SG>-<n>.json` under the conventions of verdict-schema.md. Review for satisfaction of the done criteria, consistency with the requirements, consistency with the existing code, and the presence and result of tests.
4. If there are zero high findings, set the status to done and move to the next subgoal. Otherwise send the findings back to the same implementation session with `codex exec resume`, increment `reviewLoop`, and go back to step 3. The cap is 3 rounds.
5. If it does not pass within 3 rounds, format the remaining findings, present them to the user, and ask for a decision (escalation).

## Phase 6: Completion report

When every subgoal is done, report. Include:

- the implemented subgoals and a list of changed files
- remaining medium / low findings
- verification steps and the work left to the user (e.g. git commit)

There is no approval gate here. Report and finish.

## herdr integration (optional)

Only when running inside a [herdr](https://herdr.dev) pane, emit desktop notifications at the points where a human is being waited on. Detect it with:

```bash
test "${HERDR_ENV:-}" = 1
```

If the test is false, do nothing. If true, notify at these four points:

```bash
# Approval gate 1 (waiting for requirements approval)
herdr notification show "duet: awaiting requirements approval" --body "Please review requirements.md" --sound request
# Approval gate 2 (waiting for plan approval)
herdr notification show "duet: awaiting plan approval" --body "Please review plan.md and the remaining findings" --sound request
# Escalation (not converged after 3 verification loops or 3 review rounds)
herdr notification show "duet: escalation" --body "A decision is needed on the remaining findings" --sound request
# Completion report (Phase 6)
herdr notification show "duet: done" --body "All subgoals done. Please check the report" --sound done
```

Notifications are auxiliary. If the `herdr` command fails, never stop the pipeline; keep going.

## Error handling

- If `codex exec` fails, inspect the tail of the event log and report the cause. Never silently retry the same invocation.
- If the session to resume cannot be found, restart with a fresh session, passing the verification history in plan.md (or the changes implemented so far) as context.
- Interruption is always safe. As long as state.json is current, the next run resumes from where it left off.
