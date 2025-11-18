# 完整範例 - 完整可執行程式碼

展示完整實作模式的實際範例。

## 目錄

- [完整的 Controller 範例](#完整的-controller-範例)
- [完整的 Service 與依賴注入](#完整的-service-與依賴注入)
- [完整的路由檔案](#完整的路由檔案)
- [完整的 Repository](#完整的-repository)
- [重構範例：從不良到良好](#重構範例從不良到良好)
- [端到端功能範例](#端到端功能範例)

---

## 完整的 Controller 範例

### UserController（遵循所有最佳實踐）

```typescript
// controllers/UserController.ts
import { Request, Response } from 'express';
import { BaseController } from './BaseController';
import { UserService } from '../services/userService';
import { createUserSchema, updateUserSchema } from '../validators/userSchemas';
import { z } from 'zod';

export class UserController extends BaseController {
    private userService: UserService;

    constructor() {
        super();
        this.userService = new UserService();
    }

    async getUser(req: Request, res: Response): Promise<void> {
        try {
            this.addBreadcrumb('Fetching user', 'user_controller', {
                userId: req.params.id,
            });

            const user = await this.withTransaction(
                'user.get',
                'db.query',
                () => this.userService.findById(req.params.id)
            );

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

    async listUsers(req: Request, res: Response): Promise<void> {
        try {
            const users = await this.userService.getAll();
            this.handleSuccess(res, users);
        } catch (error) {
            this.handleError(error, res, 'listUsers');
        }
    }

    async createUser(req: Request, res: Response): Promise<void> {
        try {
            // Validate input with Zod
            const validated = createUserSchema.parse(req.body);

            // Track performance
            const user = await this.withTransaction(
                'user.create',
                'db.mutation',
                () => this.userService.create(validated)
            );

            this.handleSuccess(res, user, 'User created successfully', 201);
        } catch (error) {
            if (error instanceof z.ZodError) {
                return this.handleError(error, res, 'createUser', 400);
            }
            this.handleError(error, res, 'createUser');
        }
    }

    async updateUser(req: Request, res: Response): Promise<void> {
        try {
            const validated = updateUserSchema.parse(req.body);

            const user = await this.userService.update(
                req.params.id,
                validated
            );

            this.handleSuccess(res, user, 'User updated');
        } catch (error) {
            if (error instanceof z.ZodError) {
                return this.handleError(error, res, 'updateUser', 400);
            }
            this.handleError(error, res, 'updateUser');
        }
    }

    async deleteUser(req: Request, res: Response): Promise<void> {
        try {
            await this.userService.delete(req.params.id);
            this.handleSuccess(res, null, 'User deleted', 204);
        } catch (error) {
            this.handleError(error, res, 'deleteUser');
        }
    }
}
```

---

## 完整的 Service 與依賴注入

### UserService

```typescript
// services/userService.ts
import { UserRepository } from '../repositories/UserRepository';
import { ConflictError, NotFoundError, ValidationError } from '../types/errors';
import type { CreateUserDTO, UpdateUserDTO, User } from '../types/user.types';

export class UserService {
    private userRepository: UserRepository;

    constructor(userRepository?: UserRepository) {
        this.userRepository = userRepository || new UserRepository();
    }

    async findById(id: string): Promise<User | null> {
        return await this.userRepository.findById(id);
    }

    async getAll(): Promise<User[]> {
        return await this.userRepository.findActive();
    }

    async create(data: CreateUserDTO): Promise<User> {
        // Business rule: validate age
        if (data.age < 18) {
            throw new ValidationError('User must be 18 or older');
        }

        // Business rule: check email uniqueness
        const existing = await this.userRepository.findByEmail(data.email);
        if (existing) {
            throw new ConflictError('Email already in use');
        }

        // Create user with profile
        return await this.userRepository.create({
            email: data.email,
            profile: {
                create: {
                    firstName: data.firstName,
                    lastName: data.lastName,
                    age: data.age,
                },
            },
        });
    }

    async update(id: string, data: UpdateUserDTO): Promise<User> {
        // Check exists
        const existing = await this.userRepository.findById(id);
        if (!existing) {
            throw new NotFoundError('User not found');
        }

        // Business rule: email uniqueness if changing
        if (data.email && data.email !== existing.email) {
            const emailTaken = await this.userRepository.findByEmail(data.email);
            if (emailTaken) {
                throw new ConflictError('Email already in use');
            }
        }

        return await this.userRepository.update(id, data);
    }

    async delete(id: string): Promise<void> {
        const existing = await this.userRepository.findById(id);
        if (!existing) {
            throw new NotFoundError('User not found');
        }

        await this.userRepository.delete(id);
    }
}
```

---

## 完整的路由檔案

### userRoutes.ts

```typescript
// routes/userRoutes.ts
import { Router } from 'express';
import { UserController } from '../controllers/UserController';
import { SSOMiddlewareClient } from '../middleware/SSOMiddleware';
import { auditMiddleware } from '../middleware/auditMiddleware';

const router = Router();
const controller = new UserController();

// GET /users - List all users
router.get('/',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.listUsers(req, res)
);

// GET /users/:id - Get single user
router.get('/:id',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.getUser(req, res)
);

// POST /users - Create user
router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.createUser(req, res)
);

// PUT /users/:id - Update user
router.put('/:id',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.updateUser(req, res)
);

// DELETE /users/:id - Delete user
router.delete('/:id',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.deleteUser(req, res)
);

export default router;
```

---

## 完整的 Repository

### UserRepository

```typescript
// repositories/UserRepository.ts
import { PrismaService } from '@project-lifecycle-portal/database';
import type { User, Prisma } from '@prisma/client';

export class UserRepository {
    async findById(id: string): Promise<User | null> {
        return PrismaService.main.user.findUnique({
            where: { id },
            include: { profile: true },
        });
    }

    async findByEmail(email: string): Promise<User | null> {
        return PrismaService.main.user.findUnique({
            where: { email },
            include: { profile: true },
        });
    }

    async findActive(): Promise<User[]> {
        return PrismaService.main.user.findMany({
            where: { isActive: true },
            include: { profile: true },
            orderBy: { createdAt: 'desc' },
        });
    }

    async create(data: Prisma.UserCreateInput): Promise<User> {
        return PrismaService.main.user.create({
            data,
            include: { profile: true },
        });
    }

    async update(id: string, data: Prisma.UserUpdateInput): Promise<User> {
        return PrismaService.main.user.update({
            where: { id },
            data,
            include: { profile: true },
        });
    }

    async delete(id: string): Promise<User> {
        // Soft delete
        return PrismaService.main.user.update({
            where: { id },
            data: {
                isActive: false,
                deletedAt: new Date(),
            },
        });
    }
}
```

---

## 重構範例：從不良到良好

### 重構前：商業邏輯寫在路由中 ❌

```typescript
// routes/postRoutes.ts (BAD - 200+ lines)
router.post('/posts', async (req, res) => {
    try {
        const username = res.locals.claims.preferred_username;
        const responses = req.body.responses;
        const stepInstanceId = req.body.stepInstanceId;

        // ❌ Permission check in route
        const userId = await userProfileService.getProfileByEmail(username).then(p => p.id);
        const canComplete = await permissionService.canCompleteStep(userId, stepInstanceId);
        if (!canComplete) {
            return res.status(403).json({ error: 'No permission' });
        }

        // ❌ Business logic in route
        const post = await postRepository.create({
            title: req.body.title,
            content: req.body.content,
            authorId: userId
        });

        // ❌ More business logic...
        if (res.locals.isImpersonating) {
            impersonationContextStore.storeContext(...);
        }

        // ... 100+ more lines

        res.json({ success: true, data: result });
    } catch (e) {
        handler.handleException(res, e);
    }
});
```

### 重構後：清晰的分離 ✅

**1. 清晰的路由：**
```typescript
// routes/postRoutes.ts
import { PostController } from '../controllers/PostController';

const router = Router();
const controller = new PostController();

// ✅ CLEAN: 總共只有 8 行！
router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    auditMiddleware,
    async (req, res) => controller.createPost(req, res)
);

export default router;
```

**2. Controller：**
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

**3. Service：**
```typescript
// services/postService.ts
export class PostService {
    async createPost(
        data: CreatePostDTO,
        userId: string
    ): Promise<SubmissionResult> {
        // Permission check
        const canComplete = await permissionService.canCompleteStep(
            userId,
            data.stepInstanceId
        );

        if (!canComplete) {
            throw new ForbiddenError('No permission to complete step');
        }

        // Execute workflow
        const engine = await createWorkflowEngine();
        const command = new CompleteStepCommand(
            data.stepInstanceId,
            userId,
            data.responses
        );
        const events = await engine.executeCommand(command);

        // Handle impersonation
        if (context.isImpersonating) {
            await this.handleImpersonation(data.stepInstanceId, context);
        }

        return { events, success: true };
    }

    private async handleImpersonation(stepInstanceId: number, context: any) {
        impersonationContextStore.storeContext(stepInstanceId, {
            originalUserId: context.originalUserId,
            effectiveUserId: context.effectiveUserId,
        });
    }
}
```

**結果：**
- 路由：8 行（原本超過 200 行）
- Controller：25 行
- Service：40 行
- **可測試、可維護、可重用！**

---

## 端到端功能範例

### 完整的使用者管理功能

**1. 型別定義：**
```typescript
// types/user.types.ts
export interface User {
    id: string;
    email: string;
    isActive: boolean;
    profile?: UserProfile;
}

export interface CreateUserDTO {
    email: string;
    firstName: string;
    lastName: string;
    age: number;
}

export interface UpdateUserDTO {
    email?: string;
    firstName?: string;
    lastName?: string;
}
```

**2. 驗證器：**
```typescript
// validators/userSchemas.ts
import { z } from 'zod';

export const createUserSchema = z.object({
    email: z.string().email(),
    firstName: z.string().min(1).max(100),
    lastName: z.string().min(1).max(100),
    age: z.number().int().min(18).max(120),
});

export const updateUserSchema = z.object({
    email: z.string().email().optional(),
    firstName: z.string().min(1).max(100).optional(),
    lastName: z.string().min(1).max(100).optional(),
});
```

**3. Repository：**
```typescript
// repositories/UserRepository.ts
export class UserRepository {
    async findById(id: string): Promise<User | null> {
        return PrismaService.main.user.findUnique({
            where: { id },
            include: { profile: true },
        });
    }

    async create(data: Prisma.UserCreateInput): Promise<User> {
        return PrismaService.main.user.create({
            data,
            include: { profile: true },
        });
    }
}
```

**4. Service：**
```typescript
// services/userService.ts
export class UserService {
    private userRepository: UserRepository;

    constructor() {
        this.userRepository = new UserRepository();
    }

    async create(data: CreateUserDTO): Promise<User> {
        const existing = await this.userRepository.findByEmail(data.email);
        if (existing) {
            throw new ConflictError('Email already exists');
        }

        return await this.userRepository.create({
            email: data.email,
            profile: {
                create: {
                    firstName: data.firstName,
                    lastName: data.lastName,
                    age: data.age,
                },
            },
        });
    }
}
```

**5. Controller：**
```typescript
// controllers/UserController.ts
export class UserController extends BaseController {
    private userService: UserService;

    constructor() {
        super();
        this.userService = new UserService();
    }

    async createUser(req: Request, res: Response): Promise<void> {
        try {
            const validated = createUserSchema.parse(req.body);
            const user = await this.userService.create(validated);
            this.handleSuccess(res, user, 'User created', 201);
        } catch (error) {
            this.handleError(error, res, 'createUser');
        }
    }
}
```

**6. 路由：**
```typescript
// routes/userRoutes.ts
const router = Router();
const controller = new UserController();

router.post('/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => controller.createUser(req, res)
);

export default router;
```

**7. 在 app.ts 中註冊：**
```typescript
// app.ts
import userRoutes from './routes/userRoutes';

app.use('/api/users', userRoutes);
```

**完整的請求流程：**
```
POST /api/users
  ↓
userRoutes 匹配到 /
  ↓
SSOMiddleware 進行身份驗證
  ↓
呼叫 controller.createUser
  ↓
使用 Zod 進行驗證
  ↓
呼叫 userService.create
  ↓
檢查商業規則
  ↓
呼叫 userRepository.create
  ↓
Prisma 建立使用者
  ↓
逐層往上回傳
  ↓
Controller 格式化回應
  ↓
回傳 200/201 給客戶端
```

---

**相關檔案：**
- [SKILL.md](SKILL.md)
- [routing-and-controllers.md](routing-and-controllers.md)
- [services-and-repositories.md](services-and-repositories.md)
- [validation-patterns.md](validation-patterns.md)
