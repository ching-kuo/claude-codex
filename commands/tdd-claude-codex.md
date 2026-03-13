---
description: "TDD workflow: Claude writes tests first, implements, Codex reviews"
argument-hint: "<task description or path/to/plan.md>"
model: claude-sonnet-4-6
allowed-tools: ["AskUserQuestion", "Task", "Read", "Glob", "Grep", "Write", "Edit", "Bash", "mcp__codex__codex", "mcp__codex__codex-reply"]
---

# TDD-Claude-Codex — Tests First, Claude Implements, Codex Reviews

$ARGUMENTS

---

## Core Protocols

- **TDD Mandate**: Tests must be written and verified RED before any implementation begins
- **Sovereignty**: Claude implements; Codex is the external reviewer — do not skip review
- **Stop-Loss**: Do not proceed to the next phase until the current phase output is validated
- **Language**: Use English when calling tools/models; communicate with user in their language
- **MCP for review**: Always call Codex review via `mcp__codex__codex`. Do NOT use Bash `codex review`
- **Context Sanitization**: Never pass `.env`, secrets, tokens, API keys, or credentials to any external agent or MCP. Exclude files matching `.env*`, `*secret*`, `*credential*`, `*.pem`, `*.key`. Redact inline secrets before sending.

---

## Execution Workflow

### Phase 0: Read Plan

1. If argument is a file path, read it and extract: task description, implementation steps, key files
2. If no plan file, use the argument as the task description directly
3. Confirm with user before proceeding if key context is missing

### Phase 1: Context Retrieval

1. Read key files using Read, Glob, Grep (sanitize before passing to agents)
2. Identify existing test framework, patterns, test file conventions, and configuration
3. Establish test baseline:
   - Fast suite (<2 min): run the full suite, record pass/fail counts
   - Slow suite (>2 min) or partially failing: run only tests related to the task scope, record known failures
   - No test suite: skip — note "no baseline available", regression detection relies on new tests only
4. Record `$START_SHA` via `git rev-parse HEAD` for diff scoping
5. If worktree is dirty (uncommitted changes), stop and ask the user to commit or stash their changes before running this command. Do not proceed with a dirty worktree.

### Phase 2: TDD — Write Tests First (RED)

**Constraint**: This phase outputs test files only. No production code, no interface stubs, no implementation scaffolds. Types/interfaces needed by tests must be defined inline within the test file or use `any`/equivalent.

Launch Task agent (subagent_type: "everything-claude-code:tdd-guide") with:
- Task description
- Sanitized key file contents for context
- Existing test patterns/framework info
- Instruction: "Write failing test files ONLY. Do NOT create any production code, interfaces, or stubs. Define any needed types inline within test files. Follow RED phase only."

If tdd-guide agent is unavailable, fall back to a general-purpose Task agent with the same TDD instruction above.

Apply the returned test files using Write/Edit.

**RED Verification (scoped — run new tests only, not the full suite)**:
- Run ONLY the newly created test files
- Verify they fail for the expected reason ("module not found", "not implemented", assertion failure on missing behavior)
- If tests pass unexpectedly → tests are wrong; revise and re-verify RED
- If tests fail for wrong reasons (syntax errors, broken imports unrelated to missing implementation) → fix the test file, re-run

### Phase 3: Codex Test Audit (max 2 iterations)

Before implementing, have Codex review the tests for quality, completeness, and edge case coverage.

Run `git add -N .` to mark new test files as intent-to-add.

**Call `mcp__codex__codex`** (iteration 1):
- `developer-instructions`: `"Be concise. Return only the structured verdict format. No prose."`
- `prompt`:

```
Run `git diff $START_SHA` to see newly created test files. Review the tests ONLY (no implementation exists yet).

Evaluate:
1. Do the tests cover the described task requirements comprehensively?
2. Are edge cases covered (null, empty, boundary, error paths)?
3. Are tests well-structured and following project test conventions?
4. Are there any tests that are impossible to implement or fundamentally flawed?

Return ONLY a structured verdict:

VERDICT: APPROVED | WARNING | BLOCKED

CRITICAL: <list or 'none'>
HIGH: <list or 'none'>
MEDIUM: <list or 'none'>
LOW: <list or 'none'>
```

Save the returned `threadId` (separate from the implementation review threadId).

**Parse response**:
- **APPROVED** → proceed to Phase 4
- **WARNING/BLOCKED** → Claude fixes all CRITICAL/HIGH issues in the test files, re-verifies RED, then re-calls via `mcp__codex__codex-reply` (max 2 iterations)

After 2 iterations without APPROVED, stop and ask user for direction.

### Phase 4: Claude Implements (GREEN)

**If the plan has 3+ implementation tasks**, dispatch each task to a subagent to keep the main context clean:

For each task (sequentially — next task starts only after current task passes tests):
1. Record `$TASK_SHA` via `git rev-parse HEAD` before dispatching
2. Launch a general-purpose Task agent with:
   - The specific task description and acceptance criteria
   - Relevant test file contents (the tests it must pass)
   - Sanitized key file contents it needs to read/modify
   - Instruction: "Implement only this task. Make the provided tests pass. Do not modify test files. Run the tests and confirm GREEN before finishing. Report: files changed, test result, any blockers."
3. After the subagent completes: run `git diff $TASK_SHA` (scoped to this task only) to verify what changed, run the task's tests to confirm GREEN
4. If the subagent reports a blocker or tests are still failing: fix directly with Edit/Write or re-dispatch with the failure output

**If the plan has fewer than 3 tasks**, implement directly:
- Write minimal implementation to make all new tests pass
- Run the new test files — verify all pass

After all tasks complete (either path):
- Run the baseline test scope — verify no new regressions (pre-existing failures are not regressions)
- Exception — if tests are fundamentally wrong (impossible behavior, flaky by design): revise tests with explicit justification, re-verify RED, then re-implement

### Phase 5: Refactor (IMPROVE)

- Refactor implementation while keeping tests green
- Run tests after refactoring to confirm no regressions
- If coverage tooling is available: check coverage (target 80%+); add tests if below threshold
- If coverage tooling is not available: skip and note "coverage not measured" in delivery

### Phase 6: Codex Review (max 3 iterations)

**Step R1 — Run Codex Review via MCP**

Before calling Codex, run `git add -N .` to mark any new untracked files as intent-to-add. This ensures `git diff $START_SHA` shows all created files, not just modifications.

Call `mcp__codex__codex` with:
- `developer-instructions`: `"Be concise. Return only the structured verdict format. No prose."`
- `prompt`:

```
Run `git diff $START_SHA` to see all changes made during this session, then review them.

Return ONLY a structured verdict in this exact format — no explanations, no prose:

VERDICT: APPROVED | WARNING | BLOCKED

CRITICAL: <list or 'none'>
HIGH: <list or 'none'>
MEDIUM: <list or 'none'>
LOW: <list or 'none'>
```

Save the returned `threadId` for follow-up replies.

Classify the verdict:
- **APPROVED** — no CRITICAL or HIGH → go to Phase 7
- **WARNING** — HIGH issues only → fix all, increment iteration, re-review
- **BLOCKED** — CRITICAL issues → fix all, increment iteration, re-review

**Step R2 — Fix and Re-review**

Address ALL CRITICAL and HIGH issues before re-reviewing:
- Collect every CRITICAL/HIGH finding from the last review
- Fix them all with Edit/Write in one batch
- Run available tests/lint to verify
- Re-call Codex via `mcp__codex__codex-reply` (reuse threadId) with:

```
Run `git diff $START_SHA` again to see the updated changes, then re-review.

Return ONLY the structured verdict in the same format.
```

One review per iteration, not one review per fix. Stop after 3 iterations without APPROVED.
After 3 iterations without APPROVED, stop and report remaining issues to user.

**Step R3 — MEDIUM / LOW Issues**

After CRITICAL/HIGH are resolved, collect any remaining MEDIUM and LOW issues.
If any exist, ask the user:

> "Codex flagged N MEDIUM/LOW issue(s) that were not fixed:
> - [list issues]
> Should I fix these before delivery, or proceed as-is?"

Wait for user response before proceeding.

### Phase 7: Delivery

```markdown
## Execution Complete

### Changes
| File | Operation | Description |
|------|-----------|-------------|
| path/to/file | Created/Modified | Description |

### TDD Summary
- Tests written: N test files, M test cases
- Coverage: X% (target: 80%) / not measured (no coverage tooling)
- RED phase: All new tests failed as expected
- GREEN phase: All tests passing, no regressions

### Review Result
- Codex test audit: APPROVED / N issues resolved (N/2 iterations)
- Implemented by: Claude (active model)
- Codex implementation review: APPROVED / N issues resolved (N/3 iterations)
- Remaining MEDIUM/LOW issues: N (skipped by user) / none

### Recommended Next Steps
1. [ ] Run full test suite
2. [ ] Verify coverage meets project standards
```
