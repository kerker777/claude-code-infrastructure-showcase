---
description: 對應編輯過的路由並啟動測試
argument-hint: "[/extra/path …]"
allowed-tools: Bash(cat:*), Bash(awk:*), Bash(grep:*), Bash(sort:*), Bash(xargs:*), Bash(sed:*)
model: sonnet
---

## 情境說明

本次工作階段修改的路由檔案（自動產生）：

!cat "$CLAUDE_PROJECT_DIR/.claude/tsc-cache"/\*/edited-files.log \
 | awk -F: '{print $2}' \
 | grep '/routes/' \
 | sort -u

使用者指定的額外路由：`$ARGUMENTS`

## 你的任務

請**確實**依照以下步驟執行：

1. 將自動產生的清單與 `$ARGUMENTS` 合併，去除重複項目，並解析 `src/app.ts` 中定義的所有前綴路徑。
2. 針對每個最終的路由，輸出一筆 JSON 記錄，包含路徑、方法、預期的請求/回應結構，以及有效與無效的 payload 範例。
3. **現在使用以下參數呼叫 `Task` 工具**：

```json
{
    "tool": "Task",
    "parameters": {
        "description": "route smoke tests",
        "prompt": "對上述 JSON 執行 auth-route-tester 子代理程式。"
    }
}
```
