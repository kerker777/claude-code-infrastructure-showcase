# skill-rules.json - 完整參考文件

`.claude/skills/skill-rules.json` 的完整 schema 與設定參考文件。

## 目錄

- [檔案位置](#檔案位置)
- [完整 TypeScript Schema](#完整-typescript-schema)
- [欄位指南](#欄位指南)
- [範例：Guardrail Skill](#範例guardrail-skill)
- [範例：Domain Skill](#範例domain-skill)
- [驗證](#驗證)

---

## 檔案位置

**路徑：** `.claude/skills/skill-rules.json`

這個 JSON 檔案定義了所有 skill 及其在自動啟動系統中的觸發條件。

---

## 完整 TypeScript Schema

```typescript
interface SkillRules {
    version: string;
    skills: Record<string, SkillRule>;
}

interface SkillRule {
    type: 'guardrail' | 'domain';
    enforcement: 'block' | 'suggest' | 'warn';
    priority: 'critical' | 'high' | 'medium' | 'low';

    promptTriggers?: {
        keywords?: string[];
        intentPatterns?: string[];  // Regex strings
    };

    fileTriggers?: {
        pathPatterns: string[];     // Glob patterns
        pathExclusions?: string[];  // Glob patterns
        contentPatterns?: string[]; // Regex strings
        createOnly?: boolean;       // Only trigger on file creation
    };

    blockMessage?: string;  // For guardrails, {file_path} placeholder

    skipConditions?: {
        sessionSkillUsed?: boolean;      // Skip if used in session
        fileMarkers?: string[];          // e.g., ["@skip-validation"]
        envOverride?: string;            // e.g., "SKIP_DB_VERIFICATION"
    };
}
```

---

## 欄位指南

### 最上層

| 欄位 | 類型 | 必填 | 說明 |
|-------|------|----------|-------------|
| `version` | string | 是 | Schema 版本（目前為 "1.0"） |
| `skills` | object | 是 | Skill 名稱對應到 SkillRule 的 map |

### SkillRule 欄位

| 欄位 | 類型 | 必填 | 說明 |
|-------|------|----------|-------------|
| `type` | string | 是 | "guardrail"（強制執行）或 "domain"（建議性質） |
| `enforcement` | string | 是 | "block"（PreToolUse）、"suggest"（UserPromptSubmit）或 "warn" |
| `priority` | string | 是 | "critical"、"high"、"medium" 或 "low" |
| `promptTriggers` | object | 選填 | UserPromptSubmit hook 的觸發條件 |
| `fileTriggers` | object | 選填 | PreToolUse hook 的觸發條件 |
| `blockMessage` | string | 選填* | 當 enforcement="block" 時為必填。使用 `{file_path}` 佔位符 |
| `skipConditions` | object | 選填 | 例外機制與 session 追蹤 |

*Guardrail 必填

### promptTriggers 欄位

| 欄位 | 類型 | 必填 | 說明 |
|-------|------|----------|-------------|
| `keywords` | string[] | 選填 | 精確子字串比對（不區分大小寫） |
| `intentPatterns` | string[] | 選填 | 用於意圖偵測的 regex 模式 |

### fileTriggers 欄位

| 欄位 | 類型 | 必填 | 說明 |
|-------|------|----------|-------------|
| `pathPatterns` | string[] | 是* | 檔案路徑的 glob 模式 |
| `pathExclusions` | string[] | 選填 | 要排除的 glob 模式（例如測試檔案） |
| `contentPatterns` | string[] | 選填 | 用於比對檔案內容的 regex 模式 |
| `createOnly` | boolean | 選填 | 僅在建立新檔案時觸發 |

*當有 fileTriggers 時為必填

### skipConditions 欄位

| 欄位 | 類型 | 必填 | 說明 |
|-------|------|----------|-------------|
| `sessionSkillUsed` | boolean | 選填 | 如果 skill 已在本次 session 中使用過則跳過 |
| `fileMarkers` | string[] | 選填 | 如果檔案包含註解標記則跳過 |
| `envOverride` | string | 選填 | 用於停用 skill 的環境變數名稱 |

---

## 範例：Guardrail Skill

包含所有功能的完整 blocking guardrail skill 範例：

```json
{
  "database-verification": {
    "type": "guardrail",
    "enforcement": "block",
    "priority": "critical",

    "promptTriggers": {
      "keywords": [
        "prisma",
        "database",
        "table",
        "column",
        "schema",
        "query",
        "migration"
      ],
      "intentPatterns": [
        "(add|create|implement).*?(user|login|auth|tracking|feature)",
        "(modify|update|change).*?(table|column|schema|field)",
        "database.*?(change|update|modify|migration)"
      ]
    },

    "fileTriggers": {
      "pathPatterns": [
        "**/schema.prisma",
        "**/migrations/**/*.sql",
        "database/src/**/*.ts",
        "form/src/**/*.ts",
        "email/src/**/*.ts",
        "users/src/**/*.ts",
        "projects/src/**/*.ts",
        "utilities/src/**/*.ts"
      ],
      "pathExclusions": [
        "**/*.test.ts",
        "**/*.spec.ts"
      ],
      "contentPatterns": [
        "import.*[Pp]risma",
        "PrismaService",
        "prisma\\.",
        "\\.findMany\\(",
        "\\.findUnique\\(",
        "\\.findFirst\\(",
        "\\.create\\(",
        "\\.createMany\\(",
        "\\.update\\(",
        "\\.updateMany\\(",
        "\\.upsert\\(",
        "\\.delete\\(",
        "\\.deleteMany\\("
      ]
    },

    "blockMessage": "⚠️ BLOCKED - Database Operation Detected\n\n📋 REQUIRED ACTION:\n1. Use Skill tool: 'database-verification'\n2. Verify ALL table and column names against schema\n3. Check database structure with DESCRIBE commands\n4. Then retry this edit\n\nReason: Prevent column name errors in Prisma queries\nFile: {file_path}\n\n💡 TIP: Add '// @skip-validation' comment to skip future checks",

    "skipConditions": {
      "sessionSkillUsed": true,
      "fileMarkers": [
        "@skip-validation"
      ],
      "envOverride": "SKIP_DB_VERIFICATION"
    }
  }
}
```

### Guardrail 重點

1. **type**：必須為 "guardrail"
2. **enforcement**：必須為 "block"
3. **priority**：通常為 "critical" 或 "high"
4. **blockMessage**：必填，需提供清楚的執行步驟
5. **skipConditions**：Session 追蹤可避免重複提醒
6. **fileTriggers**：通常包含路徑與內容模式
7. **contentPatterns**：捕捉技術的實際使用情況

---

## 範例：Domain Skill

完整的建議型 domain skill 範例：

```json
{
  "project-catalog-developer": {
    "type": "domain",
    "enforcement": "suggest",
    "priority": "high",

    "promptTriggers": {
      "keywords": [
        "layout",
        "layout system",
        "grid",
        "grid layout",
        "toolbar",
        "column",
        "cell editor",
        "cell renderer",
        "submission",
        "submissions",
        "blog dashboard",
        "datagrid",
        "data grid",
        "CustomToolbar",
        "GridLayoutDialog",
        "useGridLayout",
        "auto-save",
        "column order",
        "column width",
        "filter",
        "sort"
      ],
      "intentPatterns": [
        "(how does|how do|explain|what is|describe).*?(layout|grid|toolbar|column|submission|catalog)",
        "(add|create|modify|change).*?(toolbar|column|cell|editor|renderer)",
        "blog dashboard.*?"
      ]
    },

    "fileTriggers": {
      "pathPatterns": [
        "frontend/src/features/submissions/**/*.tsx",
        "frontend/src/features/submissions/**/*.ts"
      ],
      "pathExclusions": [
        "**/*.test.tsx",
        "**/*.test.ts"
      ]
    }
  }
}
```

### Domain Skill 重點

1. **type**：必須為 "domain"
2. **enforcement**：通常為 "suggest"
3. **priority**："high" 或 "medium"
4. **blockMessage**：不需要（不會阻擋）
5. **skipConditions**：選填（較不重要）
6. **promptTriggers**：通常有大量關鍵字
7. **fileTriggers**：可能只有路徑模式（內容較不重要）

---

## 驗證

### 檢查 JSON 語法

```bash
cat .claude/skills/skill-rules.json | jq .
```

如果有效，jq 會美化輸出 JSON。如果無效，會顯示錯誤。

### 常見 JSON 錯誤

**結尾逗號：**
```json
{
  "keywords": ["one", "two",]  // ❌ Trailing comma
}
```

**缺少引號：**
```json
{
  type: "guardrail"  // ❌ Missing quotes on key
}
```

**單引號（無效的 JSON）：**
```json
{
  'type': 'guardrail'  // ❌ Must use double quotes
}
```

### 驗證清單

- [ ] JSON 語法有效（使用 `jq`）
- [ ] 所有 skill 名稱與 SKILL.md 檔名一致
- [ ] Guardrail 有 `blockMessage`
- [ ] Block 訊息使用 `{file_path}` 佔位符
- [ ] Intent 模式為有效的 regex（可在 regex101.com 測試）
- [ ] 檔案路徑模式使用正確的 glob 語法
- [ ] 內容模式有跳脫特殊字元
- [ ] Priority 與 enforcement 等級相符
- [ ] 沒有重複的 skill 名稱

---

**相關檔案：**
- [SKILL.md](SKILL.md) - 主要 skill 指南
- [TRIGGER_TYPES.md](TRIGGER_TYPES.md) - 完整 trigger 文件
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - 除錯設定問題
