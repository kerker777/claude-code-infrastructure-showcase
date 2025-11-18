# Claude 整合指南

**給 CLAUDE CODE 的說明：** 當使用者要求你將本展示專案的元件整合到他們的專案中時，請仔細遵循以下指示。

---

## 概述

本專案是 Claude Code 基礎架構元件的**參考資料庫**。使用者會請你協助將特定部分整合到他們的專案中。你的職責是：

1. **詢問釐清問題**，了解他們的專案結構
2. **複製適當的檔案**
3. **自訂設定**以符合他們的配置
4. **驗證整合**是否正常運作

**核心原則：** 務必先詢問，不要假設專案結構。適用於某個專案的做法不一定適用於另一個專案。

---

## 技術堆疊相容性檢查

**重要：** 在整合 skill 之前，請先確認使用者的技術堆疊符合 skill 的需求。

### 前端 Skills

**frontend-dev-guidelines 需要：**
- React (18+)
- MUI v7
- TanStack Query
- TanStack Router
- TypeScript

**整合前請先詢問：**
「你使用 React 搭配 MUI v7 嗎？」

**如果答案是「否」：**
```
frontend-dev-guidelines skill 是專為 React + MUI v7 設計的。我可以：
1. 協助你以此為範本，建立適合 [他們的技術堆疊] 的類似 skill
2. 只萃取框架無關的模式（檔案組織、效能等）
3. 如果不相關就略過這個 skill

你希望怎麼做？
```

### 後端 Skills

**backend-dev-guidelines 需要：**
- Node.js/Express
- TypeScript
- Prisma ORM
- Sentry

**整合前請先詢問：**
「你使用 Node.js 搭配 Express 和 Prisma 嗎？」

**如果答案是「否」：**
```
backend-dev-guidelines skill 是為 Express/Prisma 設計的。我可以：
1. 協助你以此為範本，建立適合 [他們的技術堆疊] 的類似指引
2. 萃取架構模式（分層架構適用於任何框架）
3. 略過這個 skill

你希望怎麼做？
```

### 技術堆疊無關的 Skills

這些適用於**任何**技術堆疊：
- ✅ **skill-developer** - Meta-skill，無技術需求
- ✅ **route-tester** - 僅需要 JWT cookie 認證（框架無關）
- ✅ **error-tracking** - Sentry 適用於大多數堆疊

---

## 一般整合模式

當使用者說：**「將 [元件] 加到我的專案」**

1. 識別元件類型（skill/hook/agent/command）
2. **檢查技術堆疊相容性**（針對前端/後端 skills）
3. 詢問他們的專案結構
4. 複製檔案或針對他們的堆疊進行調整
5. 自訂以符合他們的配置
6. 驗證整合
7. 提供後續步驟

---

## 整合 Skills

### 逐步流程

**當使用者請求一個 skill 時**（例如「新增 backend-dev-guidelines」）：

#### 1. 了解他們的專案

**請詢問這些問題：**
- 「你的專案結構是什麼？單一應用程式、monorepo，還是多服務架構？」
- 「你的 [後端/前端] 程式碼放在哪裡？」
- 「你使用什麼框架/技術？」

#### 2. 複製 Skill

```bash
cp -r /path/to/showcase/.claude/skills/[skill-name] \
      $CLAUDE_PROJECT_DIR/.claude/skills/
```

#### 3. 處理 skill-rules.json

**檢查它是否存在：**
```bash
ls $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json
```

**如果不存在：**
- 從展示專案複製範本
- 移除使用者不需要的 skills
- 針對他們的專案進行自訂

**如果已存在：**
- 讀取他們目前的 skill-rules.json
- 新增新的 skill 項目
- 小心合併以避免破壞現有的 skills

#### 4. 自訂路徑模式

**重要：** 更新 skill-rules.json 中的 `pathPatterns` 以符合他們的結構：

**範例 - 使用者有 monorepo：**
```json
{
  "backend-dev-guidelines": {
    "fileTriggers": {
      "pathPatterns": [
        "packages/api/src/**/*.ts",
        "packages/server/src/**/*.ts",
        "apps/backend/**/*.ts"
      ]
    }
  }
}
```

**範例 - 使用者有單一後端：**
```json
{
  "backend-dev-guidelines": {
    "fileTriggers": {
      "pathPatterns": [
        "src/**/*.ts",
        "backend/**/*.ts"
      ]
    }
  }
}
```

**安全的通用模式**（不確定時使用）：
```json
{
  "pathPatterns": [
    "**/*.ts",          // 所有 TypeScript 檔案
    "src/**/*.ts",      // 常見的 src 目錄
    "backend/**/*.ts"   // 常見的 backend 目錄
  ]
}
```

#### 5. 驗證整合

```bash
# 檢查 skill 是否已複製
ls -la $CLAUDE_PROJECT_DIR/.claude/skills/[skill-name]

# 驗證 skill-rules.json 語法
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq .
```

**告知使用者：**「試著編輯 [他們的後端路徑] 中的檔案，skill 應該就會啟動。」

---

### Skill 特定注意事項

#### backend-dev-guidelines
- **技術需求：** Node.js/Express、Prisma、TypeScript、Sentry
- **請詢問：**「你使用 Express 搭配 Prisma 嗎？」「你的後端程式碼在哪裡？」
- **如果技術堆疊不同：** 提議以此為範本進行調整
- **自訂：** pathPatterns
- **範例路徑：** `api/`、`server/`、`backend/`、`services/*/src/`
- **調整提示：** 架構模式（Routes→Controllers→Services）可套用到大多數框架

#### frontend-dev-guidelines
- **技術需求：** React 18+、MUI v7、TanStack Query/Router、TypeScript
- **請詢問：**「你使用 React 搭配 MUI v7 嗎？」「你的前端程式碼在哪裡？」
- **如果技術堆疊不同：** 提議建立調整版本（Vue、Angular 等）
- **自訂：** pathPatterns + 所有框架特定的範例
- **範例路徑：** `frontend/`、`client/`、`web/`、`apps/web/src/`
- **調整提示：** 檔案組織和效能模式可移植，但元件程式碼不行

#### route-tester
- **技術需求：** JWT cookie 認證（框架無關）
- **請詢問：**「你使用 JWT cookie 認證嗎？」
- **如果答案是「否」：**「這個 skill 是為 JWT cookies 設計的。要我針對 [他們的認證類型] 調整，還是略過？」
- **自訂：** 服務 URLs、認證模式
- **適用於：** 任何使用 JWT cookies 的後端框架

#### error-tracking
- **技術需求：** Sentry（適用於大多數後端）
- **請詢問：**「你使用 Sentry 嗎？」「你的後端程式碼在哪裡？」
- **如果沒有 Sentry：**「要以此為範本用於 [他們的錯誤追蹤工具] 嗎？」
- **自訂：** pathPatterns
- **調整提示：** 錯誤追蹤理念可移植到其他工具（Rollbar、Bugsnag 等）

#### skill-developer
- **技術需求：** 無！
- **直接複製即可** - meta-skill，完全通用，教導如何為任何技術堆疊建立 skill

---

## 針對不同技術堆疊調整 Skills

當使用者的技術堆疊與 skill 需求不同時，你有以下選擇：

### 選項 1：調整現有 Skill（建議）

**適用時機：** 使用者想要類似的指引，但用於不同技術

**流程：**
1. **以現有 skill 為起點進行複製：**
   ```bash
   cp -r showcase/.claude/skills/frontend-dev-guidelines \
         $CLAUDE_PROJECT_DIR/.claude/skills/vue-dev-guidelines
   ```

2. **識別需要變更的部分：**
   - 框架特定的程式碼範例（React → Vue）
   - 函式庫 APIs（MUI → Vuetify/PrimeVue）
   - Import 陳述式
   - 元件模式

3. **保留可移植的內容：**
   - 檔案組織原則
   - 效能最佳化策略
   - TypeScript 標準
   - 一般最佳實務

4. **系統性地替換範例：**
   - 詢問使用者在他們的堆疊中的對應模式
   - 將程式碼範例更新為他們的框架
   - 保持整體結構和章節

5. **更新 skill 名稱和觸發條件：**
   - 適當地重新命名 skill
   - 更新 skill-rules.json 中針對他們堆疊的觸發條件
   - 測試啟動

**範例 - 將 frontend-dev-guidelines 調整為 Vue：**
```
我會根據 React skill 的結構建立 vue-dev-guidelines：
- 將 React.FC → Vue defineComponent
- 將 useSuspenseQuery → Vue composables
- 將 MUI 元件 → [他們的元件函式庫]
- 保留：檔案組織、效能模式、TypeScript 指引

這需要幾分鐘時間。可以嗎？
```

### 選項 2：萃取框架無關的模式

**適用時機：** 堆疊差異很大，但核心原則適用

**流程：**
1. 閱讀現有的 skill
2. 識別框架無關的模式：
   - 分層架構（後端）
   - 檔案組織策略
   - 效能最佳化原則
   - 測試策略
   - 錯誤處理理念

3. 只使用這些模式建立新的 skill
4. 使用者稍後可以新增框架特定的範例

**範例：**
```
backend-dev-guidelines 使用 Express，但分層架構
（Routes → Controllers → Services → Repositories）也適用於 Django。

我可以建立一個包含以下內容的 skill：
- 分層架構模式
- 關注點分離原則
- 錯誤處理最佳實務
- 測試策略

然後你可以在建立模式時新增 Django 特定的範例。
```

### 選項 3：僅作為參考使用

**適用時機：** 差異太大無法調整，但使用者想要靈感

**流程：**
1. 使用者瀏覽現有的 skill
2. 你協助從頭建立新的 skill
3. 使用現有 skill 的結構作為範本
4. 遵循模組化模式（主檔案 + 資源檔案）

### 通常可跨技術堆疊移植的內容

**架構與組織：**
- ✅ 分層架構（Routes/Controllers/Services 模式）
- ✅ 關注點分離
- ✅ 檔案組織策略（features/ 模式）
- ✅ 漸進式揭露（主檔案 + 資源檔案）
- ✅ Repository 資料存取模式

**開發實務：**
- ✅ 錯誤處理理念
- ✅ 輸入驗證的重要性
- ✅ 測試策略
- ✅ 效能最佳化原則
- ✅ TypeScript 最佳實務

**框架特定的程式碼：**
- ❌ React hooks → 無法移植到 Vue/Angular
- ❌ MUI 元件 → 不同的元件函式庫
- ❌ Prisma 查詢 → 不同的 ORM 語法
- ❌ Express 中介軟體 → 不同的框架模式
- ❌ 路由實作 → 框架特定

### 何時建議調整 vs 略過

**建議調整如果：**
- 使用者想要針對他們的堆疊的類似指引
- 核心模式適用（分層架構等）
- 使用者有時間協助提供框架特定的範例

**建議略過如果：**
- 堆疊完全不同
- 使用者不需要這些模式
- 調整需要太長時間
- 使用者偏好從頭建立

---

## 整合 Hooks

### 必要的 Hooks（總是可以安全複製）

#### skill-activation-prompt (UserPromptSubmit)

**用途：** 根據使用者的提示自動建議 skills

**整合（不需要自訂）：**

```bash
# 複製兩個檔案
cp showcase/.claude/hooks/skill-activation-prompt.sh \
   $CLAUDE_PROJECT_DIR/.claude/hooks/
cp showcase/.claude/hooks/skill-activation-prompt.ts \
   $CLAUDE_PROJECT_DIR/.claude/hooks/

# 設定為可執行
chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh

# 如果需要，安裝相依套件
if [ -f "showcase/.claude/hooks/package.json" ]; then
  cp showcase/.claude/hooks/package.json \
     $CLAUDE_PROJECT_DIR/.claude/hooks/
  cd $CLAUDE_PROJECT_DIR/.claude/hooks && npm install
fi
```

**加到 settings.json：**
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

**這個 hook 完全通用** - 可在任何地方使用，不需要自訂！

#### post-tool-use-tracker (PostToolUse)

**用途：** 追蹤檔案變更以管理上下文

**整合（不需要自訂）：**

```bash
# 複製檔案
cp showcase/.claude/hooks/post-tool-use-tracker.sh \
   $CLAUDE_PROJECT_DIR/.claude/hooks/

# 設定為可執行
chmod +x $CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh
```

**加到 settings.json：**
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

**這個 hook 完全通用** - 自動偵測專案結構！

---

### 選用的 Hooks（需要大量自訂）

#### tsc-check.sh 和 trigger-build-resolver.sh (Stop hooks)

⚠️ **警告：** 這些 hooks 是針對特定的多服務 monorepo 結構設定的。

**整合前請先詢問：**
1. 「你有包含多個 TypeScript 服務的 monorepo 嗎？」
2. 「你的服務目錄名稱是什麼？」
3. 「你的 tsconfig.json 檔案在哪裡？」

**對於簡單專案（單一服務）：**
- **建議略過**這些 hooks
- 對於單一服務專案來說過於複雜
- 使用者可以改為手動執行 `tsc --noEmit`

**對於複雜專案（多服務 monorepo）：**

1. 複製檔案
2. **必須編輯** tsc-check.sh - 找到這個區段：
```bash
case "$repo" in
    email|exports|form|frontend|projects|uploads|users|utilities|events|database)
        echo "$repo"
        return 0
        ;;
esac
```

3. 替換為使用者實際的服務名稱：
```bash
case "$repo" in
    api|web|auth|payments|notifications)  # ← 使用者的服務
        echo "$repo"
        return 0
        ;;
esac
```

4. 在加到 settings.json 之前先手動測試：
```bash
./.claude/hooks/tsc-check.sh
```

**重要：** 如果這個 hook 失敗，它會阻擋 Stop 事件。只有在確定它適用於他們的配置時才新增。

---

## 整合 Agents

**Agents 是獨立的** - 最容易整合！

### 標準 Agent 整合

```bash
# 複製 agent 檔案
cp showcase/.claude/agents/[agent-name].md \
   $CLAUDE_PROJECT_DIR/.claude/agents/
```

**就這樣！** Agents 會立即運作，不需要設定。

### 檢查硬編碼的路徑

某些 agents 可能會參照路徑。**複製之前，請閱讀 agent 檔案並檢查：**

- `~/git/old-project/` → 應改為 `$CLAUDE_PROJECT_DIR` 或 `.`
- `/root/git/project/` → 應改為 `$CLAUDE_PROJECT_DIR` 或 `.`
- 硬編碼的截圖路徑 → 詢問使用者想將截圖存在哪裡

**如果找到，請更新它們：**
```bash
sed -i 's|~/git/old-project/|.|g' $CLAUDE_PROJECT_DIR/.claude/agents/[agent].md
sed -i 's|/root/git/.*PROJECT.*DIR|$CLAUDE_PROJECT_DIR|g' \
    $CLAUDE_PROJECT_DIR/.claude/agents/[agent].md
```

### Agent 特定注意事項

**auth-route-tester / auth-route-debugger：**
- 需要使用者專案中的 JWT cookie 認證
- 請詢問：「你使用 JWT cookies 進行認證嗎？」
- 如果答案是「否」：「這些 agents 是為 JWT cookie 認證設計的。要略過還是要我調整？」

**frontend-error-fixer：**
- 可能會參照截圖路徑
- 請詢問：「截圖應該存在哪裡？」

**所有其他 agents：**
- 直接複製即可，它們完全通用

---

## 整合 Slash Commands

```bash
# 複製 command 檔案
cp showcase/.claude/commands/[command].md \
   $CLAUDE_PROJECT_DIR/.claude/commands/
```

### 自訂路徑

Commands 可能會參照開發文件路徑。**請檢查並更新：**

**dev-docs 和 dev-docs-update：**
- 尋找 `dev/active/` 路徑參照
- 請詢問：「你想把開發文件存在哪裡？」
- 更新 command 檔案中的路徑

**route-research-for-testing：**
- 可能會參照服務路徑
- 詢問他們的 API 結構

---

## 常見模式與最佳實務

### 模式：詢問專案結構

**不要假設：**
- ❌ 「我會為你的 blog-api 服務新增這個」
- ❌ 「為你的 frontend 目錄進行設定」

**請詢問：**
- ✅ 「你的專案結構是什麼？Monorepo 還是單一應用程式？」
- ✅ 「你的後端程式碼在哪裡？」
- ✅ 「你使用 workspaces 還是有多個服務？」

### 模式：自訂 skill-rules.json

**使用者有 monorepo 搭配 workspaces：**
```json
{
  "pathPatterns": [
    "packages/*/src/**/*.ts",
    "apps/*/src/**/*.tsx"
  ]
}
```

**使用者有 Nx monorepo：**
```json
{
  "pathPatterns": [
    "apps/api/src/**/*.ts",
    "libs/*/src/**/*.ts"
  ]
}
```

**使用者有簡單結構：**
```json
{
  "pathPatterns": [
    "src/**/*.ts",
    "backend/**/*.ts"
  ]
}
```

### 模式：settings.json 整合

**絕對不要直接複製展示專案的 settings.json！**

相反地，**萃取並合併**他們需要的區段：

1. 讀取他們現有的 settings.json
2. 新增他們需要的 hook 設定
3. 保留他們現有的設定

**範例合併：**
```json
{
  // ... 他們現有的設定 ...
  "hooks": {
    // ... 他們現有的 hooks ...
    "UserPromptSubmit": [  // ← 新增這個區段
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

---

## 驗證檢查清單

整合後，**請驗證這些項目：**

```bash
# 1. Hooks 是否可執行
ls -la $CLAUDE_PROJECT_DIR/.claude/hooks/*.sh
# 應該顯示：-rwxr-xr-x

# 2. skill-rules.json 是否為有效的 JSON
cat $CLAUDE_PROJECT_DIR/.claude/skills/skill-rules.json | jq .
# 應該能夠解析而不出錯

# 3. Hook 相依套件是否已安裝（如果有 TypeScript hooks）
ls $CLAUDE_PROJECT_DIR/.claude/hooks/node_modules/
# 如果有 package.json 應該顯示套件

# 4. Settings.json 是否為有效的 JSON
cat $CLAUDE_PROJECT_DIR/.claude/settings.json | jq .
# 應該能夠解析而不出錯
```

**然後請使用者測試：**
- 「試著編輯 [相關路徑] 中的檔案 - skill 應該會啟動」
- 「試著問我關於 [主題] 的問題 - 我應該會建議該 skill」

---

## 常見錯誤避免

### ❌ 不要：直接複製 settings.json
**原因：** Stop hooks 參照不存在的服務
**應該：** 只萃取 UserPromptSubmit 和 PostToolUse 區段

### ❌ 不要：保留範例服務名稱
**原因：** 使用者沒有 blog-api、auth-service 等
**應該：** 詢問他們實際的結構並更新

### ❌ 不要：忘記設定 hooks 為可執行
**原因：** Hooks 沒有執行權限就無法運作
**應該：** 複製後一定要執行 `chmod +x`

### ❌ 不要：假設 monorepo 結構
**原因：** 大多數專案是單一服務
**應該：** 先詢問，然後再自訂

### ❌ 不要：一次新增所有 skills
**原因：** 會讓人不知所措，而且可能不是全部都相關
**應該：** 詢問哪些 skills 與他們的工作相關

### ❌ 不要：未經測試就複製 Stop hooks
**原因：** 如果失敗會阻擋 Stop 事件
**應該：** 先手動測試，只在運作時才新增

---

## 整合對話範例

### 範例 1：新增後端 Skill

**使用者：**「將 backend-dev-guidelines skill 加到我的專案」

**Claude 應該回應：**
```
我會新增 backend-dev-guidelines skill。首先，我需要了解你的專案結構：

1. 這是有多個服務的 monorepo，還是單一後端應用程式？
2. 你的後端程式碼在哪裡？（例如 src/、backend/、server/）
3. 你使用 Express.js 嗎？
```

**使用者：**「單一應用程式，程式碼在 src/api/，是的我們使用 Express」

**Claude 應該：**
1. 複製 skill 目錄
2. 更新 skill-rules.json，pathPattern 設為：`"src/api/**/*.ts"`
3. 驗證複製正確
4. 回應：
```
✅ 已新增 backend-dev-guidelines skill！

設定：
- 編輯 src/api/ 中的檔案時會觸發 Skill
- 透過關鍵字如「controller」、「service」、「route」、「API」啟動

試試看：編輯 src/api/ 中的檔案，你應該會看到 skill 建議。
```

### 範例 2：設定 Hooks

**使用者：**「設定 skill activation hooks」

**Claude 應該回應：**
```
我會設定 skill 自動啟動系統。這需要兩個 hooks：
1. skill-activation-prompt - 根據你的提示建議 skills
2. post-tool-use-tracker - 追蹤檔案變更

你已經有 .claude/settings.json 檔案了嗎，還是我應該建立一個？
```

**使用者：**「我已經有 settings.json 了」

**Claude 應該：**
1. 讀取他們的 settings.json
2. 複製兩個 hook 檔案
3. 在 hooks 目錄執行 npm install
4. 將 hook 設定加到他們的 settings.json（保留現有設定）
5. 設定 hooks 為可執行
6. 回應：
```
✅ Skill activation hooks 已安裝！

我做了以下事項：
- 複製了 skill-activation-prompt hook
- 複製了 post-tool-use-tracker hook
- 安裝了相依套件
- 加到你的 settings.json

測試看看：問我「協助後端開發」，我應該會建議 backend-dev-guidelines skill。
```

### 範例 3：新增 Agent

**使用者：**「新增 code-architecture-reviewer agent」

**Claude 應該：**
```
code-architecture-reviewer agent 會審查程式碼的架構問題和最佳實務。
這個 agent 獨立運作，不需要設定。

我現在會複製到你的 .claude/agents/ 目錄。
```

**然後：**
1. 複製 agent 檔案
2. 回應：
```
✅ 已新增 code-architecture-reviewer agent！

使用方式是執行 Task 工具，並設定：
- subagent_type: "code-architecture-reviewer"
- prompt: 要審查的程式碼描述

Agent 會審查你的程式碼並提供架構回饋。
```

### 範例 4：技術堆疊不符

**使用者：**「將 frontend-dev-guidelines skill 加到我的 Vue 專案」

**Claude 應該：**
```
我看到你使用 Vue。frontend-dev-guidelines skill 是專為 React + MUI v7 設計的。
我有幾個選項：

選項 1（建議）：我可以使用 React skill 作為範本建立 vue-dev-guidelines skill。我會調整：
- Vue 的元件模式（Composition API、defineComponent）
- 將 MUI 範例替換為你的元件函式庫
- 針對 Vue 模式調整資料擷取
- 保留：檔案組織、效能原則、TypeScript 標準

選項 2：只萃取框架無關的模式（檔案組織、效能、測試），不包含 React 特定的程式碼。

選項 3：略過它，協助你從頭建立 Vue 指引。

你偏好哪一個？
```

**使用者：**「選項 1 聽起來不錯，我使用 Vuetify」

**Claude 應該：**
1. 複製 frontend-dev-guidelines 為 vue-dev-guidelines
2. 檢視每個資源檔案
3. 將 React 模式替換為 Vue Composition API
4. 將 MUI 元件替換為 Vuetify
5. 更新 skill-rules.json，加上 Vue 特定的觸發條件（.vue 檔案）
6. 回應：
```
✅ 已建立針對 Vue 3 + Vuetify 調整的 vue-dev-guidelines skill！

我變更的內容：
- React.FC 元件 → Vue defineComponent
- MUI v7 範例 → Vuetify 元件
- TanStack Query → 你的資料擷取方式
- 檔案模式 → .vue 檔案

我保留的內容：
- 檔案組織（features/ 模式）
- 效能最佳化策略
- TypeScript 標準
- 載入/錯誤狀態原則

試著編輯 .vue 檔案 - skill 應該會啟動。
```

---

## 快速參考表

### 什麼需要自訂？

| 元件 | 技術需求 | 自訂 | 要詢問什麼 |
|-----------|------------------|--------------|-------------|
| **skill-developer** | 無 | ✅ 無 | 直接複製即可 |
| **backend-dev-guidelines** | Express/Prisma/Node | ⚠️ 路徑 + 技術檢查 | 「使用 Express/Prisma？」「後端在哪？」 |
| **frontend-dev-guidelines** | React/MUI v7 | ⚠️⚠️ 路徑 + 框架 | 「使用 React/MUI v7？」「前端在哪？」 |
| **route-tester** | JWT cookies | ⚠️ 認證 + 路徑 | 「JWT cookie 認證？」 |
| **error-tracking** | Sentry | ⚠️ 路徑 | 「使用 Sentry？」「後端在哪？」 |
| **skill-activation-prompt** | ✅ 無 | 直接複製即可 |
| **post-tool-use-tracker** | ✅ 無 | 直接複製即可 |
| **tsc-check** | ⚠️⚠️⚠️ 大量 | 「Monorepo 還是單一服務？」 |
| **所有 agents** | ✅ 最少 | 檢查路徑 |
| **所有 commands** | ⚠️ 路徑 | 「開發文件要放哪？」 |

### 何時建議略過

| 元件 | 如果...則略過 |
|-----------|-----------|
| **tsc-check hooks** | 單一服務專案或不同的建置配置 |
| **route-tester** | 未使用 JWT cookie 認證 |
| **frontend-dev-guidelines** | 未使用 React + MUI |
| **auth agents** | 未使用 JWT cookie 認證 |

---

## 給 Claude 的最後提示

**當使用者說「新增所有東西」：**
- 從必要項目開始：skill-activation hooks + 1-2 個相關的 skills
- 不要用所有 5 個 skills + 10 個 agents 讓他們不知所措
- 詢問他們實際需要什麼

**當有東西無法運作時：**
- 檢查驗證清單
- 驗證路徑符合他們的結構
- 手動測試 hooks
- 檢查 JSON 語法錯誤

**當使用者不確定時：**
- 建議從 skill-activation hooks 開始
- 新增後端或前端 skill（他們使用哪個）
- 稍後視需要再新增更多

**總是解釋你在做什麼：**
- 顯示你正在執行的指令
- 解釋為什麼要問這些問題
- 在整合後提供清楚的後續步驟

---

**請記住：** 這是一個參考資料庫，不是一個可運作的應用程式。你的工作是協助使用者為他們特定的專案結構挑選和調整元件。
