# Plan: TDD-Integrated Commands

## Overview

Add two new commands that mirror the existing `claude-codex` and `execute-codex` commands but enforce TDD (Red-Green-Refactor) workflow before implementation.

## Existing Commands

| Command | Implementation | Review |
|---------|---------------|--------|
| `/claude-codex` | Claude | Codex |
| `/execute-codex` | Codex (large) / Claude (small) | Claude (code-reviewer) |

## New Commands

| Command | Tests | Implementation | Review |
|---------|-------|---------------|--------|
| `/tdd-claude-codex` | Claude (tdd-guide agent) | Claude | Codex |
| `/tdd-execute-codex` | Claude (tdd-guide agent) | Codex (large) / Claude (small) | Claude (code-reviewer) |

## Reused Components

- **Agent**: general-purpose Task agent with bundled `tdd-specialist-role.md` — writes failing tests and verifies RED phase.
- **Agent**: `feature-dev:code-reviewer` — used in tdd-execute-codex for review
- **MCP**: `mcp__codex__codex` / `mcp__codex__codex-reply` — Codex review loop (tdd-claude-codex) and Codex implementation (tdd-execute-codex)
- **Prompt**: `~/.claude/prompts/codex/analyzer.md` — injected as developer-instructions for Codex review
- **Prompt**: `~/.claude/prompts/codex/architect.md` — injected as developer-instructions for Codex implementation

## Worktree and Diff Scoping

Before any work begins, capture a snapshot of the current state for scoping:
- If the worktree is dirty (uncommitted changes), stop and ask the user to commit or stash before running. Do not proceed with a dirty worktree.
- Record `git rev-parse HEAD` as `$START_SHA`
- All review diffs use `git diff $START_SHA` instead of `git diff HEAD` — this scopes reviews to exactly the files changed by this command
- Run `git add -N .` before each review to mark new untracked files as intent-to-add, ensuring they appear in the diff

## Baseline Test Strategy

"Establish baseline" does NOT require running the full test suite:
- **Fast suite** (<2 min): run the full suite, record pass/fail counts
- **Slow suite** (>2 min) or **partially failing**: run only tests in directories/modules related to the task, record known failures
- **No test suite**: skip baseline, note "no baseline available" — regression detection relies on the new tests only
- **Regression rule**: a regression is a test that was passing in baseline but fails after implementation. Pre-existing failures are not regressions.

## Security: Context Sanitization

Before passing file contents to any external agent (Task agent or Codex MCP):
- NEVER include `.env`, credentials, secrets, tokens, or API keys
- Exclude files matching: `.env*`, `*secret*`, `*credential*`, `*.pem`, `*.key`
- Redact any inline secrets (API keys, passwords, JWTs) from file contents before sending
- When in doubt, omit the file and describe its interface instead

---

## Command 1: `/tdd-claude-codex`

**Description**: "TDD workflow: Claude writes tests first, implements, Codex reviews"
**Model**: claude-sonnet-4-6
**Allowed tools**: Task, Read, Glob, Grep, Write, Edit, Bash, mcp__codex__codex, mcp__codex__codex-reply

### Workflow

#### Phase 0: Read Plan
- If argument is a file path, read and extract task description, steps, key files
- If no plan file, use argument as task description
- Confirm with user if key context is missing

#### Phase 1: Context Retrieval
- Read key files using Read, Glob, Grep (apply context sanitization rules)
- Identify existing test framework, patterns, and configuration
- Establish test baseline per Baseline Test Strategy above
- Confirm complete context before proceeding

#### Phase 2: TDD - Write Tests First (RED)

**Allowed output**: Test files only. No production code, no interface stubs, no implementation scaffolds. If imports require types/interfaces that do not exist yet, use inline type definitions within the test file or `any`/equivalent.

- Launch Task agent (subagent_type: "general-purpose (with tdd-specialist-role.md)") with:
  - Task description
  - Sanitized key file contents for context
  - Existing test patterns/framework info
  - Instruction: "Write failing test files ONLY. Do NOT create any production code, interfaces, or stubs. Define any needed types inline within test files. Follow RED phase only."
- If tdd-guide agent is unavailable, fall back to a general-purpose Task agent with the TDD prompt above
- Apply the test files returned by tdd-guide using Write/Edit

**RED Verification (scoped)**:
- Run ONLY the newly created test files (not the full suite)
- Verify they fail for the expected reason (e.g., "module not found", "not implemented", assertion failure on missing behavior)
- If tests pass unexpectedly: the tests are wrong — revise tests and re-verify RED
- If tests fail for unexpected reasons (syntax errors, broken imports unrelated to missing implementation): fix the test file, re-run

#### Phase 3: Claude Implements (GREEN)
- Write minimal implementation to make all new tests pass
- Run the new test files — verify all pass
- Run the full test suite — verify no regressions against Phase 1 baseline
- If new tests fail: fix implementation (not tests) until green
- If tests are fundamentally wrong (testing impossible behavior, flaky by design): revise tests with explicit justification, re-verify RED, then re-implement

#### Phase 4: Refactor (IMPROVE)
- Refactor implementation while keeping tests green
- Run tests after refactoring to confirm no regressions
- Check coverage if tooling is available (target 80%+), add tests if below threshold
- If coverage tooling is not available, skip and note "coverage not measured" in delivery

#### Phase 5: Codex Review (max 3 iterations)
Same loop as `/claude-codex` Phase 3:

**Step R1 — Run Codex Review via MCP**
Call `mcp__codex__codex` with:
- developer-instructions: content of ~/.claude/prompts/codex/analyzer.md + "\nBe concise. Return only the structured verdict format. No prose."
- prompt: `git diff $START_SHA` review prompt requesting VERDICT format (scoped to changes since command start)

Save threadId. Classify verdict:
- APPROVED -> Phase 6
- WARNING/BLOCKED -> fix, re-review via mcp__codex__codex-reply

**Step R2 — Fix and Re-review** (max 3 iterations)
Address ALL CRITICAL and HIGH issues before re-reviewing. One review per iteration, not one review per fix.

**Step R3 — MEDIUM/LOW issues**
After CRITICAL/HIGH are resolved, collect remaining MEDIUM/LOW issues. Ask user:
> "Codex flagged N MEDIUM/LOW issue(s) that were not fixed:
> - [list issues]
> Should I fix these before delivery, or proceed as-is?"

Wait for user response before proceeding.

#### Phase 6: Delivery
```markdown
## Execution Complete

### Changes
| File | Operation | Description |
|------|-----------|-------------|

### TDD Summary
- Tests written: N test files, M test cases
- Coverage: X% (target: 80%) / not measured (no coverage tooling)
- RED phase: All new tests failed as expected
- GREEN phase: All tests passing, no regressions

### Review Result
- Implemented by: Claude (active model)
- Codex review: APPROVED / N issues resolved (N/3 iterations)
- Remaining MEDIUM/LOW issues: N (skipped by user) / none

### Recommended Next Steps
1. [ ] Verify coverage meets project standards
2. [ ] Run full test suite
```

---

## Command 2: `/tdd-execute-codex`

**Description**: "TDD workflow: Claude writes tests, Codex/Claude implements, Claude reviews"
**Model**: claude-sonnet-4-6
**Allowed tools**: mcp__codex__codex, mcp__codex__codex-reply, Task, Read, Glob, Grep, Write, Edit, Bash

### Workflow

#### Phase 0: Read Plan
Same as tdd-claude-codex Phase 0.

#### Phase 1: Context Retrieval
Same as tdd-claude-codex Phase 1 (including baseline per Baseline Test Strategy and context sanitization).

#### Phase 2: TDD - Write Tests First (RED)
Same as tdd-claude-codex Phase 2:
- Launch tdd-guide agent for test writing (no production code, no stubs)
- Apply test files
- Scoped RED verification: run only new tests, verify expected failures

#### Phase 3: Route by Change Size

**IMPORTANT**: Routing is based on estimated implementation scope ONLY — exclude test files written in Phase 2 from the size calculation. Evaluate only the production code that needs to be written/modified.

**Small change** (all true): touches <=2 production files, estimated implementation diff <=30 lines, no new abstractions
**Large change** (any true): 3+ production files, implementation diff >30 lines, new logic/modules/refactoring

Announce routing decision.

#### Route A: Small Change — Claude Implements (GREEN)
1. Write minimal code to make tests pass
2. Run new tests — verify GREEN
3. Run full test suite — verify no regressions against baseline
4. Refactor while keeping tests green
5. Check coverage if available (80%+)
6. If tests are fundamentally wrong: revise with justification, re-verify RED, re-implement
7. Launch code-reviewer agent with git diff (scoped to changed files) + task requirements
8. If CRITICAL/HIGH issues: fix and re-review (max 2 rounds)
9. After CRITICAL/HIGH resolved, collect MEDIUM/LOW issues and ask user before fixing
10. Go to Phase 4

#### Route B: Large Change — Codex Implements (GREEN, max 3 iterations)

**Step B1 — Codex Implements**
Call `mcp__codex__codex` (or codex-reply for iterations 2-3):
- prompt: "Implement the following task. Tests have already been written — your implementation must make all tests pass.\n\nTask: {task}\n\nTest files:\n{test file contents}\n\nContext:\n{sanitized key file contents}"
- sandbox: "workspace-write"
- approval-policy: "on-failure"
- developer-instructions: content of ~/.claude/prompts/codex/architect.md + "\nTests are pre-written. Write minimal implementation to pass all tests. Do not modify test files."

Save threadId.

**Step B2 — Run Tests**
Run new tests to verify GREEN. Run full suite to verify no regressions.
If failures: include failure output in next Codex iteration.

**Step B3 — Coverage Check and Test Corrections (Claude owns)**
Check coverage if available (80%+). If coverage tooling unavailable, skip and note in delivery.

**Claude owns all post-RED test changes in Route B.** Codex never modifies test files. If additional tests are needed (coverage below threshold) or existing tests need correction (flaky, impossible assertions), Claude makes those changes directly using Edit/Write:
- Add coverage tests: write new test cases, run to verify they fail (RED), then verify Codex's implementation passes them
- Correct bad tests: revise with explicit justification, re-verify RED for corrected tests, then re-verify GREEN
- After any test changes, re-run the full new test suite to confirm GREEN before proceeding

**Step B4 — Code Review (Claude)**
Launch code-reviewer agent with git diff (scoped to changed files) + task requirements.
- No CRITICAL/HIGH -> collect MEDIUM/LOW, ask user, then Phase 4
- Has issues -> codex-reply with feedback, increment iteration

After 3 iterations without approval, stop and report.

#### Phase 4: Delivery
```markdown
## Execution Complete

### Changes
| File | Operation | Description |
|------|-----------|-------------|

### TDD Summary
- Tests written: N test files, M test cases
- Coverage: X% (target: 80%) / not measured (no coverage tooling)
- RED phase: All new tests failed as expected
- GREEN phase: All tests passing, no regressions

### Review Result
- Route: Small (Claude) / Large (Codex, N/3 iterations)
- Code review: Passed / N issues resolved
- Remaining MEDIUM/LOW issues: N (skipped by user) / none

### Recommended Next Steps
1. [ ] Verify coverage meets project standards
2. [ ] Run full test suite
```

---

## Files to Create

1. `commands/tdd-claude-codex.md` — TDD + Claude implements + Codex reviews
2. `commands/tdd-execute-codex.md` — TDD + Codex/Claude implements + Claude reviews

## Key Design Decisions

1. **Tests always written by Claude (tdd-guide)** — ensures consistent test quality and proper RED phase verification regardless of who implements
2. **No production code in RED phase** — Phase 2 outputs test files only; types needed by tests are defined inline within test files, not as production stubs
3. **Scoped RED verification** — only newly created tests are run during RED verification, preventing false positives from pre-existing failures
4. **Test correction escape hatch** — if tests are fundamentally wrong (impossible behavior, flaky by design), they can be revised with explicit justification, followed by re-verification of RED before continuing
5. **Codex cannot modify test files** — in tdd-execute-codex Route B, Codex is explicitly told not to modify tests, preserving test integrity
6. **Routing excludes test files** — in tdd-execute-codex, change size routing is based on estimated implementation scope only, not inflated by test file additions
7. **Coverage is conditional** — coverage is checked if tooling is available; if not, delivery reports "not measured" instead of failing
8. **Context sanitization** — secrets, credentials, and sensitive files are never passed to external agents or Codex MCP
9. **Adaptive baseline** — baseline strategy adapts to suite speed and health: full run for fast suites, scoped run for slow/failing suites, skip for no-suite repos
10. **Diff scoping via $START_SHA + git add -N** — all reviews use `git diff $START_SHA` scoped to command changes; `git add -N .` is run before each review to mark untracked new files as intent-to-add; dirty worktrees are rejected (user must commit or stash first)
11. **Claude owns all test changes in Route B** — in tdd-execute-codex large-change route, Claude makes any post-RED test corrections or coverage additions, never Codex
12. **MEDIUM/LOW triage** — both commands ask the user before fixing lower-severity review findings, consistent with existing `/claude-codex` behavior
13. **Bundled TDD role** — TDD specialist prompt is bundled as `tdd-specialist-role.md` alongside each TDD skill, used as role context for a general-purpose Task agent
