---
name: opus-codex
version: 1.1.0
description: |
  Opus plans, Codex executes. Use Opus to produce a detailed implementation plan,
  then hand it off to `codex exec` for autonomous execution. The user should
  already be on Opus when invoking this skill (use /model opus first).
  Use when asked to "opus plan", "plan with opus", "opus then codex", or
  "plan and execute".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# Opus → Codex Workflow

## Step 1: Confirm model and understand the task

Check that the user is on Opus. If not, tell them:
> Switch to Opus first with `/model opus`, then re-run `/opus-codex`.

If the user hasn't described what to build, use AskUserQuestion to ask:
> What do you want to implement? Be as specific as you can — the more detail, the better the plan.

## Step 2: Gather codebase context

Before planning, read the relevant parts of the codebase. At minimum:

```bash
# repo structure overview
find . -type f -not -path './.git/*' -not -path './node_modules/*' -not -path './.venv/*' | head -100
```

Read key files the task touches. The plan must reference real paths and existing patterns.

## Step 3: Produce the plan

Think deeply about the task and produce a detailed implementation plan. The plan must include:

1. **Goal** — one sentence summary
2. **Codebase context** — key files, patterns, and conventions Codex needs to follow
3. **Files to create or modify** — explicit list with full paths
4. **Implementation steps** — numbered, each step actionable and specific with code snippets where helpful
5. **Edge cases** — anything to watch out for
6. **Verification** — exact commands to run and expected output

Be prescriptive. Codex will follow this literally. Reference real file paths and existing code patterns.

## Step 4: Write the plan to a temp file

Write the plan using a heredoc:

```bash
PLAN_FILE=$(mktemp /tmp/opus-plan-XXXXXX.md)
cat > "$PLAN_FILE" << 'PLAN_EOF'
... plan content here ...
PLAN_EOF
echo "Plan written to: $PLAN_FILE"
wc -l "$PLAN_FILE"
```

## Step 5: Show plan and get approval

Display the full plan to the user, then use AskUserQuestion:
> Ready to hand off to Codex. How should it run?
- A) Full-auto — Codex executes without asking (fast, uses sandbox)
- B) Manual approvals — Codex asks before each command (safer)
- C) Edit plan first — I'll revise, then re-ask
- D) Cancel

## Step 6: Execute with Codex

**If A (full-auto):**
```bash
codex exec --full-auto --sandbox workspace-write - < "$PLAN_FILE" 2>&1
```

**If B (manual approvals):**
```bash
codex exec --sandbox workspace-write - < "$PLAN_FILE" 2>&1
```

Note: pipe the plan via stdin (`-` flag) to avoid shell argument length limits.

**If C:** Let the user describe changes, update the plan file with Edit, then re-run Step 5.

**If D:** Clean up and stop.
```bash
rm -f "$PLAN_FILE"
```

## Step 7: Report results and review

After codex completes:

```bash
git diff --stat
```

Show the user:
- Files changed by Codex
- Whether the verification commands from the plan pass
- Any errors from the codex output

Run the verification commands from the plan to confirm correctness.

Then automatically invoke `/review` to review the diff. Do NOT ask the user whether to review — always review by default.
