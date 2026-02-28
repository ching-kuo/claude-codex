---
name: codex-mcp
description: How to use OpenAI Codex via MCP tools for code generation, plan review, and collaborative implementation loops. Use when calling Codex for implementation, review, or any cross-model collaboration.
---

# Codex MCP Usage Guide

## Tools

| Tool | Purpose |
|------|---------|
| `mcp__codex__codex` | Start a new Codex session. Returns a threadId + response. |
| `mcp__codex__codex-reply` | Continue an existing session using threadId. |

## Session Management

Codex sessions are **stateful** via threadId. Always:
1. Save the `threadId` from the first `mcp__codex__codex` call
2. Use `mcp__codex__codex-reply` with that threadId for all follow-ups
3. Never start a new session when continuing the same task — context carries forward

## Key Parameters

### sandbox
| Mode | When to Use |
|------|-------------|
| `read-only` | Plan review, code audit, Q&A — Codex reads but cannot write files |
| `workspace-write` | Implementation — Codex writes files in the project directory |

### approval-policy
| Policy | When to Use |
|--------|-------------|
| `never` | Review-only tasks where no shell commands are needed |
| `on-failure` | Implementation — auto-approve commands, intervene on errors |

### developer-instructions
Inject a persona or constraints into Codex without polluting the main prompt.
Always include: `"Be concise. Do not include your reasoning process in the response. Only output the result."`

## Prompting Best Practices

1. **Termination signals**: Ask Codex to reply "APPROVED" or "CHANGES NEEDED" — gives clear loop control
2. **Feedback verbatim**: When sending review feedback via codex-reply, quote specific issues exactly. Do not summarize.
3. **Concise responses**: Always instruct Codex to be concise via developer-instructions to save tokens
4. **Max iterations**: Always enforce a hard cap (default: 5) on review loops

## stderr Suppression

The MCP server config uses `2>/dev/null` at the process level.
Individual tool calls do NOT need stderr suppression — MCP returns structured JSON-RPC responses only.

See reference.md for detailed examples and patterns.
