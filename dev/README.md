# Dev Docs 模式

一套用於在 Claude Code 會話和上下文重置之間維護專案上下文的方法。

---

## 問題所在

**上下文重置會遺失所有資訊：**
- 實作決策
- 關鍵檔案及其用途
- 任務進度
- 技術限制
- 為何選擇特定做法的原因

**重置後，Claude 必須重新探索所有內容。**

---

## 解決方案：持久化的 Dev Docs

一個三檔案結構，記錄恢復工作所需的所有資訊：

```
dev/active/[task-name]/
├── [task-name]-plan.md      # 策略計畫
├── [task-name]-context.md   # 關鍵決策與檔案
└── [task-name]-tasks.md     # 檢查清單格式
```

**這些檔案能在上下文重置後保留** - Claude 讀取它們即可立即掌握進度。

---

## 三檔案結構

### 1. [task-name]-plan.md

**用途：** 實作的策略計畫

**包含內容：**
- 執行摘要
- 現狀分析
- 規劃的未來狀態
- 實作階段
- 詳細任務與驗收標準
- 風險評估
- 成功指標
- 時程預估

**何時建立：** 在複雜任務開始時

**何時更新：** 範圍變更或發現新階段時

**範例：**
```markdown
# Feature Name - Implementation Plan

## Executive Summary
What we're building and why

## Current State
Where we are now

## Implementation Phases

### Phase 1: Infrastructure (2 hours)
- Task 1.1: Set up database schema
  - Acceptance: Schema compiles, relationships correct
- Task 1.2: Create service structure
  - Acceptance: All directories created

### Phase 2: Core Functionality (3 hours)
...
```

---

### 2. [task-name]-context.md

**用途：** 恢復工作所需的關鍵資訊

**包含內容：**
- SESSION PROGRESS 區段（頻繁更新！）
- 已完成與進行中的項目
- 關鍵檔案及其用途
- 已做的重要決策
- 發現的技術限制
- 相關檔案的連結
- 快速恢復指示

**何時建立：** 任務開始時

**何時更新：** **頻繁更新** - 在重大決策、完成工作或發現問題後

**範例：**
```markdown
# Feature Name - Context

## SESSION PROGRESS (2025-10-29)

### ✅ COMPLETED
- Database schema created (User, Post, Comment models)
- PostController implemented with BaseController pattern
- Sentry integration working

### 🟡 IN PROGRESS
- Creating PostService with business logic
- File: src/services/postService.ts

### ⚠️ BLOCKERS
- Need to decide on caching strategy

## Key Files

**src/controllers/PostController.ts**
- Extends BaseController
- Handles HTTP requests for posts
- Delegates to PostService

**src/services/postService.ts** (IN PROGRESS)
- Business logic for post operations
- Next: Add caching

## Quick Resume
To continue:
1. Read this file
2. Continue implementing PostService.createPost()
3. See tasks file for remaining work
```

**重要：** 每次完成重要工作後，都要更新 SESSION PROGRESS 區段！

---

### 3. [task-name]-tasks.md

**用途：** 追蹤進度的檢查清單

**包含內容：**
- 按邏輯區段劃分的階段
- 核取方塊格式的任務
- 狀態指示器（✅/🟡/⏳）
- 驗收標準
- 快速恢復區段

**何時建立：** 任務開始時

**何時更新：** 完成每個任務或發現新任務後

**範例：**
```markdown
# Feature Name - Task Checklist

## Phase 1: Setup ✅ COMPLETE
- [x] Create database schema
- [x] Set up controllers
- [x] Configure Sentry

## Phase 2: Implementation 🟡 IN PROGRESS
- [x] Create PostController
- [ ] Create PostService (IN PROGRESS)
- [ ] Create PostRepository
- [ ] Add validation with Zod

## Phase 3: Testing ⏳ NOT STARTED
- [ ] Unit tests for service
- [ ] Integration tests
- [ ] Manual API testing
```

---

## 何時使用 Dev Docs

**適用於：**
- ✅ 複雜的多日任務
- ✅ 有許多組成部分的功能
- ✅ 可能跨越多個會話的任務
- ✅ 需要仔細規劃的工作
- ✅ 大型系統重構

**不適用於：**
- ❌ 簡單的 bug 修復
- ❌ 單一檔案的變更
- ❌ 快速更新
- ❌ 瑣碎的修改

**經驗法則：** 如果任務超過 2 小時或跨越多個會話，就使用 dev docs。

---

## Dev Docs 的工作流程

### 開始新任務

1. **使用 /dev-docs slash command：**
   ```
   /dev-docs refactor authentication system
   ```

2. **Claude 建立三個檔案：**
   - 分析需求
   - 檢查程式碼庫
   - 建立完整計畫
   - 產生 context 和 tasks 檔案

3. **檢視與調整：**
   - 檢查計畫是否合理
   - 加入任何遺漏的考量
   - 調整時程預估

### 實作期間

1. **參考 plan** 了解整體策略
2. **頻繁更新 context.md：**
   - 標記已完成的工作
   - 記錄做出的決策
   - 加入阻礙事項
3. **勾選 tasks.md 中的任務** 當你完成它們時

### 上下文重置後

1. **Claude 讀取這三個檔案**
2. **在幾秒內理解完整狀態**
3. **在你離開的地方精確恢復**

不需要解釋你在做什麼 - 一切都已記錄！

---

## 與 Slash Commands 的整合

### /dev-docs
**建立：** 為任務建立新的 dev docs

**用法：**
```
/dev-docs implement real-time notifications
```

**產生：**
- `dev/active/implement-real-time-notifications/`
  - implement-real-time-notifications-plan.md
  - implement-real-time-notifications-context.md
  - implement-real-time-notifications-tasks.md

### /dev-docs-update
**更新：** 在上下文重置前更新現有的 dev docs

**用法：**
```
/dev-docs-update
```

**更新內容：**
- 標記已完成的任務
- 加入發現的新任務
- 更新會話進度的 context
- 記錄當前狀態

**使用時機：** 接近上下文限制或結束會話時

---

## 檔案組織

```
dev/
├── README.md              # 本檔案
├── active/                # 進行中的工作
│   ├── task-1/
│   │   ├── task-1-plan.md
│   │   ├── task-1-context.md
│   │   └── task-1-tasks.md
│   └── task-2/
│       └── ...
└── archive/               # 已完成的工作（選用）
    └── old-task/
        └── ...
```

**active/**: 進行中的工作
**archive/**: 已完成的任務（供參考）

---

## 範例：實際使用

參考此儲存庫中的 **dev/active/public-infrastructure-repo/** 實際範例：
- **plan.md** - 700+ 行用於建立此展示的策略計畫
- **context.md** - 追蹤已完成的項目、做出的決策、下一步工作
- **tasks.md** - 所有階段和任務的檢查清單

這些是用於建立此展示的實際 dev docs！

---

## 最佳實務

### 頻繁更新 Context

**不良做法：** 僅在會話結束時更新
**良好做法：** 在每個主要里程碑後更新

**SESSION PROGRESS 區段應始終反映實際情況：**
```markdown
## SESSION PROGRESS (YYYY-MM-DD)

### ✅ COMPLETED (列出所有完成的項目)
### 🟡 IN PROGRESS (你現在正在進行的工作)
### ⚠️ BLOCKERS (阻礙進度的事項)
```

### 讓任務可執行

**不良做法：** "Fix the authentication"
**良好做法：** "Implement JWT token validation in AuthMiddleware.ts (Acceptance: Tokens validated, errors to Sentry)"

**包含：**
- 具體的檔案名稱
- 清楚的驗收標準
- 對其他任務的依賴關係

### 保持計畫更新

如果範圍改變：
- 更新計畫
- 加入新階段
- 調整時程預估
- 記錄範圍變更的原因

---

## 給 Claude Code 的說明

**當使用者要求建立 dev docs：**

1. **使用 /dev-docs slash command**（如果可用）
2. **或手動建立：**
   - 詢問任務範圍
   - 分析相關的程式碼庫檔案
   - 建立完整計畫
   - 產生 context 和 tasks

3. **結構化計畫包含：**
   - 清楚的階段
   - 可執行的任務
   - 驗收標準
   - 風險評估

4. **讓 context 檔案易於恢復：**
   - SESSION PROGRESS 在頂端
   - 快速恢復指示
   - 附有說明的關鍵檔案清單

**從 dev docs 恢復時：**

1. **讀取這三個檔案**（plan、context、tasks）
2. **從 context.md 開始** - 包含當前狀態
3. **檢查 tasks.md** - 查看已完成和下一步的工作
4. **參考 plan.md** - 理解整體策略

**頻繁更新：**
- 立即標記已完成的任務
- 在重要工作後更新 SESSION PROGRESS
- 發現新任務時加入

---

## 手動建立 Dev Docs

如果你沒有 /dev-docs 指令：

**1. 建立目錄：**
```bash
mkdir -p dev/active/your-task-name
```

**2. 建立 plan.md：**
- 執行摘要
- 實作階段
- 詳細任務
- 時程預估

**3. 建立 context.md：**
- SESSION PROGRESS 區段
- 關鍵檔案
- 重要決策
- 快速恢復指示

**4. 建立 tasks.md：**
- 帶核取方塊的階段
- [ ] 任務格式
- 驗收標準

---

## 優點

**使用 dev docs 之前：**
- 上下文重置 = 從頭開始
- 忘記為何做出決策
- 失去進度追蹤
- 重複工作

**使用 dev docs 之後：**
- 上下文重置 = 讀取 3 個檔案，立即恢復
- 決策已記錄
- 進度已追蹤
- 無重複工作

**節省時間：** 每次上下文重置可節省數小時

---

## 下一步

1. **在下一個複雜任務中試用此模式**
2. **使用 /dev-docs** slash command（如果可用）
3. **頻繁更新** - 特別是 context.md
4. **實際查看** - 瀏覽 dev/active/public-infrastructure-repo/

**有問題？** 參考 [CLAUDE_INTEGRATION_GUIDE.md](../CLAUDE_INTEGRATION_GUIDE.md)
