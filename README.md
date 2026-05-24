# codex-review-plugin

Claude Code 原生代碼審查 Skills — 參考 OpenAI Codex 的 `/codex:review` 和 `/codex:adversarial-review` 設計模式，以純 Claude 能力實作，零外部依賴。

## 包含 Skills

### codex-review

標準代碼審查，以資深工程師的平衡角度檢查代碼品質。

- **審查維度：** 正確性、安全性、效能、可維護性、相容性
- **輸出格式：** verdict（`approve` / `needs-attention`）+ 結構化 findings + next_steps
- **嚴重度：** critical / high / medium / low（含 confidence 分數）
- **支援 flags：** `--wait`、`--background`、`--base <ref>`、`--scope auto|working-tree|branch`
- **不支援自訂焦點文字**（如需自訂焦點，請使用 codex-adversarial-review）

### codex-adversarial-review

對抗性代碼審查，主動嘗試推翻變更而非驗證它。完全參考 Codex 的 adversarial prompt template 設計。

- **審查立場：** 預設懷疑，不給讚美，只報 material findings
- **Attack surface：** 7 大類（auth、data loss、rollback、race conditions、empty-state、version skew、observability）
- **輸出格式：** verdict（`approve` / `needs-attention`）+ findings（含 file:line 和 confidence）+ next_steps
- **支援 flags：** `--wait`、`--background`、`--base <ref>`、`--scope auto|working-tree|branch`、焦點文字
- **Review-only：** 不修改任何檔案

## 安裝

將 skill 目錄複製到 `~/.claude/skills/`：

```bash
# 複製兩個 skill
cp -r codex-adversarial-review ~/.claude/skills/
cp -r codex-review ~/.claude/skills/
```

安裝後即可在 Claude Code 中使用 `/codex-adversarial-review` 和 `/codex-review`。

## 使用方式

```
# 標準代碼審查
/codex-review
/codex-review --base main

# 對抗性審查
/codex-adversarial-review
/codex-adversarial-review security
/codex-adversarial-review --base main --scope branch
```

## 目錄結構

```
codex-review-plugin/
├── README.md
├── codex-adversarial-review/
│   ├── SKILL.md
│   └── references/
│       ├── attack-surfaces.md
│       └── finding-format.md
└── codex-review/
    ├── SKILL.md
    └── references/
        └── review-checklist.md
```

## 設計原則

- **零外部依賴：** 不需要 Codex CLI、Python 或任何外部工具
- **參考原版設計：** prompt 結構、flags、輸出格式、審查邏輯完全參考 Codex 插件
- **Review-only：** 兩個 skill 都不會修改任何檔案
- **結構化輸出：** findings 含 severity、file、line、confidence、recommendation

## License

MIT
