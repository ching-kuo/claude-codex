---
description: "Claude implements, Codex reviews uncommitted changes"
argument-hint: "<task description or path/to/plan.md>"
model: claude-sonnet-4-6
allowed-tools: ["AskUserQuestion", "Task", "Read", "Glob", "Grep", "Write", "Edit", "Bash", "mcp__codex__codex", "mcp__codex__codex-reply"]
---

# Claude-Codex — Claude Implements, Codex Reviews

$ARGUMENTS

---

## Core Protocols

- **Sovereignty**: Claude implements; Codex is the external reviewer — do not skip review
- **Stop-Loss**: Do not proceed to the next phase until the current phase output is validated
- **Language**: Use English when calling tools/models; communicate with user in their language
- **MCP for review**: Always call Codex review via `mcp__codex__codex`. Do NOT use Bash `codex review` — it produces verbose output that wastes tokens
- **Context Sanitization**: Never pass `.env`, secrets, tokens, API keys, or credentials to any external agent or MCP. Exclude files matching `.env*`, `*secret*`, `*credential*`, `*.pem`, `*.key`. Redact inline secrets before sending.

---

## Execution Workflow

### Phase 0: Read Plan

1. If argument is a file path, read it and extract: task description, implementation steps, key files
2. If no plan file, use the argument as the task description directly
3. Confirm with user before proceeding if key context is missing

### Phase 1: Context Retrieval

Read key files using Read, Glob, Grep. Confirm complete context before proceeding.

### Phase 2: Claude Implements

**If the plan has 3+ implementation tasks**, dispatch each task to a subagent to keep the main context clean:

For each task (sequentially — next task starts only after current task is verified):
1. Record `$TASK_SHA` via `git rev-parse HEAD` before dispatching
2. Launch a general-purpose Task agent with:
   - The specific task description and acceptance criteria
   - Sanitized key file contents it needs to read/modify
   - Instruction: "Implement only this task. Run available lint/tests scoped to changed files after implementing. Report: files changed, verification result, any blockers."
3. After the subagent completes: run `git diff $TASK_SHA` (scoped to this task only) to verify what changed, run scoped lint/tests to confirm no regressions
4. If the subagent reports a blocker or verification fails: fix directly with Edit/Write or re-dispatch with the failure output

**If the plan has fewer than 3 tasks**, implement directly with Edit/Write.

After all tasks complete (either path):
- Run lint / typecheck / tests if available (minimal related scope)
- Fix any regressions before proceeding to review

### Phase 3: Codex Review (max 3 iterations)

**Step R1 — Run Codex Review via MCP**

Call Codex via `mcp__codex__codex` with:
- `developer-instructions`: `"Be concise. Return only the structured verdict format. No prose."`
- `prompt`:

```
Run `git diff HEAD` to see all uncommitted changes, then review them.

Return ONLY a structured verdict in this exact format — no explanations, no prose:

VERDICT: APPROVED | WARNING | BLOCKED

CRITICAL: <list or 'none'>
HIGH: <list or 'none'>
MEDIUM: <list or 'none'>
LOW: <list or 'none'>
```

Save the returned `threadId` for follow-up replies.

Classify the verdict:
- **APPROVED** — no CRITICAL or HIGH → go to Phase 4
- **WARNING** — HIGH issues only → fix all, increment iteration, re-review
- **BLOCKED** — CRITICAL issues → fix all, increment iteration, re-review

**Step R2 — Fix and Re-review**

Address ALL CRITICAL and HIGH issues before re-reviewing:
- Collect every CRITICAL/HIGH finding from the last review
- Fix them all with Edit/Write in one batch
- Run available tests/lint to verify
- Re-call Codex via `mcp__codex__codex-reply` (reuse threadId) with:

```
Run `git diff HEAD` again to see the updated changes after fixes, then re-review.

Return ONLY the structured verdict in the same format.
```

One review per iteration, not one review per fix. Stop after 3 iterations without APPROVED.

After 3 iterations without APPROVED, stop and report remaining issues to user.

**Step R3 — MEDIUM / LOW Issues**

After CRITICAL/HIGH are resolved, collect any remaining MEDIUM and LOW issues from the last review.
If any exist, ask the user:

> "Codex flagged N MEDIUM/LOW issue(s) that were not fixed:
> - [list issues]
> Should I fix these before delivery, or proceed as-is?"

Wait for user response before proceeding.

---

### Phase 4: Delivery

```markdown
## Execution Complete

### Changes
| File | Operation | Description |
|------|-----------|-------------|
| path/to/file | Modified | Description |

### Review Result
- Implemented by: Claude (active model)
- Codex review: APPROVED / N issues resolved (N/3 iterations)
- Remaining MEDIUM/LOW issues: N (skipped by user) / none

### Recommended Next Steps
1. [ ] <test step>
2. [ ] <verification step>
```
