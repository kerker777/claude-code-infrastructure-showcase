# 非同步模式與錯誤處理

async/await 模式與自訂錯誤處理完整指南。

## 目錄

- [Async/Await 最佳實踐](#asyncawait-best-practices)
- [Promise 錯誤處理](#promise-error-handling)
- [自訂錯誤類型](#custom-error-types)
- [asyncErrorWrapper 工具函式](#asyncerrorwrapper-utility)
- [錯誤傳遞](#error-propagation)
- [常見的非同步陷阱](#common-async-pitfalls)

---

## Async/Await 最佳實踐

### 永遠使用 Try-Catch

```typescript
// ❌ 絕對不要：未處理的非同步錯誤
async function fetchData() {
    const data = await database.query(); // 如果拋出錯誤，將無法處理！
    return data;
}

// ✅ 永遠這樣做：包在 try-catch 裡
async function fetchData() {
    try {
        const data = await database.query();
        return data;
    } catch (error) {
        Sentry.captureException(error);
        throw error;
    }
}
```

### 避免 .then() 鏈

```typescript
// ❌ 避免：Promise 鏈
function processData() {
    return fetchData()
        .then(data => transform(data))
        .then(transformed => save(transformed))
        .catch(error => {
            console.error(error);
        });
}

// ✅ 建議：Async/await
async function processData() {
    try {
        const data = await fetchData();
        const transformed = await transform(data);
        return await save(transformed);
    } catch (error) {
        Sentry.captureException(error);
        throw error;
    }
}
```

---

## Promise 錯誤處理

### 平行操作

```typescript
// ✅ 在 Promise.all 中處理錯誤
try {
    const [users, profiles, settings] = await Promise.all([
        userService.getAll(),
        profileService.getAll(),
        settingsService.getAll(),
    ]);
} catch (error) {
    // 任一失敗全部失敗
    Sentry.captureException(error);
    throw error;
}

// ✅ 使用 Promise.allSettled 個別處理錯誤
const results = await Promise.allSettled([
    userService.getAll(),
    profileService.getAll(),
    settingsService.getAll(),
]);

results.forEach((result, index) => {
    if (result.status === 'rejected') {
        Sentry.captureException(result.reason, {
            tags: { operation: ['users', 'profiles', 'settings'][index] }
        });
    }
});
```

---

## 自訂錯誤類型

### 定義自訂錯誤

```typescript
// 基礎錯誤類別
export class AppError extends Error {
    constructor(
        message: string,
        public code: string,
        public statusCode: number,
        public isOperational: boolean = true
    ) {
        super(message);
        this.name = this.constructor.name;
        Error.captureStackTrace(this, this.constructor);
    }
}

// 特定錯誤類型
export class ValidationError extends AppError {
    constructor(message: string) {
        super(message, 'VALIDATION_ERROR', 400);
    }
}

export class NotFoundError extends AppError {
    constructor(message: string) {
        super(message, 'NOT_FOUND', 404);
    }
}

export class ForbiddenError extends AppError {
    constructor(message: string) {
        super(message, 'FORBIDDEN', 403);
    }
}

export class ConflictError extends AppError {
    constructor(message: string) {
        super(message, 'CONFLICT', 409);
    }
}
```

### 使用方式

```typescript
// 拋出特定錯誤
if (!user) {
    throw new NotFoundError('User not found');
}

if (user.age < 18) {
    throw new ValidationError('User must be 18+');
}

// 錯誤邊界處理
function errorBoundary(error, req, res, next) {
    if (error instanceof AppError) {
        return res.status(error.statusCode).json({
            error: {
                message: error.message,
                code: error.code
            }
        });
    }

    // 未知錯誤
    Sentry.captureException(error);
    res.status(500).json({ error: { message: 'Internal server error' } });
}
```

---

## asyncErrorWrapper 工具函式

### 模式

```typescript
export function asyncErrorWrapper(
    handler: (req: Request, res: Response, next: NextFunction) => Promise<any>
) {
    return async (req: Request, res: Response, next: NextFunction) => {
        try {
            await handler(req, res, next);
        } catch (error) {
            next(error);
        }
    };
}
```

### 使用方式

```typescript
// 沒有包裝器 - 錯誤可能無法處理
router.get('/users', async (req, res) => {
    const users = await userService.getAll(); // 如果拋出錯誤，將無法處理！
    res.json(users);
});

// 使用包裝器 - 錯誤會被捕捉
router.get('/users', asyncErrorWrapper(async (req, res) => {
    const users = await userService.getAll();
    res.json(users);
}));
```

---

## 錯誤傳遞

### 正確的錯誤鏈

```typescript
// ✅ 將錯誤向上傳遞
async function repositoryMethod() {
    try {
        return await PrismaService.main.user.findMany();
    } catch (error) {
        Sentry.captureException(error, { tags: { layer: 'repository' } });
        throw error; // 傳遞到 service
    }
}

async function serviceMethod() {
    try {
        return await repositoryMethod();
    } catch (error) {
        Sentry.captureException(error, { tags: { layer: 'service' } });
        throw error; // 傳遞到 controller
    }
}

async function controllerMethod(req, res) {
    try {
        const result = await serviceMethod();
        res.json(result);
    } catch (error) {
        this.handleError(error, res, 'controllerMethod'); // 最終處理器
    }
}
```

---

## 常見的非同步陷阱

### Fire and Forget（錯誤做法）

```typescript
// ❌ 絕對不要：Fire and forget
async function processRequest(req, res) {
    sendEmail(user.email); // 觸發非同步，錯誤無法處理！
    res.json({ success: true });
}

// ✅ 永遠這樣做：等待或處理
async function processRequest(req, res) {
    try {
        await sendEmail(user.email);
        res.json({ success: true });
    } catch (error) {
        Sentry.captureException(error);
        res.status(500).json({ error: 'Failed to send email' });
    }
}

// ✅ 或者：刻意的背景任務
async function processRequest(req, res) {
    sendEmail(user.email).catch(error => {
        Sentry.captureException(error);
    });
    res.json({ success: true });
}
```

### 未處理的 Rejection

```typescript
// ✅ 針對未處理 rejection 的全域處理器
process.on('unhandledRejection', (reason, promise) => {
    Sentry.captureException(reason, {
        tags: { type: 'unhandled_rejection' }
    });
    console.error('Unhandled Rejection:', reason);
});

process.on('uncaughtException', (error) => {
    Sentry.captureException(error, {
        tags: { type: 'uncaught_exception' }
    });
    console.error('Uncaught Exception:', error);
    process.exit(1);
});
```

---

**相關檔案：**
- [SKILL.md](SKILL.md)
- [sentry-and-monitoring.md](sentry-and-monitoring.md)
- [complete-examples.md](complete-examples.md)
