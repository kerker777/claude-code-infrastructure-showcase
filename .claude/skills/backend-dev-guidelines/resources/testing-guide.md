# 測試指南 - 後端測試策略

使用 Jest 進行後端服務測試的完整指南及最佳實踐。

## 目錄

- [單元測試](#單元測試)
- [整合測試](#整合測試)
- [Mock 策略](#mock-策略)
- [測試資料管理](#測試資料管理)
- [測試認證路由](#測試認證路由)
- [涵蓋率目標](#涵蓋率目標)

---

## 單元測試

### 測試結構

```typescript
// services/userService.test.ts
import { UserService } from './userService';
import { UserRepository } from '../repositories/UserRepository';

jest.mock('../repositories/UserRepository');

describe('UserService', () => {
    let service: UserService;
    let mockRepository: jest.Mocked<UserRepository>;

    beforeEach(() => {
        mockRepository = {
            findByEmail: jest.fn(),
            create: jest.fn(),
        } as any;

        service = new UserService();
        (service as any).userRepository = mockRepository;
    });

    afterEach(() => {
        jest.clearAllMocks();
    });

    describe('create', () => {
        it('should throw error if email exists', async () => {
            mockRepository.findByEmail.mockResolvedValue({ id: '123' } as any);

            await expect(
                service.create({ email: 'test@test.com' })
            ).rejects.toThrow('Email already in use');
        });

        it('should create user if email is unique', async () => {
            mockRepository.findByEmail.mockResolvedValue(null);
            mockRepository.create.mockResolvedValue({ id: '123' } as any);

            const user = await service.create({
                email: 'test@test.com',
                firstName: 'John',
                lastName: 'Doe',
            });

            expect(user).toBeDefined();
            expect(mockRepository.create).toHaveBeenCalledWith(
                expect.objectContaining({
                    email: 'test@test.com'
                })
            );
        });
    });
});
```

---

## 整合測試

### 使用真實資料庫測試

```typescript
import { PrismaService } from '@project-lifecycle-portal/database';

describe('UserService Integration', () => {
    let testUser: any;

    beforeAll(async () => {
        // 建立測試資料
        testUser = await PrismaService.main.user.create({
            data: {
                email: 'test@test.com',
                profile: { create: { firstName: 'Test', lastName: 'User' } },
            },
        });
    });

    afterAll(async () => {
        // 清理資料
        await PrismaService.main.user.delete({ where: { id: testUser.id } });
    });

    it('should find user by email', async () => {
        const user = await userService.findByEmail('test@test.com');
        expect(user).toBeDefined();
        expect(user?.email).toBe('test@test.com');
    });
});
```

---

## Mock 策略

### Mock PrismaService

```typescript
jest.mock('@project-lifecycle-portal/database', () => ({
    PrismaService: {
        main: {
            user: {
                findMany: jest.fn(),
                findUnique: jest.fn(),
                create: jest.fn(),
                update: jest.fn(),
            },
        },
        isAvailable: true,
    },
}));
```

### Mock Services

```typescript
const mockUserService = {
    findById: jest.fn(),
    create: jest.fn(),
    update: jest.fn(),
} as jest.Mocked<UserService>;
```

---

## 測試資料管理

### 建立與清理

```typescript
describe('PermissionService', () => {
    let instanceId: number;

    beforeAll(async () => {
        // 建立測試貼文
        const post = await PrismaService.main.post.create({
            data: { title: 'Test Post', content: 'Test', authorId: 'test-user' },
        });
        instanceId = post.id;
    });

    afterAll(async () => {
        // 清理資料
        await PrismaService.main.post.delete({
            where: { id: instanceId },
        });
    });

    beforeEach(() => {
        // 清除快取
        permissionService.clearCache();
    });

    it('should check permissions', async () => {
        const hasPermission = await permissionService.checkPermission(
            'user-id',
            instanceId,
            'VIEW_WORKFLOW'
        );
        expect(hasPermission).toBeDefined();
    });
});
```

---

## 測試認證路由

### 使用 test-auth-route.js

```bash
# 測試認證端點
node scripts/test-auth-route.js http://localhost:3002/form/api/users

# 使用 POST 資料測試
node scripts/test-auth-route.js http://localhost:3002/form/api/users POST '{"email":"test@test.com"}'
```

### 在測試中 Mock 認證

```typescript
// Mock 認證 middleware
jest.mock('../middleware/SSOMiddleware', () => ({
    SSOMiddlewareClient: {
        verifyLoginStatus: (req, res, next) => {
            res.locals.claims = {
                sub: 'test-user-id',
                preferred_username: 'testuser',
            };
            next();
        },
    },
}));
```

---

## 涵蓋率目標

### 建議涵蓋率

- **單元測試**：70% 以上涵蓋率
- **整合測試**：涵蓋關鍵路徑
- **E2E 測試**：涵蓋正常流程

### 執行涵蓋率檢查

```bash
npm test -- --coverage
```

---

**相關檔案：**
- [SKILL.md](SKILL.md)
- [services-and-repositories.md](services-and-repositories.md)
- [complete-examples.md](complete-examples.md)
