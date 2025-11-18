# 驗證模式 - 使用 Zod 進行輸入驗證

使用 Zod schemas 進行型別安全驗證的完整指南。

## 目錄

- [為什麼選擇 Zod？](#為什麼選擇-zod)
- [基本 Zod 模式](#基本-zod-模式)
- [程式碼庫中的 Schema 範例](#程式碼庫中的-schema-範例)
- [路由層級驗證](#路由層級驗證)
- [Controller 驗證](#controller-驗證)
- [DTO 模式](#dto-模式)
- [錯誤處理](#錯誤處理)
- [進階模式](#進階模式)

---

## 為什麼選擇 Zod？

### 相較於 Joi 及其他函式庫的優勢

**型別安全：**
- ✅ 完整的 TypeScript 型別推論
- ✅ 執行時期與編譯時期雙重驗證
- ✅ 自動產生型別定義

**開發體驗：**
- ✅ 直覺的 API 設計
- ✅ 可組合的 schemas
- ✅ 優秀的錯誤訊息

**效能：**
- ✅ 快速的驗證速度
- ✅ 小巧的打包體積
- ✅ 支援 Tree-shaking

### 從 Joi 遷移

現代驗證使用 Zod 取代 Joi：

```typescript
// ❌ 舊寫法 - Joi（正在淘汰中）
const schema = Joi.object({
    email: Joi.string().email().required(),
    name: Joi.string().min(3).required(),
});

// ✅ 新寫法 - Zod（推薦使用）
const schema = z.object({
    email: z.string().email(),
    name: z.string().min(3),
});
```

---

## 基本 Zod 模式

### 基本型別

```typescript
import { z } from 'zod';

// 字串
const nameSchema = z.string();
const emailSchema = z.string().email();
const urlSchema = z.string().url();
const uuidSchema = z.string().uuid();
const minLengthSchema = z.string().min(3);
const maxLengthSchema = z.string().max(100);

// 數字
const ageSchema = z.number().int().positive();
const priceSchema = z.number().positive();
const rangeSchema = z.number().min(0).max(100);

// 布林值
const activeSchema = z.boolean();

// 日期
const dateSchema = z.string().datetime(); // ISO 8601 字串
const nativeDateSchema = z.date(); // 原生 Date 物件

// 列舉
const roleSchema = z.enum(['admin', 'operations', 'user']);
const statusSchema = z.enum(['PENDING', 'APPROVED', 'REJECTED']);
```

### 物件

```typescript
// 簡單物件
const userSchema = z.object({
    email: z.string().email(),
    name: z.string(),
    age: z.number().int().positive(),
});

// 巢狀物件
const addressSchema = z.object({
    street: z.string(),
    city: z.string(),
    zipCode: z.string().regex(/^\d{5}$/),
});

const userWithAddressSchema = z.object({
    name: z.string(),
    address: addressSchema,
});

// 選填欄位
const userSchema = z.object({
    name: z.string(),
    email: z.string().email().optional(),
    phone: z.string().optional(),
});

// 可為 null 的欄位
const userSchema = z.object({
    name: z.string(),
    middleName: z.string().nullable(),
});
```

### 陣列

```typescript
// 基本型別陣列
const rolesSchema = z.array(z.string());
const numbersSchema = z.array(z.number());

// 物件陣列
const usersSchema = z.array(
    z.object({
        id: z.string(),
        name: z.string(),
    })
);

// 帶有限制條件的陣列
const tagsSchema = z.array(z.string()).min(1).max(10);
const nonEmptyArray = z.array(z.string()).nonempty();
```

---

## 程式碼庫中的 Schema 範例

### 表單驗證 Schemas

**檔案：** `/form/src/helpers/zodSchemas.ts`

```typescript
import { z } from 'zod';

// 問題類型列舉
export const questionTypeSchema = z.enum([
    'input',
    'textbox',
    'editor',
    'dropdown',
    'autocomplete',
    'checkbox',
    'radio',
    'upload',
]);

// 上傳類型
export const uploadTypeSchema = z.array(
    z.enum(['pdf', 'image', 'excel', 'video', 'powerpoint', 'word']).nullable()
);

// 輸入類型
export const inputTypeSchema = z
    .enum(['date', 'number', 'input', 'currency'])
    .nullable();

// 問題選項
export const questionOptionSchema = z.object({
    id: z.number().int().positive().optional(),
    controlTag: z.string().max(150).nullable().optional(),
    label: z.string().max(100).nullable().optional(),
    order: z.number().int().min(0).default(0),
});

// 問題 schema
export const questionSchema = z.object({
    id: z.number().int().positive().optional(),
    formID: z.number().int().positive(),
    sectionID: z.number().int().positive().optional(),
    options: z.array(questionOptionSchema).optional(),
    label: z.string().max(500),
    description: z.string().max(5000).optional(),
    type: questionTypeSchema,
    uploadTypes: uploadTypeSchema.optional(),
    inputType: inputTypeSchema.optional(),
    tags: z.array(z.string().max(150)).optional(),
    required: z.boolean(),
    isStandard: z.boolean().optional(),
    deprecatedKey: z.string().nullable().optional(),
    maxLength: z.number().int().positive().nullable().optional(),
    isOptionsSorted: z.boolean().optional(),
});

// 表單區段 schema
export const formSectionSchema = z.object({
    id: z.number().int().positive(),
    formID: z.number().int().positive(),
    questions: z.array(questionSchema).optional(),
    label: z.string().max(500),
    description: z.string().max(5000).optional(),
    isStandard: z.boolean(),
});

// 建立表單 schema
export const createFormSchema = z.object({
    id: z.number().int().positive(),
    label: z.string().max(150),
    description: z.string().max(6000).nullable().optional(),
    isPhase: z.boolean().optional(),
    username: z.string(),
});

// 更新順序 schema
export const updateOrderSchema = z.object({
    source: z.object({
        index: z.number().int().min(0),
        sectionID: z.number().int().min(0),
    }),
    destination: z.object({
        index: z.number().int().min(0),
        sectionID: z.number().int().min(0),
    }),
});

// Controller 專用的驗證 schemas
export const createQuestionValidationSchema = z.object({
    formID: z.number().int().positive(),
    sectionID: z.number().int().positive(),
    question: questionSchema,
    index: z.number().int().min(0).nullable().optional(),
    username: z.string(),
});

export const updateQuestionValidationSchema = z.object({
    questionID: z.number().int().positive(),
    username: z.string(),
    question: questionSchema,
});
```

### 代理關係 Schema

```typescript
// 代理關係驗證
const createProxySchema = z.object({
    originalUserID: z.string().min(1),
    proxyUserID: z.string().min(1),
    startsAt: z.string().datetime(),
    expiresAt: z.string().datetime(),
});

// 帶有自訂驗證
const createProxySchemaWithValidation = createProxySchema.refine(
    (data) => new Date(data.expiresAt) > new Date(data.startsAt),
    {
        message: 'expiresAt must be after startsAt',
        path: ['expiresAt'],
    }
);
```

### 工作流程驗證

```typescript
// 工作流程啟動 schema
const startWorkflowSchema = z.object({
    workflowCode: z.string().min(1),
    entityType: z.enum(['Post', 'User', 'Comment']),
    entityID: z.number().int().positive(),
    dryRun: z.boolean().optional().default(false),
});

// 工作流程步驟完成 schema
const completeStepSchema = z.object({
    stepInstanceID: z.number().int().positive(),
    answers: z.record(z.string(), z.any()),
    dryRun: z.boolean().optional().default(false),
});
```

---

## 路由層級驗證

### 模式 1：內嵌驗證

```typescript
// routes/proxyRoutes.ts
import { z } from 'zod';

const createProxySchema = z.object({
    originalUserID: z.string().min(1),
    proxyUserID: z.string().min(1),
    startsAt: z.string().datetime(),
    expiresAt: z.string().datetime(),
});

router.post(
    '/',
    SSOMiddlewareClient.verifyLoginStatus,
    async (req, res) => {
        try {
            // 在路由層級進行驗證
            const validated = createProxySchema.parse(req.body);

            // 委派給 service
            const proxy = await proxyService.createProxyRelationship(validated);

            res.status(201).json({ success: true, data: proxy });
        } catch (error) {
            if (error instanceof z.ZodError) {
                return res.status(400).json({
                    success: false,
                    error: {
                        message: 'Validation failed',
                        details: error.errors,
                    },
                });
            }
            handler.handleException(res, error);
        }
    }
);
```

**優點：**
- 快速且簡單
- 適合簡單的路由

**缺點：**
- 驗證邏輯寫在路由中
- 較難進行測試
- 無法重複使用

---

## Controller 驗證

### 模式 2：Controller 驗證（推薦）

```typescript
// validators/userSchemas.ts
import { z } from 'zod';

export const createUserSchema = z.object({
    email: z.string().email(),
    name: z.string().min(2).max(100),
    roles: z.array(z.enum(['admin', 'operations', 'user'])),
    isActive: z.boolean().default(true),
});

export const updateUserSchema = z.object({
    email: z.string().email().optional(),
    name: z.string().min(2).max(100).optional(),
    roles: z.array(z.enum(['admin', 'operations', 'user'])).optional(),
    isActive: z.boolean().optional(),
});

export type CreateUserDTO = z.infer<typeof createUserSchema>;
export type UpdateUserDTO = z.infer<typeof updateUserSchema>;
```

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

    async createUser(req: Request, res: Response): Promise<void> {
        try {
            // 驗證輸入
            const validated = createUserSchema.parse(req.body);

            // 呼叫 service
            const user = await this.userService.createUser(validated);

            this.handleSuccess(res, user, 'User created successfully', 201);
        } catch (error) {
            if (error instanceof z.ZodError) {
                // 以 400 狀態碼處理驗證錯誤
                return this.handleError(error, res, 'createUser', 400);
            }
            this.handleError(error, res, 'createUser');
        }
    }

    async updateUser(req: Request, res: Response): Promise<void> {
        try {
            // 驗證參數和 body
            const userId = req.params.id;
            const validated = updateUserSchema.parse(req.body);

            const user = await this.userService.updateUser(userId, validated);

            this.handleSuccess(res, user, 'User updated successfully');
        } catch (error) {
            if (error instanceof z.ZodError) {
                return this.handleError(error, res, 'updateUser', 400);
            }
            this.handleError(error, res, 'updateUser');
        }
    }
}
```

**優點：**
- 清楚的職責分離
- 可重複使用的 schemas
- 容易測試
- 型別安全的 DTOs

**缺點：**
- 需要管理更多檔案

---

## DTO 模式

### 從 Schemas 推論型別

```typescript
import { z } from 'zod';

// 定義 schema
const createUserSchema = z.object({
    email: z.string().email(),
    name: z.string(),
    age: z.number().int().positive(),
});

// 從 schema 推論 TypeScript 型別
type CreateUserDTO = z.infer<typeof createUserSchema>;

// 等同於：
// type CreateUserDTO = {
//     email: string;
//     name: string;
//     age: number;
// }

// 在 service 中使用
class UserService {
    async createUser(data: CreateUserDTO): Promise<User> {
        // data 是完全型別化的！
        console.log(data.email); // ✅ TypeScript 知道這個屬性存在
        console.log(data.invalid); // ❌ TypeScript 錯誤！
    }
}
```

### 輸入與輸出型別

```typescript
// 輸入 schema（API 接收的資料）
const createUserInputSchema = z.object({
    email: z.string().email(),
    name: z.string(),
    password: z.string().min(8),
});

// 輸出 schema（API 回傳的資料）
const userOutputSchema = z.object({
    id: z.string().uuid(),
    email: z.string().email(),
    name: z.string(),
    createdAt: z.string().datetime(),
    // password 不包含在內！
});

type CreateUserInput = z.infer<typeof createUserInputSchema>;
type UserOutput = z.infer<typeof userOutputSchema>;
```

---

## 錯誤處理

### Zod 錯誤格式

```typescript
try {
    const validated = schema.parse(data);
} catch (error) {
    if (error instanceof z.ZodError) {
        console.log(error.errors);
        // [
        //   {
        //     code: 'invalid_type',
        //     expected: 'string',
        //     received: 'number',
        //     path: ['email'],
        //     message: 'Expected string, received number'
        //   }
        // ]
    }
}
```

### 自訂錯誤訊息

```typescript
const userSchema = z.object({
    email: z.string().email({ message: 'Please provide a valid email address' }),
    name: z.string().min(2, { message: 'Name must be at least 2 characters' }),
    age: z.number().int().positive({ message: 'Age must be a positive number' }),
});
```

### 格式化錯誤回應

```typescript
// 格式化 Zod 錯誤的輔助函式
function formatZodError(error: z.ZodError) {
    return {
        message: 'Validation failed',
        errors: error.errors.map((err) => ({
            field: err.path.join('.'),
            message: err.message,
            code: err.code,
        })),
    };
}

// 在 controller 中
catch (error) {
    if (error instanceof z.ZodError) {
        return res.status(400).json({
            success: false,
            error: formatZodError(error),
        });
    }
}

// 回應範例：
// {
//   "success": false,
//   "error": {
//     "message": "Validation failed",
//     "errors": [
//       {
//         "field": "email",
//         "message": "Invalid email",
//         "code": "invalid_string"
//       }
//     ]
//   }
// }
```

---

## 進階模式

### 條件式驗證

```typescript
// 根據其他欄位的值進行驗證
const submissionSchema = z.object({
    type: z.enum(['NEW', 'UPDATE']),
    postId: z.number().optional(),
}).refine(
    (data) => {
        // 如果 type 是 UPDATE，則 postId 為必填
        if (data.type === 'UPDATE') {
            return data.postId !== undefined;
        }
        return true;
    },
    {
        message: 'postId is required when type is UPDATE',
        path: ['postId'],
    }
);
```

### 轉換資料

```typescript
// 將字串轉換為數字
const userSchema = z.object({
    name: z.string(),
    age: z.string().transform((val) => parseInt(val, 10)),
});

// 轉換日期
const eventSchema = z.object({
    name: z.string(),
    date: z.string().transform((str) => new Date(str)),
});
```

### 預處理資料

```typescript
// 驗證前先修剪字串
const userSchema = z.object({
    email: z.preprocess(
        (val) => typeof val === 'string' ? val.trim().toLowerCase() : val,
        z.string().email()
    ),
    name: z.preprocess(
        (val) => typeof val === 'string' ? val.trim() : val,
        z.string().min(2)
    ),
});
```

### Union 型別

```typescript
// 多種可能的型別
const idSchema = z.union([z.string(), z.number()]);

// 可辨識的 unions
const notificationSchema = z.discriminatedUnion('type', [
    z.object({
        type: z.literal('email'),
        recipient: z.string().email(),
        subject: z.string(),
    }),
    z.object({
        type: z.literal('sms'),
        phoneNumber: z.string(),
        message: z.string(),
    }),
]);
```

### 遞迴 Schemas

```typescript
// 用於樹狀結構等巢狀結構
type Category = {
    id: number;
    name: string;
    children?: Category[];
};

const categorySchema: z.ZodType<Category> = z.lazy(() =>
    z.object({
        id: z.number(),
        name: z.string(),
        children: z.array(categorySchema).optional(),
    })
);
```

### Schema 組合

```typescript
// 基礎 schemas
const timestampsSchema = z.object({
    createdAt: z.string().datetime(),
    updatedAt: z.string().datetime(),
});

const auditSchema = z.object({
    createdBy: z.string(),
    updatedBy: z.string(),
});

// 組合 schemas
const userSchema = z.object({
    id: z.string(),
    email: z.string().email(),
    name: z.string(),
}).merge(timestampsSchema).merge(auditSchema);

// 擴充 schemas
const adminUserSchema = userSchema.extend({
    adminLevel: z.number().int().min(1).max(5),
    permissions: z.array(z.string()),
});

// 選取特定欄位
const publicUserSchema = userSchema.pick({
    id: true,
    name: true,
    // email 被排除
});

// 排除欄位
const userWithoutTimestamps = userSchema.omit({
    createdAt: true,
    updatedAt: true,
});
```

### 驗證 Middleware

```typescript
// 建立可重複使用的驗證 middleware
import { Request, Response, NextFunction } from 'express';
import { z } from 'zod';

export function validateBody<T extends z.ZodType>(schema: T) {
    return (req: Request, res: Response, next: NextFunction) => {
        try {
            req.body = schema.parse(req.body);
            next();
        } catch (error) {
            if (error instanceof z.ZodError) {
                return res.status(400).json({
                    success: false,
                    error: {
                        message: 'Validation failed',
                        details: error.errors,
                    },
                });
            }
            next(error);
        }
    };
}

// 使用方式
router.post('/users',
    validateBody(createUserSchema),
    async (req, res) => {
        // req.body 已驗證並具有型別！
        const user = await userService.createUser(req.body);
        res.json({ success: true, data: user });
    }
);
```

---

**相關檔案：**
- [SKILL.md](SKILL.md) - 主要指南
- [routing-and-controllers.md](routing-and-controllers.md) - 在 controllers 中使用驗證
- [services-and-repositories.md](services-and-repositories.md) - 在 services 中使用 DTOs
- [async-and-errors.md](async-and-errors.md) - 錯誤處理模式
