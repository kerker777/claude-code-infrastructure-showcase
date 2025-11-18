# 常用模式庫

提供可直接使用的 regex 和 glob 模式，用於技能觸發器。您可以複製並依需求調整。

---

## Intent 模式（Regex）

### 功能/端點建立
```regex
(add|create|implement|build).*?(feature|endpoint|route|service|controller)
```

### 元件建立
```regex
(create|add|make|build).*?(component|UI|page|modal|dialog|form)
```

### 資料庫作業
```regex
(add|create|modify|update).*?(user|table|column|field|schema|migration)
(database|prisma).*?(change|update|query)
```

### 錯誤處理
```regex
(fix|handle|catch|debug).*?(error|exception|bug)
(add|implement).*?(try|catch|error.*?handling)
```

### 解釋說明請求
```regex
(how does|how do|explain|what is|describe|tell me about).*?
```

### 工作流程操作
```regex
(create|add|modify|update).*?(workflow|step|branch|condition)
(debug|troubleshoot|fix).*?workflow
```

### 測試
```regex
(write|create|add).*?(test|spec|unit.*?test)
```

---

## 檔案路徑模式（Glob）

### 前端
```glob
frontend/src/**/*.tsx        # 所有 React 元件
frontend/src/**/*.ts         # 所有 TypeScript 檔案
frontend/src/components/**   # 僅限 components 目錄
```

### 後端服務
```glob
form/src/**/*.ts            # Form 服務
email/src/**/*.ts           # Email 服務
users/src/**/*.ts           # Users 服務
projects/src/**/*.ts        # Projects 服務
```

### 資料庫
```glob
**/schema.prisma            # Prisma schema（任何位置）
**/migrations/**/*.sql      # Migration 檔案
database/src/**/*.ts        # 資料庫腳本
```

### 工作流程
```glob
form/src/workflow/**/*.ts              # Workflow 引擎
form/src/workflow-definitions/**/*.json # Workflow 定義
```

### 測試排除項目
```glob
**/*.test.ts                # TypeScript 測試
**/*.test.tsx               # React 元件測試
**/*.spec.ts                # Spec 檔案
```

---

## 內容模式（Regex）

### Prisma/資料庫
```regex
import.*[Pp]risma                # Prisma imports
PrismaService                    # PrismaService 使用
prisma\.                         # prisma.something
\.findMany\(                     # Prisma query 方法
\.create\(
\.update\(
\.delete\(
```

### Controllers/Routes
```regex
export class.*Controller         # Controller 類別
router\.                         # Express router
app\.(get|post|put|delete|patch) # Express app routes
```

### 錯誤處理
```regex
try\s*\{                        # Try 區塊
catch\s*\(                      # Catch 區塊
throw new                        # Throw 陳述式
```

### React/元件
```regex
export.*React\.FC               # React functional components
export default function.*       # Default function exports
useState|useEffect              # React hooks
```

---

**使用範例：**

```json
{
  "my-skill": {
    "promptTriggers": {
      "intentPatterns": [
        "(create|add|build).*?(component|UI|page)"
      ]
    },
    "fileTriggers": {
      "pathPatterns": [
        "frontend/src/**/*.tsx"
      ],
      "contentPatterns": [
        "export.*React\\.FC",
        "useState|useEffect"
      ]
    }
  }
}
```

---

**相關檔案：**
- [SKILL.md](SKILL.md) - Skill 主要指南
- [TRIGGER_TYPES.md](TRIGGER_TYPES.md) - 觸發器詳細文件
- [SKILL_RULES_REFERENCE.md](SKILL_RULES_REFERENCE.md) - 完整 schema 參考
