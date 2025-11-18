# Services 與 Repositories - 業務邏輯層

完整指南：使用 services 組織業務邏輯，以及使用 repositories 進行資料存取。

## 目錄

- [Service 層概述](#service-層概述)
- [依賴注入模式](#依賴注入模式)
- [Singleton 模式](#singleton-模式)
- [Repository 模式](#repository-模式)
- [Service 設計原則](#service-設計原則)
- [快取策略](#快取策略)
- [測試 Services](#測試-services)

---

## Service 層概述

### Services 的用途

**Services 包含業務邏輯** - 你的應用程式的「做什麼」和「為什麼」：

```
Controller 問：「我應該這樣做嗎？」
Service 回答：「可以/不可以，原因是這樣，接下來會發生這些事」
Repository 執行：「這是你要求的資料」
```

**Services 負責：**
- ✅ 強制執行業務規則
- ✅ 協調多個 repositories
- ✅ 交易管理
- ✅ 複雜計算
- ✅ 整合外部服務
- ✅ 業務驗證

**Services 不應該：**
- ❌ 了解 HTTP（Request/Response）
- ❌ 直接存取 Prisma（使用 repositories）
- ❌ 處理路由特定的邏輯
- ❌ 格式化 HTTP 回應

---

## 依賴注入模式

### 為什麼要用依賴注入？

**好處：**
- 容易測試（注入 mocks）
- 依賴關係清楚
- 配置靈活
- 促進鬆散耦合

### 優秀範例：NotificationService

**檔案：** `/blog-api/src/services/NotificationService.ts`

```typescript
// Define dependencies interface for clarity
export interface NotificationServiceDependencies {
    prisma: PrismaClient;
    batchingService: BatchingService;
    emailComposer: EmailComposer;
}

// Service with dependency injection
export class NotificationService {
    private prisma: PrismaClient;
    private batchingService: BatchingService;
    private emailComposer: EmailComposer;
    private preferencesCache: Map<string, { preferences: UserPreference; timestamp: number }> = new Map();
    private CACHE_TTL = (notificationConfig.preferenceCacheTTLMinutes || 5) * 60 * 1000;

    // Dependencies injected via constructor
    constructor(dependencies: NotificationServiceDependencies) {
        this.prisma = dependencies.prisma;
        this.batchingService = dependencies.batchingService;
        this.emailComposer = dependencies.emailComposer;
    }

    /**
     * Create a notification and route it appropriately
     */
    async createNotification(params: CreateNotificationParams) {
        const { recipientID, type, title, message, link, context = {}, channel = 'both', priority = NotificationPriority.NORMAL } = params;

        try {
            // Get template and render content
            const template = getNotificationTemplate(type);
            const rendered = renderNotificationContent(template, context);

            // Create in-app notification record
            const notificationId = await createNotificationRecord({
                instanceId: parseInt(context.instanceId || '0', 10),
                template: type,
                recipientUserId: recipientID,
                channel: channel === 'email' ? 'email' : 'inApp',
                contextData: context,
                title: finalTitle,
                message: finalMessage,
                link: finalLink,
            });

            // Route notification based on channel
            if (channel === 'email' || channel === 'both') {
                await this.routeNotification({
                    notificationId,
                    userId: recipientID,
                    type,
                    priority,
                    title: finalTitle,
                    message: finalMessage,
                    link: finalLink,
                    context,
                });
            }

            return notification;
        } catch (error) {
            ErrorLogger.log(error, {
                context: {
                    '[NotificationService] createNotification': {
                        type: params.type,
                        recipientID: params.recipientID,
                    },
                },
            });
            throw error;
        }
    }

    /**
     * Route notification based on user preferences
     */
    private async routeNotification(params: { notificationId: number; userId: string; type: string; priority: NotificationPriority; title: string; message: string; link?: string; context?: Record<string, any> }) {
        // Get user preferences with caching
        const preferences = await this.getUserPreferences(params.userId);

        // Check if we should batch or send immediately
        if (this.shouldBatchEmail(preferences, params.type, params.priority)) {
            await this.batchingService.queueNotificationForBatch({
                notificationId: params.notificationId,
                userId: params.userId,
                userPreference: preferences,
                priority: params.priority,
            });
        } else {
            // Send immediately via EmailComposer
            await this.sendImmediateEmail({
                userId: params.userId,
                title: params.title,
                message: params.message,
                link: params.link,
                context: params.context,
                type: params.type,
            });
        }
    }

    /**
     * Determine if email should be batched
     */
    shouldBatchEmail(preferences: UserPreference, notificationType: string, priority: NotificationPriority): boolean {
        // HIGH priority always immediate
        if (priority === NotificationPriority.HIGH) {
            return false;
        }

        // Check batch mode
        const batchMode = preferences.emailBatchMode || BatchMode.IMMEDIATE;
        return batchMode !== BatchMode.IMMEDIATE;
    }

    /**
     * Get user preferences with caching
     */
    async getUserPreferences(userId: string): Promise<UserPreference> {
        // Check cache first
        const cached = this.preferencesCache.get(userId);
        if (cached && Date.now() - cached.timestamp < this.CACHE_TTL) {
            return cached.preferences;
        }

        const preference = await this.prisma.userPreference.findUnique({
            where: { userID: userId },
        });

        const finalPreferences = preference || DEFAULT_PREFERENCES;

        // Update cache
        this.preferencesCache.set(userId, {
            preferences: finalPreferences,
            timestamp: Date.now(),
        });

        return finalPreferences;
    }
}
```

**在 Controller 中使用：**

```typescript
// Instantiate with dependencies
const notificationService = new NotificationService({
    prisma: PrismaService.main,
    batchingService: new BatchingService(PrismaService.main),
    emailComposer: new EmailComposer(),
});

// Use in controller
const notification = await notificationService.createNotification({
    recipientID: 'user-123',
    type: 'AFRLWorkflowNotification',
    context: { workflowName: 'AFRL Monthly Report' },
});
```

**重點整理：**
- 依賴透過建構函式傳入
- 清楚的介面定義所需的依賴
- 容易測試（注入 mocks）
- 快取邏輯封裝完整
- 業務規則與 HTTP 隔離

---

## Singleton 模式

### 何時使用 Singletons

**適用於：**
- 初始化成本高的 services
- 具有共享狀態的 services（快取）
- 從多處存取的 services
- 權限 services
- 配置 services

### 範例：PermissionService（Singleton）

**檔案：** `/blog-api/src/services/permissionService.ts`

```typescript
import { PrismaClient } from '@prisma/client';

class PermissionService {
    private static instance: PermissionService;
    private prisma: PrismaClient;
    private permissionCache: Map<string, { canAccess: boolean; timestamp: number }> = new Map();
    private CACHE_TTL = 5 * 60 * 1000; // 5 minutes

    // Private constructor prevents direct instantiation
    private constructor() {
        this.prisma = PrismaService.main;
    }

    // Get singleton instance
    public static getInstance(): PermissionService {
        if (!PermissionService.instance) {
            PermissionService.instance = new PermissionService();
        }
        return PermissionService.instance;
    }

    /**
     * Check if user can complete a workflow step
     */
    async canCompleteStep(userId: string, stepInstanceId: number): Promise<boolean> {
        const cacheKey = `${userId}:${stepInstanceId}`;

        // Check cache
        const cached = this.permissionCache.get(cacheKey);
        if (cached && Date.now() - cached.timestamp < this.CACHE_TTL) {
            return cached.canAccess;
        }

        try {
            const post = await this.prisma.post.findUnique({
                where: { id: postId },
                include: {
                    author: true,
                    comments: {
                        include: {
                            user: true,
                        },
                    },
                },
            });

            if (!post) {
                return false;
            }

            // Check if user has permission
            const canEdit = post.authorId === userId ||
                await this.isUserAdmin(userId);

            // Cache result
            this.permissionCache.set(cacheKey, {
                canAccess: isAssigned,
                timestamp: Date.now(),
            });

            return isAssigned;
        } catch (error) {
            console.error('[PermissionService] Error checking step permission:', error);
            return false;
        }
    }

    /**
     * Clear cache for user
     */
    clearUserCache(userId: string): void {
        for (const [key] of this.permissionCache) {
            if (key.startsWith(`${userId}:`)) {
                this.permissionCache.delete(key);
            }
        }
    }

    /**
     * Clear all cache
     */
    clearCache(): void {
        this.permissionCache.clear();
    }
}

// Export singleton instance
export const permissionService = PermissionService.getInstance();
```

**使用方式：**

```typescript
import { permissionService } from '../services/permissionService';

// Use anywhere in the codebase
const canComplete = await permissionService.canCompleteStep(userId, stepId);

if (!canComplete) {
    throw new ForbiddenError('You do not have permission to complete this step');
}
```

---

## Repository 模式

### Repositories 的用途

**Repositories 抽象化資料存取** - 資料操作的「如何做」：

```
Service：「給我所有按名稱排序的活躍使用者」
Repository：「這是能做到的 Prisma 查詢」
```

**Repositories 負責：**
- ✅ 所有 Prisma 操作
- ✅ 查詢建構
- ✅ 查詢優化（select、include）
- ✅ 資料庫錯誤處理
- ✅ 快取資料庫結果

**Repositories 不應該：**
- ❌ 包含業務邏輯
- ❌ 了解 HTTP
- ❌ 做決策（那是 service 層的事）

### Repository 模板

```typescript
// repositories/UserRepository.ts
import { PrismaService } from '@project-lifecycle-portal/database';
import type { User, Prisma } from '@project-lifecycle-portal/database';

export class UserRepository {
    /**
     * Find user by ID with optimized query
     */
    async findById(userId: string): Promise<User | null> {
        try {
            return await PrismaService.main.user.findUnique({
                where: { userID: userId },
                select: {
                    userID: true,
                    email: true,
                    name: true,
                    isActive: true,
                    roles: true,
                    createdAt: true,
                    updatedAt: true,
                },
            });
        } catch (error) {
            console.error('[UserRepository] Error finding user by ID:', error);
            throw new Error(`Failed to find user: ${userId}`);
        }
    }

    /**
     * Find all active users
     */
    async findActive(options?: { orderBy?: Prisma.UserOrderByWithRelationInput }): Promise<User[]> {
        try {
            return await PrismaService.main.user.findMany({
                where: { isActive: true },
                orderBy: options?.orderBy || { name: 'asc' },
                select: {
                    userID: true,
                    email: true,
                    name: true,
                    roles: true,
                },
            });
        } catch (error) {
            console.error('[UserRepository] Error finding active users:', error);
            throw new Error('Failed to find active users');
        }
    }

    /**
     * Find user by email
     */
    async findByEmail(email: string): Promise<User | null> {
        try {
            return await PrismaService.main.user.findUnique({
                where: { email },
            });
        } catch (error) {
            console.error('[UserRepository] Error finding user by email:', error);
            throw new Error(`Failed to find user with email: ${email}`);
        }
    }

    /**
     * Create new user
     */
    async create(data: Prisma.UserCreateInput): Promise<User> {
        try {
            return await PrismaService.main.user.create({ data });
        } catch (error) {
            console.error('[UserRepository] Error creating user:', error);
            throw new Error('Failed to create user');
        }
    }

    /**
     * Update user
     */
    async update(userId: string, data: Prisma.UserUpdateInput): Promise<User> {
        try {
            return await PrismaService.main.user.update({
                where: { userID: userId },
                data,
            });
        } catch (error) {
            console.error('[UserRepository] Error updating user:', error);
            throw new Error(`Failed to update user: ${userId}`);
        }
    }

    /**
     * Delete user (soft delete by setting isActive = false)
     */
    async delete(userId: string): Promise<User> {
        try {
            return await PrismaService.main.user.update({
                where: { userID: userId },
                data: { isActive: false },
            });
        } catch (error) {
            console.error('[UserRepository] Error deleting user:', error);
            throw new Error(`Failed to delete user: ${userId}`);
        }
    }

    /**
     * Check if email exists
     */
    async emailExists(email: string): Promise<boolean> {
        try {
            const count = await PrismaService.main.user.count({
                where: { email },
            });
            return count > 0;
        } catch (error) {
            console.error('[UserRepository] Error checking email exists:', error);
            throw new Error('Failed to check if email exists');
        }
    }
}

// Export singleton instance
export const userRepository = new UserRepository();
```

**在 Service 中使用 Repository：**

```typescript
// services/userService.ts
import { userRepository } from '../repositories/UserRepository';
import { ConflictError, NotFoundError } from '../utils/errors';

export class UserService {
    /**
     * Create new user with business rules
     */
    async createUser(data: { email: string; name: string; roles: string[] }): Promise<User> {
        // Business rule: Check if email already exists
        const emailExists = await userRepository.emailExists(data.email);
        if (emailExists) {
            throw new ConflictError('Email already exists');
        }

        // Business rule: Validate roles
        const validRoles = ['admin', 'operations', 'user'];
        const invalidRoles = data.roles.filter((role) => !validRoles.includes(role));
        if (invalidRoles.length > 0) {
            throw new ValidationError(`Invalid roles: ${invalidRoles.join(', ')}`);
        }

        // Create user via repository
        return await userRepository.create({
            email: data.email,
            name: data.name,
            roles: data.roles,
            isActive: true,
        });
    }

    /**
     * Get user by ID
     */
    async getUser(userId: string): Promise<User> {
        const user = await userRepository.findById(userId);

        if (!user) {
            throw new NotFoundError(`User not found: ${userId}`);
        }

        return user;
    }
}
```

---

## Service 設計原則

### 1. 單一職責

每個 service 應該有一個明確的用途：

```typescript
// ✅ 好 - 單一職責
class UserService {
    async createUser() {}
    async updateUser() {}
    async deleteUser() {}
}

class EmailService {
    async sendEmail() {}
    async sendBulkEmails() {}
}

// ❌ 壞 - 太多職責
class UserService {
    async createUser() {}
    async sendWelcomeEmail() {}  // 應該在 EmailService
    async logUserActivity() {}   // 應該在 AuditService
    async processPayment() {}    // 應該在 PaymentService
}
```

### 2. 清楚的方法名稱

方法名稱應該描述它們做什麼：

```typescript
// ✅ 好 - 意圖清楚
async createNotification()
async getUserPreferences()
async shouldBatchEmail()
async routeNotification()

// ❌ 壞 - 模糊或誤導
async process()
async handle()
async doIt()
async execute()
```

### 3. 回傳型別

永遠使用明確的回傳型別：

```typescript
// ✅ 好 - 明確的型別
async createUser(data: CreateUserDTO): Promise<User> {}
async findUsers(): Promise<User[]> {}
async deleteUser(id: string): Promise<void> {}

// ❌ 壞 - 隱含的 any
async createUser(data) {}  // 沒有型別！
```

### 4. 錯誤處理

Services 應該拋出有意義的錯誤：

```typescript
// ✅ 好 - 有意義的錯誤
if (!user) {
    throw new NotFoundError(`User not found: ${userId}`);
}

if (emailExists) {
    throw new ConflictError('Email already exists');
}

// ❌ 壞 - 通用錯誤
if (!user) {
    throw new Error('Error');  // 什麼錯誤？
}
```

### 5. 避免萬能 Services

不要建立什麼都做的 services：

```typescript
// ❌ 壞 - 萬能 service
class WorkflowService {
    async startWorkflow() {}
    async completeStep() {}
    async assignRoles() {}
    async sendNotifications() {}  // 應該在 NotificationService
    async validatePermissions() {}  // 應該在 PermissionService
    async logAuditTrail() {}  // 應該在 AuditService
    // ... 還有 50 個方法
}

// ✅ 好 - 專注的 services
class WorkflowService {
    constructor(
        private notificationService: NotificationService,
        private permissionService: PermissionService,
        private auditService: AuditService
    ) {}

    async startWorkflow() {
        // Orchestrate other services
        await this.permissionService.checkPermission();
        await this.workflowRepository.create();
        await this.notificationService.notify();
        await this.auditService.log();
    }
}
```

---

## 快取策略

### 1. 記憶體內快取

```typescript
class UserService {
    private cache: Map<string, { user: User; timestamp: number }> = new Map();
    private CACHE_TTL = 5 * 60 * 1000; // 5 minutes

    async getUser(userId: string): Promise<User> {
        // Check cache
        const cached = this.cache.get(userId);
        if (cached && Date.now() - cached.timestamp < this.CACHE_TTL) {
            return cached.user;
        }

        // Fetch from database
        const user = await userRepository.findById(userId);

        // Update cache
        if (user) {
            this.cache.set(userId, { user, timestamp: Date.now() });
        }

        return user;
    }

    clearUserCache(userId: string): void {
        this.cache.delete(userId);
    }
}
```

### 2. 快取失效

```typescript
class UserService {
    async updateUser(userId: string, data: UpdateUserDTO): Promise<User> {
        // Update in database
        const user = await userRepository.update(userId, data);

        // Invalidate cache
        this.clearUserCache(userId);

        return user;
    }
}
```

---

## 測試 Services

### 單元測試

```typescript
// tests/userService.test.ts
import { UserService } from '../services/userService';
import { userRepository } from '../repositories/UserRepository';
import { ConflictError } from '../utils/errors';

// Mock repository
jest.mock('../repositories/UserRepository');

describe('UserService', () => {
    let userService: UserService;

    beforeEach(() => {
        userService = new UserService();
        jest.clearAllMocks();
    });

    describe('createUser', () => {
        it('should create user when email does not exist', async () => {
            // Arrange
            const userData = {
                email: 'test@example.com',
                name: 'Test User',
                roles: ['user'],
            };

            (userRepository.emailExists as jest.Mock).mockResolvedValue(false);
            (userRepository.create as jest.Mock).mockResolvedValue({
                userID: '123',
                ...userData,
            });

            // Act
            const user = await userService.createUser(userData);

            // Assert
            expect(user).toBeDefined();
            expect(user.email).toBe(userData.email);
            expect(userRepository.emailExists).toHaveBeenCalledWith(userData.email);
            expect(userRepository.create).toHaveBeenCalled();
        });

        it('should throw ConflictError when email exists', async () => {
            // Arrange
            const userData = {
                email: 'existing@example.com',
                name: 'Test User',
                roles: ['user'],
            };

            (userRepository.emailExists as jest.Mock).mockResolvedValue(true);

            // Act & Assert
            await expect(userService.createUser(userData)).rejects.toThrow(ConflictError);
            expect(userRepository.create).not.toHaveBeenCalled();
        });
    });
});
```

---

**相關檔案：**
- [SKILL.md](SKILL.md) - 主要指南
- [routing-and-controllers.md](routing-and-controllers.md) - 使用 services 的 Controllers
- [database-patterns.md](database-patterns.md) - Prisma 與 repository 模式
- [complete-examples.md](complete-examples.md) - 完整的 service/repository 範例
