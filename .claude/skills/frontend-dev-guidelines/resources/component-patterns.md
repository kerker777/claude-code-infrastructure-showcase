# 元件模式

現代 React 元件架構，強調型別安全、延遲載入與 Suspense 邊界。

---

## React.FC 模式（建議使用）

### 為什麼使用 React.FC

所有元件都使用 `React.FC<Props>` 模式，原因如下：
- 明確的 props 型別安全
- 一致的元件簽名
- 清楚的 prop 介面文件
- 更好的 IDE 自動完成

### 基本模式

```typescript
import React from 'react';

interface MyComponentProps {
    /** User ID to display */
    userId: number;
    /** Optional callback when action occurs */
    onAction?: () => void;
}

export const MyComponent: React.FC<MyComponentProps> = ({ userId, onAction }) => {
    return (
        <div>
            User: {userId}
        </div>
    );
};

export default MyComponent;
```

**重點：**
- Props 介面單獨定義，並加上 JSDoc 註解
- `React.FC<Props>` 提供型別安全
- 在參數中解構 props
- 在底部放預設匯出

---

## 延遲載入模式

### 何時使用延遲載入

延遲載入適用於以下元件：
- 重量級元件（DataGrid、圖表、富文本編輯器）
- 路由層級元件
- Modal/對話框內容（初始不顯示）
- 首屏以下的內容

### 如何延遲載入

```typescript
import React from 'react';

// Lazy load heavy component
const PostDataGrid = React.lazy(() =>
    import('./grids/PostDataGrid')
);

// For named exports
const MyComponent = React.lazy(() =>
    import('./MyComponent').then(module => ({
        default: module.MyComponent
    }))
);
```

**PostTable.tsx 範例：**

```typescript
/**
 * Main post table container component
 */
import React, { useState, useCallback } from 'react';
import { Box, Paper } from '@mui/material';

// Lazy load PostDataGrid to optimize bundle size
const PostDataGrid = React.lazy(() => import('./grids/PostDataGrid'));

import { SuspenseLoader } from '~components/SuspenseLoader';

export const PostTable: React.FC<PostTableProps> = ({ formId }) => {
    return (
        <Box>
            <SuspenseLoader>
                <PostDataGrid formId={formId} />
            </SuspenseLoader>
        </Box>
    );
};

export default PostTable;
```

---

## Suspense 邊界

### SuspenseLoader 元件

**引入方式：**
```typescript
import { SuspenseLoader } from '~components/SuspenseLoader';
// Or
import { SuspenseLoader } from '@/components/SuspenseLoader';
```

**使用方式：**
```typescript
<SuspenseLoader>
    <LazyLoadedComponent />
</SuspenseLoader>
```

**功能說明：**
- 在延遲載入元件載入時顯示載入指示器
- 平滑的淡入動畫
- 一致的載入體驗
- 防止版面位移

### Suspense 邊界放置位置

**路由層級：**
```typescript
// routes/my-route/index.tsx
const MyPage = lazy(() => import('@/features/my-feature/components/MyPage'));

function Route() {
    return (
        <SuspenseLoader>
            <MyPage />
        </SuspenseLoader>
    );
}
```

**元件層級：**
```typescript
function ParentComponent() {
    return (
        <Box>
            <Header />
            <SuspenseLoader>
                <HeavyDataGrid />
            </SuspenseLoader>
        </Box>
    );
}
```

**多個邊界：**
```typescript
function Page() {
    return (
        <Box>
            <SuspenseLoader>
                <HeaderSection />
            </SuspenseLoader>

            <SuspenseLoader>
                <MainContent />
            </SuspenseLoader>

            <SuspenseLoader>
                <Sidebar />
            </SuspenseLoader>
        </Box>
    );
}
```

每個區塊獨立載入，提供更好的使用者體驗。

---

## 元件結構範本

### 建議順序

```typescript
/**
 * Component description
 * What it does, when to use it
 */
import React, { useState, useCallback, useMemo, useEffect } from 'react';
import { Box, Paper, Button } from '@mui/material';
import type { SxProps, Theme } from '@mui/material';
import { useSuspenseQuery } from '@tanstack/react-query';

// Feature imports
import { myFeatureApi } from '../api/myFeatureApi';
import type { MyData } from '~types/myData';

// Component imports
import { SuspenseLoader } from '~components/SuspenseLoader';

// Hooks
import { useAuth } from '@/hooks/useAuth';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

// 1. PROPS INTERFACE (with JSDoc)
interface MyComponentProps {
    /** The ID of the entity to display */
    entityId: number;
    /** Optional callback when action completes */
    onComplete?: () => void;
    /** Display mode */
    mode?: 'view' | 'edit';
}

// 2. STYLES (if inline and <100 lines)
const componentStyles: Record<string, SxProps<Theme>> = {
    container: {
        p: 2,
        display: 'flex',
        flexDirection: 'column',
    },
    header: {
        mb: 2,
        display: 'flex',
        justifyContent: 'space-between',
    },
};

// 3. COMPONENT DEFINITION
export const MyComponent: React.FC<MyComponentProps> = ({
    entityId,
    onComplete,
    mode = 'view',
}) => {
    // 4. HOOKS (in this order)
    // - Context hooks first
    const { user } = useAuth();
    const { showSuccess, showError } = useMuiSnackbar();

    // - Data fetching
    const { data } = useSuspenseQuery({
        queryKey: ['myEntity', entityId],
        queryFn: () => myFeatureApi.getEntity(entityId),
    });

    // - Local state
    const [selectedItem, setSelectedItem] = useState<string | null>(null);
    const [isEditing, setIsEditing] = useState(mode === 'edit');

    // - Memoized values
    const filteredData = useMemo(() => {
        return data.filter(item => item.active);
    }, [data]);

    // - Effects
    useEffect(() => {
        // Setup
        return () => {
            // Cleanup
        };
    }, []);

    // 5. EVENT HANDLERS (with useCallback)
    const handleItemSelect = useCallback((itemId: string) => {
        setSelectedItem(itemId);
    }, []);

    const handleSave = useCallback(async () => {
        try {
            await myFeatureApi.updateEntity(entityId, { /* data */ });
            showSuccess('Entity updated successfully');
            onComplete?.();
        } catch (error) {
            showError('Failed to update entity');
        }
    }, [entityId, onComplete, showSuccess, showError]);

    // 6. RENDER
    return (
        <Box sx={componentStyles.container}>
            <Box sx={componentStyles.header}>
                <h2>My Component</h2>
                <Button onClick={handleSave}>Save</Button>
            </Box>

            <Paper sx={{ p: 2 }}>
                {filteredData.map(item => (
                    <div key={item.id}>{item.name}</div>
                ))}
            </Paper>
        </Box>
    );
};

// 7. EXPORT (default export at bottom)
export default MyComponent;
```

---

## 元件拆分

### 何時拆分元件

**在以下情況拆分成多個元件：**
- 元件超過 300 行
- 有多個明確不同的職責
- 有可重用的區塊
- 有複雜的巢狀 JSX

**範例：**

```typescript
// ❌ AVOID - Monolithic
function MassiveComponent() {
    // 500+ lines
    // Search logic
    // Filter logic
    // Grid logic
    // Action panel logic
}

// ✅ PREFERRED - Modular
function ParentContainer() {
    return (
        <Box>
            <SearchAndFilter onFilter={handleFilter} />
            <DataGrid data={filteredData} />
            <ActionPanel onAction={handleAction} />
        </Box>
    );
}
```

### 何時保持在一起

**在以下情況保持在同一個檔案：**
- 元件少於 200 行
- 邏輯緊密耦合
- 不會在其他地方重用
- 簡單的呈現型元件

---

## 匯出模式

### 具名 Const + 預設匯出（建議使用）

```typescript
export const MyComponent: React.FC<Props> = ({ ... }) => {
    // Component logic
};

export default MyComponent;
```

**原因：**
- 具名匯出方便測試與重構
- 預設匯出方便延遲載入
- 提供兩種選項給使用者

### 延遲載入具名匯出

```typescript
const MyComponent = React.lazy(() =>
    import('./MyComponent').then(module => ({
        default: module.MyComponent
    }))
);
```

---

## 元件通訊

### Props 向下，事件向上

```typescript
// Parent
function Parent() {
    const [selectedId, setSelectedId] = useState<string | null>(null);

    return (
        <Child
            data={data}                    // Props down
            onSelect={setSelectedId}       // Events up
        />
    );
}

// Child
interface ChildProps {
    data: Data[];
    onSelect: (id: string) => void;
}

export const Child: React.FC<ChildProps> = ({ data, onSelect }) => {
    return (
        <div onClick={() => onSelect(data[0].id)}>
            {/* Content */}
        </div>
    );
};
```

### 避免 Prop Drilling

**深度巢狀時使用 context：**
```typescript
// ❌ AVOID - Prop drilling 5+ levels
<A prop={x}>
  <B prop={x}>
    <C prop={x}>
      <D prop={x}>
        <E prop={x} />  // Finally uses it here
      </D>
    </C>
  </B>
</A>

// ✅ PREFERRED - Context or TanStack Query
const MyContext = createContext<MyData | null>(null);

function Provider({ children }) {
    const { data } = useSuspenseQuery({ ... });
    return <MyContext.Provider value={data}>{children}</MyContext.Provider>;
}

function DeepChild() {
    const data = useContext(MyContext);
    // Use data directly
}
```

---

## 進階模式

### 複合元件

```typescript
// Card.tsx
export const Card: React.FC<CardProps> & {
    Header: typeof CardHeader;
    Body: typeof CardBody;
    Footer: typeof CardFooter;
} = ({ children }) => {
    return <Paper>{children}</Paper>;
};

Card.Header = CardHeader;
Card.Body = CardBody;
Card.Footer = CardFooter;

// Usage
<Card>
    <Card.Header>Title</Card.Header>
    <Card.Body>Content</Card.Body>
    <Card.Footer>Actions</Card.Footer>
</Card>
```

### Render Props（少用，但有用）

```typescript
interface DataProviderProps {
    children: (data: Data) => React.ReactNode;
}

export const DataProvider: React.FC<DataProviderProps> = ({ children }) => {
    const { data } = useSuspenseQuery({ ... });
    return <>{children(data)}</>;
};

// Usage
<DataProvider>
    {(data) => <Display data={data} />}
</DataProvider>
```

---

## 總結

**現代元件配方：**
1. 使用 TypeScript 的 `React.FC<Props>`
2. 如果是重量級元件，使用延遲載入：`React.lazy(() => import())`
3. 用 `<SuspenseLoader>` 包裝以顯示載入狀態
4. 使用 `useSuspenseQuery` 取得資料
5. 使用匯入別名（@/、~types、~components）
6. 事件處理器使用 `useCallback`
7. 在底部放預設匯出
8. 不要為載入狀態提早 return

**另請參閱：**
- [data-fetching.md](data-fetching.md) - useSuspenseQuery 詳細說明
- [loading-and-error-states.md](loading-and-error-states.md) - Suspense 最佳實踐
- [complete-examples.md](complete-examples.md) - 完整實作範例
