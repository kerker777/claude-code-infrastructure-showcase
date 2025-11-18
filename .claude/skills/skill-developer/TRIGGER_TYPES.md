# Trigger Types - 完整指南

Claude Code 技能自動啟用系統中觸發器設定的完整參考文件。

## 目錄

- [關鍵字觸發器 (明確式)](#關鍵字觸發器-明確式)
- [意圖模式觸發器 (隱含式)](#意圖模式觸發器-隱含式)
- [檔案路徑觸發器](#檔案路徑觸發器)
- [內容模式觸發器](#內容模式觸發器)
- [最佳實踐摘要](#最佳實踐摘要)

---

## 關鍵字觸發器 (明確式)

### 運作方式

對使用者提示進行不區分大小寫的子字串比對。

### 適用情境

當使用者明確提及主題時，基於主題的啟用機制。

### 設定方式

```json
"promptTriggers": {
  "keywords": ["layout", "grid", "toolbar", "submission"]
}
```

### 範例

- 使用者提示：「how does the **layout** system work?」
- 比對到：「layout」關鍵字
- 啟用技能：`project-catalog-developer`

### 最佳實踐

- 使用具體、明確的詞彙
- 包含常見變體（「layout」、「layout system」、「grid layout」）
- 避免過於籠統的字詞（「system」、「work」、「create」）
- 以實際提示進行測試

---

## 意圖模式觸發器 (隱含式)

### 運作方式

使用正規表達式模式比對來偵測使用者意圖，即使他們沒有明確提及特定主題。

### 適用情境

當使用者描述他們想做什麼，而非特定主題時，基於行為的啟用機制。

### 設定方式

```json
"promptTriggers": {
  "intentPatterns": [
    "(create|add|implement).*?(feature|endpoint)",
    "(how does|explain).*?(layout|workflow)"
  ]
}
```

### 範例

**資料庫相關工作：**
- 使用者提示：「add user tracking feature」
- 比對到：`(add).*?(feature)`
- 啟用技能：`database-verification`、`error-tracking`

**元件建立：**
- 使用者提示：「create a dashboard widget」
- 比對到：`(create).*?(component)`（若模式中包含 component）
- 啟用技能：`frontend-dev-guidelines`

### 最佳實踐

- 捕捉常見動作動詞：`(create|add|modify|build|implement)`
- 包含領域特定名詞：`(feature|endpoint|component|workflow)`
- 使用非貪婪比對：`.*?` 而非 `.*`
- 使用正規表達式測試工具徹底測試模式 (https://regex101.com/)
- 不要讓模式過於廣泛（會造成誤判）
- 不要讓模式過於具體（會造成漏判）

### 常見模式範例

```regex
# 資料庫工作
(add|create|implement).*?(user|login|auth|feature)

# 解釋說明
(how does|explain|what is|describe).*?

# 前端工作
(create|add|make|build).*?(component|UI|page|modal|dialog)

# 錯誤處理
(fix|handle|catch|debug).*?(error|exception|bug)

# 工作流程操作
(create|add|modify).*?(workflow|step|branch|condition)
```

---

## 檔案路徑觸發器

### 運作方式

使用 Glob 模式比對正在編輯的檔案路徑。

### 適用情境

根據專案中檔案的位置進行特定領域或區域的啟用。

### 設定方式

```json
"fileTriggers": {
  "pathPatterns": [
    "frontend/src/**/*.tsx",
    "form/src/**/*.ts"
  ],
  "pathExclusions": [
    "**/*.test.ts",
    "**/*.spec.ts"
  ]
}
```

### Glob 模式語法

- `**` = 任意數量的目錄（包含零個）
- `*` = 目錄名稱中的任意字元
- 範例：
  - `frontend/src/**/*.tsx` = frontend/src 及其子目錄中所有 .tsx 檔案
  - `**/schema.prisma` = 專案中任何位置的 schema.prisma
  - `form/src/**/*.ts` = form/src 子目錄中所有 .ts 檔案

### 範例

- 正在編輯的檔案：`frontend/src/components/Dashboard.tsx`
- 比對到：`frontend/src/**/*.tsx`
- 啟用技能：`frontend-dev-guidelines`

### 最佳實踐

- 使用具體條件以避免誤判
- 使用排除規則排除測試檔案：`**/*.test.ts`
- 考慮子目錄結構
- 使用實際檔案路徑測試模式
- 盡可能使用更窄的模式：`form/src/services/**` 而非 `form/**`

### 常見路徑模式

```glob
# 前端
frontend/src/**/*.tsx        # 所有 React 元件
frontend/src/**/*.ts         # 所有 TypeScript 檔案
frontend/src/components/**   # 僅 components 目錄

# 後端服務
form/src/**/*.ts            # Form 服務
email/src/**/*.ts           # Email 服務
users/src/**/*.ts           # Users 服務

# 資料庫
**/schema.prisma            # Prisma schema（任何位置）
**/migrations/**/*.sql      # Migration 檔案
database/src/**/*.ts        # 資料庫腳本

# 工作流程
form/src/workflow/**/*.ts              # 工作流程引擎
form/src/workflow-definitions/**/*.json # 工作流程定義

# 測試檔案排除
**/*.test.ts                # TypeScript 測試
**/*.test.tsx               # React 元件測試
**/*.spec.ts                # Spec 檔案
```

---

## 內容模式觸發器

### 運作方式

使用正規表達式模式比對檔案的實際內容（檔案裡面的內容）。

### 適用情境

根據程式碼引入或使用的技術（Prisma、controllers、特定函式庫等）進行技術特定的啟用。

### 設定方式

```json
"fileTriggers": {
  "contentPatterns": [
    "import.*[Pp]risma",
    "PrismaService",
    "\\.findMany\\(",
    "\\.create\\("
  ]
}
```

### 範例

**Prisma 偵測：**
- 檔案內容包含：`import { PrismaService } from '@project/database'`
- 比對到：`import.*[Pp]risma`
- 啟用技能：`database-verification`

**Controller 偵測：**
- 檔案內容包含：`export class UserController {`
- 比對到：`export class.*Controller`
- 啟用技能：`error-tracking`

### 最佳實踐

- 比對 imports：`import.*[Pp]risma`（使用 [Pp] 進行大小寫不敏感比對）
- 跳脫特殊正規表達式字元：`\\.findMany\\(` 而非 `.findMany(`
- 模式使用不區分大小寫的旗標
- 針對實際檔案內容進行測試
- 讓模式足夠具體以避免誤判

### 常見內容模式

```regex
# Prisma/資料庫
import.*[Pp]risma                # Prisma imports
PrismaService                    # PrismaService 使用
prisma\.                         # prisma.something
\.findMany\(                     # Prisma 查詢方法
\.create\(
\.update\(
\.delete\(

# Controllers/Routes
export class.*Controller         # Controller 類別
router\.                         # Express router
app\.(get|post|put|delete|patch) # Express app routes

# 錯誤處理
try\s*\{                        # Try 區塊
catch\s*\(                      # Catch 區塊
throw new                        # Throw 陳述式

# React/元件
export.*React\.FC               # React functional components
export default function.*       # Default function exports
useState|useEffect              # React hooks
```

---

## 最佳實踐摘要

### 應該做的事：
✅ 使用具體、明確的關鍵字
✅ 使用實際範例測試所有模式
✅ 包含常見變體
✅ 使用非貪婪正規表達式：`.*?`
✅ 在內容模式中跳脫特殊字元
✅ 加入測試檔案的排除規則
✅ 讓檔案路徑模式窄且具體

### 不應該做的事：
❌ 使用過於籠統的關鍵字（「system」、「work」）
❌ 讓意圖模式過於廣泛（造成誤判）
❌ 讓模式過於具體（造成漏判）
❌ 忘記使用正規表達式測試工具進行測試 (https://regex101.com/)
❌ 使用貪婪正規表達式：`.*` 而非 `.*?`
❌ 在檔案路徑中比對過於廣泛

### 測試你的觸發器

**測試關鍵字/意圖觸發器：**
```bash
echo '{"session_id":"test","prompt":"your test prompt"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

**測試檔案路徑/內容觸發器：**
```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{
  "session_id": "test",
  "tool_name": "Edit",
  "tool_input": {"file_path": "/path/to/test/file.ts"}
}
EOF
```

---

**相關檔案：**
- [SKILL.md](SKILL.md) - 主要技能指南
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 完整 skill-rules.json schema
- [PATTERNS_LIBRARY.md](PATTERNS_LIBRARY.md) - 即用模式庫
