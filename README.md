# claude-codex

A Claude Code setup for collaborative AI development: Claude plans with Opus, Codex implements or reviews, Sonnet implements or reviews. Five skills (and their equivalent commands) drive a structured plan → implement → review loop.

## Skills (Recommended)

Skills are the primary interface. They support eval-based testing via skill-creator and are easier to iterate on than commands.

| Skill | Trigger | Description |
|-------|---------|-------------|
| `plan-codex` | `/plan-codex <task>` | Claude plans with Opus, Codex audits for correctness/completeness/security until approved |
| `claude-codex` | `/claude-codex <task or plan>` | Claude implements, Codex reviews with APPROVED/WARNING/BLOCKED verdicts |
| `execute-codex` | `/execute-codex <task or plan>` | Smart size-based routing: Claude (small) or Codex (large) implements, code-reviewer reviews |
| `tdd-claude-codex` | `/tdd-claude-codex <task or plan>` | TDD: Claude writes tests, Codex audits tests, Claude implements, Codex reviews implementation |
| `tdd-execute-codex` | `/tdd-execute-codex <task or plan>` | TDD: Claude writes tests, Codex audits tests, smart routing for implementation, code-reviewer reviews |

Commands (`commands/`) are retained for backward compatibility — they support model pinning and explicit `allowed-tools` restrictions that skills cannot enforce.

## How It Works

```
/plan-codex <task>
  Claude Opus + Plan agent    → creates structured plan
  Codex audits the plan       → flags issues (max 3 rounds)
  Plan saved to .claude/plan/<feature>.md

/execute-codex <task or plan file>
  Small change (≤2 files, ≤30 lines) → Claude implements directly
  Large change                        → Codex implements
  feature-dev code-reviewer agent      → reviews diff (max 3 rounds)

/claude-codex <task or plan file>
  Claude implements (any model)       → Edit/Write + self-verify
  Codex reviews uncommitted changes   → returns structured verdict (APPROVED/WARNING/BLOCKED)
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

### 3. feature-dev plugin

This setup uses the `code-reviewer` agent from [claude-plugins-official](https://github.com/anthropics/claude-plugins-official).

Install the plugin in Claude Code:

```
claude plugin add claude-plugins-official/feature-dev
```

## Installation

### 1. Clone and copy files

```bash
git clone https://github.com/<your-username>/claude-codex
cd claude-codex

cp -r skills/*   ~/.claude/skills/
cp -r prompts/*  ~/.claude/prompts/
cp -r commands/* ~/.claude/commands/   # optional: for model pinning / tool restrictions
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

Claude Opus uses the built-in `Plan` agent to research your codebase and produce a structured plan. Codex audits it for correctness, security, and completeness. The approved plan is saved to `.claude/plan/<feature>.md`.

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

Claude implements using the active model (default: Sonnet; override with `/model opus` for heavier tasks). Codex reviews the uncommitted diff via MCP, returning a structured verdict (APPROVED / WARNING / BLOCKED):
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
├── skills/
│   ├── codex-mcp/
│   │   ├── SKILL.md                  # Auto-loaded Codex MCP usage knowledge
│   │   └── reference.md              # Detailed patterns and examples
│   ├── plan-codex/
│   │   ├── SKILL.md                  # /plan-codex skill
│   │   ├── codex-analyzer-role.md    # Bundled Codex role for plan auditing
│   │   └── evals/evals.json          # Eval scenarios for skill-creator
│   ├── claude-codex/
│   │   ├── SKILL.md                  # /claude-codex skill
│   │   └── evals/evals.json
│   ├── execute-codex/
│   │   ├── SKILL.md                  # /execute-codex skill
│   │   ├── codex-architect-role.md   # Bundled Codex role for implementation
│   │   └── evals/evals.json
│   ├── tdd-claude-codex/
│   │   ├── SKILL.md                  # /tdd-claude-codex skill
│   │   ├── codex-analyzer-role.md    # Bundled Codex role for test auditing
│   │   ├── tdd-specialist-role.md    # Bundled TDD specialist role for test writing
│   │   └── evals/evals.json
│   └── tdd-execute-codex/
│       ├── SKILL.md                  # /tdd-execute-codex skill
│       ├── codex-architect-role.md   # Bundled Codex role for implementation
│       ├── tdd-specialist-role.md    # Bundled TDD specialist role for test writing
│       └── evals/evals.json
├── commands/                         # Deprecated — retained for backward compatibility
│   ├── plan-codex.md
│   ├── execute-codex.md
│   ├── claude-codex.md
│   ├── tdd-claude-codex.md
│   └── tdd-execute-codex.md
└── prompts/
    └── codex/
        ├── analyzer.md      # Codex role: plan auditor (source for skill bundles)
        ├── architect.md     # Codex role: implementer (source for skill bundles)
        └── reviewer.md      # Codex role: code reviewer
```

The `prompts/codex/` files are injected into `developer-instructions` on each Codex MCP call to shape its behaviour per phase.

## Credits

- `prompts/codex/` role prompts (`analyzer.md`, `architect.md`, `reviewer.md`) are sourced from [ccg-workflow](https://github.com/fengshao1227/ccg-workflow) by fengshao1227, licensed under MIT.
- TDD specialist prompt adapted from [everything-claude-code](https://github.com/affaan-m/everything-claude-code) by affaan-m, licensed under MIT. `code-reviewer` agent from [claude-plugins-official/feature-dev](https://github.com/anthropics/claude-plugins-official).

## Notes

- Plans are saved to `.claude/plan/` in your project directory — add to `.gitignore` if you don't want them committed
- Max 3 iterations on all loops to keep token usage predictable
- The `codex-mcp` skill auto-loads when Codex MCP tools are relevant, providing Claude with session management patterns and best practices
- `/claude-codex` uses the Codex MCP server for review — Codex runs `git diff HEAD` itself and returns a structured verdict only, minimising token usage
