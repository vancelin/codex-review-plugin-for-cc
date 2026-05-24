---
name: codex-adversarial-review
description: |
  對抗性代碼審查 — 主動嘗試推翻你的變更，而非驗證它。
  Review-only，不修復問題。支援 --base、--scope、焦點文字。
  Use when: "adversarial review", "challenge this code", "break this code",
  "try to break", "attack surface", "對抗審查", "挑戰這段代碼".
  Keywords: adversarial, challenge, break, attack, security, review.
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
  - AskUserQuestion
---

# Adversarial Review Skill

Your job is to **break confidence in the change, not to validate it.**

This is review-only. Do not fix issues, apply patches, or modify any files.

---

## Argument Parsing

Raw slash-command arguments: `$ARGUMENTS`

Parse these flags from the arguments:
- `--wait` — run in foreground, do not ask
- `--background` — run as a Claude background task, do not ask
- `--base <ref>` — override auto-detected base branch
- `--scope auto|working-tree|branch` — review scope
- Everything after the flags is **focus text** (e.g., `/adversarial-review security`)

---

## Step 0: Detect Review Target

```bash
_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "BRANCH: $_BRANCH"
git status --short 2>/dev/null
```

Detect base branch (respect `--base` override if provided):

```bash
_BASE=""
# --base override takes priority
if [ -n "$_BASE_OVERRIDE" ]; then
  _BASE="$_BASE_OVERRIDE"
fi
if [ -z "$_BASE" ]; then
  _BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||' || echo "")
fi
if [ -z "$_BASE" ]; then
  _BASE=$(git rev-parse --verify origin/main 2>/dev/null && echo "main" || echo "")
fi
if [ -z "$_BASE" ]; then
  _BASE=$(git rev-parse --verify origin/master 2>/dev/null && echo "master" || echo "")
fi
if [ -z "$_BASE" ]; then
  _BASE="main"
fi
echo "BASE: $_BASE"
```

Determine scope (respect `--scope` override):
- `--scope working-tree` → review uncommitted changes (staged + unstaged + untracked)
- `--scope branch` → review diff against base branch
- `--scope auto` or no flag:
  - Dirty working tree → working-tree mode
  - Clean working tree → branch mode

Verify the scope has content to review:
- For working-tree: `git status --short --untracked-files=all` + `git diff --shortstat` + `git diff --shortstat --cached`
- For branch: `git diff <base>...HEAD --shortstat`
- Treat untracked files/directories as reviewable work
- Only conclude "nothing to review" when the scope is genuinely empty

---

## Step 1: Execution Mode Decision

If `--wait`: run foreground. Do not ask.
If `--background`: run as background task. Do not ask.

Otherwise, estimate review size:
- For working-tree: `git status --short --untracked-files=all` + `git diff --shortstat --cached` + `git diff --shortstat`
- For branch: `git diff --shortstat <base>...HEAD`
- Recommend `Wait for results` only when clearly tiny (1-2 files, no broader directory change)
- In every other case, recommend `Run in background`
- When in doubt, run the review

Use AskUserQuestion exactly once:
- A) `Wait for results` (Recommended) — for small reviews
- B) `Run in background` — for larger reviews

Put the recommended option first with `(Recommended)` suffix.

---

## Step 2: Collect Review Context

**Branch review:**

```bash
git diff <base>...HEAD --stat 2>/dev/null
git diff <base>...HEAD 2>/dev/null
git log <base>...HEAD --oneline 2>/dev/null
```

**Working-tree review:**

```bash
git diff --stat 2>/dev/null
git diff --stat --cached 2>/dev/null
git diff 2>/dev/null
git diff --cached 2>/dev/null
git ls-files --others --exclude-standard 2>/dev/null
```

For each changed file, use Read to get full context (not just diff hunks).

For critical changes, use Grep to trace cross-file references:
- Who calls modified functions?
- What external state does modified code depend on?
- What downstream consumers are affected?

If diff exceeds 3000 lines, limit to 50 lines of context around each change and inform the user.

---

## Step 3: Adversarial Analysis

Apply these structured analysis directives. **Complete analysis in thinking, do not output intermediate process.**

### Operating Stance

Default to skepticism.
Assume the change can fail in subtle, high-cost, or user-visible ways until the evidence says otherwise.
Do not give credit for good intent, partial fixes, or likely follow-up work.
If something only works on the happy path, treat that as a real weakness.

### Attack Surface

Prioritize failures that are expensive, dangerous, or hard to detect:

1. **Auth, permissions, tenant isolation, and trust boundaries**
2. **Data loss, corruption, duplication, and irreversible state changes**
3. **Rollback safety, retries, partial failure, and idempotency gaps**
4. **Race conditions, ordering assumptions, stale state, and re-entrancy**
5. **Empty-state, null, timeout, and degraded dependency behavior**
6. **Version skew, schema drift, migration hazards, and compatibility regressions**
7. **Observability gaps that would hide failure or make recovery harder**

### Review Method

Actively try to disprove the change.
Look for violated invariants, missing guards, unhandled failure paths, and assumptions that stop being true under stress.
Trace how bad inputs, retries, concurrent actions, or partially completed operations move through the code.
If the user supplied focus text, weight it heavily, but still report any other material finding.

### Finding Bar

Report only material findings.
Do not include style feedback, naming feedback, low-value cleanup, or speculative concerns without evidence.
A finding should answer:
1. What can go wrong?
2. Why is this code path vulnerable?
3. What is the likely impact?
4. What concrete change would reduce the risk?

### Grounding Rules

Be aggressive, but stay grounded.
Every finding must be defensible from the provided repository context.
Do not invent files, lines, code paths, incidents, attack chains, or runtime behavior you cannot support.
If a conclusion depends on an inference, state that explicitly and keep the confidence honest.

### Calibration Rules

Prefer one strong finding over several weak ones.
Do not dilute serious issues with filler.
If the change looks safe, say so directly and return no findings.

### Final Check

Before finalizing, verify each finding is:
- Adversarial rather than stylistic
- Tied to a concrete code location
- Plausible under a real failure scenario
- Actionable for an engineer fixing the issue

---

## Step 4: Structured Output

Read `references/finding-format.md` for the complete output schema.

Render findings sorted by severity (critical first):

```
════════════════════════════════════════════════════════════
ADVERSARIAL REVIEW: <target label>
════════════════════════════════════════════════════════════

<summary — terse ship/no-ship assessment>

- [critical] <title> (<file>:<line_start>[-<line_end>])
  <body>
  Recommendation: <recommendation>

- [high] <title> (<file>:<line_start>[-<line_end>])
  <body>
  Recommendation: <recommendation>

- [medium] <title> (<file>:<line_start>[-<line_end>])
  <body>
  Recommendation: <recommendation>

- [low] <title> (<file>:<line_start>[-<line_end>])
  <body>
  Recommendation: <recommendation>

Next steps:
- <actionable follow-up 1>
- <actionable follow-up 2>

VERDICT: approve | needs-attention
════════════════════════════════════════════════════════════
```

If no findings:

```
════════════════════════════════════════════════════════════
ADVERSARIAL REVIEW: <target label>
════════════════════════════════════════════════════════════

<summary: change overview + "No adversarial findings.">

VERDICT: approve
════════════════════════════════════════════════════════════
```
