# Codex MCP Reference — Detailed Patterns

## Starting a Review Session

```
mcp__codex__codex:
  prompt: |
    Review this implementation plan for correctness, completeness,
    security, and edge cases. Reply APPROVED if solid, or list
    specific issues to fix.

    <plan>
    {plan content here}
    </plan>
  sandbox: "read-only"
  approval-policy: "never"
  developer-instructions: |
    You are a senior architect reviewing implementation plans.
    Be thorough but practical. Only flag real issues, not stylistic preferences.
    Be concise. Do not include your reasoning process in the response.
    Only output the result.
```

## Starting an Implementation Session

```
mcp__codex__codex:
  prompt: |
    Implement the following task. Write production-quality code.

    {task description or plan content}
  sandbox: "workspace-write"
  approval-policy: "on-failure"
  developer-instructions: |
    Write clean, tested, production code. Follow existing project conventions.
    Do not modify unrelated files. Be concise.
    Do not include your reasoning process in the response.
    Only output the result.
```

## Continuing with Feedback

```
mcp__codex__codex-reply:
  threadId: "{saved threadId from initial call}"
  prompt: |
    Fix these issues from code review:
    - auth.ts:42 — missing null check on user.email
    - api.ts:15 — no rate limiting on /login endpoint
    Reimplement with these fixes.
```

## Loop Termination Signals

| Loop Type | Termination Signal | Who Decides |
|-----------|-------------------|-------------|
| Plan review | Codex response contains "APPROVED" | Codex |
| Execute review | Claude reviewer response contains "APPROVED" | Claude |

Always enforce max iteration count (default: 5).
After iteration 3, consider asking the user if the approach is viable.

## Anti-Patterns

- **Don't start new sessions per iteration** — wastes context, loses conversation history
- **Don't summarize reviewer feedback** — Codex needs exact file:line references to fix issues
- **Don't use `danger-full-access` sandbox** — `workspace-write` is sufficient for implementation
- **Don't skip `developer-instructions`** — it shapes response quality significantly
- **Don't omit conciseness instruction** — Codex may include lengthy reasoning that wastes context tokens
- **Don't add `2>/dev/null` to tool calls** — MCP handles this at the server process level

## Parameter Quick Reference

| Parameter | Plan Review | Implementation |
|-----------|------------|----------------|
| sandbox | `read-only` | `workspace-write` |
| approval-policy | `never` | `on-failure` |
| developer-instructions | Architect reviewer persona | Coding standards + conciseness |
| cwd | Project root | Project root |
