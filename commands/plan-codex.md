---
description: "Claude plans with Opus, Codex audits, loop until approved"
argument-hint: "<task description>"
model: claude-opus-4-6
allowed-tools: ["AskUserQuestion", "mcp__codex__codex", "mcp__codex__codex-reply", "Task", "Read", "Glob", "Grep", "Write", "Bash"]
---

> **Deprecated**: This command has been converted to a skill (`/plan-codex`).
> The skill version is recommended for new usage and supports eval-based testing.
> This command is retained for backward compatibility (model pinning, tool restrictions).

# Plan-Codex — Claude Plans, Codex Audits

$ARGUMENTS

---

## Core Protocols

- **Code Sovereignty**: Codex has zero filesystem write access during planning — all file operations by Claude only
- **Stop-Loss**: Do not proceed to next phase until current phase output is validated
- **Language**: Use English when calling tools/models; communicate with user in their language
- **Planning Only**: Never modify production code during this command

---

## Execution Workflow

### Phase 1: Plan Creation

Launch a Task agent (subagent_type: "Plan") with the task description.
The Plan agent will research the codebase and return a structured implementation plan.

If requirements are ambiguous before launching the planner, ask clarifying questions first.

Save the returned plan to `.claude/plan/<feature-name>.md`.

### Phase 2: Codex Audit Loop (max 3 iterations)

Read `~/.claude/prompts/codex/analyzer.md` and inject as `developer-instructions`.

**Call `mcp__codex__codex`** (iteration 1):
- prompt: "Read the plan file at `.claude/plan/<feature-name>.md` and audit it for correctness, completeness, security, and edge cases. Reply APPROVED if solid, or list specific issues to fix."
- sandbox: "read-only"
- approval-policy: "never"
- developer-instructions: {content of ~/.claude/prompts/codex/analyzer.md} + "\nBe concise. Output result only, no reasoning process."

Save the returned `threadId`.

**Parse response**:
- Contains "APPROVED" → update `.claude/plan/<feature-name>.md` with final version, go to Phase 3
- Contains issues → address every issue, revise plan, update `.claude/plan/<feature-name>.md`

**Call `mcp__codex__codex-reply`** (iterations 2-3):
- threadId: {saved threadId}
- prompt: "The plan has been revised to address your feedback. Re-read the plan file at `.claude/plan/<feature-name>.md` and audit it again for correctness, completeness, security, and edge cases. Reply APPROVED if solid, or list specific issues to fix."

After 3 iterations without APPROVED, stop and ask user for direction.

### Phase 3: Deliver

Present the final plan and output:

---
**Plan saved to `.claude/plan/<feature-name>.md`**

Review the plan above. When ready, start a new session and run:
```
/execute-codex .claude/plan/<feature-name>.md
```
---

**Stop here. Do not auto-execute.**
