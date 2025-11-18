---
name: error-tracking
description: Add Sentry v8 error tracking and performance monitoring to your project services. Use this skill when adding error handling, creating new controllers, instrumenting cron jobs, or tracking database performance. ALL ERRORS MUST BE CAPTURED TO SENTRY - no exceptions.
---

# your project Sentry 整合技能

## 目的
此技能強制在所有 your project 服務中實施全面的 Sentry 錯誤追蹤和效能監控，遵循 Sentry v8 模式。

## 何時使用此技能
- 為任何程式碼加入錯誤處理
- 建立新的控制器或路由
- 為 cron job 加入監控
- 追蹤資料庫效能
- 加入效能 span
- 處理工作流程錯誤

## 🚨 重要規則

**所有錯誤都必須傳送到 Sentry** - 沒有例外。絕對不要單獨使用 console.error。

## 目前狀態

### Form Service ✅ 已完成
- Sentry v8 已完全整合
- 所有工作流程錯誤已追蹤
- SystemActionQueueProcessor 已加入監控
- 測試端點可用

### Email Service 🟡 進行中
- 第 1-2 階段完成（6/22 項任務）
- 189 個 ErrorLogger.log() 呼叫待處理

## Sentry 整合模式

### 1. 控制器錯誤處理

```typescript
// ✅ 正確 - 使用 BaseController
import { BaseController } from '../controllers/BaseController';

export class MyController extends BaseController {
    async myMethod() {
        try {
            // ... your code
        } catch (error) {
            this.handleError(error, 'myMethod'); // Automatically sends to Sentry
        }
    }
}
```

### 2. 路由錯誤處理（不使用 BaseController）

```typescript
import * as Sentry from '@sentry/node';

router.get('/route', async (req, res) => {
    try {
        // ... your code
    } catch (error) {
        Sentry.captureException(error, {
            tags: { route: '/route', method: 'GET' },
            extra: { userId: req.user?.id }
        });
        res.status(500).json({ error: 'Internal server error' });
    }
});
```

### 3. 工作流程錯誤處理

```typescript
import { WorkflowSentryHelper } from '../workflow/utils/sentryHelper';

// ✅ 正確 - 使用 WorkflowSentryHelper
WorkflowSentryHelper.captureWorkflowError(error, {
    workflowCode: 'DHS_CLOSEOUT',
    instanceId: 123,
    stepId: 456,
    userId: 'user-123',
    operation: 'stepCompletion',
    metadata: { additionalInfo: 'value' }
});
```

### 4. Cron Jobs（必要模式）

```typescript
#!/usr/bin/env node
// FIRST LINE after shebang - CRITICAL!
import '../instrument';
import * as Sentry from '@sentry/node';

async function main() {
    return await Sentry.startSpan({
        name: 'cron.job-name',
        op: 'cron',
        attributes: {
            'cron.job': 'job-name',
            'cron.startTime': new Date().toISOString(),
        }
    }, async () => {
        try {
            // Your cron job logic
        } catch (error) {
            Sentry.captureException(error, {
                tags: {
                    'cron.job': 'job-name',
                    'error.type': 'execution_error'
                }
            });
            console.error('[Job] Error:', error);
            process.exit(1);
        }
    });
}

main()
    .then(() => {
        console.log('[Job] Completed successfully');
        process.exit(0);
    })
    .catch((error) => {
        console.error('[Job] Fatal error:', error);
        process.exit(1);
    });
```

### 5. 資料庫效能監控

```typescript
import { DatabasePerformanceMonitor } from '../utils/databasePerformance';

// ✅ 正確 - 包裝資料庫操作
const result = await DatabasePerformanceMonitor.withPerformanceTracking(
    'findMany',
    'UserProfile',
    async () => {
        return await PrismaService.main.userProfile.findMany({
            take: 5,
        });
    }
);
```

### 6. 非同步操作與 Span

```typescript
import * as Sentry from '@sentry/node';

const result = await Sentry.startSpan({
    name: 'operation.name',
    op: 'operation.type',
    attributes: {
        'custom.attribute': 'value'
    }
}, async () => {
    // Your async operation
    return await someAsyncOperation();
});
```

## 錯誤等級

使用適當的嚴重性等級：

- **fatal**: 系統無法使用（資料庫當機、關鍵服務故障）
- **error**: 操作失敗，需要立即處理
- **warning**: 可恢復的問題、效能降低
- **info**: 資訊訊息、成功操作
- **debug**: 詳細除錯資訊（僅開發環境）

## 必要的上下文

```typescript
import * as Sentry from '@sentry/node';

Sentry.withScope((scope) => {
    // 如果有可用資訊，務必包含這些
    scope.setUser({ id: userId });
    scope.setTag('service', 'form'); // or 'email', 'users', etc.
    scope.setTag('environment', process.env.NODE_ENV);

    // 加入操作特定的上下文
    scope.setContext('operation', {
        type: 'workflow.start',
        workflowCode: 'DHS_CLOSEOUT',
        entityId: 123
    });

    Sentry.captureException(error);
});
```

## 服務特定整合

### Form Service

**位置**: `./blog-api/src/instrument.ts`

```typescript
import * as Sentry from '@sentry/node';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
    dsn: process.env.SENTRY_DSN,
    environment: process.env.NODE_ENV || 'development',
    integrations: [
        nodeProfilingIntegration(),
    ],
    tracesSampleRate: 0.1,
    profilesSampleRate: 0.1,
});
```

**主要輔助工具**:
- `WorkflowSentryHelper` - 工作流程特定錯誤
- `DatabasePerformanceMonitor` - 資料庫查詢追蹤
- `BaseController` - 控制器錯誤處理

### Email Service

**位置**: `./notifications/src/instrument.ts`

```typescript
import * as Sentry from '@sentry/node';
import { nodeProfilingIntegration } from '@sentry/profiling-node';

Sentry.init({
    dsn: process.env.SENTRY_DSN,
    environment: process.env.NODE_ENV || 'development',
    integrations: [
        nodeProfilingIntegration(),
    ],
    tracesSampleRate: 0.1,
    profilesSampleRate: 0.1,
});
```

**主要輔助工具**:
- `EmailSentryHelper` - 郵件特定錯誤
- `BaseController` - 控制器錯誤處理

## 設定檔 (config.ini)

```ini
[sentry]
dsn = your-sentry-dsn
environment = development
tracesSampleRate = 0.1
profilesSampleRate = 0.1

[databaseMonitoring]
enableDbTracing = true
slowQueryThreshold = 100
logDbQueries = false
dbErrorCapture = true
enableN1Detection = true
```

## 測試 Sentry 整合

### Form Service 測試端點

```bash
# Test basic error capture
curl http://localhost:3002/blog-api/api/sentry/test-error

# Test workflow error
curl http://localhost:3002/blog-api/api/sentry/test-workflow-error

# Test database performance
curl http://localhost:3002/blog-api/api/sentry/test-database-performance

# Test error boundary
curl http://localhost:3002/blog-api/api/sentry/test-error-boundary
```

### Email Service 測試端點

```bash
# Test basic error capture
curl http://localhost:3003/notifications/api/sentry/test-error

# Test email-specific error
curl http://localhost:3003/notifications/api/sentry/test-email-error

# Test performance tracking
curl http://localhost:3003/notifications/api/sentry/test-performance
```

## 效能監控

### 需求

1. **所有 API 端點**必須有交易追蹤
2. **超過 100ms 的資料庫查詢**會自動標記
3. **N+1 查詢**會被偵測並回報
4. **Cron jobs** 必須追蹤執行時間

### 交易追蹤

```typescript
import * as Sentry from '@sentry/node';

// Automatic transaction tracking for Express routes
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// Manual transaction for custom operations
const transaction = Sentry.startTransaction({
    op: 'operation.type',
    name: 'Operation Name',
});

try {
    // Your operation
} finally {
    transaction.finish();
}
```

## 常見錯誤

❌ **絕對不要**只使用 console.error 而不傳送到 Sentry
❌ **絕對不要**靜默地吞掉錯誤
❌ **絕對不要**在錯誤上下文中暴露敏感資料
❌ **絕對不要**使用缺乏上下文的通用錯誤訊息
❌ **絕對不要**跳過非同步操作的錯誤處理
❌ **絕對不要**在 cron job 中忘記第一行引入 instrument.ts

## 實作檢查清單

為新程式碼加入 Sentry 時：

- [ ] 已引入 Sentry 或適當的輔助工具
- [ ] 所有 try/catch 區塊都有傳送到 Sentry
- [ ] 已為錯誤加入有意義的上下文
- [ ] 使用了適當的錯誤等級
- [ ] 錯誤訊息中沒有敏感資料
- [ ] 已為緩慢操作加入效能追蹤
- [ ] 已測試錯誤處理路徑
- [ ] 對於 cron job：已在第一行引入 instrument.ts

## 關鍵檔案

### Form Service
- `/blog-api/src/instrument.ts` - Sentry 初始化
- `/blog-api/src/workflow/utils/sentryHelper.ts` - 工作流程錯誤
- `/blog-api/src/utils/databasePerformance.ts` - 資料庫監控
- `/blog-api/src/controllers/BaseController.ts` - 控制器基礎類別

### Email Service
- `/notifications/src/instrument.ts` - Sentry 初始化
- `/notifications/src/utils/EmailSentryHelper.ts` - 郵件錯誤
- `/notifications/src/controllers/BaseController.ts` - 控制器基礎類別

### 設定檔
- `/blog-api/config.ini` - Form service 設定
- `/notifications/config.ini` - Email service 設定
- `/sentry.ini` - 共用 Sentry 設定

## 文件

- 完整實作說明：`/dev/active/email-sentry-integration/`
- Form service 文件：`/blog-api/docs/sentry-integration.md`
- Email service 文件：`/notifications/docs/sentry-integration.md`

## 相關技能

- 在資料庫操作前使用 **database-verification**
- 使用 **workflow-builder** 取得工作流程錯誤上下文
- 使用 **database-scripts** 處理資料庫錯誤
