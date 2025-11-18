# Hooks 配置指南

本指南說明如何為您的專案配置和自訂 hooks 系統。

## 快速開始配置

### 1. 在 .claude/settings.json 中註冊 Hooks

在專案根目錄建立或更新 `.claude/settings.json`：

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
    ],
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
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-prettier-formatter.sh"
          },
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-build-check-enhanced.sh"
          },
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/error-handling-reminder.sh"
          }
        ]
      }
    ]
  }
}
```

### 2. 安裝相依套件

```bash
cd .claude/hooks
npm install
```

### 3. 設定執行權限

```bash
chmod +x .claude/hooks/*.sh
```

## 自訂選項

### 專案結構偵測

預設情況下，hooks 會偵測以下目錄模式：

**Frontend：** `frontend/`、`client/`、`web/`、`app/`、`ui/`
**Backend：** `backend/`、`server/`、`api/`、`src/`、`services/`
**Database：** `database/`、`prisma/`、`migrations/`
**Monorepo：** `packages/*`、`examples/*`

#### 新增自訂目錄模式

編輯 `.claude/hooks/post-tool-use-tracker.sh`，修改 `detect_repo()` 函數：

```bash
case "$repo" in
    # 在此新增您的自訂目錄
    my-custom-service)
        echo "$repo"
        ;;
    admin-panel)
        echo "$repo"
        ;;
    # ... 既有的模式
esac
```

### 建置指令偵測

hooks 會根據以下條件自動偵測建置指令：
1. 是否存在包含 "build" script 的 `package.json`
2. 套件管理工具（優先順序：pnpm > npm > yarn）
3. 特殊情況（Prisma schemas）

#### 自訂建置指令

編輯 `.claude/hooks/post-tool-use-tracker.sh`，修改 `get_build_command()` 函數：

```bash
# 新增自訂建置邏輯
if [[ "$repo" == "my-service" ]]; then
    echo "cd $repo_path && make build"
    return
fi
```

### TypeScript 配置

hooks 會自動偵測：
- 標準 TypeScript 專案的 `tsconfig.json`
- Vite/React 專案的 `tsconfig.app.json`

#### 自訂 TypeScript 配置

編輯 `.claude/hooks/post-tool-use-tracker.sh`，修改 `get_tsc_command()` 函數：

```bash
if [[ "$repo" == "my-service" ]]; then
    echo "cd $repo_path && npx tsc --project tsconfig.build.json --noEmit"
    return
fi
```

### Prettier 配置

prettier hook 會按以下順序搜尋配置檔：
1. 目前檔案所在目錄（向上查找）
2. 專案根目錄
3. 使用 Prettier 預設值

#### 自訂 Prettier 配置搜尋

編輯 `.claude/hooks/stop-prettier-formatter.sh`，修改 `get_prettier_config()` 函數：

```bash
# 新增自訂配置位置
if [[ -f "$project_root/config/.prettierrc" ]]; then
    echo "$project_root/config/.prettierrc"
    return
fi
```

### 錯誤處理提醒

在 `.claude/hooks/error-handling-reminder.ts` 中配置檔案類別偵測：

```typescript
function getFileCategory(filePath: string): 'backend' | 'frontend' | 'database' | 'other' {
    // 新增自訂模式
    if (filePath.includes('/my-custom-dir/')) return 'backend';
    // ... 既有的模式
}
```

### 錯誤閾值配置

變更何時建議使用 auto-error-resolver agent。

編輯 `.claude/hooks/stop-build-check-enhanced.sh`：

```bash
# 預設為 5 個錯誤 - 可改為您偏好的數值
if [[ $total_errors -ge 10 ]]; then  # 現在需要 10 個以上的錯誤
    # 建議使用 agent
fi
```

## 環境變數

### 全域環境變數

在您的 shell 設定檔（`.bashrc`、`.zshrc` 等）中設定：

```bash
# 停用錯誤處理提醒
export SKIP_ERROR_REMINDER=1

# 自訂專案目錄（如果不使用預設值）
export CLAUDE_PROJECT_DIR=/path/to/your/project
```

### 單次 Session 環境變數

在啟動 Claude Code 之前設定：

```bash
SKIP_ERROR_REMINDER=1 claude-code
```

## Hook 執行順序

Stop hooks 會按照 `settings.json` 中指定的順序執行：

```json
"Stop": [
  {
    "hooks": [
      { "command": "...formatter.sh" },    // 第一個執行
      { "command": "...build-check.sh" },  // 第二個執行
      { "command": "...reminder.sh" }      // 第三個執行
    ]
  }
]
```

**為什麼順序很重要：**
1. 先格式化檔案（整理程式碼）
2. 然後檢查錯誤
3. 最後顯示提醒

## 選擇性啟用 Hooks

您不需要啟用所有 hooks。選擇適合您專案的：

### 最小設定（僅 Skill Activation）

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

### 僅建置檢查（不格式化）

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
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-build-check-enhanced.sh"
          }
        ]
      }
    ]
  }
}
```

### 僅格式化（不建置檢查）

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
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/stop-prettier-formatter.sh"
          }
        ]
      }
    ]
  }
}
```

## 快取管理

### 快取位置

```
$CLAUDE_PROJECT_DIR/.claude/tsc-cache/[session_id]/
```

### 手動清除快取

```bash
# 移除所有快取資料
rm -rf $CLAUDE_PROJECT_DIR/.claude/tsc-cache/*

# 移除特定 session
rm -rf $CLAUDE_PROJECT_DIR/.claude/tsc-cache/[session-id]
```

### 自動清理

build-check hook 會在建置成功後自動清理 session 快取。

## 配置疑難排解

### Hook 未執行

1. **檢查註冊：** 確認 hook 已加入 `.claude/settings.json`
2. **檢查權限：** 執行 `chmod +x .claude/hooks/*.sh`
3. **檢查路徑：** 確保 `$CLAUDE_PROJECT_DIR` 設定正確
4. **檢查 TypeScript：** 執行 `cd .claude/hooks && npx tsc` 檢查是否有錯誤

### 誤判偵測

**問題：** Hook 對不該處理的檔案觸發

**解決方法：** 在相關的 hook 中新增略過條件：

```bash
# 在 post-tool-use-tracker.sh 中
if [[ "$file_path" =~ /generated/ ]]; then
    exit 0  # 略過自動產生的檔案
fi
```

### 效能問題

**問題：** Hooks 執行緩慢

**解決方法：**
1. 限制 TypeScript 檢查僅針對變更的檔案
2. 使用較快的套件管理工具（pnpm > npm）
3. 新增更多略過條件
4. 對大型檔案停用 Prettier

```bash
# 在 stop-prettier-formatter.sh 中略過大型檔案
file_size=$(wc -c < "$file" 2>/dev/null || echo 0)
if [[ $file_size -gt 100000 ]]; then  # 略過大於 100KB 的檔案
    continue
fi
```

### 除錯 Hooks

在任何 hook 中新增除錯輸出：

```bash
# 在 hook script 的開頭
set -x  # 啟用除錯模式

# 或新增特定的除錯行
echo "DEBUG: file_path=$file_path" >&2
echo "DEBUG: repo=$repo" >&2
```

在 Claude Code 的日誌中查看 hook 執行情況。

## 進階配置

### 自訂 Hook 事件處理器

您可以為其他事件建立自己的 hooks：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/my-custom-bash-guard.sh"
          }
        ]
      }
    ]
  }
}
```

### Monorepo 配置

針對包含多個套件的 monorepo：

```bash
# 在 post-tool-use-tracker.sh 的 detect_repo() 中
case "$repo" in
    packages)
        # 取得套件名稱
        local package=$(echo "$relative_path" | cut -d'/' -f2)
        if [[ -n "$package" ]]; then
            echo "packages/$package"
        else
            echo "$repo"
        fi
        ;;
esac
```

### Docker/Container 專案

如果您的建置指令需要在容器中執行：

```bash
# 在 post-tool-use-tracker.sh 的 get_build_command() 中
if [[ "$repo" == "api" ]]; then
    echo "docker-compose exec api npm run build"
    return
fi
```

## 最佳實務

1. **從簡單開始** - 一次啟用一個 hook
2. **充分測試** - 進行修改並確認 hooks 正常運作
3. **記錄自訂內容** - 新增註解說明自訂邏輯
4. **版本控制** - 將 `.claude/` 目錄提交到 git
5. **團隊一致性** - 在團隊間共享配置

## 延伸閱讀

- [README.md](./README.md) - Hooks 概覽
- [../../docs/HOOKS_SYSTEM.md](../../docs/HOOKS_SYSTEM.md) - 完整 hooks 參考文件
- [../../docs/SKILLS_SYSTEM.md](../../docs/SKILLS_SYSTEM.md) - Skills 整合說明
