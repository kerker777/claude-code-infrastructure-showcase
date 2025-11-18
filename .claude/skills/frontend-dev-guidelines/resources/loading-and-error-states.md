# 載入與錯誤狀態

**重要**：正確處理載入與錯誤狀態可避免版面位移並提供更好的使用者體驗。

---

## ⚠️ 重要規則：絕不使用提早返回

### 問題說明

```typescript
// ❌ 絕對不要這樣做 - 提早返回並顯示載入旋轉圖示
const Component = () => {
    const { data, isLoading } = useQuery();

    // 錯誤：這會造成版面位移和糟糕的使用者體驗
    if (isLoading) {
        return <LoadingSpinner />;
    }

    return <Content data={data} />;
};
```

**為什麼這樣不好：**
1. **版面位移**：載入完成時內容位置會跳動
2. **CLS (Cumulative Layout Shift)**：Core Web Vital 分數變差
3. **不順暢的使用者體驗**：頁面結構突然改變
4. **失去捲動位置**：使用者會失去在頁面上的位置

### 解決方案

**方案 1：SuspenseLoader（新元件的首選方式）**

```typescript
import { SuspenseLoader } from '~components/SuspenseLoader';

const HeavyComponent = React.lazy(() => import('./HeavyComponent'));

export const MyComponent: React.FC = () => {
    return (
        <SuspenseLoader>
            <HeavyComponent />
        </SuspenseLoader>
    );
};
```

**方案 2：LoadingOverlay（用於舊有的 useQuery 模式）**

```typescript
import { LoadingOverlay } from '~components/LoadingOverlay';

export const MyComponent: React.FC = () => {
    const { data, isLoading } = useQuery({ ... });

    return (
        <LoadingOverlay loading={isLoading}>
            <Content data={data} />
        </LoadingOverlay>
    );
};
```

---

## SuspenseLoader 元件

### 功能說明

- 在延遲載入元件載入時顯示載入指示器
- 平滑的淡入動畫
- 避免版面位移
- 在整個應用程式中提供一致的載入體驗

### 引入方式

```typescript
import { SuspenseLoader } from '~components/SuspenseLoader';
// 或
import { SuspenseLoader } from '@/components/SuspenseLoader';
```

### 基本用法

```typescript
<SuspenseLoader>
    <LazyLoadedComponent />
</SuspenseLoader>
```

### 搭配 useSuspenseQuery 使用

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { SuspenseLoader } from '~components/SuspenseLoader';

const Inner: React.FC = () => {
    // 不需要 isLoading！
    const { data } = useSuspenseQuery({
        queryKey: ['data'],
        queryFn: () => api.getData(),
    });

    return <Display data={data} />;
};

// 外層元件包裹 Suspense
export const Outer: React.FC = () => {
    return (
        <SuspenseLoader>
            <Inner />
        </SuspenseLoader>
    );
};
```

### 多個 Suspense 邊界

**模式**：為獨立區塊分別載入

```typescript
export const Dashboard: React.FC = () => {
    return (
        <Box>
            <SuspenseLoader>
                <Header />
            </SuspenseLoader>

            <SuspenseLoader>
                <MainContent />
            </SuspenseLoader>

            <SuspenseLoader>
                <Sidebar />
            </SuspenseLoader>
        </Box>
    );
};
```

**優點：**
- 每個區塊獨立載入
- 使用者可以更早看到部分內容
- 更好的感知效能

### 巢狀 Suspense

```typescript
export const ParentComponent: React.FC = () => {
    return (
        <SuspenseLoader>
            {/* 父元件在載入時暫停 */}
            <ParentContent>
                <SuspenseLoader>
                    {/* 子元件的巢狀 suspense */}
                    <ChildComponent />
                </SuspenseLoader>
            </ParentContent>
        </SuspenseLoader>
    );
};
```

---

## LoadingOverlay 元件

### 使用時機

- 尚未重構為 Suspense 的舊有 `useQuery` 元件
- 需要覆蓋式載入狀態
- 無法使用 Suspense 邊界的情況

### 用法

```typescript
import { LoadingOverlay } from '~components/LoadingOverlay';

export const MyComponent: React.FC = () => {
    const { data, isLoading } = useQuery({
        queryKey: ['data'],
        queryFn: () => api.getData(),
    });

    return (
        <LoadingOverlay loading={isLoading}>
            <Box sx={{ p: 2 }}>
                {data && <Content data={data} />}
            </Box>
        </LoadingOverlay>
    );
};
```

**功能：**
- 顯示半透明覆蓋層與旋轉圖示
- 保留內容區域（無版面位移）
- 載入時防止互動

---

## 錯誤處理

### useMuiSnackbar Hook（必須使用）

**絕對不要使用 react-toastify** - 專案標準是 MUI Snackbar

```typescript
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const MyComponent: React.FC = () => {
    const { showSuccess, showError, showInfo, showWarning } = useMuiSnackbar();

    const handleAction = async () => {
        try {
            await api.doSomething();
            showSuccess('操作成功完成');
        } catch (error) {
            showError('操作失敗');
        }
    };

    return <Button onClick={handleAction}>執行動作</Button>;
};
```

**可用方法：**
- `showSuccess(message)` - 綠色成功訊息
- `showError(message)` - 紅色錯誤訊息
- `showWarning(message)` - 橘色警告訊息
- `showInfo(message)` - 藍色資訊訊息

### TanStack Query 錯誤回呼

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';

export const MyComponent: React.FC = () => {
    const { showError } = useMuiSnackbar();

    const { data } = useSuspenseQuery({
        queryKey: ['data'],
        queryFn: () => api.getData(),

        // 處理錯誤
        onError: (error) => {
            showError('載入資料失敗');
            console.error('Query error:', error);
        },
    });

    return <Content data={data} />;
};
```

### 錯誤邊界

```typescript
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }) {
    return (
        <Box sx={{ p: 4, textAlign: 'center' }}>
            <Typography variant='h5' color='error'>
                發生錯誤
            </Typography>
            <Typography>{error.message}</Typography>
            <Button onClick={resetErrorBoundary}>重試</Button>
        </Box>
    );
}

export const MyPage: React.FC = () => {
    return (
        <ErrorBoundary
            FallbackComponent={ErrorFallback}
            onError={(error) => console.error('Boundary caught:', error)}
        >
            <SuspenseLoader>
                <ComponentThatMightError />
            </SuspenseLoader>
        </ErrorBoundary>
    );
};
```

---

## 完整範例

### 範例 1：使用 Suspense 的現代元件

```typescript
import React from 'react';
import { Box, Paper } from '@mui/material';
import { useSuspenseQuery } from '@tanstack/react-query';
import { SuspenseLoader } from '~components/SuspenseLoader';
import { myFeatureApi } from '../api/myFeatureApi';

// 內層元件使用 useSuspenseQuery
const InnerComponent: React.FC<{ id: number }> = ({ id }) => {
    const { data } = useSuspenseQuery({
        queryKey: ['entity', id],
        queryFn: () => myFeatureApi.getEntity(id),
    });

    // data 永遠有定義 - 不需要 isLoading！
    return (
        <Paper sx={{ p: 2 }}>
            <h2>{data.title}</h2>
            <p>{data.description}</p>
        </Paper>
    );
};

// 外層元件提供 Suspense 邊界
export const OuterComponent: React.FC<{ id: number }> = ({ id }) => {
    return (
        <Box>
            <SuspenseLoader>
                <InnerComponent id={id} />
            </SuspenseLoader>
        </Box>
    );
};

export default OuterComponent;
```

### 範例 2：使用 LoadingOverlay 的舊有模式

```typescript
import React from 'react';
import { Box } from '@mui/material';
import { useQuery } from '@tanstack/react-query';
import { LoadingOverlay } from '~components/LoadingOverlay';
import { myFeatureApi } from '../api/myFeatureApi';

export const LegacyComponent: React.FC<{ id: number }> = ({ id }) => {
    const { data, isLoading, error } = useQuery({
        queryKey: ['entity', id],
        queryFn: () => myFeatureApi.getEntity(id),
    });

    return (
        <LoadingOverlay loading={isLoading}>
            <Box sx={{ p: 2 }}>
                {error && <ErrorDisplay error={error} />}
                {data && <Content data={data} />}
            </Box>
        </LoadingOverlay>
    );
};
```

### 範例 3：使用 Snackbar 的錯誤處理

```typescript
import React from 'react';
import { useSuspenseQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { Button } from '@mui/material';
import { useMuiSnackbar } from '@/hooks/useMuiSnackbar';
import { myFeatureApi } from '../api/myFeatureApi';

export const EntityEditor: React.FC<{ id: number }> = ({ id }) => {
    const queryClient = useQueryClient();
    const { showSuccess, showError } = useMuiSnackbar();

    const { data } = useSuspenseQuery({
        queryKey: ['entity', id],
        queryFn: () => myFeatureApi.getEntity(id),
        onError: () => {
            showError('載入實體失敗');
        },
    });

    const updateMutation = useMutation({
        mutationFn: (updates) => myFeatureApi.update(id, updates),

        onSuccess: () => {
            queryClient.invalidateQueries({ queryKey: ['entity', id] });
            showSuccess('實體更新成功');
        },

        onError: () => {
            showError('實體更新失敗');
        },
    });

    return (
        <Button onClick={() => updateMutation.mutate({ name: 'New' })}>
            更新
        </Button>
    );
};
```

---

## 載入狀態反模式

### ❌ 不該做的事

```typescript
// ❌ 絕對不要 - 提早返回
if (isLoading) {
    return <CircularProgress />;
}

// ❌ 絕對不要 - 條件式渲染
{isLoading ? <Spinner /> : <Content />}

// ❌ 絕對不要 - 版面改變
if (isLoading) {
    return (
        <Box sx={{ height: 100 }}>
            <Spinner />
        </Box>
    );
}
return (
    <Box sx={{ height: 500 }}>  // 不同的高度！
        <Content />
    </Box>
);
```

### ✅ 該做的事

```typescript
// ✅ 最佳 - useSuspenseQuery + SuspenseLoader
<SuspenseLoader>
    <ComponentWithSuspenseQuery />
</SuspenseLoader>

// ✅ 可接受 - LoadingOverlay
<LoadingOverlay loading={isLoading}>
    <Content />
</LoadingOverlay>

// ✅ 可以 - 相同版面的內嵌骨架屏
<Box sx={{ height: 500 }}>
    {isLoading ? <Skeleton variant='rectangular' height='100%' /> : <Content />}
</Box>
```

---

## 骨架屏載入（替代方案）

### MUI Skeleton 元件

```typescript
import { Skeleton, Box } from '@mui/material';

export const MyComponent: React.FC = () => {
    const { data, isLoading } = useQuery({ ... });

    return (
        <Box sx={{ p: 2 }}>
            {isLoading ? (
                <>
                    <Skeleton variant='text' width={200} height={40} />
                    <Skeleton variant='rectangular' width='100%' height={200} />
                    <Skeleton variant='text' width='100%' />
                </>
            ) : (
                <>
                    <Typography variant='h5'>{data.title}</Typography>
                    <img src={data.image} />
                    <Typography>{data.description}</Typography>
                </>
            )}
        </Box>
    );
};
```

**關鍵**：骨架屏必須與實際內容有**相同的版面**（無位移）

---

## 總結

**載入狀態：**
- ✅ **首選**：SuspenseLoader + useSuspenseQuery（現代模式）
- ✅ **可接受**：LoadingOverlay（舊有模式）
- ✅ **可以**：相同版面的骨架屏
- ❌ **絕對不要**：提早返回或條件式版面

**錯誤處理：**
- ✅ **永遠使用**：useMuiSnackbar 提供使用者回饋
- ❌ **絕對不要**：react-toastify
- ✅ 在查詢/變更中使用 onError 回呼
- ✅ 使用錯誤邊界處理元件層級錯誤

**參考資料：**
- [component-patterns.md](component-patterns.md) - Suspense 整合
- [data-fetching.md](data-fetching.md) - useSuspenseQuery 詳細說明
