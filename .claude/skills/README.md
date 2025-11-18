# Skills

經過實戰測試的 Claude Code 技能，能根據情境自動啟動。

---

## 什麼是 Skills？

Skills 是模組化的知識庫，Claude 會在需要時載入。它們提供：
- 特定領域的指引
- 最佳實務
- 程式碼範例
- 應避免的反模式

**問題：** Skills 預設不會自動啟動。

**解決方案：** 此 showcase 包含讓它們自動啟動所需的 hooks 與配置。

---

## 可用的 Skills

### skill-developer（Meta-Skill）
**用途：** 建立和管理 Claude Code skills

**檔案：** 7 個資源檔案（共 426 行）

**使用時機：**
- 建立新的 skills
- 理解 skill 結構
- 處理 skill-rules.json
- 除錯 skill 啟動問題

**客製化需求：** ✅ 無需修改 - 直接複製即可

**[查看 Skill →](skill-developer/)**

---

### backend-dev-guidelines
**用途：** Node.js/Express/TypeScript 開發模式

**檔案：** 12 個資源檔案（主檔案 304 行 + 資源檔案）

**涵蓋內容：**
- 分層架構（Routes → Controllers → Services → Repositories）
- BaseController 模式
- Prisma 資料庫存取
- Sentry 錯誤追蹤
- Zod 驗證
- UnifiedConfig 模式
- 依賴注入
- 測試策略

**使用時機：**
- 建立或修改 API routes
- 建立 controllers 或 services
- 使用 Prisma 進行資料庫操作
- 設定錯誤追蹤

**客製化需求：** ⚠️ 更新 skill-rules.json 中的 `pathPatterns` 以符合你的後端目錄

**pathPatterns 範例：**
```json
{
  "pathPatterns": [
    "src/api/**/*.ts",       // Single app with src/api
    "backend/**/*.ts",       // Backend directory
    "services/*/src/**/*.ts" // Multi-service monorepo
  ]
}
```

**[查看 Skill →](backend-dev-guidelines/)**

---

### frontend-dev-guidelines
**用途：** React/TypeScript/MUI v7 開發模式

**檔案：** 11 個資源檔案（主檔案 398 行 + 資源檔案）

**涵蓋內容：**
- 現代 React 模式（Suspense、lazy loading）
- 使用 useSuspenseQuery 進行資料擷取
- MUI v7 樣式（使用 `size={{}}` prop 的 Grid）
- TanStack Router
- 檔案組織（features/ 模式）
- 效能最佳化
- TypeScript 最佳實務

**使用時機：**
- 建立 React 元件
- 使用 TanStack Query 擷取資料
- 使用 MUI v7 進行樣式設定
- 設定路由

**客製化需求：** ⚠️ 更新 `pathPatterns` 並確認你使用 React/MUI

**pathPatterns 範例：**
```json
{
  "pathPatterns": [
    "src/**/*.tsx",          // Single React app
    "frontend/src/**/*.tsx", // Frontend directory
    "apps/web/**/*.tsx"      // Monorepo web app
  ]
}
```

**注意：** 此 skill 被設定為 **guardrail**（enforcement: "block"），以防止 MUI v6→v7 的不相容問題。

**[查看 Skill →](frontend-dev-guidelines/)**

---

### route-tester
**用途：** 使用 JWT cookie 驗證測試需要身份驗證的 API routes

**檔案：** 1 個主檔案（389 行）

**涵蓋內容：**
- 基於 JWT cookie 的身份驗證測試
- test-auth-route.js 腳本模式
- 使用 cookie 身份驗證的 cURL
- 除錯身份驗證問題
- 測試 POST/PUT/DELETE 操作

**使用時機：**
- 測試 API endpoints
- 除錯身份驗證
- 驗證 route 功能

**客製化需求：** ⚠️ 需要設定 JWT cookie 身份驗證

**先確認：** 「你是否使用基於 JWT cookie 的身份驗證？」
- 如果是：複製並自訂 service URLs
- 如果否：略過或改用你的身份驗證方法

**[查看 Skill →](route-tester/)**

---

### error-tracking
**用途：** Sentry 錯誤追蹤與監控模式

**檔案：** 1 個主檔案（約 250 行）

**涵蓋內容：**
- Sentry v8 初始化
- 錯誤捕獲模式
- Breadcrumbs 與使用者 context
- 效能監控
- 與 Express 和 React 的整合

**使用時機：**
- 設定錯誤追蹤
- 捕獲例外
- 新增錯誤 context
- 除錯正式環境問題

**客製化需求：** ⚠️ 為你的後端更新 `pathPatterns`

**[查看 Skill →](error-tracking/)**

---

## 如何將 Skill 加入你的專案

### 快速整合

**給 Claude Code：**
```
使用者：「將 backend-dev-guidelines skill 加入我的專案」

Claude 應該：
1. 詢問專案結構
2. 複製 skill 目錄
3. 以他們的路徑更新 skill-rules.json
4. 驗證整合
```

完整說明請參閱 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)。

### 手動整合

**步驟 1：複製 skill 目錄**
```bash
cp -r claude-code-infrastructure-showcase/.claude/skills/backend-dev-guidelines \
      your-project/.claude/skills/
```

**步驟 2：更新 skill-rules.json**

如果你還沒有這個檔案，請建立它：
```bash
cp claude-code-infrastructure-showcase/.claude/skills/skill-rules.json \
   your-project/.claude/skills/
```

接著為你的專案自訂 `pathPatterns`：
```json
{
  "skills": {
    "backend-dev-guidelines": {
      "fileTriggers": {
        "pathPatterns": [
          "YOUR_BACKEND_PATH/**/*.ts"  // ← Update this!
        ]
      }
    }
  }
}
```

**步驟 3：測試**
- 編輯後端目錄中的檔案
- Skill 應該會自動啟動

---

## skill-rules.json 配置

### 功能說明

定義 skills 應該在何時啟動，依據：
- **關鍵字**：使用者提示中的關鍵字（"backend"、"API"、"route"）
- **Intent patterns**：使用 regex 比對使用者意圖
- **檔案路徑模式**：編輯後端檔案
- **內容模式**：程式碼包含 Prisma 查詢

### 配置格式

```json
{
  "skill-name": {
    "type": "domain" | "guardrail",
    "enforcement": "suggest" | "block",
    "priority": "high" | "medium" | "low",
    "promptTriggers": {
      "keywords": ["list", "of", "keywords"],
      "intentPatterns": ["regex patterns"]
    },
    "fileTriggers": {
      "pathPatterns": ["path/to/files/**/*.ts"],
      "contentPatterns": ["import.*Prisma"]
    }
  }
}
```

### Enforcement 層級

- **suggest**：Skill 以建議方式出現，不會阻擋
- **block**：必須使用 skill 才能繼續（guardrail）

**使用 "block" 的時機：**
- 防止破壞性變更（MUI v6→v7）
- 關鍵的資料庫操作
- 安全敏感的程式碼

**使用 "suggest" 的時機：**
- 一般最佳實務
- 領域指引
- 程式碼組織

---

## 建立你自己的 Skills

完整指南請參閱 **skill-developer** skill：
- Skill YAML frontmatter 結構
- 資源檔案組織
- Trigger pattern 設計
- 測試 skill 啟動

**快速範本：**
```markdown
---
name: my-skill
description: What this skill does
---

# My Skill Title

## Purpose
[Why this skill exists]

## When to Use This Skill
[Auto-activation scenarios]

## Quick Reference
[Key patterns and examples]

## Resource Files
- [topic-1.md](resources/topic-1.md)
- [topic-2.md](resources/topic-2.md)
```

---

## 疑難排解

### Skill 沒有啟動

**檢查項目：**
1. Skill 目錄是否在 `.claude/skills/` 中？
2. Skill 是否列在 `skill-rules.json` 中？
3. `pathPatterns` 是否符合你的檔案？
4. Hooks 是否已安裝且正常運作？
5. settings.json 是否正確配置？

**除錯：**
```bash
# Check skill exists
ls -la .claude/skills/

# Validate skill-rules.json
cat .claude/skills/skill-rules.json | jq .

# Check hooks are executable
ls -la .claude/hooks/*.sh

# Test hook manually
./.claude/hooks/skill-activation-prompt.sh
```

### Skill 啟動太頻繁

更新 skill-rules.json：
- 讓關鍵字更具體
- 縮小 `pathPatterns` 範圍
- 提高 `intentPatterns` 的具體性

### Skill 從不啟動

更新 skill-rules.json：
- 新增更多關鍵字
- 擴大 `pathPatterns` 範圍
- 新增更多 `intentPatterns`

---

## 給 Claude Code

**為使用者整合 skill 時：**

1. **先閱讀 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)**
2. 詢問他們的專案結構
3. 在 skill-rules.json 中自訂 `pathPatterns`
4. 驗證 skill 檔案沒有寫死的路徑
5. 整合後測試啟動

**常見錯誤：**
- 保留範例路徑（blog-api/、frontend/）
- 沒有詢問是 monorepo 還是單一應用程式
- 複製 skill-rules.json 時沒有客製化

---

## 下一步

1. **從簡單開始：** 加入一個符合你工作的 skill
2. **驗證啟動：** 編輯相關檔案，skill 應該會提示
3. **再加更多：** 第一個 skill 運作後，再加入其他的
4. **客製化：** 根據你的工作流程調整 triggers

**有問題嗎？** 完整的整合說明請參閱 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)。
