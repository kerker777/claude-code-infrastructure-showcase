# Database Patterns - Prisma 最佳實踐

在後端微服務中使用 Prisma 進行資料庫存取的完整指南。

## 目錄

- [PrismaService 使用方式](#prismaservice-使用方式)
- [Repository 模式](#repository-模式)
- [Transaction 模式](#transaction-模式)
- [查詢最佳化](#查詢最佳化)
- [N+1 查詢預防](#n1-查詢預防)
- [錯誤處理](#錯誤處理)

---

## PrismaService 使用方式

### 基本模式

```typescript
import { PrismaService } from '@project-lifecycle-portal/database';

// Always use PrismaService.main
const users = await PrismaService.main.user.findMany();
```

### 檢查可用性

```typescript
if (!PrismaService.isAvailable) {
    throw new Error('Prisma client not initialized');
}

const user = await PrismaService.main.user.findUnique({ where: { id } });
```

---

## Repository 模式

### 為什麼要使用 Repository

✅ **在以下情況使用 repository：**
- 包含 join/include 的複雜查詢
- 查詢在多個地方使用
- 需要快取層
- 想要模擬測試

❌ **在以下情況可以跳過 repository：**
- 簡單的一次性查詢
- 原型開發階段（之後可以重構）

### Repository 範本

```typescript
export class UserRepository {
    async findById(id: string): Promise<User | null> {
        return PrismaService.main.user.findUnique({
            where: { id },
            include: { profile: true },
        });
    }

    async findActive(): Promise<User[]> {
        return PrismaService.main.user.findMany({
            where: { isActive: true },
            orderBy: { createdAt: 'desc' },
        });
    }

    async create(data: Prisma.UserCreateInput): Promise<User> {
        return PrismaService.main.user.create({ data });
    }
}
```

---

## Transaction 模式

### 簡單 Transaction

```typescript
const result = await PrismaService.main.$transaction(async (tx) => {
    const user = await tx.user.create({ data: userData });
    const profile = await tx.userProfile.create({ data: { userId: user.id } });
    return { user, profile };
});
```

### 互動式 Transaction

```typescript
const result = await PrismaService.main.$transaction(
    async (tx) => {
        const user = await tx.user.findUnique({ where: { id } });
        if (!user) throw new Error('User not found');

        return await tx.user.update({
            where: { id },
            data: { lastLogin: new Date() },
        });
    },
    {
        maxWait: 5000,
        timeout: 10000,
    }
);
```

---

## 查詢最佳化

### 使用 select 限制欄位

```typescript
// ❌ Fetches all fields
const users = await PrismaService.main.user.findMany();

// ✅ Only fetch needed fields
const users = await PrismaService.main.user.findMany({
    select: {
        id: true,
        email: true,
        profile: { select: { firstName: true, lastName: true } },
    },
});
```

### 謹慎使用 include

```typescript
// ❌ Excessive includes
const user = await PrismaService.main.user.findUnique({
    where: { id },
    include: {
        profile: true,
        posts: { include: { comments: true } },
        workflows: { include: { steps: { include: { actions: true } } } },
    },
});

// ✅ Only include what you need
const user = await PrismaService.main.user.findUnique({
    where: { id },
    include: { profile: true },
});
```

---

## N+1 查詢預防

### 問題：N+1 查詢

```typescript
// ❌ N+1 Query Problem
const users = await PrismaService.main.user.findMany(); // 1 query

for (const user of users) {
    // N queries (one per user)
    const profile = await PrismaService.main.userProfile.findUnique({
        where: { userId: user.id },
    });
}
```

### 解決方案：使用 include 或批次處理

```typescript
// ✅ Single query with include
const users = await PrismaService.main.user.findMany({
    include: { profile: true },
});

// ✅ Or batch query
const userIds = users.map(u => u.id);
const profiles = await PrismaService.main.userProfile.findMany({
    where: { userId: { in: userIds } },
});
```

---

## 錯誤處理

### Prisma 錯誤類型

```typescript
import { Prisma } from '@prisma/client';

try {
    await PrismaService.main.user.create({ data });
} catch (error) {
    if (error instanceof Prisma.PrismaClientKnownRequestError) {
        // Unique constraint violation
        if (error.code === 'P2002') {
            throw new ConflictError('Email already exists');
        }

        // Foreign key constraint
        if (error.code === 'P2003') {
            throw new ValidationError('Invalid reference');
        }

        // Record not found
        if (error.code === 'P2025') {
            throw new NotFoundError('Record not found');
        }
    }

    // Unknown error
    Sentry.captureException(error);
    throw error;
}
```

---

**相關檔案：**
- [SKILL.md](SKILL.md)
- [services-and-repositories.md](services-and-repositories.md)
- [async-and-errors.md](async-and-errors.md)
