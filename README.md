# claude-codex

A Claude Code setup for collaborative AI development: Claude plans with Opus, Codex implements or reviews, Sonnet implements or reviews. Three commands drive a structured plan → implement → review loop.

## How It Works

```
/plan-codex <task>
  Claude Opus + planner agent → creates structured plan
  Codex audits the plan       → flags issues (max 3 rounds)
  Plan saved to .claude/plan/<feature>.md

/execute-codex <task or plan file>
  Small change (≤2 files, ≤30 lines) → Claude implements directly
  Large change                        → Codex implements
  Sonnet code-reviewer agent          → reviews diff (max 3 rounds)

/claude-codex <task or plan file>
  Claude implements (any model)       → Edit/Write + self-verify
  Codex reviews uncommitted changes   → structured CRITICAL/HIGH/MEDIUM/LOW output
  Claude fixes CRITICAL/HIGH issues   → re-reviews (max 3 rounds)
  MEDIUM/LOW issues                   → user decides before delivery
```

## Dependencies

### 1. Claude Code

Install from [claude.ai/code](https://claude.ai/code).

### 2. OpenAI Codex CLI

```bash
npm install -g @openai/codex
```

Authenticate with your OpenAI API key:

```bash
export OPENAI_API_KEY=your-key-here
```

### 3. everything-claude-code plugin

This setup uses the `planner` and `code-reviewer` agents from [everything-claude-code](https://github.com/affaan-m/everything-claude-code).

Follow the installation instructions in that repo, then install the plugin in Claude Code:

```
/plugin install everything-claude-code@everything-claude-code
```

## Installation

### 1. Clone and copy files

```bash
git clone https://github.com/<your-username>/claude-codex
cd claude-codex

cp -r commands/* ~/.claude/commands/
cp -r skills/*   ~/.claude/skills/
cp -r prompts/*  ~/.claude/prompts/
```

### 2. Configure Codex

Suppress reasoning tokens by adding the following to `~/.codex/config.toml`:

```toml
hide_agent_reasoning = true
```

### 3. Add Codex as an MCP server

```bash
claude mcp add codex -s user -- codex -c model=gpt-5.3-codex -c model_reasoning_effort=high mcp-server
```

Adjust `model` and `model_reasoning_effort` (`low`/`medium`/`high`/`xhigh`) to your preference.

### 4. Restart Claude Code

```
/exit
claude
```

Verify the MCP server is connected:

```
/mcp
```

## Usage

### Planning

```
/plan-codex Add JWT authentication to the API
```

Claude Opus uses the `planner` agent to research your codebase and produce a structured plan. Codex audits it for correctness, security, and completeness. The approved plan is saved to `.claude/plan/<feature>.md`.

### Execution (Codex implements, Sonnet reviews)

```
/execute-codex .claude/plan/jwt-auth.md
```

Or skip the plan step for a direct task:

```
/execute-codex Fix the null pointer in user.service.ts
```

Claude routes automatically:
- **Small change** (≤2 files, ≤30 lines, no new logic) — Claude implements directly, Sonnet reviews
- **Large change** — Codex implements, Sonnet reviews and feeds back (max 3 rounds)

### Execution (Claude implements, Codex reviews)

```
/claude-codex .claude/plan/jwt-auth.md
```

Or for a direct task:

```
/claude-codex Add input validation to the registration endpoint
```

Claude implements using the active model (default: Sonnet; override with `/model opus` for heavier tasks). Codex reviews the uncommitted diff using `codex review --uncommitted`:
- **BLOCKED** (CRITICAL issues) — Claude fixes, Codex re-reviews (max 3 rounds)
- **WARNING** (HIGH issues) — Claude fixes, Codex re-reviews (max 3 rounds)
- **MEDIUM/LOW issues** — surfaced to user; user decides whether to fix before delivery

### Model Override

Commands have defaults (`plan-codex` → Opus, `execute-codex` → Sonnet). Override per session:

```
/model opus
/execute-codex .claude/plan/complex-feature.md
```

## File Structure

```
~/.claude/
├── commands/
│   ├── plan-codex.md        # /plan-codex command
│   ├── execute-codex.md     # /execute-codex command
│   └── claude-codex.md      # /claude-codex command
├── skills/
│   └── codex-mcp/
│       ├── SKILL.md         # Auto-loaded Codex MCP usage knowledge
│       └── reference.md     # Detailed patterns and examples
└── prompts/
    └── codex/
        ├── analyzer.md      # Codex role: plan auditor
        ├── architect.md     # Codex role: implementer
        └── reviewer.md      # Codex role: code reviewer
```

The `prompts/codex/` files are injected into `developer-instructions` on each Codex MCP call to shape its behaviour per phase.

## Credits

- `prompts/codex/` role prompts (`analyzer.md`, `architect.md`, `reviewer.md`) are sourced from [ccg-workflow](https://github.com/fengshao1227/ccg-workflow) by fengshao1227, licensed under MIT.
- `planner` and `code-reviewer` agents are from [everything-claude-code](https://github.com/affaan-m/everything-claude-code), licensed under MIT.

## Notes

- Plans are saved to `.claude/plan/` in your project directory — add to `.gitignore` if you don't want them committed
- Max 3 iterations on all loops to keep token usage predictable
- The `codex-mcp` skill auto-loads when Codex MCP tools are relevant, providing Claude with session management patterns and best practices
- `/claude-codex` uses the `codex review --uncommitted` CLI (not the MCP server) — no MCP connection required for that command
