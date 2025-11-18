# 路由指南

基於資料夾結構的 TanStack Router 實作，包含延遲載入模式。

---

## TanStack Router 概觀

**TanStack Router** 採用檔案式路由：
- 資料夾結構定義路由
- 延遲載入以進行程式碼分割
- 型別安全的路由
- 麵包屑載入器

---

## 資料夾式路由

### 目錄結構

```
routes/
  __root.tsx                    # Root layout
  index.tsx                     # Home route (/)
  posts/
    index.tsx                   # /posts
    create/
      index.tsx                 # /posts/create
    $postId.tsx                 # /posts/:postId (dynamic)
  comments/
    index.tsx                   # /comments
```

**模式**：
- `index.tsx` = 該路徑的路由
- `$param.tsx` = 動態參數
- 巢狀資料夾 = 巢狀路由

---

## 基本路由模式

### posts/index.tsx 範例

```typescript
/**
 * Posts route component
 * Displays the main blog posts list
 */

import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';

// Lazy load the page component
const PostsList = lazy(() =>
    import('@/features/posts/components/PostsList').then(
        (module) => ({ default: module.PostsList }),
    ),
);

export const Route = createFileRoute('/posts/')({
    component: PostsPage,
    // Define breadcrumb data
    loader: () => ({
        crumb: 'Posts',
    }),
});

function PostsPage() {
    return (
        <PostsList
            title='All Posts'
            showFilters={true}
        />
    );
}

export default PostsPage;
```

**重點：**
- 延遲載入大型元件
- 使用 `createFileRoute` 搭配路由路徑
- 使用 `loader` 載入麵包屑資料
- 頁面元件渲染內容
- 匯出 Route 和元件

---

## 延遲載入路由

### 具名匯出模式

```typescript
import { lazy } from 'react';

// For named exports, use .then() to map to default
const MyPage = lazy(() =>
    import('@/features/my-feature/components/MyPage').then(
        (module) => ({ default: module.MyPage })
    )
);
```

### 預設匯出模式

```typescript
import { lazy } from 'react';

// For default exports, simpler syntax
const MyPage = lazy(() => import('@/features/my-feature/components/MyPage'));
```

### 為什麼要延遲載入路由？

- 程式碼分割 - 減少初始載入檔案大小
- 加快初始頁面載入速度
- 僅在導航至該路由時才載入程式碼
- 提升效能

---

## createFileRoute

### 基本設定

```typescript
export const Route = createFileRoute('/my-route/')({
    component: MyRoutePage,
});

function MyRoutePage() {
    return <div>My Route Content</div>;
}
```

### 使用麵包屑載入器

```typescript
export const Route = createFileRoute('/my-route/')({
    component: MyRoutePage,
    loader: () => ({
        crumb: 'My Route Title',
    }),
});
```

麵包屑會自動顯示在導覽列/應用程式列中。

### 使用資料載入器

```typescript
export const Route = createFileRoute('/my-route/')({
    component: MyRoutePage,
    loader: async () => {
        // Can prefetch data here
        const data = await api.getData();
        return { crumb: 'My Route', data };
    },
});
```

### 使用搜尋參數

```typescript
export const Route = createFileRoute('/search/')({
    component: SearchPage,
    validateSearch: (search: Record<string, unknown>) => {
        return {
            query: (search.query as string) || '',
            page: Number(search.page) || 1,
        };
    },
});

function SearchPage() {
    const { query, page } = Route.useSearch();
    // Use query and page
}
```

---

## 動態路由

### 參數路由

```typescript
// routes/users/$userId.tsx

export const Route = createFileRoute('/users/$userId')({
    component: UserPage,
});

function UserPage() {
    const { userId } = Route.useParams();

    return <UserProfile userId={userId} />;
}
```

### 多個參數

```typescript
// routes/posts/$postId/comments/$commentId.tsx

export const Route = createFileRoute('/posts/$postId/comments/$commentId')({
    component: CommentPage,
});

function CommentPage() {
    const { postId, commentId } = Route.useParams();

    return <CommentEditor postId={postId} commentId={commentId} />;
}
```

---

## 導航

### 程式化導航

```typescript
import { useNavigate } from '@tanstack/react-router';

export const MyComponent: React.FC = () => {
    const navigate = useNavigate();

    const handleClick = () => {
        navigate({ to: '/posts' });
    };

    return <Button onClick={handleClick}>View Posts</Button>;
};
```

### 帶參數

```typescript
const handleNavigate = () => {
    navigate({
        to: '/users/$userId',
        params: { userId: '123' },
    });
};
```

### 帶搜尋參數

```typescript
const handleSearch = () => {
    navigate({
        to: '/search',
        search: { query: 'test', page: 1 },
    });
};
```

---

## 路由版面配置模式

### 根版面配置 (__root.tsx)

```typescript
import { createRootRoute, Outlet } from '@tanstack/react-router';
import { Box } from '@mui/material';
import { CustomAppBar } from '~components/CustomAppBar';

export const Route = createRootRoute({
    component: RootLayout,
});

function RootLayout() {
    return (
        <Box>
            <CustomAppBar />
            <Box sx={{ p: 2 }}>
                <Outlet />  {/* Child routes render here */}
            </Box>
        </Box>
    );
}
```

### 巢狀版面配置

```typescript
// routes/dashboard/index.tsx
export const Route = createFileRoute('/dashboard/')({
    component: DashboardLayout,
});

function DashboardLayout() {
    return (
        <Box>
            <DashboardSidebar />
            <Box sx={{ flex: 1 }}>
                <Outlet />  {/* Nested routes */}
            </Box>
        </Box>
    );
}
```

---

## 完整路由範例

```typescript
/**
 * User profile route
 * Path: /users/:userId
 */

import { createFileRoute } from '@tanstack/react-router';
import { lazy } from 'react';
import { SuspenseLoader } from '~components/SuspenseLoader';

// Lazy load heavy component
const UserProfile = lazy(() =>
    import('@/features/users/components/UserProfile').then(
        (module) => ({ default: module.UserProfile })
    )
);

export const Route = createFileRoute('/users/$userId')({
    component: UserPage,
    loader: () => ({
        crumb: 'User Profile',
    }),
});

function UserPage() {
    const { userId } = Route.useParams();

    return (
        <SuspenseLoader>
            <UserProfile userId={userId} />
        </SuspenseLoader>
    );
}

export default UserPage;
```

---

## 總結

**路由檢查清單：**
- ✅ 資料夾式：`routes/my-route/index.tsx`
- ✅ 延遲載入元件：`React.lazy(() => import())`
- ✅ 使用 `createFileRoute` 搭配路由路徑
- ✅ 在 `loader` 函式中加入麵包屑
- ✅ 使用 `SuspenseLoader` 包裝以處理載入狀態
- ✅ 使用 `Route.useParams()` 取得動態參數
- ✅ 使用 `useNavigate()` 進行程式化導航

**另見：**
- [component-patterns.md](component-patterns.md) - 延遲載入模式
- [loading-and-error-states.md](loading-and-error-states.md) - SuspenseLoader 用法
- [complete-examples.md](complete-examples.md) - 完整路由範例
