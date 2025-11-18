# TypeScript 標準

React 前端程式碼中的 TypeScript 最佳實踐，確保型別安全與可維護性。

---

## 嚴格模式

### 設定

專案已**啟用** TypeScript 嚴格模式：

```json
// tsconfig.json
{
    "compilerOptions": {
        "strict": true,
        "noImplicitAny": true,
        "strictNullChecks": true
    }
}
```

**這代表：**
- 不允許隱式的 `any` 型別
- 必須明確處理 null/undefined
- 強制執行型別安全

---

## 禁用 `any` 型別

### 規則

```typescript
// ❌ 絕不使用 any
function handleData(data: any) {
    return data.something;
}

// ✅ 使用明確的型別
interface MyData {
    something: string;
}

function handleData(data: MyData) {
    return data.something;
}

// ✅ 或針對真正未知的資料使用 unknown
function handleUnknown(data: unknown) {
    if (typeof data === 'object' && data !== null && 'something' in data) {
        return (data as MyData).something;
    }
}
```

**如果你真的不知道型別：**
- 使用 `unknown`（強制進行型別檢查）
- 使用型別守衛來縮小範圍
- 記錄為何型別未知

---

## 明確的回傳型別

### 函式回傳型別

```typescript
// ✅ 正確 - 明確的回傳型別
function getUser(id: number): Promise<User> {
    return apiClient.get(`/users/${id}`);
}

function calculateTotal(items: Item[]): number {
    return items.reduce((sum, item) => sum + item.price, 0);
}

// ❌ 避免 - 隱式回傳型別（較不清楚）
function getUser(id: number) {
    return apiClient.get(`/users/${id}`);
}
```

### 元件回傳型別

```typescript
// React.FC 已提供回傳型別 (ReactElement)
export const MyComponent: React.FC<Props> = ({ prop }) => {
    return <div>{prop}</div>;
};

// 針對自訂 Hook
function useMyData(id: number): { data: Data; isLoading: boolean } {
    const [data, setData] = useState<Data | null>(null);
    const [isLoading, setIsLoading] = useState(true);

    return { data: data!, isLoading };
}
```

---

## 型別匯入

### 使用 'type' 關鍵字

```typescript
// ✅ 正確 - 明確標記為型別匯入
import type { User } from '~types/user';
import type { Post } from '~types/post';
import type { SxProps, Theme } from '@mui/material';

// ❌ 避免 - 混合值與型別的匯入
import { User } from '~types/user';  // 不清楚是型別還是值
```

**好處：**
- 清楚區分型別與值
- 更好的 tree-shaking
- 避免循環依賴
- TypeScript 編譯器最佳化

---

## 元件 Prop 介面

### 介面模式

```typescript
/**
 * MyComponent 的 Props
 */
interface MyComponentProps {
    /** 要顯示的使用者 ID */
    userId: number;

    /** 動作完成時的選用回呼函式 */
    onComplete?: () => void;

    /** 元件的顯示模式 */
    mode?: 'view' | 'edit';

    /** 額外的 CSS 類別 */
    className?: string;
}

export const MyComponent: React.FC<MyComponentProps> = ({
    userId,
    onComplete,
    mode = 'view',  // 預設值
    className,
}) => {
    return <div>...</div>;
};
```

**重點：**
- 為 props 建立獨立的介面
- 為每個 prop 加上 JSDoc 註解
- 選用的 props 使用 `?`
- 在解構時提供預設值

### 包含 Children 的 Props

```typescript
interface ContainerProps {
    children: React.ReactNode;
    title: string;
}

// React.FC 會自動包含 children 型別，但明確定義更好
export const Container: React.FC<ContainerProps> = ({ children, title }) => {
    return (
        <div>
            <h2>{title}</h2>
            {children}
        </div>
    );
};
```

---

## 實用型別

### Partial<T>

```typescript
// 將所有屬性設為選用
type UserUpdate = Partial<User>;

function updateUser(id: number, updates: Partial<User>) {
    // updates 可以包含 User 屬性的任意子集
}
```

### Pick<T, K>

```typescript
// 選擇特定的屬性
type UserPreview = Pick<User, 'id' | 'name' | 'email'>;

const preview: UserPreview = {
    id: 1,
    name: 'John',
    email: 'john@example.com',
    // 不允許其他 User 屬性
};
```

### Omit<T, K>

```typescript
// 排除特定的屬性
type UserWithoutPassword = Omit<User, 'password' | 'passwordHash'>;

const publicUser: UserWithoutPassword = {
    id: 1,
    name: 'John',
    email: 'john@example.com',
    // 不允許 password 和 passwordHash
};
```

### Required<T>

```typescript
// 將所有屬性設為必填
type RequiredConfig = Required<Config>;  // 所有選用的 props 變為必填
```

### Record<K, V>

```typescript
// 型別安全的物件/映射
const userMap: Record<string, User> = {
    'user1': { id: 1, name: 'John' },
    'user2': { id: 2, name: 'Jane' },
};

// 用於樣式
import type { SxProps, Theme } from '@mui/material';

const styles: Record<string, SxProps<Theme>> = {
    container: { p: 2 },
    header: { mb: 1 },
};
```

---

## 型別守衛

### 基本型別守衛

```typescript
function isUser(data: unknown): data is User {
    return (
        typeof data === 'object' &&
        data !== null &&
        'id' in data &&
        'name' in data
    );
}

// 使用方式
if (isUser(response)) {
    console.log(response.name);  // TypeScript 知道這是 User
}
```

### 可辨識聯合型別

```typescript
type LoadingState =
    | { status: 'idle' }
    | { status: 'loading' }
    | { status: 'success'; data: Data }
    | { status: 'error'; error: Error };

function Component({ state }: { state: LoadingState }) {
    // TypeScript 根據 status 縮小型別範圍
    if (state.status === 'success') {
        return <Display data={state.data} />;  // 這裡可以使用 data
    }

    if (state.status === 'error') {
        return <Error error={state.error} />;  // 這裡可以使用 error
    }

    return <Loading />;
}
```

---

## 泛型

### 泛型函式

```typescript
function getById<T>(items: T[], id: number): T | undefined {
    return items.find(item => (item as any).id === id);
}

// 使用型別推論
const users: User[] = [...];
const user = getById(users, 123);  // 型別：User | undefined
```

### 泛型元件

```typescript
interface ListProps<T> {
    items: T[];
    renderItem: (item: T) => React.ReactNode;
}

export function List<T>({ items, renderItem }: ListProps<T>): React.ReactElement {
    return (
        <div>
            {items.map((item, index) => (
                <div key={index}>{renderItem(item)}</div>
            ))}
        </div>
    );
}

// 使用方式
<List<User>
    items={users}
    renderItem={(user) => <UserCard user={user} />}
/>
```

---

## 型別斷言（謹慎使用）

### 何時使用

```typescript
// ✅ 可接受 - 當你比 TypeScript 更了解狀況時
const element = document.getElementById('my-element') as HTMLInputElement;
const value = element.value;

// ✅ 可接受 - 已驗證的 API 回應
const response = await api.getData();
const user = response.data as User;  // 你知道資料結構
```

### 何時不使用

```typescript
// ❌ 避免 - 規避型別安全
const data = getData() as any;  // 錯誤 - 破壞了 TypeScript 的用途

// ❌ 避免 - 不安全的斷言
const value = unknownValue as string;  // 實際上可能不是 string
```

---

## Null/Undefined 處理

### 可選鏈

```typescript
// ✅ 正確
const name = user?.profile?.name;

// 等同於：
const name = user && user.profile && user.profile.name;
```

### 空值合併

```typescript
// ✅ 正確
const displayName = user?.name ?? 'Anonymous';

// 只在 null 或 undefined 時使用預設值
// （與 || 不同，|| 會在 '', 0, false 時觸發）
```

### 非空斷言（謹慎使用）

```typescript
// ✅ 可接受 - 當你確定值存在時
const data = queryClient.getQueryData<Data>(['data'])!;

// ⚠️ 小心 - 只在你確定不是 null 時使用
// 更好的做法是明確檢查：
const data = queryClient.getQueryData<Data>(['data']);
if (data) {
    // 使用 data
}
```

---

## 總結

**TypeScript 檢查清單：**
- ✅ 啟用嚴格模式
- ✅ 禁用 `any` 型別（需要時使用 `unknown`）
- ✅ 函式使用明確的回傳型別
- ✅ 型別匯入使用 `import type`
- ✅ Prop 介面加上 JSDoc 註解
- ✅ 實用型別（Partial、Pick、Omit、Required、Record）
- ✅ 使用型別守衛來縮小範圍
- ✅ 可選鏈與空值合併
- ❌ 除非必要，避免使用型別斷言

**另請參閱：**
- [component-patterns.md](component-patterns.md) - 元件型別定義
- [data-fetching.md](data-fetching.md) - API 型別定義
