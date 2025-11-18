# 故障排除 - Skill 啟動問題

Skill 啟動問題的完整除錯指南。

## 目錄

- [Skill 無法觸發](#skill-無法觸發)
  - [UserPromptSubmit 未建議](#userpromptsubmit-未建議)
  - [PreToolUse 未阻擋](#pretooluse-未阻擋)
- [誤判](#誤判)
- [Hook 未執行](#hook-未執行)
- [效能問題](#效能問題)

---

## Skill 無法觸發

### UserPromptSubmit 未建議

**症狀：**提出問題後，輸出中沒有出現 skill 建議。

**常見原因：**

####  1. 關鍵字不符

**檢查：**
- 查看 skill-rules.json 中的 `promptTriggers.keywords`
- 你的提示詞中是否真的包含這些關鍵字？
- 記住：不區分大小寫的子字串比對

**範例：**
```json
"keywords": ["layout", "grid"]
```
- "how does the layout work?" → ✅ 符合 "layout"
- "how does the grid system work?" → ✅ 符合 "grid"
- "how do layouts work?" → ✅ 符合 "layout"
- "how does it work?" → ❌ 不符合

**修正：**在 skill-rules.json 中新增更多關鍵字變化

#### 2. 意圖模式太過具體

**檢查：**
- 查看 `promptTriggers.intentPatterns`
- 在 https://regex101.com/ 測試 regex
- 可能需要更廣泛的模式

**範例：**
```json
"intentPatterns": [
  "(create|add).*?(database.*?table)"  // 太過具體
]
```
- "create a database table" → ✅ 符合
- "add new table" → ❌ 不符合（缺少 "database"）

**修正：**擴大模式範圍：
```json
"intentPatterns": [
  "(create|add).*?(table|database)"  // 較好
]
```

#### 3. Skill 名稱拼寫錯誤

**檢查：**
- SKILL.md frontmatter 中的 skill 名稱
- skill-rules.json 中的 skill 名稱
- 必須完全一致

**範例：**
```yaml
# SKILL.md
name: project-catalog-developer
```
```json
// skill-rules.json
"project-catalogue-developer": {  // ❌ 拼寫錯誤：catalogue vs catalog
  ...
}
```

**修正：**確保名稱完全一致

#### 4. JSON 語法錯誤

**檢查：**
```bash
cat .claude/skills/skill-rules.json | jq .
```

如果 JSON 無效，jq 會顯示錯誤。

**常見錯誤：**
- 結尾多餘的逗號
- 缺少引號
- 使用單引號而非雙引號
- 字串中未跳脫的字元

**修正：**修正 JSON 語法，使用 jq 驗證

#### 除錯指令

手動測試 hook：

```bash
echo '{"session_id":"debug","prompt":"your test prompt here"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

預期結果：輸出中應該出現你的 skill。

---

### PreToolUse 未阻擋

**症狀：**編輯應該觸發防護機制的檔案，但沒有發生阻擋。

**常見原因：**

#### 1. 檔案路徑不符合模式

**檢查：**
- 正在編輯的檔案路徑
- skill-rules.json 中的 `fileTriggers.pathPatterns`
- Glob 模式語法

**範例：**
```json
"pathPatterns": [
  "frontend/src/**/*.tsx"
]
```
- 編輯：`frontend/src/components/Dashboard.tsx` → ✅ 符合
- 編輯：`frontend/tests/Dashboard.test.tsx` → ✅ 符合（需新增排除規則！）
- 編輯：`backend/src/app.ts` → ❌ 不符合

**修正：**調整 glob 模式或新增缺少的路徑

#### 2. 被 pathExclusions 排除

**檢查：**
- 你是否在編輯測試檔案？
- 查看 `fileTriggers.pathExclusions`

**範例：**
```json
"pathExclusions": [
  "**/*.test.ts",
  "**/*.spec.ts"
]
```
- 編輯：`services/user.test.ts` → ❌ 被排除
- 編輯：`services/user.ts` → ✅ 未被排除

**修正：**如果測試排除規則太廣，縮小範圍或移除

#### 3. 找不到內容模式

**檢查：**
- 檔案中是否真的包含該模式？
- 查看 `fileTriggers.contentPatterns`
- Regex 是否正確？

**範例：**
```json
"contentPatterns": [
  "import.*[Pp]risma"
]
```
- 檔案內容：`import { PrismaService } from './prisma'` → ✅ 符合
- 檔案內容：`import { Database } from './db'` → ❌ 不符合

**除錯：**
```bash
# 檢查檔案中是否存在該模式
grep -i "prisma" path/to/file.ts
```

**修正：**調整內容模式或新增缺少的 import

#### 4. Session 已使用過該 Skill

**檢查 session 狀態：**
```bash
ls .claude/hooks/state/
cat .claude/hooks/state/skills-used-{session-id}.json
```

**範例：**
```json
{
  "skills_used": ["database-verification"],
  "files_verified": []
}
```

如果 skill 在 `skills_used` 中，它在此 session 中不會再次阻擋。

**修正：**刪除狀態檔案以重置：
```bash
rm .claude/hooks/state/skills-used-{session-id}.json
```

#### 5. 檔案中存在標記

**檢查檔案中的跳過標記：**
```bash
grep "@skip-validation" path/to/file.ts
```

如果找到，該檔案會被永久跳過。

**修正：**如果需要再次驗證，移除該標記

#### 6. 環境變數覆寫

**檢查：**
```bash
echo $SKIP_DB_VERIFICATION
echo $SKIP_SKILL_GUARDRAILS
```

如果有設定，該 skill 會被停用。

**修正：**取消設定環境變數：
```bash
unset SKIP_DB_VERIFICATION
```

#### 除錯指令

手動測試 hook：

```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts 2>&1
{
  "session_id": "debug",
  "tool_name": "Edit",
  "tool_input": {"file_path": "/root/git/your-project/form/src/services/user.ts"}
}
EOF
echo "Exit code: $?"
```

預期結果：
- 如果應該阻擋：exit code 2 + stderr 訊息
- 如果應該允許：exit code 0 + 無輸出

---

## 誤判

**症狀：**Skill 在不應該觸發時觸發了。

**常見原因與解決方案：**

### 1. 關鍵字太過廣泛

**問題：**
```json
"keywords": ["user", "system", "create"]  // 太廣泛
```
- 會觸發於："user manual"、"file system"、"create directory"

**解決方案：**讓關鍵字更具體
```json
"keywords": [
  "user authentication",
  "user tracking",
  "create feature"
]
```

### 2. 意圖模式太過廣泛

**問題：**
```json
"intentPatterns": [
  "(create)"  // 符合所有包含 "create" 的內容
]
```
- 會觸發於："create file"、"create folder"、"create account"

**解決方案：**為模式增加上下文
```json
"intentPatterns": [
  "(create|add).*?(database|table|feature)"  // 更具體
]
```

**進階：**使用 negative lookaheads 排除特定情況
```regex
(create)(?!.*test).*?(feature)  // 如果出現 "test" 則不符合
```

### 3. 檔案路徑太過廣泛

**問題：**
```json
"pathPatterns": [
  "form/**"  // 符合 form/ 下的所有內容
]
```
- 會觸發於：測試檔案、設定檔、所有內容

**解決方案：**使用更窄的模式
```json
"pathPatterns": [
  "form/src/services/**/*.ts",  // 僅 service 檔案
  "form/src/controllers/**/*.ts"
]
```

### 4. 內容模式捕捉到無關程式碼

**問題：**
```json
"contentPatterns": [
  "Prisma"  // 會在註解、字串等處符合
]
```
- 會觸發於：`// Don't use Prisma here`
- 會觸發於：`const note = "Prisma is cool"`

**解決方案：**讓模式更具體
```json
"contentPatterns": [
  "import.*[Pp]risma",        // 僅 import
  "PrismaService\\.",         // 僅實際使用
  "prisma\\.(findMany|create)" // 特定方法
]
```

### 5. 調整強制等級

**最後手段：**如果誤判很頻繁：

```json
{
  "enforcement": "block"  // 改為 "suggest"
}
```

這會讓它變成建議性質而非阻擋性質。

---

## Hook 未執行

**症狀：**Hook 完全沒有執行 - 沒有建議，沒有阻擋。

**常見原因：**

### 1. Hook 未註冊

**檢查 `.claude/settings.json`：**
```bash
cat .claude/settings.json | jq '.hooks.UserPromptSubmit'
cat .claude/settings.json | jq '.hooks.PreToolUse'
```

預期結果：應該有 hook 項目

**修正：**新增缺少的 hook 註冊：
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

### 2. Bash Wrapper 無執行權限

**檢查：**
```bash
ls -l .claude/hooks/*.sh
```

預期結果：`-rwxr-xr-x`（可執行）

**修正：**
```bash
chmod +x .claude/hooks/*.sh
```

### 3. Shebang 不正確

**檢查：**
```bash
head -1 .claude/hooks/skill-activation-prompt.sh
```

預期結果：`#!/bin/bash`

**修正：**在第一行新增正確的 shebang

### 4. npx/tsx 不可用

**檢查：**
```bash
npx tsx --version
```

預期結果：版本編號

**修正：**安裝依賴套件：
```bash
cd .claude/hooks
npm install
```

### 5. TypeScript 編譯錯誤

**檢查：**
```bash
cd .claude/hooks
npx tsc --noEmit skill-activation-prompt.ts
```

預期結果：無輸出（無錯誤）

**修正：**修正 TypeScript 語法錯誤

---

## 效能問題

**症狀：**Hooks 執行緩慢，在提示詞/編輯前有明顯延遲。

**常見原因：**

### 1. 模式數量過多

**檢查：**
- 計算 skill-rules.json 中的模式數量
- 每個模式 = regex 編譯 + 比對

**解決方案：**減少模式
- 合併相似的模式
- 移除重複的模式
- 使用更具體的模式（比對更快）

### 2. 複雜的 Regex

**問題：**
```regex
(create|add|modify|update|implement|build).*?(feature|endpoint|route|service|controller|component|UI|page)
```
- 長的選擇列表 = 慢

**解決方案：**簡化
```regex
(create|add).*?(feature|endpoint)  // 較少的選擇項
```

### 3. 檢查太多檔案

**問題：**
```json
"pathPatterns": [
  "**/*.ts"  // 檢查所有 TypeScript 檔案
]
```

**解決方案：**更具體
```json
"pathPatterns": [
  "form/src/services/**/*.ts",  // 僅特定目錄
  "form/src/controllers/**/*.ts"
]
```

### 4. 大型檔案

內容模式比對會讀取整個檔案 - 對大型檔案很慢。

**解決方案：**
- 只在必要時使用內容模式
- 考慮檔案大小限制（未來改進項目）

### 測量效能

```bash
# UserPromptSubmit
time echo '{"prompt":"test"}' | npx tsx .claude/hooks/skill-activation-prompt.ts

# PreToolUse
time cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

**目標指標：**
- UserPromptSubmit：< 100ms
- PreToolUse：< 200ms

---

**相關檔案：**
- [SKILL.md](SKILL.md) - 主要 skill 指南
- [HOOK_MECHANISMS.md](HOOK_MECHANISMS.md) - Hook 運作方式
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 設定參考
