# Hook 機制 - 深入探討

關於 UserPromptSubmit 和 PreToolUse hook 如何運作的技術深入探討。

## 目錄

- [UserPromptSubmit Hook 流程](#userpromptsubmit-hook-流程)
- [PreToolUse Hook 流程](#pretooluse-hook-流程)
- [結束代碼行為（重要）](#結束代碼行為重要)
- [Session 狀態管理](#session-狀態管理)
- [效能考量](#效能考量)

---

## UserPromptSubmit Hook 流程

### 執行順序

```
User submits prompt
    ↓
.claude/settings.json registers hook
    ↓
skill-activation-prompt.sh executes
    ↓
npx tsx skill-activation-prompt.ts
    ↓
Hook reads stdin (JSON with prompt)
    ↓
Loads skill-rules.json
    ↓
Matches keywords + intent patterns
    ↓
Groups matches by priority (critical → high → medium → low)
    ↓
Outputs formatted message to stdout
    ↓
stdout becomes context for Claude (injected before prompt)
    ↓
Claude sees: [skill suggestion] + user's prompt
```

### 重點說明

- **結束代碼（Exit code）**：永遠為 0（允許）
- **stdout**：→ Claude 的 context（以系統訊息形式注入）
- **時機點**：在 Claude 處理提示詞之前執行
- **行為**：非阻塞性，僅提供建議
- **目的**：讓 Claude 知道相關的 skill

### 輸入格式

```json
{
  "session_id": "abc-123",
  "transcript_path": "/path/to/transcript.json",
  "cwd": "/root/git/your-project",
  "permission_mode": "normal",
  "hook_event_name": "UserPromptSubmit",
  "prompt": "how does the layout system work?"
}
```

### 輸出格式（到 stdout）

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 SKILL ACTIVATION CHECK
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📚 RECOMMENDED SKILLS:
  → project-catalog-developer

ACTION: Use Skill tool BEFORE responding
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Claude 會在處理使用者提示詞之前，先看到這個輸出作為額外的 context。

---

## PreToolUse Hook 流程

### 執行順序

```
Claude calls Edit/Write tool
    ↓
.claude/settings.json registers hook (matcher: Edit|Write)
    ↓
skill-verification-guard.sh executes
    ↓
npx tsx skill-verification-guard.ts
    ↓
Hook reads stdin (JSON with tool_name, tool_input)
    ↓
Loads skill-rules.json
    ↓
Checks file path patterns (glob matching)
    ↓
Reads file for content patterns (if file exists)
    ↓
Checks session state (was skill already used?)
    ↓
Checks skip conditions (file markers, env vars)
    ↓
IF MATCHED AND NOT SKIPPED:
  Update session state (mark skill as enforced)
  Output block message to stderr
  Exit with code 2 (BLOCK)
ELSE:
  Exit with code 0 (ALLOW)
    ↓
IF BLOCKED:
  stderr → Claude sees message
  Edit/Write tool does NOT execute
  Claude must use skill and retry
IF ALLOWED:
  Tool executes normally
```

### 重點說明

- **結束代碼 2（Exit code 2）**：阻擋（stderr → Claude）
- **結束代碼 0（Exit code 0）**：允許
- **時機點**：在工具執行之前執行
- **Session 追蹤**：防止在同一個 session 中重複阻擋
- **Fail open**：發生錯誤時允許操作（避免中斷工作流程）
- **目的**：強制執行關鍵的防護機制

### 輸入格式

```json
{
  "session_id": "abc-123",
  "transcript_path": "/path/to/transcript.json",
  "cwd": "/root/git/your-project",
  "permission_mode": "normal",
  "hook_event_name": "PreToolUse",
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "/root/git/your-project/form/src/services/user.ts",
    "old_string": "...",
    "new_string": "..."
  }
}
```

### 輸出格式（被阻擋時輸出到 stderr）

```
⚠️ BLOCKED - Database Operation Detected

📋 REQUIRED ACTION:
1. Use Skill tool: 'database-verification'
2. Verify ALL table and column names against schema
3. Check database structure with DESCRIBE commands
4. Then retry this edit

Reason: Prevent column name errors in Prisma queries
File: form/src/services/user.ts

💡 TIP: Add '// @skip-validation' comment to skip future checks
```

Claude 會收到這個訊息，並理解它需要先使用 skill 才能重試編輯。

---

## 結束代碼行為（重要）

### 結束代碼參考表

| Exit Code | stdout | stderr | Tool Execution | Claude Sees |
|-----------|--------|--------|----------------|-------------|
| 0 (UserPromptSubmit) | → Context | → User only | N/A | stdout content |
| 0 (PreToolUse) | → User only | → User only | **Proceeds** | Nothing |
| 2 (PreToolUse) | → User only | → **CLAUDE** | **BLOCKED** | stderr content |
| Other | → User only | → User only | Blocked | Nothing |

### 為什麼結束代碼 2 很重要

這是執行強制機制的關鍵：

1. 從 PreToolUse 傳送訊息給 Claude 的**唯一方法**
2. stderr 的內容會「自動回饋給 Claude」
3. Claude 看到阻擋訊息並理解該怎麼做
4. 工具執行被阻止
5. 對於強制執行防護機制至關重要

### 對話流程範例

```
User: "Add a new user service with Prisma"

Claude: "I'll create the user service..."
    [Attempts to Edit form/src/services/user.ts]

PreToolUse Hook: [Exit code 2]
    stderr: "⚠️ BLOCKED - Use database-verification"

Claude sees error, responds:
    "I need to verify the database schema first."
    [Uses Skill tool: database-verification]
    [Verifies column names]
    [Retries Edit - now allowed (session tracking)]
```

---

## Session 狀態管理

### 目的

防止在同一個 session 中重複提醒 - 一旦 Claude 使用了某個 skill，就不再阻擋。

### 狀態檔案位置

`.claude/hooks/state/skills-used-{session_id}.json`

### 狀態檔案結構

```json
{
  "skills_used": [
    "database-verification",
    "error-tracking"
  ],
  "files_verified": []
}
```

### 運作方式

1. **第一次編輯**包含 Prisma 的檔案：
   - Hook 以結束代碼 2 阻擋
   - 更新 session 狀態：將 "database-verification" 加入 skills_used
   - Claude 看到訊息，使用 skill

2. **第二次編輯**（同一個 session）：
   - Hook 檢查 session 狀態
   - 在 skills_used 中找到 "database-verification"
   - 以代碼 0 結束（允許）
   - 不傳送訊息給 Claude

3. **不同的 session**：
   - 新的 session ID = 新的狀態檔案
   - Hook 再次阻擋

### 限制

Hook 無法偵測 skill 是否*真的*被呼叫 - 它只是在每個 session 中對每個 skill 阻擋一次。這意味著：

- 如果 Claude 沒有使用 skill 但進行了不同的編輯，不會再次阻擋
- 信任 Claude 會遵循指示
- 未來改進：偵測實際的 Skill tool 使用情況

---

## 效能考量

### 目標指標

- **UserPromptSubmit**：< 100ms
- **PreToolUse**：< 200ms

### 效能瓶頸

1. **載入 skill-rules.json**（每次執行都要）
   - 未來：在記憶體中快取
   - 未來：監看變更，只在需要時重新載入

2. **讀取檔案內容**（PreToolUse）
   - 只在設定 contentPatterns 時
   - 只在檔案存在時
   - 大型檔案可能會很慢

3. **Glob 比對**（PreToolUse）
   - 為每個 pattern 編譯正規表達式
   - 未來：編譯一次，快取

4. **正規表達式比對**（兩個 hook）
   - Intent patterns（UserPromptSubmit）
   - Content patterns（PreToolUse）
   - 未來：延遲編譯，快取編譯過的正規表達式

### 最佳化策略

**減少 pattern：**
- 使用更具體的 pattern（需要檢查的更少）
- 盡可能合併類似的 pattern

**檔案路徑 pattern：**
- 更具體 = 需要檢查的檔案更少
- 範例：`form/src/services/**` 優於 `form/**`

**內容 pattern：**
- 只在真正需要時才加入
- 更簡單的正規表達式 = 更快的比對

---

**相關檔案：**
- [SKILL.md](SKILL.md) - 主要 skill 指南
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Debug hook 問題
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 設定參考
