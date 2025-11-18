# Claude Code 基礎建設展示

**實戰驗證的 Claude Code 基礎建設參考資源庫。**

這個展示專案源自 6 個月的實際使用經驗，用於管理複雜的 TypeScript 微服務專案。提供了解決「技能無法自動啟動」問題的模式與系統，讓 Claude Code 能夠應用於企業級開發。

> **這不是一個可運行的應用程式** - 這是一個參考資源庫。請複製您需要的部分到自己的專案中。

---

## 內容說明

**實戰驗證的基礎建設包括：**
- ✅ **透過 hooks 自動啟動的 skills**
- ✅ **模組化 skill 模式**（500 行規則與漸進式揭露）
- ✅ **專門處理複雜任務的 agents**
- ✅ **能夠承受 context 重置的 dev docs 系統**
- ✅ **使用通用部落格領域的完整範例**

**建置所需時間：** 6 個月的反覆調整
**整合到您的專案所需時間：** 15-30 分鐘

---

## 快速開始 - 選擇您的路徑

### 🤖 使用 Claude Code 來整合？

**Claude：** 請閱讀 [`CLAUDE_INTEGRATION_GUIDE.md`](CLAUDE_INTEGRATION_GUIDE.md) 獲取為 AI 輔助設置量身定做的逐步整合說明。

### 🎯 我想要 skill 自動啟動

**突破性功能：** 真正能在您需要時自動啟動的 Skills。

**您需要的東西：**
1. skill 啟動的 hooks（2 個檔案）
2. 一兩個與您工作相關的 skill
3. 15 分鐘

**👉 [設置指南：.claude/hooks/README.md](.claude/hooks/README.md)**

### 📚 我想新增一個 skill

瀏覽 [skills 目錄](.claude/skills/) 並複製您需要的內容。

**可用的 skills：**
- **backend-dev-guidelines** - Node.js/Express/TypeScript 模式
- **frontend-dev-guidelines** - React/TypeScript/MUI v7 模式
- **skill-developer** - 用於建立 skills 的 meta-skill
- **route-tester** - 測試需要驗證的 API routes
- **error-tracking** - Sentry 整合模式

**👉 [Skills 指南：.claude/skills/README.md](.claude/skills/README.md)**

### 🤖 我想要專門的 agents

10 個實戰驗證的 agents 用於處理複雜任務：
- 程式碼架構審查
- 重構協助
- 文件生成
- 錯誤除錯
- 以及更多...

**👉 [Agents 指南：.claude/agents/README.md](.claude/agents/README.md)**

---

## 有什麼不同？

### 自動啟動的突破

**問題：** Claude Code skills 只是靜靜地待在那裡。您必須記得去使用它們。

**解決方案：** UserPromptSubmit hook 能夠：
- 分析您的提示
- 檢查檔案 context
- 自動建議相關的 skills
- 透過 `skill-rules.json` 配置運作

**結果：** Skills 在您需要時啟動，而不是在您記得時才啟動。

### 實戰驗證的模式

這些不是理論範例 - 它們是從實際專案中提取的：
- ✅ 6 個正在運作的微服務
- ✅ 50,000+ 行 TypeScript
- ✅ 具有複雜資料網格的 React 前端
- ✅ 精密的工作流程引擎
- ✅ 6 個月的每日 Claude Code 使用

這些模式能夠運作，是因為它們解決了真實的問題。

### 模組化 Skills（500 行規則）

大型 skills 會碰到 context 限制。解決方案：

```
skill-name/
  SKILL.md                  # <500 行，高階指南
  resources/
    topic-1.md              # 每個 <500 行
    topic-2.md
    topic-3.md
```

**漸進式揭露：** Claude 首先載入主要的 skill，只在需要時才載入資源檔案。

---

## 專案結構

```
.claude/
├── skills/                 # 5 個實戰 skills
│   ├── backend-dev-guidelines/  (12 個資源檔案)
│   ├── frontend-dev-guidelines/ (11 個資源檔案)
│   ├── skill-developer/         (7 個資源檔案)
│   ├── route-tester/
│   ├── error-tracking/
│   └── skill-rules.json    # Skill 啟動配置
├── hooks/                  # 6 個自動化 hooks
│   ├── skill-activation-prompt.*  (必要)
│   ├── post-tool-use-tracker.sh   (必要)
│   ├── tsc-check.sh        (選用，需要自訂)
│   └── trigger-build-resolver.sh  (選用)
├── agents/                 # 10 個專門的 agents
│   ├── code-architecture-reviewer.md
│   ├── refactor-planner.md
│   ├── frontend-error-fixer.md
│   └── ... 另外 7 個
└── commands/               # 3 個 slash commands
    ├── dev-docs.md
    └── ...

dev/
└── active/                 # Dev docs 模式範例
    └── public-infrastructure-repo/
```

---

## 元件目錄

### 🎨 Skills (5)

| Skill | 行數 | 用途 | 最適合 |
|-------|-------|---------|----------|
| [**skill-developer**](.claude/skills/skill-developer/) | 426 | 建立和管理 skills | Meta-開發 |
| [**backend-dev-guidelines**](.claude/skills/backend-dev-guidelines/) | 304 | Express/Prisma/Sentry 模式 | Backend APIs |
| [**frontend-dev-guidelines**](.claude/skills/frontend-dev-guidelines/) | 398 | React/MUI v7/TypeScript | React 前端 |
| [**route-tester**](.claude/skills/route-tester/) | 389 | 測試需要驗證的 routes | API 測試 |
| [**error-tracking**](.claude/skills/error-tracking/) | ~250 | Sentry 整合 | 錯誤監控 |

**所有 skills 都遵循模組化模式** - 主檔案 + 資源檔案以達成漸進式揭露。

**👉 [如何整合 skills →](.claude/skills/README.md)**

### 🪝 Hooks (6)

| Hook | 類型 | 必要？ | 自訂程度 |
|------|------|-----------|---------------|
| skill-activation-prompt | UserPromptSubmit | ✅ 是 | ✅ 不需要 |
| post-tool-use-tracker | PostToolUse | ✅ 是 | ✅ 不需要 |
| tsc-check | Stop | ⚠️ 選用 | ⚠️ 重度 - 僅限 monorepo |
| trigger-build-resolver | Stop | ⚠️ 選用 | ⚠️ 重度 - 僅限 monorepo |
| error-handling-reminder | Stop | ⚠️ 選用 | ⚠️ 中度 |
| stop-build-check-enhanced | Stop | ⚠️ 選用 | ⚠️ 中度 |

**從兩個必要的 hooks 開始** - 它們能啟用 skill 自動啟動並且可以直接使用。

**👉 [Hook 設置指南 →](.claude/hooks/README.md)**

### 🤖 Agents (10)

**獨立使用 - 直接複製即可！**

| Agent | 用途 |
|-------|---------|
| code-architecture-reviewer | 審查程式碼的架構一致性 |
| code-refactor-master | 規劃並執行重構 |
| documentation-architect | 生成完整的文件 |
| frontend-error-fixer | 除錯前端錯誤 |
| plan-reviewer | 審查開發計劃 |
| refactor-planner | 建立重構策略 |
| web-research-specialist | 線上研究技術問題 |
| auth-route-tester | 測試需要驗證的端點 |
| auth-route-debugger | 除錯驗證問題 |
| auto-error-resolver | 自動修復 TypeScript 錯誤 |

**👉 [Agents 運作方式 →](.claude/agents/README.md)**

### 💬 Slash Commands (3)

| Command | 用途 |
|---------|---------|
| /dev-docs | 建立結構化的開發文件 |
| /dev-docs-update | 在 context 重置前更新文件 |
| /route-research-for-testing | 研究用於測試的 route 模式 |

---

## 核心概念

### Hooks + skill-rules.json = 自動啟動

**系統運作方式：**
1. **skill-activation-prompt hook** 在每次使用者提示時執行
2. 檢查 **skill-rules.json** 尋找觸發模式
3. 自動建議相關的 skills
4. Skills 只在需要時載入

**這解決了第一大問題** - Claude Code skills 不會自己啟動。

### 漸進式揭露（500 行規則）

**問題：** 大型 skills 會碰到 context 限制

**解決方案：** 模組化結構
- 主要的 SKILL.md <500 行（概觀 + 導覽）
- 資源檔案每個 <500 行（深入探討）
- Claude 根據需要逐步載入

**範例：** backend-dev-guidelines 有 12 個資源檔案，涵蓋 routing、controllers、services、repositories、testing 等。

### Dev Docs 模式

**問題：** Context 重置會失去專案 context

**解決方案：** 三檔案結構
- `[task]-plan.md` - 策略計劃
- `[task]-context.md` - 關鍵決策和檔案
- `[task]-tasks.md` - 檢查清單格式

**搭配使用：** `/dev-docs` slash command 自動生成這些檔案

---

## ⚠️ 重要：哪些無法直接使用

### settings.json
包含的 `settings.json` **僅為範例**：
- Stop hooks 參照特定的 monorepo 結構
- 服務名稱（blog-api 等）是範例
- MCP servers 可能不存在於您的設置中

**如何使用：**
1. 僅提取 UserPromptSubmit 和 PostToolUse hooks
2. 自訂或略過 Stop hooks
3. 根據您的設置更新 MCP server 清單

### 部落格領域範例
Skills 使用通用的部落格範例（Post/Comment/User）：
- 這些是**教學範例**，不是必要條件
- 模式適用於任何領域（電子商務、SaaS 等）
- 將模式調整為您的商業邏輯

### Hook 目錄結構
某些 hooks 預期特定的結構：
- `tsc-check.sh` 預期有 service 目錄
- 根據您的專案配置進行自訂

---

## 整合流程

**建議方式：**

### 階段 1：Skill 啟動（15 分鐘）
1. 複製 skill-activation-prompt hook
2. 複製 post-tool-use-tracker hook
3. 更新 settings.json
4. 安裝 hook 相依套件

### 階段 2：新增第一個 Skill（10 分鐘）
1. 挑選一個相關的 skill
2. 複製 skill 目錄
3. 建立/更新 skill-rules.json
4. 自訂路徑模式

### 階段 3：測試與調整（5 分鐘）
1. 編輯檔案 - skill 應該要啟動
2. 提出問題 - skill 應該要被建議
3. 根據需要新增更多 skills

### 階段 4：選用的增強功能
- 新增您覺得有用的 agents
- 新增 slash commands
- 自訂 Stop hooks（進階）

---

## 取得協助

### 給使用者
**整合遇到問題？**
1. 檢查 [CLAUDE_INTEGRATION_GUIDE.md](CLAUDE_INTEGRATION_GUIDE.md)
2. 詢問 Claude：「為什麼 [skill] 沒有啟動？」
3. 提出 issue 並附上您的專案結構

### 給 Claude Code
協助使用者整合時：
1. **先閱讀 CLAUDE_INTEGRATION_GUIDE.md**
2. 詢問他們的專案結構
3. 自訂，不要盲目複製
4. 整合後驗證

---

## 這解決了什麼問題

### 使用這個基礎建設之前

❌ Skills 不會自動啟動
❌ 必須記得要使用哪個 skill
❌ 大型 skills 會碰到 context 限制
❌ Context 重置會失去專案知識
❌ 開發過程缺乏一致性
❌ 每次都需要手動調用 agent

### 使用這個基礎建設之後

✅ Skills 根據 context 自己建議
✅ Hooks 在正確時機觸發 skills
✅ 模組化 skills 保持在 context 限制內
✅ Dev docs 在重置後保留知識
✅ 透過防護機制確保一致的模式
✅ Agents 簡化複雜任務

---

## 社群

**覺得有用嗎？**

- ⭐ 為這個專案加星
- 🐛 回報問題或建議改進
- 💬 分享您自己的 skills/hooks/agents
- 📝 貢獻來自您領域的範例

**背景：**
這個基礎建設在我發表到 Reddit 的文章 ["Claude Code is a Beast – Tips from 6 Months of Hardcore Use"](https://www.reddit.com/r/ClaudeAI/comments/1oivjvm/claude_code_is_a_beast_tips_from_6_months_of/) 中有詳細說明。在收到數百個請求後，建立了這個展示專案來協助社群實作這些模式。


---

## 授權

MIT License - 可自由用於您的專案，商業或個人皆可。

---

## 快速連結

- 📖 [Claude 整合指南](CLAUDE_INTEGRATION_GUIDE.md) - 用於 AI 輔助設置
- 🎨 [Skills 文件](.claude/skills/README.md)
- 🪝 [Hooks 設置](.claude/hooks/README.md)
- 🤖 [Agents 指南](.claude/agents/README.md)
- 📝 [Dev Docs 模式](dev/README.md)

**從這裡開始：** 複製兩個必要的 hooks，新增一個 skill，然後見證自動啟動的魔法。
