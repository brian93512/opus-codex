# opus-codex

Opus plans, Codex executes. A [Claude Code](https://claude.ai/code) skill that uses Claude Opus to produce detailed implementation plans, then hands them off to [OpenAI Codex CLI](https://github.com/openai/codex) for autonomous execution.

## Install — 30 seconds

**Requirements:** [Claude Code](https://claude.ai/code), [Git](https://git-scm.com), [OpenAI Codex CLI](https://github.com/openai/codex) (`npm i -g @openai/codex`)

### Step 1: Install on your machine

Open Claude Code and paste this. Claude does the rest.

> Install opus-codex: run `git clone --single-branch --depth 1 https://github.com/brian93512/opus-codex.git ~/.claude/skills/opus-codex && cd ~/.claude/skills/opus-codex && ./setup` then add an "opus-codex" section to CLAUDE.md that lists the available skill: `/opus-codex`, and mentions it requires `/model opus`. Add a "Default workflow" section that says: for all implementation tasks, use `/opus-codex` (Opus plans, Codex executes).

### Step 2: Add to your repo so teammates get it (optional)

> Add opus-codex to this project: run `cp -Rf ~/.claude/skills/opus-codex .claude/skills/opus-codex && rm -rf .claude/skills/opus-codex/.git && cd .claude/skills/opus-codex && ./setup` then add an "opus-codex" section to this project's CLAUDE.md that lists the available skill: `/opus-codex`, and mentions it requires `/model opus`.

Real files get committed to your repo (not a submodule), so `git clone` just works.

## Usage

1. Start Claude Code
2. Switch to Opus: `/model opus`
3. Run: `/opus-codex`
4. Describe your task
5. Review the plan, choose execution mode (full-auto or manual approvals)
6. Codex executes the plan autonomously
7. `/review` runs automatically on the result

## See it work

```
You:    /model opus
You:    /opus-codex
You:    Add rate limiting middleware to our Express API

Claude: [reads codebase — finds existing middleware patterns, route structure]
        [produces detailed plan: 3 files, code snippets, edge cases, test cases]

        Ready to hand off to Codex. How should it run?
        A) Full-auto    B) Manual approvals    C) Edit plan    D) Cancel

You:    A

Claude: [hands plan to codex exec --full-auto --sandbox workspace-write]
        ...
        Codex executed successfully. Changes to 3 files (65 insertions):

        src/middleware/rate-limit.ts   (new, 38 lines)
        src/app.ts                    (3 lines added)
        tests/rate-limit.test.ts      (new, 24 lines)

        Running verification... all tests pass.

        [auto-runs /review]
        Review: PASS — no issues found. Ready to commit.
```

Opus thinks. Codex types. You ship.

## How it works

1. **Opus plans** — reads your codebase, produces a detailed implementation plan with file paths, code snippets, edge cases, and verification steps
2. **You approve** — review the plan, choose full-auto or manual approvals
3. **Codex executes** — `codex exec` runs the plan autonomously in a sandbox
4. **Auto-review** — runs verification commands and `/review` on the diff

## License

MIT
