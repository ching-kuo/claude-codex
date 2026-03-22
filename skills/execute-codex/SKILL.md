---
name: execute-codex
description: "Smart size-based routing: Claude implements small changes (<=2 files, <=30 lines), Codex implements large changes, code-reviewer agent reviews all. Best for: executing a plan or task where you want automatic routing by complexity. Triggers on: /execute-codex, execute plan, implement with routing, run the plan."
---

# Execute-Codex — Smart Routing: Claude (small) / Codex (large)

## Input

The user's message after the trigger is either:
1. A file path to a plan (e.g. `.claude/plan/feature.md`) — read it and extract task description, implementation steps, key files
2. A direct task description — use as-is

Confirm with user before proceeding if key context is missing.

## Model Recommendation

Works with `claude-sonnet-4-6` (default). No model override needed — routing handles complexity automatically.

## Tool Requirements

Advisory checklist — ensure these tools are available:
- `AskUserQuestion` — confirm missing context
- `mcp__codex__codex` and `mcp__codex__codex-reply` — Codex implementation (Route B)
- `Task` — dispatch subagents
- `Read`, `Glob`, `Grep` — context retrieval
- `Write`, `Edit` — Claude implementation (Route A)
- `Bash` — run lint/tests/git commands

## Core Protocols

- **Code Sovereignty**: Claude is the final authority — review and approve all changes before delivery
- **Stop-Loss**: Do not proceed to next phase until current phase output is validated
- **Language**: Use English when calling tools/models; communicate with user in their language
- **Context Sanitization**: Never pass `.env`, secrets, tokens, API keys, or credentials to any external agent or MCP. Exclude files matching `.env*`, `*secret*`, `*credential*`, `*.pem`, `*.key`. Redact inline secrets before sending.

## Execution Workflow

### Phase 0: Read Plan

1. If argument is a file path, read it and extract: task description, implementation steps, key files
2. If no plan file, use the argument as the task description directly
3. Confirm with user before proceeding if key context is missing

### Phase 1: Context Retrieval

Read key files using Read, Glob, Grep. Confirm complete context before proceeding.

### Phase 2: Route by Change Size

Classify the task before doing any implementation:

**Small change** — ALL of the following must be true:
- Touches ≤ 2 files
- Estimated diff ≤ 30 lines
- No new abstractions, modules, or significant logic (e.g. config edits, wording fixes, minor additions, single-function tweaks)

**Large change** — any of the following:
- Touches 3+ files
- Estimated diff > 30 lines
- Introduces new logic, new files, refactoring, or cross-cutting concerns

Announce your routing decision before implementing (e.g. "Small change — implementing directly" or "Large change — routing to Codex").

---

### Route A: Small Change — Claude Implements Directly

**If the plan has 3+ implementation tasks**, dispatch each task to a subagent (sequentially):
- Before each task: record `$TASK_SHA` via `git rev-parse HEAD`
- For each task: launch a general-purpose Task agent with the task description, sanitized context files, and instruction: "Implement only this task. Run available lint/tests scoped to changed files. Report: files changed, verification result, any blockers."
- After each subagent: run `git diff $TASK_SHA` (scoped to this task only) to verify changes, run scoped lint/tests to confirm no regressions
- If the subagent reports a blocker: fix directly with Edit/Write or re-dispatch with failure output

**If fewer than 3 tasks**: apply changes directly with Edit/Write.

After all tasks complete (either path):
1. Run self-verification: lint / typecheck / tests if available
2. Launch Task agent (subagent_type: "feature-dev:code-reviewer") with:
   - `git diff HEAD`
   - Original task requirements
3. **Critical Evaluation**: Before fixing, critically evaluate each CRITICAL/HIGH finding:
   - **Assess correctness**: Is it technically accurate? Check actual code and project conventions.
   - **Check context**: Does the reviewer have full context, or is it flagging intentional design?
   - **Verify applicability**: Does it improve correctness/security, or is it a style preference / false positive?
   - **If a finding seems incorrect**: Do NOT fix it or count it as an iteration. Launch a new code-reviewer agent with the original diff, the contested findings, and your technical reasoning. Ask it to re-evaluate only the contested items. Accept confirmed findings; drop withdrawn ones. Discussion rounds do NOT count toward the iteration cap.
4. If reviewer finds CRITICAL/HIGH issues (after evaluation): fix directly with Edit/Write and re-review (max 2 rounds)
5. Go to Phase 3

---

### Route B: Large Change — Codex First (max 3 iterations)

**MANDATORY**: When Route B is selected, you MUST NOT use Edit/Write tools for implementation. All implementation MUST go through `mcp__codex__codex`. Claude's role in Route B is context retrieval, verification, and review — not implementation. The ONLY exception is fixing minor regressions found during self-verification (Step B2) that are clearly caused by Codex output (e.g. a missing import, a typo in a generated file).

**No post-routing rationalization**: Once Route B is selected, the routing decision applies to ALL tasks in the plan. Do NOT re-evaluate individual tasks for complexity. If you find yourself thinking "this task is simple enough to do directly" or "this is just a deletion" — STOP. That is the routing decision being second-guessed. The routing already accounted for overall complexity. Every task goes through Codex.

**Codex availability check**: `mcp__codex__codex` MUST be listed in the available tools (either in the tool list or in `<available-deferred-tools>`). If the tool is genuinely absent from both locations (i.e. the MCP server is not configured or not responding), fall back to Route A and announce: "Codex MCP unavailable — falling back to Claude direct implementation." This fallback is ONLY for when `mcp__codex__codex` is technically inaccessible. It MUST NOT be used because a task seems "too simple for Codex" or because Claude judges direct editing would be faster.

**Note on agent spawning**: Codex runs in a sandboxed shell and cannot spawn Claude agents directly. Instead, Claude spawns a subagent per task — each subagent owns its Codex MCP session for that task, keeping both the main context and each Codex session scoped and clean.

Read `codex-architect-role.md` from this skill directory and inject as `developer-instructions` for all Codex calls.

**Step B1 — Dispatch per-task subagents (if plan has 3+ tasks)**

For each implementation task (sequentially):

1. Record `$TASK_SHA` via `git rev-parse HEAD` before dispatching
2. Launch a general-purpose Task agent with the task description, sanitized context files, the `mcp__codex__codex` and `mcp__codex__codex-reply` tools, and instruction:

```
Call mcp__codex__codex with:
- prompt: "Implement the following task.\n\nTask: {task}\n\nContext:\n{sanitized key file contents}"
- sandbox: "workspace-write"
- approval-policy: "on-failure"
- developer-instructions: {content of codex-architect-role.md} + "\nBe concise. Output result only, no reasoning process."

Save the threadId. Run lint/tests scoped to changed files to verify. If failures, call mcp__codex__codex-reply with the threadId and failure output (max 3 retries). Report: files changed, verification result, any unresolved failures.
```

3. After each subagent: run `git diff $TASK_SHA` (scoped to this task only) to verify changes, run scoped lint/tests to confirm no regressions

**Important**: Each subagent owns its own Codex `threadId`. There is no shared threadId across tasks. The outer review loop (Step B3) therefore uses the `code-reviewer` Task agent — not `mcp__codex__codex-reply` — since there is no single session to reply to.

**If plan has fewer than 3 tasks**, call Codex directly:

Call `mcp__codex__codex` (iteration 1) or `mcp__codex__codex-reply` (iterations 2-3):
- prompt (first call): "Implement the following task.\n\nTask: {task}\n\nContext:\n{key file contents}"
- prompt (retry): "Fix the following issues found in code review:\n\n{reviewer feedback verbatim}"
- sandbox: "workspace-write"
- approval-policy: "on-failure"
- developer-instructions: {content of codex-architect-role.md} + "\nBe concise. Output result only, no reasoning process."

Save the returned `threadId`. Reuse it for all subsequent calls.

**Step B2 — Self-Verification**

Run existing lint / typecheck / tests if available (minimal related scope).
If failures: fix regressions before proceeding to review.

**Step B3 — Code Review (Claude Sonnet)**

Launch Task agent (subagent_type: "feature-dev:code-reviewer") with:
- `git diff HEAD`
- Original task requirements

Parse reviewer response:
- No CRITICAL/HIGH issues → approved, go to Phase 3

**Critical Evaluation** (before fixing): Critically evaluate each CRITICAL/HIGH finding:
- **Assess correctness**: Is it technically accurate? Check actual code and project conventions.
- **Check context**: Does the reviewer have full context, or is it flagging intentional design?
- **Verify applicability**: Does it improve correctness/security, or is it a style preference / false positive?
- **If a finding seems incorrect**: Do NOT fix it or count it as an iteration. Launch a new code-reviewer agent with the original diff, the contested findings, and your technical reasoning. Ask it to re-evaluate only the contested items. Accept confirmed findings; drop withdrawn ones. Discussion rounds do NOT count toward the iteration cap.

After evaluation:
- Has CRITICAL/HIGH (3+ task path) → dispatch a Claude subagent per affected task with the accepted reviewer feedback verbatim; there is no shared `threadId` to reply to, so do not call `mcp__codex__codex-reply`. After fixes, re-run `code-reviewer` (max 3 total iterations).
- Has CRITICAL/HIGH (single Codex session path) → call `mcp__codex__codex-reply` with accepted reviewer feedback verbatim, increment iteration count (max 3 total iterations).

After 3 iterations without approval, stop and report status to user.

---

### Phase 3: Delivery

```markdown
## Execution Complete

### Changes
| File | Operation | Description |
|------|-----------|-------------|
| path/to/file | Modified | Description |

### Review Result
- Route: Small (Claude) / Large (Codex, N/3 iterations)
- Code review: Passed / N issues resolved

### Recommended Next Steps
1. [ ] <test step>
2. [ ] <verification step>
```
