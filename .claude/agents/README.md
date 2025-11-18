# Agents

專門處理複雜、多步驟任務的 Agent。

---

## 什麼是 Agent？

Agent 是能夠處理特定複雜任務的自主 Claude 實例。與 Skill（提供即時指導）不同，Agent：
- 以獨立子任務方式執行
- 能夠自主運作，僅需最少監督
- 擁有專用的工具存取權限
- 完成後回傳完整報告

**主要優勢：** Agent 是**獨立的** - 只需複製 `.md` 檔案即可立即使用！

---

## 可用的 Agent（10 個）

### code-architecture-reviewer
**用途：** 檢視程式碼的架構一致性與最佳實務

**使用時機：**
- 實作新功能後
- 合併重大變更前
- 重構程式碼時
- 驗證架構決策時

**整合方式：** ✅ 直接複製即可

---

### code-refactor-master
**用途：** 規劃與執行全面性重構

**使用時機：**
- 重組檔案結構
- 拆解大型元件
- 移動檔案後更新 import 路徑
- 改善程式碼可維護性

**整合方式：** ✅ 直接複製即可

---

### documentation-architect
**用途：** 建立完整文件

**使用時機：**
- 撰寫新功能文件
- 建立 API 文件
- 撰寫開發者指南
- 產生架構概觀

**整合方式：** ✅ 直接複製即可

---

### frontend-error-fixer
**用途：** 除錯與修復前端錯誤

**使用時機：**
- 瀏覽器 console 錯誤
- 前端的 TypeScript 編譯錯誤
- React 錯誤
- 建置失敗

**整合方式：** ⚠️ 可能參照截圖路徑 - 需要時請更新

---

### plan-reviewer
**用途：** 在實作前檢視開發計畫

**使用時機：**
- 開始複雜功能前
- 驗證架構計畫
- 及早發現潛在問題
- 尋求第二意見時

**整合方式：** ✅ 直接複製即可

---

### refactor-planner
**用途：** 建立全面的重構策略

**使用時機：**
- 規劃程式碼重組
- 現代化舊有程式碼
- 拆解大型檔案
- 改善程式碼結構

**整合方式：** ✅ 直接複製即可

---

### web-research-specialist
**用途：** 線上研究技術問題

**使用時機：**
- 除錯罕見錯誤
- 尋找問題解決方案
- 研究最佳實務
- 比較實作方法

**整合方式：** ✅ 直接複製即可

---

### auth-route-tester
**用途：** 測試需要驗證的 API 端點

**使用時機：**
- 測試使用 JWT cookie 驗證的路由
- 驗證端點功能
- 除錯驗證問題

**整合方式：** ⚠️ 需要 JWT cookie 驗證機制

---

### auth-route-debugger
**用途：** 除錯驗證問題

**使用時機：**
- 驗證失敗
- Token 問題
- Cookie 問題
- 權限錯誤

**整合方式：** ⚠️ 需要 JWT cookie 驗證機制

---

### auto-error-resolver
**用途：** 自動修復 TypeScript 編譯錯誤

**使用時機：**
- TypeScript 錯誤導致建置失敗
- 重構後破壞型別
- 需要系統性錯誤解決

**整合方式：** ⚠️ 可能需要更新路徑

---

## 如何整合 Agent

### 標準整合方式（大多數 Agent）

**步驟 1：複製檔案**
```bash
cp showcase/.claude/agents/agent-name.md \
   your-project/.claude/agents/
```

**步驟 2：驗證（選用）**
```bash
# 檢查是否有寫死的路徑
grep -n "~/git/\|/root/git/\|/Users/" your-project/.claude/agents/agent-name.md
```

**步驟 3：使用**
詢問 Claude：「使用 [agent-name] agent 來 [任務]」

就這樣！Agent 可以立即運作。

---

### 需要客製化的 Agent

**frontend-error-fixer：**
- 可能參照截圖路徑
- 詢問使用者：「截圖應該存放在哪裡？」
- 在 agent 檔案中更新路徑

**auth-route-tester / auth-route-debugger：**
- 需要 JWT cookie 驗證機制
- 從範例更新服務 URL
- 依據使用者的驗證設定進行客製化

**auto-error-resolver：**
- 可能有寫死的專案路徑
- 更新為使用 `$CLAUDE_PROJECT_DIR` 或相對路徑

---

## 何時使用 Agent vs Skill

| 使用 Agent 的時機... | 使用 Skill 的時機... |
|-------------------|-------------------|
| 任務需要多個步驟 | 需要即時指導 |
| 需要複雜分析 | 檢查最佳實務 |
| 偏好自主運作 | 希望保持控制權 |
| 任務有明確終點 | 持續開發工作 |
| 範例：「檢視所有 controller」 | 範例：「建立新的 route」 |

**兩者可以搭配使用：**
- Skill 在開發期間提供模式指導
- Agent 在完成後檢視結果

---

## Agent 快速參考

| Agent | 複雜度 | 客製化需求 | 需要驗證 |
|-------|-----------|---------------|---------------|
| code-architecture-reviewer | 中等 | ✅ 無 | 否 |
| code-refactor-master | 高 | ✅ 無 | 否 |
| documentation-architect | 中等 | ✅ 無 | 否 |
| frontend-error-fixer | 中等 | ⚠️ 截圖路徑 | 否 |
| plan-reviewer | 低 | ✅ 無 | 否 |
| refactor-planner | 中等 | ✅ 無 | 否 |
| web-research-specialist | 低 | ✅ 無 | 否 |
| auth-route-tester | 中等 | ⚠️ 驗證設定 | JWT cookies |
| auth-route-debugger | 中等 | ⚠️ 驗證設定 | JWT cookies |
| auto-error-resolver | 低 | ⚠️ 路徑 | 否 |

---

## 給 Claude Code

**為使用者整合 agent 時：**

1. **閱讀 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)**
2. **直接複製 .md 檔案** - agent 是獨立的
3. **檢查是否有寫死的路徑：**
   ```bash
   grep "~/git/\|/root/" agent-name.md
   ```
4. **如有發現則更新路徑**為 `$CLAUDE_PROJECT_DIR` 或 `.`
5. **針對驗證 agent：** 先詢問是否使用 JWT cookie 驗證

**就這樣！** Agent 是最容易整合的元件。

---

## 建立自己的 Agent

Agent 是 markdown 檔案，可包含選用的 YAML frontmatter：

```markdown
# Agent Name

## Purpose
What this agent does

## Instructions
Step-by-step instructions for autonomous execution

## Tools Available
List of tools this agent can use

## Expected Output
What format to return results in
```

**提示：**
- 指示要非常具體
- 將複雜任務拆解為編號步驟
- 明確指定要回傳的內容
- 包含良好輸出的範例
- 明確列出可用的工具

---

## 疑難排解

### 找不到 Agent

**檢查：**
```bash
# agent 檔案是否存在？
ls -la .claude/agents/[agent-name].md
```

### Agent 因路徑錯誤而失敗

**檢查是否有寫死的路徑：**
```bash
grep "~/\|/root/\|/Users/" .claude/agents/[agent-name].md
```

**修復：**
```bash
sed -i 's|~/git/.*project|$CLAUDE_PROJECT_DIR|g' .claude/agents/[agent-name].md
```

---

## 下一步

1. **瀏覽上述 agent** - 找出對你有用的
2. **複製需要的** - 只需要 .md 檔案
3. **請 Claude 使用它們** - 「使用 [agent] 來 [任務]」
4. **建立自己的** - 依據這個模式滿足你的特定需求

**有問題？** 請參閱 [CLAUDE_INTEGRATION_GUIDE.md](../../CLAUDE_INTEGRATION_GUIDE.md)
