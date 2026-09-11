# Counterpart CLI invocation reference

Based on options measured against codex-cli 0.151.0. Never specify a model (leave it to each CLI's config).

## Invoking Codex (codex exec) — always used for verification and implementation

Codex's role (verification and implementation) always runs in a `codex exec` child process, whether the launcher is Claude Code or Codex. This gives it a dedicated session, separate from orchestration, that can be continued with resume.

### Plan verification (first run)

```bash
codex exec --json \
  -s read-only \
  --skip-git-repo-check \
  --output-schema "<SKILL_DIR>/references/verdict.schema.json" \
  -o .duet/verification-1.json \
  "$(cat .duet/prompts/verify-1.md)" \
  > .duet/codex-events-1.jsonl
```

- `--output-schema` forces the final response into verdict JSON.
- `-o` writes the final message (the verdict JSON) to a file.
- `--json` streams execution events as JSONL to stdout. The session ID is taken from here.

### Capturing and storing the session ID

The first line of the event JSONL is `{"type":"thread.started","thread_id":"<UUID>"}` (measured on 0.151.0). Store this `thread_id` in `verifySessionId` in `state.json`. The fallback when it cannot be captured is `codex exec resume --last`, but that risks mixing with other codex usage, so prefer storing the ID.

### Plan verification (second run onwards)

```bash
codex exec resume "$VERIFY_SESSION_ID" --json \
  --skip-git-repo-check \
  -c 'sandbox_mode="read-only"' \
  --output-schema "<SKILL_DIR>/references/verdict.schema.json" \
  -o .duet/verification-2.json \
  "$(cat .duet/prompts/verify-2.md)" \
  > .duet/codex-events-2.jsonl
```

The resume subcommand does not accept the `-s` flag (measured on 0.151.0). Specify the sandbox with `-c sandbox_mode="..."`. `--skip-git-repo-check` / `--json` / `-o` / `--output-schema` all work with resume.

### Implementation (per subgoal)

```bash
codex exec --json \
  -s workspace-write \
  -c sandbox_workspace_write.network_access=true \
  --skip-git-repo-check \
  -C "<target repository>" \
  "$(cat .duet/prompts/impl-SG1.md)" \
  > .duet/codex-events-impl-SG1.jsonl
```

- The sandbox is workspace-write. Writes are confined to the target repository.
- Network access is allowed (to support adding dependency packages).
- State the git-operation prohibition explicitly in the implementation prompt (the sandbox cannot enforce it).

Send review-finding fixes back to the same implementation session.

```bash
codex exec resume "$IMPL_SESSION_ID" --json \
  --skip-git-repo-check \
  -c 'sandbox_mode="workspace-write"' \
  -c sandbox_workspace_write.network_access=true \
  "$(cat .duet/prompts/fix-SG1-1.md)" \
  > .duet/codex-events-fix-SG1-1.jsonl
```

## Invoking Claude Code (claude -p) — only when launched from Codex

When the launcher is Claude Code, its role (planning and review) runs in the current session. Only when the launcher is Codex, delegate it to `claude -p`.

### Planning

```bash
claude -p "$(cat .duet/prompts/plan.md)" > .duet/plan-draft.md
```

Include the path to requirements.md and the plan.md format in the prompt.

### Implementation review

```bash
claude -p "$(cat .duet/prompts/review-SG1.md)" > .duet/review-SG1-1.json
```

Include the following in the prompt:

- the subgoal's spec (the relevant part of plan.md) and the path to requirements.md
- the list of files Codex changed
- an instruction to output only JSON conforming to verdict.schema.json

## Shared cautions

- Invoke both CLIs assuming they run without approval prompts. Do not include operations that require interaction.
- Write prompts to files under `.duet/prompts/` before passing them. This avoids shell-escaping accidents.
- Replace `<SKILL_DIR>` with the actual directory of this skill.
