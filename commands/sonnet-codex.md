---
description: "Claude implements, Codex reviews uncommitted changes"
argument-hint: "<task description or path/to/plan.md>"
model: claude-sonnet-4-6
allowed-tools: ["Task", "Read", "Glob", "Grep", "Write", "Edit", "Bash"]
---

# Sonnet-Codex — Claude Implements, Codex Reviews

$ARGUMENTS

---

## Core Protocols

- **Sovereignty**: Claude implements; Codex is the external reviewer — do not skip review
- **Stop-Loss**: Do not proceed to the next phase until the current phase output is validated
- **Language**: Use English when calling tools/models; communicate with user in their language

---

## Execution Workflow

### Phase 0: Read Plan

1. If argument is a file path, read it and extract: task description, implementation steps, key files
2. If no plan file, use the argument as the task description directly
3. Confirm with user before proceeding if key context is missing

### Phase 1: Context Retrieval

Read key files using Read, Glob, Grep. Confirm complete context before proceeding.

### Phase 2: Claude Implements

Apply all changes using Edit/Write tools directly.

Run self-verification after implementing:
- Lint / typecheck / tests if available (minimal related scope)
- Fix any regressions before proceeding to review

### Phase 3: Codex Review (max 3 iterations)

**Step R1 — Run Codex Review**

```bash
codex review --uncommitted "Review for correctness, security, and code quality. Group findings by severity: CRITICAL, HIGH, MEDIUM, LOW. End with a single verdict line: APPROVED (no CRITICAL/HIGH issues), WARNING (HIGH issues only), or BLOCKED (CRITICAL issues found)."
```

Parse the verdict line at the end of Codex output:
- **APPROVED** → go to Phase 4
- **WARNING** → fix HIGH issues with Edit/Write, re-run review, increment iteration count
- **BLOCKED** → fix CRITICAL issues with Edit/Write, re-run review, increment iteration count

**Step R2 — Fix and Re-review**

Address each CRITICAL or HIGH issue found by Codex:
- Fix directly with Edit/Write
- Re-run the same `codex review` command
- Repeat until APPROVED or 3 iterations reached

After 3 iterations without APPROVED, stop and report remaining issues to user.

**Step R3 — MEDIUM / LOW Issues**

After CRITICAL/HIGH are resolved, collect any remaining MEDIUM and LOW issues from the last review output.
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
- Implemented by: Claude Sonnet
- Codex review: APPROVED / N issues resolved (N/3 iterations)
- Remaining MEDIUM/LOW issues: N (skipped by user) / none

### Recommended Next Steps
1. [ ] <test step>
2. [ ] <verification step>
```
