---
name: opus-codex
version: 1.3.0
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

## Preamble (run first)

```bash
_UPD=$(~/.claude/skills/opus-codex/bin/update-check 2>/dev/null || .claude/skills/opus-codex/bin/update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
```

### If output contains `AUTO_UPGRADE`

The user has opted into auto-upgrade. Immediately run:

```bash
cd ~/.claude/skills/opus-codex && git pull origin main 2>/dev/null || cd .claude/skills/opus-codex && git pull origin main 2>/dev/null || true
```

Tell the user: "opus-codex auto-updated (v{old} → v{new})." and continue with the workflow.

### If output contains `UPGRADE_AVAILABLE`

Use AskUserQuestion to ask:
> opus-codex update available (v{old} → v{new}). How would you like to proceed?
> - A) Yes, upgrade now
> - B) Always keep me up to date (auto-upgrade from now on)
> - C) Not now (snooze 24h)
> - D) Never ask again

**If A (upgrade now):**
```bash
cd ~/.claude/skills/opus-codex && git pull origin main 2>/dev/null || cd .claude/skills/opus-codex && git pull origin main 2>/dev/null || true
```
Tell the user "Updated!" and continue.

**If B (auto-upgrade):**
```bash
~/.claude/skills/opus-codex/bin/update-config set auto_upgrade true 2>/dev/null || .claude/skills/opus-codex/bin/update-config set auto_upgrade true 2>/dev/null || true
cd ~/.claude/skills/opus-codex && git pull origin main 2>/dev/null || cd .claude/skills/opus-codex && git pull origin main 2>/dev/null || true
```
Tell the user "Auto-upgrade enabled. Future updates will apply automatically." and continue.

**If C (not now):**
```bash
~/.claude/skills/opus-codex/bin/update-config snooze 24 2>/dev/null || .claude/skills/opus-codex/bin/update-config snooze 24 2>/dev/null || true
```
Continue with the workflow on the current version.

**If D (never ask again):**
```bash
~/.claude/skills/opus-codex/bin/update-config set update_check false 2>/dev/null || .claude/skills/opus-codex/bin/update-config set update_check false 2>/dev/null || true
```
Tell the user "Update checks disabled. Re-enable anytime by running: `~/.claude/skills/opus-codex/bin/update-config set update_check true`" and continue.

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

## Step 7: Bail-out check

After codex completes, check whether it actually produced useful changes:

```bash
git diff --stat
```

**Bail out if ANY of these are true:**
- Codex exit code was non-zero
- `git diff --stat` shows no changes
- Codex output contains repeated error messages or signs of looping

If bailing out, tell the user:
> Codex execution failed. Want me to implement this directly with Opus instead?

Clean up the plan file and stop. Do NOT attempt to fix Codex's mistakes — that wastes more tokens than just doing it with Opus from the start.

## Step 8: Report results and review

If Codex succeeded (changes exist and no errors):

Show the user:
- Files changed by Codex
- Any warnings from the codex output

Run the verification commands from the plan to confirm correctness.

Then automatically invoke `/review` to review the diff. Do NOT ask the user whether to review — always review by default.
