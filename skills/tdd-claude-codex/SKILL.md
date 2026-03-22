---
name: tdd-claude-codex
description: "Full TDD workflow: Claude writes tests first (RED), Codex audits tests, Claude implements (GREEN), refactors (IMPROVE), Codex reviews final implementation. Best for: test-driven development where you want Codex reviewing both tests and code. Triggers on: /tdd-claude-codex, TDD with review, test-driven implement, tests first then implement."
---

# TDD-Claude-Codex — Tests First, Claude Implements, Codex Reviews

## Input

The user's message after the trigger is either:
1. A file path to a plan (e.g. `.claude/plan/feature.md`) — read it and extract task description, implementation steps, key files
2. A direct task description — use as-is

Confirm with user before proceeding if key context is missing.

## Model Recommendation

Works with `claude-sonnet-4-6` (default). Override with `/model opus` for tasks requiring deep architectural reasoning.

## Tool Requirements

Advisory checklist — ensure these tools are available:
- `AskUserQuestion` — confirm missing context
- `Task` — dispatch tdd-guide and implementation subagents
- `Read`, `Glob`, `Grep` — context retrieval
- `Write`, `Edit` — write tests and implement
- `Bash` — run tests/lint/git commands
- `mcp__codex__codex` and `mcp__codex__codex-reply` — Codex test audit and implementation review

## Core Protocols

- **TDD Mandate**: Tests must be written and verified RED before any implementation begins
- **Sovereignty**: Claude implements; Codex is the external reviewer — do not skip review
- **Stop-Loss**: Do not proceed to the next phase until the current phase output is validated
- **Language**: Use English when calling tools/models; communicate with user in their language
- **MCP for review**: Always call Codex review via `mcp__codex__codex`. Do NOT use Bash `codex review`
- **Context Sanitization**: Never pass `.env`, secrets, tokens, API keys, or credentials to any external agent or MCP. Exclude files matching `.env*`, `*secret*`, `*credential*`, `*.pem`, `*.key`. Redact inline secrets before sending.

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
5. If worktree is dirty (uncommitted changes), stop and ask the user to commit or stash their changes before running this skill. Do not proceed with a dirty worktree.

### Phase 2: TDD — Write Tests First (RED)

**Constraint**: This phase outputs test files only. No production code, no interface stubs, no implementation scaffolds. Types/interfaces needed by tests must be defined inline within the test file or use `any`/equivalent.

Read `tdd-specialist-role.md` from this skill directory.

Launch a general-purpose Task agent with:
- Task description
- Sanitized key file contents for context
- Existing test patterns/framework info
- Content of tdd-specialist-role.md as the agent's role context
- Instruction: "Write failing test files ONLY. Do NOT create any production code, interfaces, or stubs. Define any needed types inline within test files. Follow RED phase only."

Apply the returned test files using Write/Edit.

**RED Verification (scoped — run new tests only, not the full suite)**:
- Run ONLY the newly created test files
- Verify they fail for the expected reason ("module not found", "not implemented", assertion failure on missing behavior)
- If tests pass unexpectedly → tests are wrong; revise and re-verify RED
- If tests fail for wrong reasons (syntax errors, broken imports unrelated to missing implementation) → fix the test file, re-run

### Phase 3: Codex Test Audit (max 2 iterations)

**MANDATORY Codex availability check**: `mcp__codex__codex` MUST be listed in the available tools (either in the tool list or in `<available-deferred-tools>`). If the tool is genuinely absent from both locations, **degrade gracefully**: note "Codex unavailable — test audit skipped" and proceed to Phase 4. Do not stop — implementation can continue without the test audit.

Before implementing, have Codex review the tests for quality, completeness, and edge case coverage.

Run `git add -N .` to mark new test files as intent-to-add.

Read `codex-analyzer-role.md` from this skill directory and inject as `developer-instructions`.

**Call `mcp__codex__codex`** (iteration 1):
- `developer-instructions`: {content of codex-analyzer-role.md} + `"\nBe concise. Return only the structured verdict format. No prose."`
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
- **WARNING/BLOCKED** → critically evaluate findings before fixing (see below)

**Critical Evaluation of Test Audit Findings**

Before fixing any issues, critically evaluate each CRITICAL/HIGH finding:

1. **Assess correctness**: Is the finding technically accurate? Check the test code and project test conventions.
2. **Check context**: Does the reviewer have full context about the testing strategy and requirements?
3. **Verify applicability**: Does the suggestion improve test quality, or is it a preference / false positive?

**If a finding seems incorrect or questionable:**
- Do NOT fix it or count it as an iteration. Instead, reply to the reviewer with your technical reasoning.
- Call `mcp__codex__codex-reply` (reuse threadId) explaining why the finding appears incorrect, with specific test code references.
- If the reviewer provides additional justification that is convincing, accept the finding.
- If the reviewer concedes or cannot justify further, drop the finding.
- Discussion replies do NOT increment the iteration counter — only fix-and-re-review cycles count.

**If findings are accepted:** Claude fixes all accepted CRITICAL/HIGH issues in the test files, re-verifies RED, then re-calls via `mcp__codex__codex-reply` (max 2 iterations).

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

**MANDATORY Codex availability check**: `mcp__codex__codex` MUST be listed in the available tools (either in the tool list or in `<available-deferred-tools>`). Do NOT skip or bypass this phase. If the tool is genuinely absent from both locations, **stop and tell the user**: "Codex MCP is not available for implementation review. This skill requires Codex for the final review. Please add the Codex MCP server or use a different skill." Do not proceed without Codex — the final Codex review is this skill's differentiator from plain TDD.

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

**Step R1.5 — Critical Evaluation of Review Findings**

Before fixing any issues, critically evaluate each CRITICAL/HIGH finding:

1. **Assess correctness**: Is the finding technically accurate? Check the actual code and project conventions.
2. **Check context**: Does the reviewer have full context, or is it flagging intentional design decisions?
3. **Verify applicability**: Does the suggestion improve correctness/security, or is it a style preference / false positive?

**If a finding seems incorrect or questionable:**
- Do NOT fix it or count it as an iteration. Instead, reply to the reviewer with your technical reasoning.
- Call `mcp__codex__codex-reply` (reuse threadId) explaining why the finding appears incorrect, with specific code references.
- If the reviewer provides additional justification that is convincing, accept the finding and include it in the fix batch.
- If the reviewer concedes or cannot justify further, drop the finding.
- Discussion replies do NOT increment the iteration counter — only fix-and-re-review cycles count.

**If all findings appear correct:** proceed directly to Step R2.

**Step R2 — Fix and Re-review**

Address ALL accepted CRITICAL and HIGH issues before re-reviewing:
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
