---
name: opus-codex
version: 1.6.0
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

## Step 0: Preflight check (run BEFORE any planning)

Verify Codex CLI is installed and authenticated before spending tokens on planning:

```bash
codex --version 2>&1 || echo "FAIL: codex CLI not installed"
AUTH_MARKER="$HOME/.opus-codex/auth-verified"
if [ -f "$AUTH_MARKER" ]; then
  AGE=$(( $(date +%s) - $(stat -f %m "$AUTH_MARKER" 2>/dev/null || stat -c %Y "$AUTH_MARKER" 2>/dev/null || echo 0) ))
  if [ "$AGE" -lt 604800 ]; then
    echo "AUTH: cached (valid)"
  else
    echo "AUTH: expired"
  fi
else
  echo "AUTH: not verified"
fi
```

If codex is not installed (output contains "FAIL"), tell the user:
> Codex CLI is not installed. Install it first: `npm i -g @openai/codex`

If AUTH is "cached (valid)", skip auth test and proceed to Step 1.

If AUTH is "expired" or "not verified", run a quick auth test:

```bash
codex exec --full-auto --sandbox workspace-write - <<< "echo hello" 2>&1 | head -5
```

If the auth test succeeds, write the marker:
```bash
mkdir -p ~/.opus-codex && touch ~/.opus-codex/auth-verified
```

If it fails with an auth error, tell the user:
> Codex is not authenticated. Either:
> - Run `! codex auth login` to log in interactively, or
> - Set your API key: `! export OPENAI_API_KEY=sk-...`
>
> Then re-run `/opus-codex`.

**STOP here if either check fails.** Do NOT proceed to planning — it would waste Opus tokens on a plan that can't execute.

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
4. **Implementation steps** — numbered, each step actionable and specific. Describe INTENT and structure, not full code. Codex is smart enough to write code from clear descriptions. Bad: "Create store.py with this exact code: [150 lines]". Good: "Create store.py: JobStore class, thread-safe with Lock, CRUD methods, supports filtering + pagination."
5. **Edge cases** — anything to watch out for
6. **Verification** — exact commands to run and expected output
7. **Cleanup** — end every plan with: "Delete any files not listed above that were created during execution (e.g., sitecustomize.py, __pycache__, shim directories)."

Keep plans lean. Codex input tokens cost money too. A 200-line intent-based plan works as well as a 1000-line code-heavy plan.

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

**IMPORTANT: Pipe codex output to a file, NOT into the conversation.** Codex stdout can be 500+ lines. If it lands in the conversation, it gets re-sent as cache read on every subsequent turn, inflating cost by ~20k tokens per turn. Keep it out of context.

```bash
CODEX_LOG=$(mktemp /tmp/codex-log-XXXXXX.txt)
```

**If A (full-auto):**
```bash
codex exec --full-auto --sandbox workspace-write - < "$PLAN_FILE" > "$CODEX_LOG" 2>&1
echo "EXIT_CODE: $?"
```

**If B (manual approvals):**
```bash
codex exec --sandbox workspace-write - < "$PLAN_FILE" > "$CODEX_LOG" 2>&1
echo "EXIT_CODE: $?"
```

**If C:** Let the user describe changes, update the plan file with Edit, then re-run Step 5.

**If D:** Clean up and stop.
```bash
rm -f "$PLAN_FILE"
```

## Step 7: Check results, review, and report (ONE combined step)

Run this single command block to get everything you need in ONE turn:

```bash
echo "=== CODEX SUMMARY ==="
echo "--- Errors ---"
grep -i -n 'error\|fail\|traceback\|exception\|abort' "$CODEX_LOG" 2>/dev/null || echo "(none)"
echo "--- Test results ---"
grep -i -n 'passed\|failed\| ok\|PASSED\|FAILED\|tests\? ran\|test.*complete' "$CODEX_LOG" 2>/dev/null || echo "(none found)"
echo "--- Last 15 lines ---"
tail -15 "$CODEX_LOG"
echo ""
echo "=== GIT DIFF STAT ==="
git diff --stat
echo ""
echo "=== GIT DIFF ==="
git diff
echo ""
echo "=== CLEANUP ==="
rm -f sitecustomize.py 2>/dev/null
rm -rf __pycache__ .codex 2>/dev/null
rm -f "$PLAN_FILE" "$CODEX_LOG"
echo "Done"
```

This extracts the important signals from Codex output (errors, test results, final status) without dumping 500+ lines into the conversation context. The grep lines are small and targeted.

**NEVER run additional commands after this.** No `git status`, no `git log`, no reading individual files. This one command is all you need.

### Bail out if ANY of these are true:
- Codex exit code was non-zero (from Step 6)
- `git diff --stat` shows no changes
- Codex tail shows repeated errors or looping

If bailing out, tell the user:
> Codex execution failed. Want me to implement this directly with Opus instead?

### If Codex succeeded, review the diff and report:

1. **Test results** — scan the Codex tail for test result lines (e.g., "X passed", "OK"). Report: "Tests: X passed (verified by Codex)". **NEVER re-run tests.** Codex already ran them.
2. **Diff review** — check the `git diff` output for correctness, missing imports, bugs, files that should have changed but didn't.
3. **Report** — show files changed, test results, and review findings.

Do NOT invoke `/review` or any external skill — this skill must work standalone.
