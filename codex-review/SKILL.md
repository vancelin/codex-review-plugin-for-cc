---
name: codex-review
description: |
  標準代碼審查 — 資深工程師的平衡審查，檢查正確性、安全性、效能、可維護性、相容性。
  Review-only，不修復問題。
  Use when: "code review", "review this", "review the diff", "codex review",
  "check my code", "代碼審查", "審查代碼".
  Keywords: review, code review, diff review, check code, 審查.
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
  - AskUserQuestion
---

# Codex Review Skill

你正在執行標準代碼審查。你是資深工程師，以平衡、建設性的角度審查代碼變更。

This is review-only. Do not fix issues, apply patches, or modify any files.

---

## Argument Parsing

Raw slash-command arguments: `$ARGUMENTS`

Parse these flags from the arguments:
- `--wait` — run in foreground, do not ask
- `--background` — run as a Claude background task, do not ask
- `--base <ref>` — override auto-detected base branch
- `--scope auto|working-tree|branch` — review scope
- This review does NOT support custom focus text. If the user needs custom instructions, suggest using `/codex-adversarial-review` instead.

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
- Treat untracked files or directories as reviewable work even when `git diff --shortstat` is empty
- Only conclude there is nothing to review when the relevant scope is actually empty
- When in doubt, run the review instead of declaring that there is nothing to review

---

## Step 0.5: Execution Mode Decision

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

## Operating Stance

- **平衡審查。** 找問題但也認可好的實作。不是每段代碼都有問題。
- **精確定位。** 每個 finding 必須有具體的檔案和行號。
- **可操作。** 每個 finding 都要附帶具體的修復建議。
- **分級準確。** 嚴重度反映真實影響，不誇大也不淡化。
- **不修復。** 這是 review-only skill。不修改任何檔案。

---

## Step 1: Collect Review Context

**Branch review：**

```bash
git diff <base>...HEAD --stat 2>/dev/null
git diff <base>...HEAD 2>/dev/null
git log <base>...HEAD --oneline 2>/dev/null
```

**Working-tree review：**

```bash
git diff --stat 2>/dev/null
git diff --stat --cached 2>/dev/null
git diff 2>/dev/null
git diff --cached 2>/dev/null
git ls-files --others --exclude-standard 2>/dev/null
```

對每個變更的檔案，使用 Read 讀取完整內容以理解上下文。

對關鍵變更，使用 Grep 追蹤跨檔案引用（誰呼叫了被修改的函數、下游影響）。

若 diff 超過 3000 行，限制為變更區域前後各 50 行，並告知用戶。

---

## Step 2: 逐維度審查

載入 `references/review-checklist.md`，按五個維度逐一檢查：

### 1. Correctness（正確性）
邏輯錯誤、邊界情況、錯誤處理、空值處理、非同步正確性。

### 2. Security（安全性）
輸入驗證、SQL 注入、command injection、認證授權、敏感資料暴露。

### 3. Performance（效能）
N+1 查詢、重複計算、記憶體使用、不必要的同步等待。

### 4. Maintainability（可維護性）
函數職責、命名清晰度、重複代碼、可測試性、型別安全。

### 5. Compatibility（相容性）
API 合約向後相容、遷移安全、回滾風險、滾動部署安全。

**在思考過程中完成分析，不要輸出中間過程。**

只回報有實質影響的 finding。風格偏好、命名建議、低價值清理不納入 findings。

---

## Step 3: 結構化輸出

輸出格式貼近原版 codex:review 的 schema：

```
════════════════════════════════════════════════════════════
CODEX REVIEW: <target label>
════════════════════════════════════════════════════════════

<summary: 1-3 句話概述變更和審查結論>

---

### [critical] <title>

**File:** <file>:<line_start>-<line_end>
**Body:** <詳細說明問題>
**Confidence:** <0.0-1.0>
**Recommendation:** <具體修復建議>

---

### [high] <title>
(same format)

---

### [medium] <title>
(same format)

---

### [low] <title>
(same format)

---

════════════════════════════════════════════════════════════
VERDICT: approve | needs-attention
════════════════════════════════════════════════════════════

Next steps:
- <actionable follow-up 1>
- <actionable follow-up 2>
```

**Verdict 規則：**
- `approve` — 無 critical/high findings，代碼可以合併
- `needs-attention` — 有 critical 或 high finding，需要處理後再合併

**Finding 嚴重度定義：**
- `critical` — 可被外部利用、導致資料遺失或安全漏洞
- `high` — 在真實場景中會導致功能中斷或資料損壞
- `medium` — 邊緣情況下的失敗，有合理觸發路徑
- `low` — 理論風險或防禦性改進建議

**若無任何 findings：**

```
════════════════════════════════════════════════════════════
CODEX REVIEW: <target label>
════════════════════════════════════════════════════════════

<summary: 變更概述 + "No issues found.">

VERDICT: approve
════════════════════════════════════════════════════════════
```

---

## 最終檢查

在輸出前，驗證每個 finding：
- 有具體的檔案路徑和行號範圍
- confidence 分數反映真實確定程度（不確定就標低）
- recommendation 是具體可操作的（不只是「應該修正」）
- 不是風格偏好或低價值的清理建議

不通過以上檢查的 finding，刪除它。
