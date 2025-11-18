---
name: backend-dev-guidelines
description: Node.js/Express/TypeScript 微服務的完整後端開發指南。適用於建立 routes、controllers、services、repositories、middleware，或處理 Express API、Prisma 資料庫存取、Sentry 錯誤追蹤、Zod 驗證、unifiedConfig、依賴注入或非同步模式。涵蓋分層架構（routes → controllers → services → repositories）、BaseController 模式、錯誤處理、效能監控、測試策略，以及從舊有模式的遷移。
---

# 後端開發指南

## 目的

在後端微服務（blog-api、auth-service、notifications-service）中建立一致性和最佳實務，使用現代化的 Node.js/Express/TypeScript 模式。

## 何時使用此技能

在以下情況下會自動啟動：
- 建立或修改 routes、endpoints、APIs
- 建立 controllers、services、repositories
- 實作 middleware（身份驗證、資料驗證、錯誤處理）
- 使用 Prisma 進行資料庫操作
- 使用 Sentry 進行錯誤追蹤
- 使用 Zod 進行輸入驗證
- 設定管理
- 後端測試與重構

---

## 快速開始

### 新後端功能檢查清單

- [ ] **Route**：清楚定義，委派給 controller
- [ ] **Controller**：繼承 BaseController
- [ ] **Service**：使用 DI 的業務邏輯
- [ ] **Repository**：資料庫存取（如果複雜的話）
- [ ] **Validation**：Zod schema
- [ ] **Sentry**：錯誤追蹤
- [ ] **Tests**：單元測試與整合測試
- [ ] **Config**：使用 unifiedConfig

### 新微服務檢查清單

- [ ] 目錄結構（參見 [architecture-overview.md](architecture-overview.md)）
- [ ] instrument.ts 用於 Sentry
- [ ] unifiedConfig 設定
- [ ] BaseController 類別
- [ ] Middleware 堆疊
- [ ] 錯誤邊界
- [ ] 測試框架

---

## 架構概觀

### 分層架構

```
HTTP Request
    ↓
Routes (routing only)
    ↓
Controllers (request handling)
    ↓
Services (business logic)
    ↓
Repositories (data access)
    ↓
Database (Prisma)
```

**核心原則：**每一層只有一個職責。

詳細內容請參見 [architecture-overview.md](architecture-overview.md)。

---

## 目錄結構

```
service/src/
├── config/              # UnifiedConfig
├── controllers/         # Request handlers
├── services/            # Business logic
├── repositories/        # Data access
├── routes/              # Route definitions
├── middleware/          # Express middleware
├── types/               # TypeScript types
├── validators/          # Zod schemas
├── utils/               # Utilities
├── tests/               # Tests
├── instrument.ts        # Sentry (FIRST IMPORT)
├── app.ts               # Express setup
└── server.ts            # HTTP server
```

**命名慣例：**
- Controllers：`PascalCase` - `UserController.ts`
- Services：`camelCase` - `userService.ts`
- Routes：`camelCase + Routes` - `userRoutes.ts`
- Repositories：`PascalCase + Repository` - `UserRepository.ts`

---

## 核心原則（7 個關鍵規則）

### 1. Routes 只負責路由，Controllers 負責控制

```typescript
// ❌ 絕對不要：在 routes 中寫業務邏輯
router.post('/submit', async (req, res) => {
    // 200 行的邏輯
});

// ✅ 永遠這樣做：委派給 controller
router.post('/submit', (req, res) => controller.submit(req, res));
```

### 2. 所有 Controllers 都要繼承 BaseController

```typescript
export class UserController extends BaseController {
    async getUser(req: Request, res: Response): Promise<void> {
        try {
            const user = await this.userService.findById(req.params.id);
            this.handleSuccess(res, user);
        } catch (error) {
            this.handleError(error, res, 'getUser');
        }
    }
}
```

### 3. 所有錯誤都要傳送到 Sentry

```typescript
try {
    await operation();
} catch (error) {
    Sentry.captureException(error);
    throw error;
}
```

### 4. 使用 unifiedConfig，絕對不要用 process.env

```typescript
// ❌ 絕對不要
const timeout = process.env.TIMEOUT_MS;

// ✅ 永遠這樣做
import { config } from './config/unifiedConfig';
const timeout = config.timeouts.default;
```

### 5. 使用 Zod 驗證所有輸入

```typescript
const schema = z.object({ email: z.string().email() });
const validated = schema.parse(req.body);
```

### 6. 使用 Repository 模式進行資料存取

```typescript
// Service → Repository → Database
const users = await userRepository.findActive();
```

### 7. 需要完整的測試

```typescript
describe('UserService', () => {
    it('should create user', async () => {
        expect(user).toBeDefined();
    });
});
```

---

## 常用 Imports

```typescript
// Express
import express, { Request, Response, NextFunction, Router } from 'express';

// Validation
import { z } from 'zod';

// Database
import { PrismaClient } from '@prisma/client';
import type { Prisma } from '@prisma/client';

// Sentry
import * as Sentry from '@sentry/node';

// Config
import { config } from './config/unifiedConfig';

// Middleware
import { SSOMiddlewareClient } from './middleware/SSOMiddleware';
import { asyncErrorWrapper } from './middleware/errorBoundary';
```

---

## 快速參考

### HTTP 狀態碼

| 代碼 | 使用情境 |
|------|----------|
| 200 | 成功 |
| 201 | 已建立 |
| 400 | 錯誤的請求 |
| 401 | 未經授權 |
| 403 | 禁止存取 |
| 404 | 找不到 |
| 500 | 伺服器錯誤 |

### 服務範本

**Blog API**（✅ 成熟）- 作為 REST API 的範本
**Auth Service**（✅ 成熟）- 作為身份驗證模式的範本

---

## 應避免的反模式

❌ 在 routes 中寫業務邏輯
❌ 直接使用 process.env
❌ 缺少錯誤處理
❌ 沒有輸入驗證
❌ 到處直接使用 Prisma
❌ 使用 console.log 而非 Sentry

---

## 導覽指南

| 需要... | 閱讀此文件 |
|------------|-----------|
| 了解架構 | [architecture-overview.md](architecture-overview.md) |
| 建立 routes/controllers | [routing-and-controllers.md](routing-and-controllers.md) |
| 組織業務邏輯 | [services-and-repositories.md](services-and-repositories.md) |
| 驗證輸入 | [validation-patterns.md](validation-patterns.md) |
| 新增錯誤追蹤 | [sentry-and-monitoring.md](sentry-and-monitoring.md) |
| 建立 middleware | [middleware-guide.md](middleware-guide.md) |
| 資料庫存取 | [database-patterns.md](database-patterns.md) |
| 管理設定 | [configuration.md](configuration.md) |
| 處理非同步/錯誤 | [async-and-errors.md](async-and-errors.md) |
| 撰寫測試 | [testing-guide.md](testing-guide.md) |
| 查看範例 | [complete-examples.md](complete-examples.md) |

---

## 資源檔案

### [architecture-overview.md](architecture-overview.md)
分層架構、請求生命週期、關注點分離

### [routing-and-controllers.md](routing-and-controllers.md)
Route 定義、BaseController、錯誤處理、範例

### [services-and-repositories.md](services-and-repositories.md)
Service 模式、DI、repository 模式、快取

### [validation-patterns.md](validation-patterns.md)
Zod schemas、驗證、DTO 模式

### [sentry-and-monitoring.md](sentry-and-monitoring.md)
Sentry 初始化、錯誤捕捉、效能監控

### [middleware-guide.md](middleware-guide.md)
身份驗證、稽核、錯誤邊界、AsyncLocalStorage

### [database-patterns.md](database-patterns.md)
PrismaService、repositories、交易、最佳化

### [configuration.md](configuration.md)
UnifiedConfig、環境設定、機密資料

### [async-and-errors.md](async-and-errors.md)
非同步模式、自訂錯誤、asyncErrorWrapper

### [testing-guide.md](testing-guide.md)
單元/整合測試、模擬、覆蓋率

### [complete-examples.md](complete-examples.md)
完整範例、重構指南

---

## 相關技能

- **database-verification** - 驗證欄位名稱與 schema 一致性
- **error-tracking** - Sentry 整合模式
- **skill-developer** - 建立與管理技能的元技能

---

**技能狀態**：完成 ✅
**行數**：< 500 ✅
**漸進式揭露**：11 個資源檔案 ✅
