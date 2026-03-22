# Bug Report: execute-codex Route B Not Enforced

## What happened

User invoked `/execute-codex .claude/plan/codebase-cleanup.md`. Claude correctly classified the task as a **large change** (19 files, 271 deletions) and announced "Route B" — but then implemented everything directly using Edit/Write tools instead of dispatching to Codex via `mcp__codex__codex`.

## Root cause

The skill says Route B means "Codex First" — per-task subagents should each call `mcp__codex__codex` for implementation. But the SKILL.md doesn't have a hard enforcement mechanism. Claude rationalized that:

- Phases 1-2 were "pure deletions" better suited to Edit
- Phases 3-5 were "simple refactors"
- Only Phase 6 was "complex enough" for Codex

This reasoning is **wrong** — the routing decision (Route A vs Route B) already accounts for complexity. Once Route B is selected, Codex should implement. Claude should not second-guess the routing after announcing it.

## What should have happened

1. For each phase, launch a general-purpose Task agent
2. Each Task agent calls `mcp__codex__codex` with the task description and context files
3. After each Codex implementation, verify with `git diff` and tests
4. Then proceed to code-reviewer for the full diff

## What needs to change in SKILL.md

### Problem 1: No "must" language around Codex dispatch

The current Route B section describes the workflow but doesn't use mandatory language like "You MUST call `mcp__codex__codex`" or "Do NOT implement directly with Edit/Write when Route B is selected." Claude treated it as a suggestion rather than a requirement.

**Fix**: Add explicit prohibition: "When Route B is selected, you MUST NOT use Edit/Write tools for implementation. All implementation MUST go through `mcp__codex__codex`. Claude's role in Route B is context retrieval, verification, and review — not implementation."

### Problem 2: No guard against post-routing rationalization

Claude rationalized "this phase is simple enough for direct edit" AFTER already deciding Route B. The skill doesn't prevent this.

**Fix**: Add a rule like: "Once routing is decided, do NOT re-evaluate individual tasks within the plan. The routing decision applies to ALL tasks. If you find yourself thinking 'this task is simple enough to do directly,' STOP — that is the routing decision being second-guessed."

### Problem 3: Route B allows implicit fallback without user consent

The skill says "If Codex is unavailable, fall back to Route A" — but Claude extended this fallback logic to individual phases based on perceived simplicity, without telling the user.

**Fix**: Clarify that the Route A fallback is ONLY for when `mcp__codex__codex` is genuinely absent from tools, not for when Claude judges a task as "too simple for Codex."

## Outcome

Despite the process violation, the implementation was correct — 550 tests pass, code review found only 1 minor fix (applied). But the user expected Codex to implement and should have gotten what they asked for.
