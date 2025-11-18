# Hooks

Claude Code hooks 可啟用 skill 自動觸發、檔案追蹤及驗證功能。

---

## 什麼是 Hooks？

Hooks 是在 Claude 工作流程中特定時間點執行的腳本：
- **UserPromptSubmit**：當使用者提交提示訊息時
- **PreToolUse**：在工具執行之前
- **PostToolUse**：在工具完成之後
- **Stop**：當使用者要求停止時

**關鍵概念：** Hooks 可以修改提示訊息、阻止動作並追蹤狀態 — 這些功能是 Claude 單獨無法做到的。

---

## 必要的 Hooks（從這裡開始）

### skill-activation-prompt (UserPromptSubmit)

**用途：** 根據使用者提示訊息和檔案情境自動建議相關的 skills

**運作方式：**
1. 讀取 `skill-rules.json`
2. 將使用者提示訊息與觸發規則進行比對
3. 檢查使用者正在處理哪些檔案
4. 將 skill 建議注入到 Claude 的情境中

**為什麼必要：** 這是讓 skills 能夠自動啟動的關鍵 hook。

**整合方式：**
```bash
# 複製兩個檔案
cp skill-activation-prompt.sh your-project/.claude/hooks/
cp skill-activation-prompt.ts your-project/.claude/hooks/

# 設為可執行
chmod +x your-project/.claude/hooks/skill-activation-prompt.sh

# 安裝相依套件
cd your-project/.claude/hooks
npm install
```

**加入到 settings.json：**
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

**客製化：** ✅ 無需修改 - 會自動讀取 skill-rules.json

---

### post-tool-use-tracker (PostToolUse)

**用途：** 追蹤檔案變更，以維持跨會話的情境

**運作方式：**
1. 監控 Edit/Write/MultiEdit 工具呼叫
2. 記錄哪些檔案被修改
3. 建立快取以管理情境
4. 自動偵測專案結構（frontend、backend、packages 等）

**為什麼必要：** 幫助 Claude 理解你的程式碼庫中哪些部分正在使用。

**整合方式：**
```bash
# 複製檔案
cp post-tool-use-tracker.sh your-project/.claude/hooks/

# 設為可執行
chmod +x your-project/.claude/hooks/post-tool-use-tracker.sh
```

**加入到 settings.json：**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
          }
        ]
      }
    ]
  }
}
```

**客製化：** ✅ 無需修改 - 會自動偵測結構

---

## 選用的 Hooks（需要客製化）

### tsc-check (Stop)

**用途：** 當使用者停止時進行 TypeScript 編譯檢查

**⚠️ 警告：** 此設定是針對多服務 monorepo 結構

**整合方式：**

**首先，確認這是否適合你：**
- ✅ 適用：多服務 TypeScript monorepo
- ❌ 跳過：單一服務專案或不同的建置設定

**如果要使用：**
1. 複製 tsc-check.sh
2. **編輯服務偵測部分（約第 28 行）：**
   ```bash
   # 將範例服務替換成你的服務：
   case "$repo" in
       api|web|auth|payments|...)  # ← 你實際的服務
   ```
3. 在加入到 settings.json 之前先手動測試

**客製化：** ⚠️⚠️⚠️ 需要大幅修改

---

### trigger-build-resolver (Stop)

**用途：** 當編譯失敗時自動啟動 build-error-resolver agent

**依賴於：** tsc-check hook 正常運作

**客製化：** ✅ 無需修改（但 tsc-check 必須先正常運作）

---

## 給 Claude Code

**為使用者設定 hooks 時：**

1. **先閱讀 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)**
2. **務必從兩個必要的 hooks 開始**
3. **在加入 Stop hooks 之前先詢問** - 設定錯誤可能會造成阻塞
4. **設定後進行驗證：**
   ```bash
   ls -la .claude/hooks/*.sh | grep rwx
   ```

**有問題？** 請參閱 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)
