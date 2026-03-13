---
description: "TDD workflow: Claude writes tests, Codex/Claude implements, Claude reviews"
argument-hint: "<task description or path/to/plan.md>"
model: claude-sonnet-4-6
allowed-tools: ["AskUserQuestion", "mcp__codex__codex", "mcp__codex__codex-reply", "Task", "Read", "Glob", "Grep", "Write", "Edit", "Bash"]
---

# TDD-Execute-Codex — Tests First, Smart Routing, Claude Reviews

$ARGUMENTS

---

## Core Protocols

- **TDD Mandate**: Tests must be written and verified RED before any implementation begins
- **Code Sovereignty**: Claude is the final authority — review and approve all changes before delivery
- **Stop-Loss**: Do not proceed to next phase until current phase output is validated
- **Language**: Use English when calling tools/models; communicate with user in their language
- **Test Ownership**: Claude owns all test files. Codex never modifies test files.
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

**MANDATORY Codex availability check**: `mcp__codex__codex` MUST be listed in the available tools (either in the tool list or in `<available-deferred-tools>`). Do NOT skip or bypass this phase. Do NOT assume Codex is unavailable without checking. If the tool is present, use it — no exceptions. If it is genuinely absent from both locations, note "Codex unavailable — skipping audit" and proceed to Phase 4.

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

### Phase 4: Route by Change Size

**IMPORTANT**: Routing is based on estimated **implementation** scope only. Exclude test files written in Phase 2 from the size calculation. Evaluate only the production code that needs to be written/modified.

**Small change** — ALL of the following must be true:
- Touches ≤ 2 production files
- Estimated implementation diff ≤ 30 lines
- No new abstractions, modules, or significant logic

**Large change** — any of the following:
- Touches 3+ production files
- Implementation diff > 30 lines
- Introduces new logic, new files, refactoring, or cross-cutting concerns

Announce routing decision (e.g. "Small change — implementing directly" or "Large change — routing to Codex").

---

### Route A: Small Change — Claude Implements (GREEN)

**If the plan has 3+ implementation tasks**, dispatch each task to a subagent (sequentially):
- Before each task: record `$TASK_SHA` via `git rev-parse HEAD`
- For each task: launch a general-purpose Task agent with the task description, relevant test files it must pass, sanitized context files, and instruction: "Implement only this task. Make the provided tests pass. Do not modify test files. Run tests and confirm GREEN before finishing. Report: files changed, test result, any blockers."
- After each subagent: run `git diff $TASK_SHA` (scoped to this task only) to verify changes, run the task's tests to confirm GREEN
- If the subagent reports a blocker: fix directly with Edit/Write or re-dispatch with failure output

**If fewer than 3 tasks**: implement directly with Edit/Write.

After all tasks complete (either path):
1. Run new tests — verify GREEN
2. Run baseline test scope — verify no new regressions
3. Refactor while keeping tests green
4. If coverage tooling available: check coverage (80%+)
5. Exception — if tests are fundamentally wrong: revise with justification, re-verify RED, re-implement
6. Run `git add -N .` to mark any new untracked files as intent-to-add (ensures `git diff $START_SHA` includes new files)
7. Launch Task agent (subagent_type: "everything-claude-code:code-reviewer") with:
   - `git diff $START_SHA` output
   - Original task requirements
8. If CRITICAL/HIGH issues: fix with Edit/Write, re-review (max 2 rounds)
9. After CRITICAL/HIGH resolved, collect MEDIUM/LOW issues and ask user:
   > "Reviewer flagged N MEDIUM/LOW issue(s) that were not fixed:
   > - [list issues]
   > Should I fix these before delivery, or proceed as-is?"
10. Go to Phase 5

---

### Route B: Large Change — Codex Implements (GREEN, max 3 iterations)

**MANDATORY**: Route B MUST use `mcp__codex__codex` when it is available. Check the tool list and `<available-deferred-tools>` — if `mcp__codex__codex` is present, use it. Only fall back to Claude direct implementation if the tool is genuinely absent from both locations.

**Note on agent spawning**: Codex runs in a sandboxed shell and cannot spawn Claude agents directly. Instead, Claude spawns a subagent per task — each subagent owns its Codex MCP session for that task, keeping both the main context and each Codex session scoped and clean.

**Step B1 — Dispatch per-task subagents (if plan has 3+ tasks)**

For each implementation task (sequentially — next starts only after current passes tests):

1. Record `$TASK_SHA` via `git rev-parse HEAD` before dispatching
2. Launch a general-purpose Task agent with:
   - The specific task description
   - Relevant test file contents (tests it must pass)
   - Sanitized key file contents
   - The `mcp__codex__codex` and `mcp__codex__codex-reply` tools
   - Instruction:

```
You are implementing one task using Codex via MCP. Follow these steps:

1. Call mcp__codex__codex with:
   - prompt: "Implement the following task. Tests are pre-written — make them pass.\n\nTask: {task}\n\nTest files:\n{test file contents}\n\nContext:\n{sanitized key file contents}"
   - sandbox: "workspace-write"
   - approval-policy: "on-failure"
   - developer-instructions: {content of ~/.claude/prompts/codex/architect.md} + "\nTests are pre-written. Write minimal implementation to pass all tests. Do not modify test files."

2. Save the returned threadId.

3. Run the task's tests to verify GREEN.

4. If tests fail: call mcp__codex__codex-reply with the threadId and failure output. Retry up to 3 times.

5. Report back: files changed, final test result, any unresolved failures.
```

3. After each subagent: run `git diff $TASK_SHA` (scoped to this task only) to verify what changed, run the task's tests to confirm GREEN

**Important**: Each subagent owns its own Codex `threadId`. There is no shared threadId across tasks. Step B4 (code review) therefore uses the `code-reviewer` Task agent — not `mcp__codex__codex-reply` — since there is no single session to reply to.

**If plan has fewer than 3 tasks**, call `mcp__codex__codex` directly (iteration 1) or `mcp__codex__codex-reply` (iterations 2-3):
- prompt: "Implement the following task. Tests have already been written — your implementation must make all tests pass.\n\nTask: {task}\n\nTest files:\n{test file contents}\n\nContext:\n{sanitized key file contents}"
- prompt (retry): "Fix the following issues:\n\n{failure output or reviewer feedback verbatim}"
- sandbox: "workspace-write"
- approval-policy: "on-failure"
- developer-instructions: {content of ~/.claude/prompts/codex/architect.md} + "\nTests are pre-written. Write minimal implementation to pass all tests. Do not modify test files."

Save the returned `threadId`. Reuse for all subsequent calls.

**Step B2 — Run Tests**

Run new tests to verify GREEN. Run baseline test scope to verify no new regressions.
If failures: include failure output verbatim in the next Codex iteration prompt.

**Step B3 — Coverage Check and Test Corrections (Claude owns)**

Check coverage if tooling is available (target 80%+). If unavailable, skip and note in delivery.

**Claude owns all post-RED test changes.** Codex never modifies test files. If additional tests are needed or existing tests need correction:
- Add coverage tests: write new test cases using Edit/Write, run to verify RED, then verify Codex's implementation passes them
- Correct bad tests: revise with explicit justification using Edit/Write, re-verify RED, then re-verify GREEN
- After any test changes: re-run the full new test suite to confirm GREEN before proceeding

**Step B4 — Code Review (Claude)**

Run `git add -N .` to mark any new untracked files as intent-to-add (ensures `git diff $START_SHA` includes new files).

Launch Task agent (subagent_type: "everything-claude-code:code-reviewer") with:
- `git diff $START_SHA` output
- Original task requirements

Parse reviewer response:
- No CRITICAL/HIGH → collect MEDIUM/LOW, ask user (same prompt as Route A step 9), then Phase 5
- Has CRITICAL/HIGH (3+ task path) → dispatch a Claude subagent per affected task with the reviewer feedback verbatim; there is no shared `threadId` to reply to, so do not call `mcp__codex__codex-reply`. After fixes, re-run `code-reviewer` (max 3 total iterations).
- Has CRITICAL/HIGH (single Codex session path) → call `mcp__codex__codex-reply` with reviewer feedback verbatim, increment iteration (max 3 total iterations).

After 3 iterations without approval, stop and report remaining issues to user.

---

### Phase 5: Delivery

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
- Route: Small (Claude) / Large (Codex, N/3 iterations)
- Code review: Passed / N issues resolved
- Remaining MEDIUM/LOW issues: N (skipped by user) / none

### Recommended Next Steps
1. [ ] Run full test suite
2. [ ] Verify coverage meets project standards
```
