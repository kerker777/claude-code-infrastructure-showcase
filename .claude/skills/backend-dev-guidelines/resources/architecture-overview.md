# Architecture Overview - Backend Services

後端微服務分層架構模式完整指南。

## Table of Contents

- [Layered Architecture Pattern](#layered-architecture-pattern)
- [Request Lifecycle](#request-lifecycle)
- [Service Comparison](#service-comparison)
- [Directory Structure Rationale](#directory-structure-rationale)
- [Module Organization](#module-organization)
- [Separation of Concerns](#separation-of-concerns)

---

## Layered Architecture Pattern

### The Four Layers

```
┌─────────────────────────────────────┐
│         HTTP Request                │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│  Layer 1: ROUTES                    │
│  - Route definitions only           │
│  - Middleware registration          │
│  - Delegate to controllers          │
│  - NO business logic                │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│  Layer 2: CONTROLLERS               │
│  - Request/response handling        │
│  - Input validation                 │
│  - Call services                    │
│  - Format responses                 │
│  - Error handling                   │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│  Layer 3: SERVICES                  │
│  - Business logic                   │
│  - Orchestration                    │
│  - Call repositories                │
│  - No HTTP knowledge                │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│  Layer 4: REPOSITORIES              │
│  - Data access abstraction          │
│  - Prisma operations                │
│  - Query optimization               │
│  - Caching                          │
└───────────────┬─────────────────────┘
                ↓
┌─────────────────────────────────────┐
│         Database (MySQL)            │
└─────────────────────────────────────┘
```

### Why This Architecture?

**可測試性 (Testability)：**
- 每一層都可以獨立測試
- 容易 mock 依賴項目
- 測試邊界清楚

**可維護性 (Maintainability)：**
- 變更可以隔離在特定層級
- 商業邏輯與 HTTP 關注點分離
- 容易定位 bug

**可重用性 (Reusability)：**
- Service 可以被 route、cron job、script 使用
- Repository 隱藏資料庫實作細節
- 商業邏輯不與 HTTP 綁定

**可擴展性 (Scalability)：**
- 容易新增 endpoint
- 有清楚的模式可以遵循
- 結構一致

---

## Request Lifecycle

### Complete Flow Example

```typescript
1. HTTP POST /api/users
   ↓
2. Express matches route in userRoutes.ts
   ↓
3. Middleware chain executes:
   - SSOMiddleware.verifyLoginStatus (authentication)
   - auditMiddleware (context tracking)
   ↓
4. Route handler delegates to controller:
   router.post('/users', (req, res) => userController.create(req, res))
   ↓
5. Controller validates and calls service:
   - Validate input with Zod
   - Call userService.create(data)
   - Handle success/error
   ↓
6. Service executes business logic:
   - Check business rules
   - Call userRepository.create(data)
   - Return result
   ↓
7. Repository performs database operation:
   - PrismaService.main.user.create({ data })
   - Handle database errors
   - Return created user
   ↓
8. Response flows back:
   Repository → Service → Controller → Express → Client
```

### Middleware Execution Order

**重要：** Middleware 依照註冊順序執行

```typescript
app.use(Sentry.Handlers.requestHandler());  // 1. Sentry tracing (FIRST)
app.use(express.json());                     // 2. Body parsing
app.use(express.urlencoded({ extended: true })); // 3. URL encoding
app.use(cookieParser());                     // 4. Cookie parsing
app.use(SSOMiddleware.initialize());         // 5. Auth initialization
// ... routes registered here
app.use(auditMiddleware);                    // 6. Audit (if global)
app.use(errorBoundary);                      // 7. Error handler (LAST)
app.use(Sentry.Handlers.errorHandler());     // 8. Sentry errors (LAST)
```

**規則：** Error handler 必須在 route 之後註冊！

---

## Service Comparison

### Email Service (Mature Pattern ✅)

**優點：**
- 完整的 BaseController，整合 Sentry
- 乾淨的 route 委派（route 中沒有商業邏輯）
- 一致的依賴注入模式
- 良好的 middleware 組織
- 完整的型別安全
- 優秀的錯誤處理

**範例結構：**
```
email/src/
├── controllers/
│   ├── BaseController.ts          ✅ Excellent template
│   ├── NotificationController.ts  ✅ Extends BaseController
│   └── EmailController.ts         ✅ Clean patterns
├── routes/
│   ├── notificationRoutes.ts      ✅ Clean delegation
│   └── emailRoutes.ts             ✅ No business logic
├── services/
│   ├── NotificationService.ts     ✅ Dependency injection
│   └── BatchingService.ts         ✅ Clear responsibility
└── middleware/
    ├── errorBoundary.ts           ✅ Comprehensive
    └── DevImpersonationSSOMiddleware.ts
```

**新服務請參考這個架構！**

### Form Service (Transitioning ⚠️)

**優點：**
- 優秀的 workflow 架構（event sourcing）
- 良好的 Sentry 整合
- 創新的 audit middleware（AsyncLocalStorage）
- 完整的權限系統

**缺點：**
- 某些 route 有 200 多行商業邏輯
- Controller 命名不一致
- 直接使用 process.env（60 多處）
- Repository pattern 使用不足

**範例：**
```
form/src/
├── routes/
│   ├── responseRoutes.ts          ❌ Business logic in routes
│   └── proxyRoutes.ts             ✅ Good validation pattern
├── controllers/
│   ├── formController.ts          ⚠️ Lowercase naming
│   └── UserProfileController.ts   ✅ PascalCase naming
├── workflow/                      ✅ Excellent architecture!
│   ├── core/
│   │   ├── WorkflowEngineV3.ts   ✅ Event sourcing
│   │   └── DryRunWrapper.ts      ✅ Innovative
│   └── services/
└── middleware/
    └── auditMiddleware.ts         ✅ AsyncLocalStorage pattern
```

**可學習：** workflow/、middleware/auditMiddleware.ts
**應避免：** responseRoutes.ts、直接使用 process.env

---

## Directory Structure Rationale

### Controllers Directory

**用途：** 處理 HTTP request/response 相關事務

**內容：**
- `BaseController.ts` - 包含共用方法的基礎類別
- `{Feature}Controller.ts` - 功能專屬 controller

**命名：** PascalCase + Controller

**職責：**
- 解析 request 參數
- 驗證輸入（Zod）
- 呼叫對應的 service method
- 格式化 response
- 處理錯誤（透過 BaseController）
- 設定 HTTP status code

### Services Directory

**用途：** 商業邏輯和協調（orchestration）

**內容：**
- `{feature}Service.ts` - 功能的商業邏輯

**命名：** camelCase + Service（或 PascalCase + Service）

**職責：**
- 實作商業規則
- 協調多個 repository
- 交易管理
- 商業驗證
- 不包含 HTTP 知識（Request/Response 型別）

### Repositories Directory

**用途：** 資料存取抽象層

**內容：**
- `{Entity}Repository.ts` - Entity 的資料庫操作

**命名：** PascalCase + Repository

**職責：**
- Prisma query 操作
- Query 優化
- 資料庫錯誤處理
- Caching 層
- 隱藏 Prisma 實作細節

**目前狀況：** 只有 1 個 repository 存在（WorkflowRepository）

### Routes Directory

**用途：** 僅用於註冊 route

**內容：**
- `{feature}Routes.ts` - 功能的 Express router

**命名：** camelCase + Routes

**職責：**
- 在 Express 註冊 route
- 套用 middleware
- 委派給 controller
- **不可有商業邏輯！**

### Middleware Directory

**用途：** 橫切關注點（cross-cutting concerns）

**內容：**
- Authentication middleware
- Audit middleware
- Error boundary
- Validation middleware
- 自訂 middleware

**命名：** camelCase

**類型：**
- Request 處理（handler 之前）
- Response 處理（handler 之後）
- 錯誤處理（error boundary）

### Config Directory

**用途：** 設定管理

**內容：**
- `unifiedConfig.ts` - 型別安全的設定
- 環境專屬設定

**模式：** 單一事實來源（single source of truth）

### Types Directory

**用途：** TypeScript 型別定義

**內容：**
- `{feature}.types.ts` - 功能專屬型別
- DTO（Data Transfer Object）
- Request/Response 型別
- Domain model

---

## Module Organization

### Feature-Based Organization

大型功能使用子目錄：

```
src/workflow/
├── core/              # Core engine
├── services/          # Workflow-specific services
├── actions/           # System actions
├── models/            # Domain models
├── validators/        # Workflow validation
└── utils/             # Workflow utilities
```

**何時使用：**
- 功能有 5 個以上檔案
- 有明確的子領域
- 邏輯分組能提升清晰度

### Flat Organization

簡單功能使用扁平結構：

```
src/
├── controllers/UserController.ts
├── services/userService.ts
├── routes/userRoutes.ts
└── repositories/UserRepository.ts
```

**何時使用：**
- 簡單功能（少於 5 個檔案）
- 沒有明確的子領域
- 扁平結構更清楚

---

## Separation of Concerns

### What Goes Where

**Routes Layer：**
- ✅ Route 定義
- ✅ Middleware 註冊
- ✅ Controller 委派
- ❌ 商業邏輯
- ❌ 資料庫操作
- ❌ 驗證邏輯（應在 validator 或 controller）

**Controllers Layer：**
- ✅ Request 解析（params、body、query）
- ✅ 輸入驗證（Zod）
- ✅ Service 呼叫
- ✅ Response 格式化
- ✅ 錯誤處理
- ❌ 商業邏輯
- ❌ 資料庫操作

**Services Layer：**
- ✅ 商業邏輯
- ✅ 商業規則執行
- ✅ 協調（多個 repository）
- ✅ 交易管理
- ❌ HTTP 關注點（Request/Response）
- ❌ 直接呼叫 Prisma（使用 repository）

**Repositories Layer：**
- ✅ Prisma 操作
- ✅ Query 建構
- ✅ 資料庫錯誤處理
- ✅ Caching
- ❌ 商業邏輯
- ❌ HTTP 關注點

### Example: User Creation

**Route：**
```typescript
router.post('/users',
    SSOMiddleware.verifyLoginStatus,
    auditMiddleware,
    (req, res) => userController.create(req, res)
);
```

**Controller：**
```typescript
async create(req: Request, res: Response): Promise<void> {
    try {
        const validated = createUserSchema.parse(req.body);
        const user = await this.userService.create(validated);
        this.handleSuccess(res, user, 'User created');
    } catch (error) {
        this.handleError(error, res, 'create');
    }
}
```

**Service：**
```typescript
async create(data: CreateUserDTO): Promise<User> {
    // Business rule: check if email already exists
    const existing = await this.userRepository.findByEmail(data.email);
    if (existing) throw new ConflictError('Email already exists');

    // Create user
    return await this.userRepository.create(data);
}
```

**Repository：**
```typescript
async create(data: CreateUserDTO): Promise<User> {
    return PrismaService.main.user.create({ data });
}

async findByEmail(email: string): Promise<User | null> {
    return PrismaService.main.user.findUnique({ where: { email } });
}
```

**注意：** 每一層都有清楚且獨特的職責！

---

**Related Files:**
- [SKILL.md](SKILL.md) - Main guide
- [routing-and-controllers.md](routing-and-controllers.md) - Routes and controllers details
- [services-and-repositories.md](services-and-repositories.md) - Service and repository patterns
