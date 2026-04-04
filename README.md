# opus-codex

Opus plans, Codex executes. A [Claude Code](https://claude.ai/code) skill that uses Claude Opus to produce detailed implementation plans, then hands them off to [OpenAI Codex CLI](https://github.com/openai/codex) for autonomous execution.

## Prerequisites

- [Claude Code](https://claude.ai/code)
- [OpenAI Codex CLI](https://github.com/openai/codex) — `npm i -g @openai/codex`
- Claude Opus model — `/model opus` in Claude Code

## Quickstart

### Global install (all projects)

```bash
git clone https://github.com/AgentSafe-AI/opus-codex.git ~/.claude/skills/opus-codex && cd ~/.claude/skills/opus-codex && ./setup
```

### Project install (single repo)

```bash
git clone https://github.com/AgentSafe-AI/opus-codex.git .claude/skills/opus-codex && cd .claude/skills/opus-codex && ./setup
```

## Usage

1. Start Claude Code
2. Switch to Opus: `/model opus`
3. Run: `/opus-codex`
4. Describe your task
5. Review the plan, choose execution mode (full-auto or manual approvals)
6. Codex executes the plan autonomously
7. `/review` runs automatically on the result

### Optional: make it the default workflow

Add to your `CLAUDE.md` (global `~/.claude/CLAUDE.md` or project-level):

```markdown
## Default workflow
For all implementation tasks, use `/opus-codex` (Opus plans, Codex executes).
```

## How it works

1. **Opus plans** — reads your codebase, produces a detailed implementation plan with file paths, code snippets, edge cases, and verification steps
2. **You approve** — review the plan, choose full-auto or manual approvals
3. **Codex executes** — `codex exec` runs the plan autonomously in a sandbox
4. **Auto-review** — runs verification commands and `/review` on the diff

## License

MIT
