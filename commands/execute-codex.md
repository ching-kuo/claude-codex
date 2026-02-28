---
description: "Codex implements (large), Claude implements (small), Sonnet reviews"
argument-hint: "<task description or path/to/plan.md>"
model: claude-sonnet-4-6
allowed-tools: ["mcp__codex__codex", "mcp__codex__codex-reply", "Task", "Read", "Glob", "Grep", "Write", "Edit", "Bash"]
---

# Execute-Codex — Smart Routing: Claude (small) / Codex (large)

$ARGUMENTS

---

## Core Protocols

- **Code Sovereignty**: Claude is the final authority — review and approve all changes before delivery
- **Stop-Loss**: Do not proceed to next phase until current phase output is validated
- **Language**: Use English when calling tools/models; communicate with user in their language

---

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

1. Apply changes using Edit/Write
2. Run self-verification: lint / typecheck / tests if available
3. Launch Task agent (subagent_type: "everything-claude-code:code-reviewer") with:
   - `git diff HEAD`
   - Original task requirements
4. If reviewer finds CRITICAL/HIGH issues: fix directly with Edit/Write and re-review (max 2 rounds)
5. Go to Phase 3

---

### Route B: Large Change — Codex First (max 3 iterations)

Read `~/.claude/prompts/codex/architect.md` and inject as `developer-instructions` for implementation calls.

**Step B1 — Codex Implements**

Call `mcp__codex__codex` (iteration 1) or `mcp__codex__codex-reply` (iterations 2-3):
- prompt (first call): "Implement the following task.\n\nTask: {task}\n\nContext:\n{key file contents}"
- prompt (retry): "Fix the following issues found in code review:\n\n{reviewer feedback verbatim}"
- sandbox: "workspace-write"
- approval-policy: "on-failure"
- developer-instructions: {content of ~/.claude/prompts/codex/architect.md} + "\nBe concise. Output result only, no reasoning process."

Save the returned `threadId`. Reuse it for all subsequent calls.

**Step B2 — Self-Verification**

Run existing lint / typecheck / tests if available (minimal related scope).
If failures: fix regressions before proceeding to review.

**Step B3 — Code Review (Claude Sonnet)**

Launch Task agent (subagent_type: "everything-claude-code:code-reviewer") with:
- `git diff HEAD`
- Original task requirements

Parse reviewer response:
- No CRITICAL/HIGH issues → approved, go to Phase 3
- Has issues → call `mcp__codex__codex-reply` with reviewer feedback verbatim, increment iteration count

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
