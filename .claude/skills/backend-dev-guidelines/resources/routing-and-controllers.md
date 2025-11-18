# 路由與控制器 - 最佳實踐

完整指南：乾淨的路由定義與控制器模式。

## 目錄

- [路由：只做路由](#路由只做路由)
- [BaseController 模式](#basecontroller-模式)
- [良好範例](#良好範例)
- [反模式](#反模式)
- [重構指南](#重構指南)
- [錯誤處理](#錯誤處理)
- [HTTP 狀態碼](#http-狀態碼)

---

## 路由：只做路由

### 黃金法則

**路由應該只做：**
- ✅ 定義路由路徑
- ✅ 註冊中介層
- ✅ 委派給控制器

**路由絕對不應該：**
- ❌ 包含業務邏輯
- ❌ 直接存取資料庫
- ❌ 實作驗證邏輯（使用 Zod + 控制器）
- ❌ 格式化複雜的回應
- ❌ 處理複雜的錯誤情境

### 乾淨的路由模式

```typescript
// routes/userRoutes.ts
import { Router } from 'express';
import { UserController } from '../controllers/UserController';
import { SSOMiddlewareClient } from '../middleware/SSOMiddleware';
import { auditMiddleware } from '../middleware/auditMiddleware';

const router = Router();
const controller = new UserController();

// ✅ 乾淨：只有路由定義
router.get('/:id',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.getUser(req, res)
);

router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.createUser(req, res)
);

router.put('/:id',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.updateUser(req, res)
);

export default router;
```

**重點：**
- 每個路由：HTTP 方法、路徑、中介層鏈、控制器委派
- 不需要 try-catch（控制器會處理錯誤）
- 乾淨、可讀、易維護
- 一眼就能看到所有端點

---

## BaseController 模式

### 為什麼要用 BaseController？

**優點：**
- 所有控制器的錯誤處理方式一致
- 自動整合 Sentry
- 標準化的回應格式
- 可重用的輔助方法
- 效能追蹤工具
- 日誌記錄與麵包屑輔助工具

### BaseController 模式（範本）

**檔案：** `/email/src/controllers/BaseController.ts`

```typescript
import * as Sentry from '@sentry/node';
import { Response } from 'express';

export abstract class BaseController {
    /**
     * 處理錯誤並整合 Sentry
     */
    protected handleError(
        error: unknown,
        res: Response,
        context: string,
        statusCode = 500
    ): void {
        Sentry.withScope((scope) => {
            scope.setTag('controller', this.constructor.name);
            scope.setTag('operation', context);
            scope.setUser({ id: res.locals?.claims?.userId });

            if (error instanceof Error) {
                scope.setContext('error_details', {
                    message: error.message,
                    stack: error.stack,
                });
            }

            Sentry.captureException(error);
        });

        res.status(statusCode).json({
            success: false,
            error: {
                message: error instanceof Error ? error.message : 'An error occurred',
                code: statusCode,
            },
        });
    }

    /**
     * 處理成功回應
     */
    protected handleSuccess<T>(
        res: Response,
        data: T,
        message?: string,
        statusCode = 200
    ): void {
        res.status(statusCode).json({
            success: true,
            message,
            data,
        });
    }

    /**
     * 效能追蹤包裝器
     */
    protected async withTransaction<T>(
        name: string,
        operation: string,
        callback: () => Promise<T>
    ): Promise<T> {
        return await Sentry.startSpan(
            { name, op: operation },
            callback
        );
    }

    /**
     * 驗證必填欄位
     */
    protected validateRequest(
        required: string[],
        actual: Record<string, any>,
        res: Response
    ): boolean {
        const missing = required.filter((field) => !actual[field]);

        if (missing.length > 0) {
            Sentry.captureMessage(
                `Missing required fields: ${missing.join(', ')}`,
                'warning'
            );

            res.status(400).json({
                success: false,
                error: {
                    message: 'Missing required fields',
                    code: 'VALIDATION_ERROR',
                    details: { missing },
                },
            });
            return false;
        }
        return true;
    }

    /**
     * 日誌記錄輔助工具
     */
    protected logInfo(message: string, context?: Record<string, any>): void {
        Sentry.addBreadcrumb({
            category: this.constructor.name,
            message,
            level: 'info',
            data: context,
        });
    }

    protected logWarning(message: string, context?: Record<string, any>): void {
        Sentry.captureMessage(message, {
            level: 'warning',
            tags: { controller: this.constructor.name },
            extra: context,
        });
    }

    /**
     * 新增 Sentry 麵包屑
     */
    protected addBreadcrumb(
        message: string,
        category: string,
        data?: Record<string, any>
    ): void {
        Sentry.addBreadcrumb({ message, category, level: 'info', data });
    }

    /**
     * 擷取自訂指標
     */
    protected captureMetric(name: string, value: number, unit: string): void {
        Sentry.metrics.gauge(name, value, { unit });
    }
}
```

### 使用 BaseController

```typescript
// controllers/UserController.ts
import { Request, Response } from 'express';
import { BaseController } from './BaseController';
import { UserService } from '../services/userService';
import { createUserSchema } from '../validators/userSchemas';

export class UserController extends BaseController {
    private userService: UserService;

    constructor() {
        super();
        this.userService = new UserService();
    }

    async getUser(req: Request, res: Response): Promise<void> {
        try {
            this.addBreadcrumb('Fetching user', 'user_controller', { userId: req.params.id });

            const user = await this.userService.findById(req.params.id);

            if (!user) {
                return this.handleError(
                    new Error('User not found'),
                    res,
                    'getUser',
                    404
                );
            }

            this.handleSuccess(res, user);
        } catch (error) {
            this.handleError(error, res, 'getUser');
        }
    }

    async createUser(req: Request, res: Response): Promise<void> {
        try {
            // 驗證輸入
            const validated = createUserSchema.parse(req.body);

            // 追蹤效能
            const user = await this.withTransaction(
                'user.create',
                'db.query',
                () => this.userService.create(validated)
            );

            this.handleSuccess(res, user, 'User created successfully', 201);
        } catch (error) {
            this.handleError(error, res, 'createUser');
        }
    }

    async updateUser(req: Request, res: Response): Promise<void> {
        try {
            const validated = updateUserSchema.parse(req.body);
            const user = await this.userService.update(req.params.id, validated);
            this.handleSuccess(res, user, 'User updated');
        } catch (error) {
            this.handleError(error, res, 'updateUser');
        }
    }
}
```

**優點：**
- 一致的錯誤處理
- 自動整合 Sentry
- 效能追蹤
- 乾淨、可讀的程式碼
- 容易測試

---

## 良好範例

### 範例 1：Email 通知路由（優秀 ✅）

**檔案：** `/email/src/routes/notificationRoutes.ts`

```typescript
import { Router } from 'express';
import { NotificationController } from '../controllers/NotificationController';
import { SSOMiddlewareClient } from '../middleware/SSOMiddleware';

const router = Router();
const controller = new NotificationController();

// ✅ 優秀：乾淨的委派
router.get('/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => controller.getNotifications(req, res)
);

router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => controller.createNotification(req, res)
);

router.put('/:id/read',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => controller.markAsRead(req, res)
);

export default router;
```

**為什麼這是優秀的：**
- 路由中沒有業務邏輯
- 清晰的中介層鏈
- 一致的模式
- 容易理解

### 範例 2：帶驗證的 Proxy 路由（良好 ✅）

**檔案：** `/form/src/routes/proxyRoutes.ts`

```typescript
import { z } from 'zod';

const createProxySchema = z.object({
    originalUserID: z.string().min(1),
    proxyUserID: z.string().min(1),
    startsAt: z.string().datetime(),
    expiresAt: z.string().datetime(),
});

router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => {
        try {
            const validated = createProxySchema.parse(req.body);
            const proxy = await proxyService.createProxyRelationship(validated);
            res.status(201).json({ success: true, data: proxy });
        } catch (error) {
            handler.handleException(res, error);
        }
    }
);
```

**為什麼這是良好的：**
- Zod 驗證
- 委派給服務層
- 適當的 HTTP 狀態碼
- 錯誤處理

**可以更好的地方：**
- 將驗證移到控制器
- 使用 BaseController

---

## 反模式

### 反模式 1：路由中的業務邏輯（糟糕 ❌）

**檔案：** `/form/src/routes/responseRoutes.ts`（實際的 production 程式碼）

```typescript
// ❌ 反模式：路由中有 200+ 行業務邏輯
router.post('/:formID/submit', async (req: Request, res: Response) => {
    try {
        const username = res.locals.claims.preferred_username;
        const responses = req.body.responses;
        const stepInstanceId = req.body.stepInstanceId;

        // ❌ 路由中的權限檢查
        const userId = await userProfileService.getProfileByEmail(username).then(p => p.id);
        const canComplete = await permissionService.canCompleteStep(userId, stepInstanceId);
        if (!canComplete) {
            return res.status(403).json({ error: 'No permission' });
        }

        // ❌ 路由中的工作流程邏輯
        const { createWorkflowEngine, CompleteStepCommand } = require('../workflow/core/WorkflowEngineV3');
        const engine = await createWorkflowEngine();
        const command = new CompleteStepCommand(
            stepInstanceId,
            userId,
            responses,
            additionalContext
        );
        const events = await engine.executeCommand(command);

        // ❌ 路由中的模擬處理
        if (res.locals.isImpersonating) {
            impersonationContextStore.storeContext(stepInstanceId, {
                originalUserId: res.locals.originalUserId,
                effectiveUserId: userId,
            });
        }

        // ❌ 路由中的回應處理
        const post = await PrismaService.main.post.findUnique({
            where: { id: postData.id },
            include: { comments: true },
        });

        // ❌ 路由中的權限檢查
        await checkPostPermissions(post, userId);

        // ... 還有 100+ 行業務邏輯

        res.json({ success: true, data: result });
    } catch (e) {
        handler.handleException(res, e);
    }
});
```

**為什麼這很糟糕：**
- 200+ 行的業務邏輯
- 難以測試（需要 HTTP mocking）
- 難以重用（綁定在路由上）
- 職責混雜
- 難以除錯
- 效能追蹤困難

### 如何重構（逐步說明）

**步驟 1：建立控制器**

```typescript
// controllers/PostController.ts
export class PostController extends BaseController {
    private postService: PostService;

    constructor() {
        super();
        this.postService = new PostService();
    }

    async createPost(req: Request, res: Response): Promise<void> {
        try {
            const validated = createPostSchema.parse({
                ...req.body,
            });

            const result = await this.postService.createPost(
                validated,
                res.locals.userId
            );

            this.handleSuccess(res, result, 'Post created successfully');
        } catch (error) {
            this.handleError(error, res, 'createPost');
        }
    }
}
```

**步驟 2：建立服務**

```typescript
// services/postService.ts
export class PostService {
    async createPost(
        data: CreatePostDTO,
        userId: string
    ): Promise<PostResult> {
        // 權限檢查
        const canCreate = await permissionService.canCreatePost(userId);
        if (!canCreate) {
            throw new ForbiddenError('No permission to create post');
        }

        // 執行工作流程
        const engine = await createWorkflowEngine();
        const command = new CompleteStepCommand(/* ... */);
        const events = await engine.executeCommand(command);

        // 如需要，處理模擬
        if (context.isImpersonating) {
            await this.handleImpersonation(data.stepInstanceId, context);
        }

        // 同步角色
        await this.synchronizeRoles(events, userId);

        return { events, success: true };
    }

    private async handleImpersonation(stepInstanceId: number, context: any) {
        impersonationContextStore.storeContext(stepInstanceId, {
            originalUserId: context.originalUserId,
            effectiveUserId: context.effectiveUserId,
        });
    }

    private async synchronizeRoles(events: WorkflowEvent[], userId: string) {
        // 角色同步邏輯
    }
}
```

**步驟 3：更新路由**

```typescript
// routes/postRoutes.ts
import { PostController } from '../controllers/PostController';

const router = Router();
const controller = new PostController();

// ✅ 乾淨：只做路由
router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.createPost(req, res)
);
```

**結果：**
- 路由：8 行（原本 200+ 行）
- 控制器：25 行（請求處理）
- 服務：50 行（業務邏輯）
- 可測試、可重用、易維護！

---

## 錯誤處理

### 控制器的錯誤處理

```typescript
async createUser(req: Request, res: Response): Promise<void> {
    try {
        const result = await this.userService.create(req.body);
        this.handleSuccess(res, result, 'User created', 201);
    } catch (error) {
        // BaseController.handleError 會自動：
        // - 擷取到 Sentry 並附上 context
        // - 設定適當的狀態碼
        // - 回傳格式化的錯誤回應
        this.handleError(error, res, 'createUser');
    }
}
```

### 自訂錯誤狀態碼

```typescript
async getUser(req: Request, res: Response): Promise<void> {
    try {
        const user = await this.userService.findById(req.params.id);

        if (!user) {
            // 自訂 404 狀態
            return this.handleError(
                new Error('User not found'),
                res,
                'getUser',
                404  // 自訂狀態碼
            );
        }

        this.handleSuccess(res, user);
    } catch (error) {
        this.handleError(error, res, 'getUser');
    }
}
```

### 驗證錯誤

```typescript
async createUser(req: Request, res: Response): Promise<void> {
    try {
        const validated = createUserSchema.parse(req.body);
        const user = await this.userService.create(validated);
        this.handleSuccess(res, user, 'User created', 201);
    } catch (error) {
        // Zod 錯誤會使用 400 狀態碼
        if (error instanceof z.ZodError) {
            return this.handleError(error, res, 'createUser', 400);
        }
        this.handleError(error, res, 'createUser');
    }
}
```

---

## HTTP 狀態碼

### 標準狀態碼

| 代碼 | 使用場景 | 範例 |
|------|----------|---------|
| 200 | 成功（GET、PUT） | 使用者已取得、已更新 |
| 201 | 已建立（POST） | 使用者已建立 |
| 204 | 無內容（DELETE） | 使用者已刪除 |
| 400 | 錯誤請求 | 無效的輸入資料 |
| 401 | 未授權 | 未驗證 |
| 403 | 禁止 | 無權限 |
| 404 | 找不到 | 資源不存在 |
| 409 | 衝突 | 資源重複 |
| 422 | 無法處理的實體 | 驗證失敗 |
| 500 | 內部伺服器錯誤 | 未預期的錯誤 |

### 使用範例

```typescript
// 200 - 成功（預設）
this.handleSuccess(res, user);

// 201 - 已建立
this.handleSuccess(res, user, 'Created', 201);

// 400 - 錯誤請求
this.handleError(error, res, 'operation', 400);

// 404 - 找不到
this.handleError(new Error('Not found'), res, 'operation', 404);

// 403 - 禁止
this.handleError(new ForbiddenError('No permission'), res, 'operation', 403);
```

---

## 重構指南

### 識別需要重構的路由

**警訊：**
- 路由檔案 > 100 行
- 一個路由中有多個 try-catch 區塊
- 直接存取資料庫（Prisma 呼叫）
- 複雜的業務邏輯（if 陳述式、迴圈）
- 路由中的權限檢查

**檢查你的路由：**
```bash
# 找出大型路由檔案
wc -l form/src/routes/*.ts | sort -n

# 找出使用 Prisma 的路由
grep -r "PrismaService" form/src/routes/
```

### 重構流程

**1. 提取到控制器：**
```typescript
// 之前：路由中有邏輯
router.post('/action', async (req, res) => {
    try {
        // 50 行邏輯
    } catch (e) {
        handler.handleException(res, e);
    }
});

// 之後：乾淨的路由
router.post('/action', (req, res) => controller.performAction(req, res));

// 新的控制器方法
async performAction(req: Request, res: Response): Promise<void> {
    try {
        const result = await this.service.performAction(req.body);
        this.handleSuccess(res, result);
    } catch (error) {
        this.handleError(error, res, 'performAction');
    }
}
```

**2. 提取到服務：**
```typescript
// 控制器保持精簡
async performAction(req: Request, res: Response): Promise<void> {
    try {
        const validated = actionSchema.parse(req.body);
        const result = await this.actionService.execute(validated);
        this.handleSuccess(res, result);
    } catch (error) {
        this.handleError(error, res, 'performAction');
    }
}

// 服務包含業務邏輯
export class ActionService {
    async execute(data: ActionDTO): Promise<Result> {
        // 所有業務邏輯都在這裡
        // 權限檢查
        // 資料庫操作
        // 複雜的轉換
        return result;
    }
}
```

**3. 新增倉儲（如需要）：**
```typescript
// 服務呼叫倉儲
export class ActionService {
    constructor(private actionRepository: ActionRepository) {}

    async execute(data: ActionDTO): Promise<Result> {
        // 業務邏輯
        const entity = await this.actionRepository.findById(data.id);
        // 更多邏輯
        return await this.actionRepository.update(data.id, changes);
    }
}

// 倉儲處理資料存取
export class ActionRepository {
    async findById(id: number): Promise<Entity | null> {
        return PrismaService.main.entity.findUnique({ where: { id } });
    }

    async update(id: number, data: Partial<Entity>): Promise<Entity> {
        return PrismaService.main.entity.update({ where: { id }, data });
    }
}
```

---

**相關檔案：**
- [SKILL.md](SKILL.md) - 主要指南
- [services-and-repositories.md](services-and-repositories.md) - 服務層細節
- [complete-examples.md](complete-examples.md) - 完整重構範例
