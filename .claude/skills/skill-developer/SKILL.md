---
name: skill-developer
description: 建立與管理遵循 Anthropic 最佳實務的 Claude Code 技能。適用於建立新技能、修改 skill-rules.json、理解觸發模式、使用 hooks、除錯技能啟動問題，或實作漸進式揭露。涵蓋技能結構、YAML frontmatter、觸發類型（關鍵字、意圖模式、檔案路徑、內容模式）、執行層級（block、suggest、warn）、hook 機制（UserPromptSubmit、PreToolUse）、會話追蹤，以及 500 行規則。
---

# Skill 開發指南

## 目的

這是一份在 Claude Code 中建立與管理自動啟動系統技能的完整指南，遵循 Anthropic 官方最佳實務，包含 500 行規則與漸進式揭露模式。

## 何時使用此技能

當你提及以下內容時會自動啟動：
- 建立或新增技能
- 修改技能觸發器或規則
- 理解技能啟動的運作方式
- 除錯技能啟動問題
- 處理 skill-rules.json
- Hook 系統機制
- Claude Code 最佳實務
- 漸進式揭露
- YAML frontmatter
- 500 行規則

---

## 系統概覽

### 雙 Hook 架構

**1. UserPromptSubmit Hook**（主動建議）
- **檔案**：`.claude/hooks/skill-activation-prompt.ts`
- **觸發時機**：在 Claude 看到使用者提示之前
- **目的**：根據關鍵字 + 意圖模式建議相關技能
- **方法**：注入格式化的提醒作為上下文（stdout → Claude 的輸入）
- **使用情境**：主題式技能、隱式工作偵測

**2. Stop Hook - 錯誤處理提醒**（溫和提醒）
- **檔案**：`.claude/hooks/error-handling-reminder.ts`
- **觸發時機**：在 Claude 完成回應之後
- **目的**：溫和提醒自我檢視所寫程式碼的錯誤處理
- **方法**：分析編輯過的檔案是否有風險模式，必要時顯示提醒
- **使用情境**：在不阻礙工作流程的情況下維持錯誤處理意識

**理念轉變（2025-10-27）：** 我們不再使用阻擋式的 PreToolUse 來處理 Sentry/錯誤處理。改為使用不會阻礙工作流程但能維持程式碼品質意識的溫和回應後提醒。

### 設定檔

**位置**：`.claude/skills/skill-rules.json`

定義內容：
- 所有技能及其觸發條件
- 執行層級（block、suggest、warn）
- 檔案路徑模式（glob）
- 內容偵測模式（regex）
- 跳過條件（會話追蹤、檔案標記、環境變數）

---

## 技能類型

### 1. Guardrail 技能

**目的：** 強制執行能預防錯誤的關鍵最佳實務

**特性：**
- 類型：`"guardrail"`
- 執行：`"block"`
- 優先級：`"critical"` 或 `"high"`
- 在使用技能前阻擋檔案編輯
- 預防常見錯誤（欄位名稱、關鍵錯誤）
- 會話感知（同一會話中不重複提醒）

**範例：**
- `database-verification` - 在 Prisma 查詢前驗證資料表/欄位名稱
- `frontend-dev-guidelines` - 強制執行 React/TypeScript 模式

**何時使用：**
- 會造成執行期錯誤的失誤
- 資料完整性考量
- 關鍵相容性問題

### 2. Domain 技能

**目的：** 為特定領域提供全面指引

**特性：**
- 類型：`"domain"`
- 執行：`"suggest"`
- 優先級：`"high"` 或 `"medium"`
- 建議性質，非強制性
- 主題或領域專屬
- 完整文件

**範例：**
- `backend-dev-guidelines` - Node.js/Express/TypeScript 模式
- `frontend-dev-guidelines` - React/TypeScript 最佳實務
- `error-tracking` - Sentry 整合指引

**何時使用：**
- 需要深入知識的複雜系統
- 最佳實務文件
- 架構模式
- 操作指南

---

## 快速入門：建立新技能

### 步驟 1：建立技能檔案

**位置：** `.claude/skills/{skill-name}/SKILL.md`

**範本：**
```markdown
---
name: my-new-skill
description: 簡短描述，包含觸發此技能的關鍵字。提及主題、檔案類型和使用情境。明確說明觸發詞彙。
---

# My New Skill

## 目的
此技能的用途

## 何時使用
具體情境與條件

## 重點資訊
實際指引、文件、模式、範例
```

**最佳實務：**
- ✅ **名稱**：小寫、連字號、偏好動名詞形式（動詞 + -ing）
- ✅ **描述**：包含所有觸發關鍵字/片語（最多 1024 字元）
- ✅ **內容**：500 行以下 - 使用參考檔案來記錄細節
- ✅ **範例**：真實程式碼範例
- ✅ **結構**：清楚的標題、列表、程式碼區塊

### 步驟 2：加到 skill-rules.json

完整架構請參閱 [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md)。

**基本範本：**
```json
{
  "my-new-skill": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "medium",
    "promptTriggers": {
      "keywords": ["keyword1", "keyword2"],
      "intentPatterns": ["(create|add).*?something"]
    }
  }
}
```

### 步驟 3：測試觸發器

**測試 UserPromptSubmit：**
```bash
echo '{"session_id":"test","prompt":"your test prompt"}' | \
  npx tsx .claude/hooks/skill-activation-prompt.ts
```

**測試 PreToolUse：**
```bash
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"session_id":"test","tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

### 步驟 4：優化模式

根據測試結果：
- 新增遺漏的關鍵字
- 優化意圖模式以減少誤判
- 調整檔案路徑模式
- 針對實際檔案測試內容模式

### 步驟 5：遵循 Anthropic 最佳實務

✅ 保持 SKILL.md 在 500 行以下
✅ 使用參考檔案進行漸進式揭露
✅ 為超過 100 行的參考檔案加上目錄
✅ 撰寫包含觸發關鍵字的詳細描述
✅ 在文件化之前先用 3 個以上的真實情境測試
✅ 根據實際使用情況反覆改進

---

## 執行層級

### BLOCK（關鍵 Guardrails）

- 實際阻止 Edit/Write 工具執行
- Hook 回傳退出碼 2，stderr → Claude
- Claude 看到訊息後必須使用技能才能繼續
- **適用於**：關鍵錯誤、資料完整性、安全性問題

**範例：** 資料庫欄位名稱驗證

### SUGGEST（建議）

- 在 Claude 看到提示前注入提醒
- Claude 知道相關技能
- 不強制執行，僅供建議
- **適用於**：領域指引、最佳實務、操作指南

**範例：** 前端開發指南

### WARN（選用）

- 低優先級建議
- 僅供建議，最低限度的執行
- **適用於**：錦上添花的建議、資訊性提醒

**很少使用** - 大多數技能不是 BLOCK 就是 SUGGEST。

---

## 跳過條件與使用者控制

### 1. 會話追蹤

**目的：** 避免在同一會話中重複提醒

**運作方式：**
- 第一次編輯 → Hook 阻擋，更新會話狀態
- 第二次編輯（同一會話）→ Hook 允許
- 不同會話 → 再次阻擋

**狀態檔案：** `.claude/hooks/state/skills-used-{session_id}.json`

### 2. 檔案標記

**目的：** 已驗證檔案的永久跳過

**標記：** `// @skip-validation`

**用法：**
```typescript
// @skip-validation
import { PrismaService } from './prisma';
// This file has been manually verified
```

**注意：** 謹慎使用 - 過度使用會失去原本的作用

### 3. 環境變數

**目的：** 緊急停用、暫時覆寫

**全域停用：**
```bash
export SKIP_SKILL_GUARDRAILS=true  # 停用所有 PreToolUse 阻擋
```

**特定技能：**
```bash
export SKIP_DB_VERIFICATION=true
export SKIP_ERROR_REMINDER=true
```

---

## 測試檢查清單

建立新技能時，請驗證：

- [ ] 已在 `.claude/skills/{name}/SKILL.md` 建立技能檔案
- [ ] 有正確的 frontmatter，包含 name 和 description
- [ ] 已在 `skill-rules.json` 新增條目
- [ ] 已用真實提示測試關鍵字
- [ ] 已用變化形式測試意圖模式
- [ ] 已用實際檔案測試檔案路徑模式
- [ ] 已針對檔案內容測試內容模式
- [ ] 阻擋訊息清楚且可執行（如果是 guardrail）
- [ ] 跳過條件已適當設定
- [ ] 優先級符合重要性
- [ ] 測試中無誤判
- [ ] 測試中無漏判
- [ ] 效能可接受（<100ms 或 <200ms）
- [ ] JSON 語法已驗證：`jq . skill-rules.json`
- [ ] **SKILL.md 在 500 行以下** ⭐
- [ ] 必要時已建立參考檔案
- [ ] 超過 100 行的檔案已加上目錄

---

## 參考檔案

特定主題的詳細資訊，請參閱：

### [TRIGGER_TYPES.md](TRIGGER_TYPES.md)
所有觸發類型的完整指南：
- 關鍵字觸發器（明確主題匹配）
- 意圖模式（隱式動作偵測）
- 檔案路徑觸發器（glob 模式）
- 內容模式（檔案中的 regex）
- 各類型的最佳實務與範例
- 常見陷阱與測試策略

### [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md)
完整的 skill-rules.json 架構：
- 完整 TypeScript 介面定義
- 逐欄位說明
- 完整 guardrail 技能範例
- 完整 domain 技能範例
- 驗證指南與常見錯誤

### [HOOK_MECHANISMS.md](HOOK_MECHANISMS.md)
Hook 內部機制深入探討：
- UserPromptSubmit 流程（詳細）
- PreToolUse 流程（詳細）
- 退出碼行為表（重要）
- 會話狀態管理
- 效能考量

### [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
完整的除錯指南：
- 技能未觸發（UserPromptSubmit）
- PreToolUse 未阻擋
- 誤判（觸發次數過多）
- Hook 完全未執行
- 效能問題

### [PATTERNS_LIBRARY.md](PATTERNS_LIBRARY.md)
可直接使用的模式集合：
- 意圖模式庫（regex）
- 檔案路徑模式庫（glob）
- 內容模式庫（regex）
- 按使用情境分類
- 可直接複製貼上

### [ADVANCED.md](ADVANCED.md)
未來改進與想法：
- 動態規則更新
- 技能相依性
- 條件式執行
- 技能分析
- 技能版本控制

---

## 快速參考摘要

### 建立新技能（5 步驟）

1. 建立 `.claude/skills/{name}/SKILL.md`，包含 frontmatter
2. 在 `.claude/skills/skill-rules.json` 新增條目
3. 使用 `npx tsx` 指令測試
4. 根據測試結果優化模式
5. 保持 SKILL.md 在 500 行以下

### 觸發類型

- **關鍵字**：明確主題提及
- **意圖**：隱式動作偵測
- **檔案路徑**：基於位置的啟動
- **內容**：特定技術偵測

完整細節請參閱 [TRIGGER_TYPES.md](TRIGGER_TYPES.md)。

### 執行

- **BLOCK**：退出碼 2，僅限關鍵情況
- **SUGGEST**：注入上下文，最常見
- **WARN**：建議性質，很少使用

### 跳過條件

- **會話追蹤**：自動（防止重複提醒）
- **檔案標記**：`// @skip-validation`（永久跳過）
- **環境變數**：`SKIP_SKILL_GUARDRAILS`（緊急停用）

### Anthropic 最佳實務

✅ **500 行規則**：保持 SKILL.md 在 500 行以下
✅ **漸進式揭露**：使用參考檔案記錄細節
✅ **目錄**：為超過 100 行的參考檔案加上目錄
✅ **單層深度**：不要深層巢狀參考
✅ **豐富描述**：包含所有觸發關鍵字（最多 1024 字元）
✅ **先測試**：在撰寫大量文件之前先建立 3 個以上的評估
✅ **動名詞命名**：偏好動詞 + -ing（例如 "processing-pdfs"）

### 疑難排解

手動測試 hooks：
```bash
# UserPromptSubmit
echo '{"prompt":"test"}' | npx tsx .claude/hooks/skill-activation-prompt.ts

# PreToolUse
cat <<'EOF' | npx tsx .claude/hooks/skill-verification-guard.ts
{"tool_name":"Edit","tool_input":{"file_path":"test.ts"}}
EOF
```

完整除錯指南請參閱 [TROUBLESHOOTING.md](TROUBLESHOOTING.md)。

---

## 相關檔案

**設定：**
- `.claude/skills/skill-rules.json` - 主要設定檔
- `.claude/hooks/state/` - 會話追蹤
- `.claude/settings.json` - Hook 註冊

**Hooks：**
- `.claude/hooks/skill-activation-prompt.ts` - UserPromptSubmit
- `.claude/hooks/error-handling-reminder.ts` - Stop 事件（溫和提醒）

**所有技能：**
- `.claude/skills/*/SKILL.md` - 技能內容檔案

---

**技能狀態**：完成 - 已按照 Anthropic 最佳實務重新架構 ✅
**行數**：< 500（遵循 500 行規則）✅
**漸進式揭露**：詳細資訊使用參考檔案 ✅

**下一步**：建立更多技能，根據使用情況優化模式
