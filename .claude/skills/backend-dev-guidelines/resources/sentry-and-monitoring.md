# Sentry 整合與監控

Sentry v8 錯誤追蹤與效能監控完整指南。

## 目錄

- [核心原則](#核心原則)
- [Sentry 初始化](#sentry-初始化)
- [錯誤捕捉模式](#錯誤捕捉模式)
- [效能監控](#效能監控)
- [Cron 任務監控](#cron-任務監控)
- [錯誤上下文最佳實務](#錯誤上下文最佳實務)
- [常見錯誤](#常見錯誤)

---

## 核心原則

**必須遵守**：所有錯誤都必須被捕捉到 Sentry。無一例外。

**所有錯誤都必須被捕捉** - 在所有服務中使用 Sentry v8 進行全面的錯誤追蹤。

---

## Sentry 初始化

### instrument.ts 模式

**位置：** `src/instrument.ts`（必須是 server.ts 和所有 cron 任務中的第一個 import）

**微服務範本：**

```typescript
import * as Sentry from '@sentry/node';
import * as fs from 'fs';
import * as path from 'path';
import * as ini from 'ini';

const sentryConfigPath = path.join(__dirname, '../sentry.ini');
const sentryConfig = ini.parse(fs.readFileSync(sentryConfigPath, 'utf-8'));

Sentry.init({
    dsn: sentryConfig.sentry?.dsn,
    environment: process.env.NODE_ENV || 'development',
    tracesSampleRate: parseFloat(sentryConfig.sentry?.tracesSampleRate || '0.1'),
    profilesSampleRate: parseFloat(sentryConfig.sentry?.profilesSampleRate || '0.1'),

    integrations: [
        ...Sentry.getDefaultIntegrations({}),
        Sentry.extraErrorDataIntegration({ depth: 5 }),
        Sentry.localVariablesIntegration(),
        Sentry.requestDataIntegration({
            include: {
                cookies: false,
                data: true,
                headers: true,
                ip: true,
                query_string: true,
                url: true,
                user: { id: true, email: true, username: true },
            },
        }),
        Sentry.consoleIntegration(),
        Sentry.contextLinesIntegration(),
        Sentry.prismaIntegration(),
    ],

    beforeSend(event, hint) {
        // 過濾健康檢查
        if (event.request?.url?.includes('/healthcheck')) {
            return null;
        }

        // 清除敏感標頭
        if (event.request?.headers) {
            delete event.request.headers['authorization'];
            delete event.request.headers['cookie'];
        }

        // 遮罩電子郵件以保護個人資訊
        if (event.user?.email) {
            event.user.email = event.user.email.replace(/^(.{2}).*(@.*)$/, '$1***$2');
        }

        return event;
    },

    ignoreErrors: [
        /^Invalid JWT/,
        /^JWT expired/,
        'NetworkError',
    ],
});

// 設定服務上下文
Sentry.setTags({
    service: 'form',
    version: '1.0.1',
});

Sentry.setContext('runtime', {
    node_version: process.version,
    platform: process.platform,
});
```

**重點事項：**
- 內建個人資訊保護（beforeSend）
- 過濾非關鍵錯誤
- 完整的整合
- Prisma 監測
- 服務特定標籤

---

## 錯誤捕捉模式

### 1. BaseController 模式

```typescript
// Use BaseController.handleError
protected handleError(error: unknown, res: Response, context: string, statusCode = 500): void {
    Sentry.withScope((scope) => {
        scope.setTag('controller', this.constructor.name);
        scope.setTag('operation', context);
        scope.setUser({ id: res.locals?.claims?.userId });
        Sentry.captureException(error);
    });

    res.status(statusCode).json({
        success: false,
        error: { message: error instanceof Error ? error.message : 'Error occurred' }
    });
}
```

### 2. 工作流程錯誤處理

```typescript
import { SentryHelper } from '../utils/sentryHelper';

try {
    await businessOperation();
} catch (error) {
    SentryHelper.captureOperationError(error, {
        operationType: 'POST_CREATION',
        entityId: 123,
        userId: 'user-123',
        operation: 'createPost',
    });
    throw error;
}
```

### 3. 服務層錯誤處理

```typescript
try {
    await someOperation();
} catch (error) {
    Sentry.captureException(error, {
        tags: {
            service: 'form',
            operation: 'someOperation'
        },
        extra: {
            userId: currentUser.id,
            entityId: 123
        }
    });
    throw error;
}
```

---

## 效能監控

### 資料庫效能追蹤

```typescript
import { DatabasePerformanceMonitor } from '../utils/databasePerformance';

const result = await DatabasePerformanceMonitor.withPerformanceTracking(
    'findMany',
    'UserProfile',
    async () => {
        return await PrismaService.main.userProfile.findMany({ take: 5 });
    }
);
```

### API 端點 Span

```typescript
router.post('/operation', async (req, res) => {
    return await Sentry.startSpan({
        name: 'operation.execute',
        op: 'http.server',
        attributes: {
            'http.method': 'POST',
            'http.route': '/operation'
        }
    }, async () => {
        const result = await performOperation();
        res.json(result);
    });
});
```

---

## Cron 任務監控

### 必須遵守的模式

```typescript
#!/usr/bin/env node
import '../instrument'; // shebang 之後的第一行
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
            // Cron 任務邏輯放在這裡
        } catch (error) {
            Sentry.captureException(error, {
                tags: {
                    'cron.job': 'job-name',
                    'error.type': 'execution_error'
                }
            });
            console.error('[Cron] Error:', error);
            process.exit(1);
        }
    });
}

main().then(() => {
    console.log('[Cron] Completed successfully');
    process.exit(0);
}).catch((error) => {
    console.error('[Cron] Fatal error:', error);
    process.exit(1);
});
```

---

## 錯誤上下文最佳實務

### 豐富上下文範例

```typescript
Sentry.withScope((scope) => {
    // 使用者上下文
    scope.setUser({
        id: user.id,
        email: user.email,
        username: user.username
    });

    // 用於過濾的標籤
    scope.setTag('service', 'form');
    scope.setTag('endpoint', req.path);
    scope.setTag('method', req.method);

    // 結構化上下文
    scope.setContext('operation', {
        type: 'workflow.complete',
        workflowId: 123,
        stepId: 456
    });

    // 時間軸麵包屑
    scope.addBreadcrumb({
        category: 'workflow',
        message: 'Starting step completion',
        level: 'info',
        data: { stepId: 456 }
    });

    Sentry.captureException(error);
});
```

---

## 常見錯誤

```typescript
// ❌ 吞掉錯誤
try {
    await riskyOperation();
} catch (error) {
    // 靜默失敗
}

// ❌ 通用錯誤訊息
throw new Error('Error occurred');

// ❌ 暴露敏感資料
Sentry.captureException(error, {
    extra: { password: user.password } // NEVER
});

// ❌ 缺少非同步錯誤處理
async function bad() {
    fetchData().then(data => processResult(data)); // 未處理
}

// ✅ 正確的非同步處理
async function good() {
    try {
        const data = await fetchData();
        processResult(data);
    } catch (error) {
        Sentry.captureException(error);
        throw error;
    }
}
```

---

**相關檔案：**
- [SKILL.md](SKILL.md)
- [routing-and-controllers.md](routing-and-controllers.md)
- [async-and-errors.md](async-and-errors.md)
